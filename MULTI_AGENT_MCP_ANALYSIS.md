# Multi-Agent MCP Tool Access Analysis

## Executive Summary

**Issue Found**: Spawned sub-agents (like review agents, compact agents) cannot reliably access MCP tools due to a race condition in the initialization process.

## Root Cause

When a sub-agent is spawned, the following sequence occurs:

1. `run_codex_conversation_one_shot()` is called with a cloned config (including MCP server configurations)
2. `Codex::spawn()` → `Session::new()` is invoked
3. `Session::new()` calls `mcp_connection_manager.initialize()` (line 609 in `codex.rs`)
4. **CRITICAL**: `initialize()` spawns **asynchronous background tasks** to connect to MCP servers and returns immediately
5. Sub-agent conversation starts **immediately** without waiting for MCP initialization
6. When the sub-agent's first turn executes and calls `list_all_tools()`, MCP servers may not be ready yet

## Evidence

### Location 1: MCP Initialization (codex.rs:605-616)
```rust
sess.services
    .mcp_connection_manager
    .write()
    .await
    .initialize(
        config.mcp_servers.clone(),
        config.mcp_oauth_credentials_store_mode,
        auth_statuses.clone(),
        tx_event.clone(),
        sess.services.mcp_startup_cancellation_token.clone(),
    )
    .await;  // Returns immediately, spawns background tasks
```

### Location 2: MCP Initialize Method (mcp_connection_manager.rs:269-335)
```rust
pub async fn initialize(
    &mut self,
    mcp_servers: HashMap<String, McpServerConfig>,
    // ... other params
) {
    // ... setup code ...

    // Spawns background tasks for each server
    for (server_name, cfg) in mcp_servers.into_iter().filter(|(_, cfg)| cfg.enabled) {
        join_set.spawn(async move {
            let outcome = async_managed_client.client().await;
            // ...
        });
    }

    self.clients = clients;

    // THIS spawns async task, doesn't wait!
    tokio::spawn(async move {
        let outcomes = join_set.join_all().await;
        // ... sends McpStartupCompleteEvent later
    });
}
```

### Location 3: Immediate Submission (codex_delegate.rs:98-113)
```rust
let io = run_codex_conversation_interactive(
    config,
    auth_manager,
    parent_session,
    parent_ctx,
    child_cancel.clone(),
    initial_history,
)
.await?;

// Immediately submits input - doesn't wait for MCP servers!
io.submit(Op::UserInput { items: input }).await?;
```

### Location 4: Per-Turn Tool Collection (codex.rs:2030-2046)
```rust
let mcp_tools = sess.services.mcp_connection_manager
    .read().await
    .list_all_tools()  // May return empty if servers not ready!
    .or_cancel(&cancellation_token)
    .await?;

let router = Arc::new(ToolRouter::from_config(
    &turn_context.tools_config,
    Some(
        mcp_tools
            .into_iter()
            .map(|(name, tool)| (name, tool.tool))
            .collect(),
    ),
));
```

## Impact

Sub-agents will have **no MCP tools available** on their first turn (and potentially subsequent turns) if:
- MCP servers take longer than a few milliseconds to initialize
- The sub-agent starts executing immediately after spawn
- This affects Review agents, Compact agents, and any other spawned agents

## Verification

Main agents work because:
1. They are spawned at startup
2. User typically waits before first interaction
3. MCP servers have time to initialize in the background

Sub-agents fail because:
1. They are spawned on-demand during execution
2. They start executing immediately after spawn
3. No waiting mechanism for MCP initialization

## Config Inheritance

Sub-agents **DO** inherit the parent's MCP server configuration:
- Line 93 in `review.rs`: `let mut sub_agent_config = config.as_ref().clone();`
- This clones the entire config including `mcp_servers`
- The issue is **timing**, not configuration

## Proposed Solutions

### Option 1: Wait for MCP Startup Complete (Recommended)
Modify `Session::new()` or the spawning logic to wait for `McpStartupCompleteEvent` before proceeding with the first turn.

```rust
// In Session::new, after initialize():
let startup_rx = /* subscribe to McpStartupComplete event */;
timeout(Duration::from_secs(10), startup_rx.recv()).await?;
```

### Option 2: Share MCP Connection Manager
Modify sub-agent spawning to share the parent's `McpConnectionManager` instead of creating a new one.

```rust
// In run_codex_conversation_interactive:
let services = SessionServices {
    mcp_connection_manager: parent_session.services.mcp_connection_manager.clone(),
    // ... other fields
};
```

### Option 3: Lazy Tool Loading with Retry
Retry `list_all_tools()` with exponential backoff if empty on first attempt.

### Option 4: Explicit Initialization Gate
Add a `ReadinessFlag` for MCP services similar to `tool_call_gate` and wait for it before executing turns.

## Recommended Fix

**Option 2 (Share MCP Connection Manager)** is the cleanest solution because:
1. Avoids duplicate server connections
2. Eliminates race condition entirely
3. Sub-agents get instant access to parent's connected servers
4. Reduces resource usage (fewer connections)
5. Simpler implementation

## Files Requiring Changes

1. `codex-rs/core/src/codex_delegate.rs` - Modify spawning to accept and pass parent's SessionServices
2. `codex-rs/core/src/codex.rs` - Update `Session::new()` to optionally accept existing SessionServices
3. `codex-rs/core/src/state/service.rs` - No changes needed (already uses Arc for McpConnectionManager)

## Testing Strategy

1. Create a test with slow-initializing MCP server
2. Spawn sub-agent immediately after main agent startup
3. Verify sub-agent has access to MCP tools
4. Verify no duplicate server connections are created
