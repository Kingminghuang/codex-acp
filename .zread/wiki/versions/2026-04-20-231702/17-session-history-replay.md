When a client reconnects to a previously active conversation, it expects to see the full transcript of prior turns — user messages, agent responses, reasoning, and tool calls — as if they had been streamed live. **Session history replay** is the mechanism that reads persisted rollout data from disk, translates each stored entry into the corresponding ACP notification format, and streams the result to the client before the session becomes interactive again. It is the bridge between Codex's durable conversation log and the ACP client's expectation of real-time notification delivery.

Sources: [thread.rs](src/thread.rs#L3380-L3400)

## When Replay Is Triggered

Replay is exclusively triggered during the `load_session` flow in `CodexAgent`. After authenticating and resolving the session's rollout file path, the agent calls `RolloutRecorder::get_rollout_history` to read the persisted history from disk. The returned `InitialHistory` enum determines what items are available for replay:

| `InitialHistory` Variant | Replay Items | Meaning |
|---|---|---|
| `Resumed(resumed)` | `resumed.history` | Session was resumed intact — full conversation history |
| `Forked(items)` | `items` | Session was forked from another — subset of original history |
| `New` | `Vec::new()` | No prior history (empty session) |

The replay step occurs **after** the underlying Codex thread is resumed via `thread_manager.resume_thread_from_rollout` but **before** the `LoadSessionResponse` is returned to the client. This ordering ensures the client receives all historical notifications before it begins interacting with the session.

Sources: [codex_agent.rs](src/codex_agent.rs#L380-L448)

## The Replay Pipeline

The following diagram illustrates the end-to-end flow from rollout file to ACP notifications:

```mermaid
sequenceDiagram
    participant Client as ACP Client
    participant Agent as CodexAgent
    participant Recorder as RolloutRecorder
    participant Thread as Thread
    participant Actor as ThreadActor
    participant SessionClient as SessionClient

    Client->>Agent: load_session(request)
    Agent->>Recorder: get_rollout_history(rollout_path)
    Recorder-->>Agent: InitialHistory (Resumed/Forked/New)
    Agent->>Agent: resume_thread_from_rollout()
    Agent->>Thread: replay_history(rollout_items)
    Thread->>Actor: ThreadMessage::ReplayHistory { history }
    
    loop For each RolloutItem
        Actor->>Actor: handle_replay_history()
        alt EventMsg
            Actor->>SessionClient: replay_event_msg()
            SessionClient->>Client: SessionUpdate (UserMessageChunk / AgentMessageChunk / AgentThoughtChunk)
        else ResponseItem
            Actor->>SessionClient: replay_response_item()
            SessionClient->>Client: SessionUpdate (ToolCall / ToolCallUpdate)
        else SessionMeta / TurnContext / Compacted
            Actor-->>Actor: Skip
        end
    end
    
    Actor-->>Thread: Result<()>
    Thread-->>Agent: Result<()>
    Agent->>Thread: load()
    Thread-->>Agent: LoadSessionResponse
    Agent-->>Client: LoadSessionResponse
```

The `Thread::replay_history` public method wraps the asynchronous dispatch through the `ThreadMessage::ReplayHistory` variant, ensuring the replay is processed on the `ThreadActor`'s single-threaded event loop — the same loop that handles live events. This guarantees that replayed notifications are delivered in order and without interleaving with any subsequent live prompt processing.

Sources: [thread.rs](src/thread.rs#L301-L313), [thread.rs](src/thread.rs#L2726-L2732)

## RolloutItem Dispatch: Two Parallel Paths

The `handle_replay_history` method iterates over the `Vec<RolloutItem>` and dispatches each item to one of two specialized replay functions. The dual-path design reflects how Codex persists conversation state: **message content** (user input, agent output, reasoning) is stored as `EventMsg` variants, while **tool call records** are stored as `ResponseItem` variants. This separation means that messages and tool calls are never conflated during persistence or replay.

| RolloutItem Variant | Handler | What It Covers |
|---|---|---|
| `EventMsg(event_msg)` | `replay_event_msg` | User messages, agent text, agent reasoning |
| `ResponseItem(response_item)` | `replay_response_item` | Shell commands, patch edits, MCP calls, web search, custom tools |
| `SessionMeta` / `TurnContext` / `Compacted` | *(skipped)* | Metadata only — no ACP equivalent |

Sources: [thread.rs](src/thread.rs#L3386-L3399)

## EventMsg Replay: Messages and Reasoning

The `replay_event_msg` handler translates Codex protocol events into ACP `SessionUpdate` notifications. Only four `EventMsg` variants produce output — all others are silently skipped because they are either transient (deltas, turn lifecycle events) or have no direct ACP equivalent.

| EventMsg Variant | ACP Notification | SessionUpdate Type |
|---|---|---|
| `UserMessage` | `send_user_message` | `UserMessageChunk` |
| `AgentMessage` | `send_agent_text` | `AgentMessageChunk` |
| `AgentReasoning` | `send_agent_thought` | `AgentThoughtChunk` |
| `AgentReasoningRawContent` | `send_agent_thought` | `AgentThoughtChunk` |
| All other variants | *(skipped)* | — |

The symmetry with live event handling is deliberate: during a live prompt, the same `SessionClient` methods (`send_user_message`, `send_agent_text`, `send_agent_thought`) are called as agent events stream in. Replay replays the same notification sequence, allowing the client to render the conversation identically whether it was received live or loaded from history.

Sources: [thread.rs](src/thread.rs#L3402-L3428)

## ResponseItem Replay: Tool Calls and Their Outputs

The `replay_response_item` handler is more intricate than its `EventMsg` counterpart because tool calls require rich parsing to produce the same detailed ACP notifications that live events generate. The handler processes six `ResponseItem` variants:

| ResponseItem Variant | ACP Notification | Key Behavior |
|---|---|---|
| `Message` / `Reasoning` | *(skipped)* | Already covered by `EventMsg` replay |
| `FunctionCall` | `send_tool_call` or `send_completed_tool_call` | Shell commands get rich parsing; others use generic fallback |
| `FunctionCallOutput` | `send_tool_call_completed` | Attaches raw output to a previously sent tool call |
| `LocalShellCall` | `send_tool_call` | Parses command for title, kind, and file locations |
| `CustomToolCall` | `send_tool_call` or `send_completed_tool_call` | `apply_patch` gets diff extraction; others use generic fallback |
| `CustomToolCallOutput` | `send_tool_call_completed` | Attaches raw output to a previously sent tool call |
| `WebSearchCall` | `send_tool_call` | Extracts title/call_id from the `WebSearchAction` |
| Other variants | *(skipped)* | `GhostSnapshot`, `Compaction`, `Other`, `LocalShellCall` without `call_id` |

### Rich Parsing for Shell Commands

Both `FunctionCall` (with name `shell`, `container.exec`, or `shell_command`) and `LocalShellCall` undergo command parsing via `parse_command` and `parse_command_tool_call`. This produces the same human-readable title, `ToolKind` classification, and file path locations that live shell events generate — ensuring the client sees "Read src/main.rs" rather than a raw command vector.

Sources: [thread.rs](src/thread.rs#L3558-L3704)

### Rich Parsing for Apply Patch

When a `CustomToolCall` has the name `apply_patch`, the handler invokes `parse_apply_patch_call` to extract the patch's file-level hunks. Each hunk — whether an `AddFile`, `DeleteFile`, or `UpdateFile` — is converted into a `ToolCallContent::Diff` with old and new text, and the file paths become `ToolCallLocation` entries. The result is a tool call notification with structured diff content, matching the live `PatchApplyBegin`/`PatchApplyEnd` event flow.

Sources: [thread.rs](src/thread.rs#L3430-L3502)

### Web Search Replay

`WebSearchCall` items carry an optional `WebSearchAction` that determines the display title and fallback call ID. The `web_search_action_to_title_and_id` helper maps each action variant:

| WebSearchAction | Title Source | Fallback ID Prefix |
|---|---|---|
| `Search { query, queries }` | `queries.join(", ")` or `query` | `web_search` |
| `OpenPage { url }` | `url` | `web_open` |
| `FindInPage { pattern }` | `pattern` | `web_find` |
| `Other` | `"Unknown"` | `web_search` |

When the `id` field is `None`, a UUID-based fallback ID is generated using `generate_fallback_id`.

Sources: [thread.rs](src/thread.rs#L3993-L4035)

## Notification Delivery via SessionClient

All replayed notifications pass through `SessionClient::send_notification`, which wraps each `SessionUpdate` in a `SessionNotification` addressed to the session's `SessionId` and delivers it via the ACP `Client::session_notification` method. The three core notification helpers used during replay are:

| SessionClient Method | SessionUpdate Variant | Purpose |
|---|---|---|
| `send_user_message` | `UserMessageChunk` | Replayed user input |
| `send_agent_text` | `AgentMessageChunk` | Replayed agent output |
| `send_agent_thought` | `AgentThoughtChunk` | Replayed reasoning |
| `send_tool_call` | `ToolCall` | Replayed tool call (with status, kind, locations, content) |
| `send_tool_call_update` | `ToolCallUpdate` | Replayed tool call output or completion |

Two convenience methods — `send_completed_tool_call` and `send_tool_call_completed` — are specifically annotated as "used for replay." The former creates a `ToolCall` with `Completed` status and optional `raw_input`; the latter sends a `ToolCallUpdate` with `Completed` status and optional `raw_output`.

Sources: [thread.rs](src/thread.rs#L2464-L2524)

## Design Rationale: Why Two Storage Paths

The split between `EventMsg` and `ResponseItem` in the rollout history mirrors the Codex protocol's dual event model. During live execution, the TUI consumes `EventMsg` variants (which are streamed incrementally), while the underlying OpenAI-style conversation state accumulates `ResponseItem` variants. When persisting to the rollout file, both are recorded as `RolloutItem` entries. During replay, messages and reasoning come from `EventMsg` (which captures the final, assembled text), while tool calls come from `ResponseItem` (which captures the structured call/output pair). This avoids duplicating message content and ensures tool call metadata — particularly `call_id` linkage between calls and their outputs — is preserved with full fidelity.

Sources: [thread.rs](src/thread.rs#L3380-L3399)

## What Gets Skipped and Why

The `handle_replay_history` loop explicitly skips three `RolloutItem` variants — `SessionMeta`, `TurnContext`, and `Compacted` — because they are bookkeeping records with no ACP notification equivalent. Similarly, `replay_event_msg` skips all `EventMsg` variants except the four message/reasoning types, and `replay_response_item` skips `Message`, `Reasoning`, `GhostSnapshot`, `Compaction`, `Other`, and `LocalShellCall` entries without a `call_id`. These omissions are intentional: transient events (deltas, turn boundaries, background events) have no meaningful representation in a replayed conversation, and including them would produce duplicate or confusing notifications.

Sources: [thread.rs](src/thread.rs#L3395-L3397), [thread.rs](src/thread.rs#L3422-L3426), [thread.rs](src/thread.rs#L3701-L3702)

## Sequencing: Replay Before Load

A subtle but important ordering constraint exists in `load_session`: `replay_history` is called **before** `thread.load()`. This means all historical notifications are delivered to the client before the `LoadSessionResponse` — which carries the session's modes, models, and config options — is sent. The client therefore receives the full conversation transcript first, then the session metadata that enables interactive use. The `ThreadMessage::ReplayHistory` is processed synchronously on the actor loop (the `oneshot::Sender` blocks until replay completes), so no live events can interleave with the replayed history.

Sources: [codex_agent.rs](src/codex_agent.rs#L435-L437)

## Related Pages

- [Session Lifecycle: New, Load, Close, and List](8-session-lifecycle-new-load-close-and-list) — The broader `load_session` flow within which replay is embedded
- [Translating Codex Events to ACP Notifications](11-translating-codex-events-to-acp-notifications) — How live events map to ACP notifications (the same mappings used during replay)
- [SessionClient: The ACP Notification Gateway](18-sessionclient-the-acp-notification-gateway) — The `SessionClient` layer that delivers all replayed notifications
- [Thread and ThreadActor: Event Loop and Codex-to-ACP Translation](7-thread-and-threadactor-event-loop-and-codex-to-acp-translation) — The actor infrastructure that processes `ThreadMessage::ReplayHistory`