# maw-rs: Distributed Terminal Multiplexing & Fleet Management for AI Agent Oracles (Rust)

**Repository**: https://github.com/Soul-Brews-Studio/maw-rs

A comprehensive Rust implementation of distributed terminal multiplexing and fleet management for AI agent oracles, mirroring and extending the maw-js JavaScript/TypeScript codebase.

---

## Overview

**maw-rs** is a multi-crate Rust monorepo implementing terminal session management (tmux), peer discovery/probing, network transport with failover routing, and worktree management. The architecture follows a modular plugin-based design with clear separation between core logic (testable pure functions) and IO seams (injected traits).

### Key Architectural Principles

1. **Pure Logic & Injected IO**: Core algorithms are deterministic pure functions. I/O is injected through traits (`TmuxRunner`, `Transport`) for testability.
2. **Modular Crates**: Each functional domain (tmux, peer, transport, worktree) lives in its own crate with minimal cross-dependencies.
3. **Code Generation**: Build-time code generation (build.rs) assembles CLI dispatchers and fragment arrays to keep command registration declarative.
4. **Async/Sync Separation**: Sync dispatch for most commands; async handlers explicitly separated in dispatcher entries.
5. **Audit Trail**: All commands are logged to `audit.jsonl` for observability.

---

## Main Entry Point

**File**: `/crates/maw-cli/src/main.rs`

The entry point handles three concerns:

1. **Tmux Attach Fast Path**: Intercepts `maw attach` commands and directly invokes tmux if the target is live (avoid CLI overhead)
2. **mawx Symlink Shim**: Automatically converts `mawx cost` to `maw x cost` for convenience
3. **Signal/Output Handling**: Manages broken pipe errors and exit codes

```rust
#[tokio::main(flavor = "multi_thread")]
async fn main() {
    let program = std::env::args().next();
    let argv: Vec<String> = std::env::args().skip(1).collect();
    let argv = mawx_shim_argv(program.as_deref(), argv);
    std::process::exit(main_code_async(&argv).await);
}

/// mawx argv[0] shim: when invoked as "mawx", inject "x" as the first argument
/// so `mawx costs` ≡ `maw x costs`
fn mawx_shim_argv(program: Option<&str>, mut argv: Vec<String>) -> Vec<String> {
    let is_mawx = program.is_some_and(|program| {
        std::path::Path::new(program)
            .file_name()
            .and_then(OsStr::to_str)
            .unwrap_or(program)
            .starts_with("mawx")
    });
    if is_mawx {
        argv.insert(0, "x".to_owned());
    }
    argv
}

async fn main_code_async(argv: &[String]) -> i32 {
    main_code_async_with(argv, maybe_exec_attach).await
}
```

### Tmux Attach Fast Path

Intercepts `maw a[ttach]` to jump directly to tmux without parsing the full CLI:

```rust
fn maybe_exec_attach(argv: &[String]) -> Option<i32> {
    let mut client = TmuxClient::local();
    let alive_sessions = client.list_session_names();
    maybe_exec_attach_with(
        argv,
        std::io::stdout().is_terminal(),
        std::env::var_os("TMUX").is_some(),
        &alive_sessions,
        run_tmux_attach,
    )
}

fn attach_exec_tmux_args(
    argv: &[String],
    stdout_is_terminal: bool,
    inside_tmux: bool,
    alive_sessions: &[String],
) -> Option<Vec<String>> {
    let verb = argv.first()?.as_str();
    if !matches!(verb, "a" | "attach") { return None; }
    
    // Parse flags and target
    let mut readonly = false;
    let mut target: Option<&str> = None;
    for arg in argv.iter().skip(1).map(String::as_str) {
        match arg {
            "--readonly" | "--read-only" | "-r" => readonly = true,
            arg if arg.starts_with('-') => return None,
            value => target = Some(value),
        }
    }
    
    // Resolve session name
    let session_query = target?.split(':').next().unwrap_or_default();
    let alive = alive_sessions.iter().cloned().collect::<BTreeSet<_>>();
    let session = match resolve_tmux_attach_session(session_query, &alive) {
        TmuxAttachSessionResolution::Match { session } => session,
        _ => return None,
    };
    
    Some(if readonly {
        vec!["attach".to_owned(), "-r".to_owned(), "-t".to_owned(), session]
    } else if inside_tmux {
        vec!["switch-client".to_owned(), "-t".to_owned(), session]
    } else {
        vec!["attach".to_owned(), "-t".to_owned(), session]
    })
}
```

---

## CLI Architecture & Dispatching

**File**: `/crates/maw-cli/src/core_impl/dispatcher.rs`

### Handler Types

The dispatcher supports both synchronous and asynchronous command handlers:

```rust
type NativeHandler = fn(&[String]) -> CliOutput;
type AsyncHandler = fn(Vec<String>) -> Pin<Box<dyn Future<Output = CliOutput> + Send>>;

#[derive(Clone, Copy)]
enum Handler {
    Sync(NativeHandler),
    Async(AsyncHandler),
}

#[derive(Clone, Copy)]
pub(crate) struct DispatcherEntry {
    command: &'static str,
    handler: Handler,
}
```

### Build-Time Dispatcher Assembly

The `build.rs` collects command fragments (from individual `core_impl/*.rs` files) and generates dispatcher arrays:

```rust
// From build.rs excerpt:
fn generate() -> io::Result<()> {
    let core_impl_dir = manifest_dir.join("src").join("core_impl");
    let parts = collect_core_files(&core_impl_dir)?;
    
    let mut includes = String::new();
    for part in &parts {
        writeln!(includes, "include!({:?});", part.path.display().to_string())?;
    }
    
    let mut fragments = String::from(
        "#[allow(clippy::needless_borrow)]\npub(crate) const DISPATCHER_FRAGMENTS: &[&[DispatcherEntry]] = &[\n"
    );
    for number in dispatch_numbers {
        writeln!(fragments, "    &DISPATCH_{number:02},")?;
    }
    fragments.push_str("];\n");
    
    fs::write(out_dir.join("parts_includes.rs"), includes)?;
    fs::write(out_dir.join("dispatch_fragments.rs"), fragments)?;
    Ok(())
}
```

### Dispatcher Pattern

Each command file declares a `DISPATCH_NN` fragment:

```rust
// Example from session.rs
const DISPATCH_75: &[DispatcherEntry] = &[DispatcherEntry {
    command: "session",
    handler: Handler::Sync(session_run_command),
}];
```

### Run Dispatch with Plugin Fallback

```rust
pub fn run_cli(argv: &[String]) -> CliOutput {
    let Some(command) = argv.first().map(String::as_str) else {
        return usage_ok();
    };

    match dispatcher_target(command) {
        DispatchTarget::Native(handler) => {
            cli_dispatch_log_command(command, &argv[1..]);
            native_or_plugin_fallback(argv, || handler(&argv[1..]))
        }
        DispatchTarget::AsyncNative(handler) => native_or_plugin_fallback(argv, || {
            run_async_handler_blocking(handler, &argv[1..])
        }),
        DispatchTarget::UnknownCommand => dispatch_cli_plugin_or_unknown(argv, command),
    }
}

pub async fn run_cli_async(argv: &[String]) -> CliOutput {
    let Some(command) = argv.first().map(String::as_str) else {
        return usage_ok();
    };

    match dispatcher_target(command) {
        DispatchTarget::Native(handler) => {
            cli_dispatch_log_command(command, &argv[1..]);
            native_or_plugin_fallback(argv, || handler(&argv[1..]))
        }
        DispatchTarget::AsyncNative(handler) => {
            plugin_fallback_for_native_miss(argv, handler(argv[1..].to_vec()).await)
        }
        DispatchTarget::UnknownCommand => dispatch_cli_plugin_or_unknown(argv, command),
    }
}
```

### Async Handler Runtime Management

Async handlers run on tokio; the sync dispatcher creates a new runtime if needed:

```rust
fn run_async_handler_blocking(handler: AsyncHandler, args: &[String]) -> CliOutput {
    // If already inside a tokio runtime, error (avoid nested block_on)
    if tokio::runtime::Handle::try_current().is_ok() {
        return CliOutput {
            code: 1,
            stdout: String::new(),
            stderr: "cannot block_on inside runtime; call run_cli_async for async commands\n".to_owned(),
        };
    }

    let runtime = match tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
    {
        Ok(runtime) => runtime,
        Err(error) => {
            return CliOutput {
                code: 1,
                stdout: String::new(),
                stderr: format!("failed to start tokio runtime: {error}\n"),
            };
        }
    };
    runtime.block_on(handler(args.to_vec()))
}
```

### Audit Trail

Every command is logged to `audit.jsonl`:

```rust
fn cli_dispatch_log_command(command: &str, args: &[String]) {
    let row = serde_json::json!({
        "ts": cli_dispatch_now_iso(),
        "cmd": command,
        "args": args,
        "binary": "maw-rs",
        "version": MAW_RS_BUILD_VERSION,
    });
    let path = audit_jsonl_path(&current_xdg_env());
    let _ = append_jsonl_atomic(&path, &row);
}
```

---

## Core Modules

### 1. maw-tmux: Terminal Multiplexing

**Files**: `/crates/maw-tmux/src/core_impl/types_runner_parts/tmux_domain_types.rs`

Core types for tmux interaction:

```rust
/// Tmux session metadata
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct TmuxSession {
    pub name: String,
    pub windows: Vec<TmuxWindow>,
}

/// Tmux pane metadata
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct TmuxPane {
    pub id: String,
    pub command: String,
    pub target: String,
    pub title: String,
    pub pid: Option<u32>,
    pub cwd: Option<String>,
    pub last_activity: Option<u64>,
}

/// Injectable tmux execution seam
pub trait TmuxRunner {
    /// Run `tmux <subcommand> <args...>` and return stdout
    fn run(&mut self, subcommand: &str, args: &[String]) -> Result<String, TmuxError>;

    /// Run `tmux <subcommand> <args...>` with stdin
    fn run_with_stdin(
        &mut self,
        subcommand: &str,
        args: &[String],
        _stdin: &[u8],
    ) -> Result<String, TmuxError> {
        self.run(subcommand, args)
    }
}
```

#### Send Tracker: Rate Limiting & Cooldown

Ports maw-js heartbeat tracking to prevent spam:

```rust
/// Per-pane heartbeat throttle state for `maw tmux send`
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct SendTrackerEntry {
    pub last_ts: u64,
    pub count: u32,
    pub window_start: u64,
}

/// In-memory cooldown + quota tracker
#[derive(Debug, Clone, Default, PartialEq, Eq)]
pub struct TmuxSendTracker {
    entries: BTreeMap<String, SendTrackerEntry>,
}

impl TmuxSendTracker {
    /// Apply maw-js heartbeat cooldown and quota gates
    pub fn check(&mut self, resolved: &str, now_ms: u64, force: bool) -> SendThrottle {
        if force { return SendThrottle::Allowed; }
        
        let Some(prev) = self.entries.get_mut(resolved) else {
            self.entries.insert(resolved.to_owned(), SendTrackerEntry {
                last_ts: now_ms,
                count: 1,
                window_start: now_ms,
            });
            return SendThrottle::Allowed;
        };
        
        if now_ms.saturating_sub(prev.last_ts) < COOLDOWN_MS {
            return SendThrottle::Cooldown { cooldown_ms: COOLDOWN_MS };
        }
        if now_ms.saturating_sub(prev.window_start) > QUOTA_WINDOW_MS {
            prev.count = 0;
            prev.window_start = now_ms;
        }
        if prev.count >= QUOTA_PER_MINUTE {
            return SendThrottle::Quota { quota_per_minute: QUOTA_PER_MINUTE };
        }
        
        prev.last_ts = now_ms;
        prev.count += 1;
        SendThrottle::Allowed
    }
}
```

### 2. maw-peer: Peer Discovery & TOFU

**File**: `/crates/maw-peer/src/core_impl/peer_store_parts/peer_store_types.rs`

Peer discovery with Trust-On-First-Use (TOFU) pubkey validation:

```rust
/// Peer store record subset used by maw-js `probe-all`
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct PeerRecord {
    pub url: String,
    #[serde(default)]
    pub node: Option<String>,
    #[serde(rename = "addedAt")]
    pub added_at: String,
    #[serde(default, rename = "lastSeen")]
    pub last_seen: Option<String>,
    #[serde(default, rename = "lastError")]
    pub last_error: Option<ProbeLastError>,
    #[serde(default)]
    pub nickname: Option<String>,
    #[serde(default)]
    pub pubkey: Option<String>,
    #[serde(default, rename = "pubkeyFirstSeen")]
    pub pubkey_first_seen: Option<String>,
    #[serde(default)]
    pub identity: Option<PeerIdentity>,
}

/// Stale peer row used by doctor `--fix-stale` preview and mutation
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct StalePeer {
    pub alias: String,
    pub url: String,
    pub age_ms: Option<u64>,
}

/// TOFU decision kinds during peer validation
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TofuDecisionKind {
    TofuBootstrap,
    Match,
    Mismatch,
    LegacyFirstContact,
    LegacyAfterPinned,
}

/// Pubkey mismatch error (when peer's key changes after first contact)
impl std::fmt::Display for PeerPubkeyMismatchError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(
            f,
            "peer pubkey changed for {}: {}… → {}…; manually `maw peers forget {}` to re-TOFU",
            self.alias,
            prefix16(&self.cached),
            prefix16(&self.observed),
            self.alias
        )
    }
}
```

### 3. maw-transport: Network Transport & Failover

**File**: `/crates/maw-transport/src/core_impl/transport_router_parts/transport_models.rs`

Portable error classification and failover routing:

```rust
/// Transport failure reasons (for retry logic)
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TransportFailureReason {
    Timeout,
    Unreachable,
    Auth,
    RateLimit,
    Rejected,
    ParseError,
    Unknown,
}

impl TransportFailureReason {
    #[must_use]
    pub const fn as_str(self) -> &'static str {
        match self {
            Self::Timeout => "timeout",
            Self::Unreachable => "unreachable",
            Self::Auth => "auth",
            Self::RateLimit => "rate_limit",
            Self::Rejected => "rejected",
            Self::ParseError => "parse_error",
            Self::Unknown => "unknown",
        }
    }
}

/// Classified transport failure with retryability
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct ClassifiedError {
    pub reason: TransportFailureReason,
    pub retryable: bool,
}

/// Classify common error strings into portable failure reasons
#[must_use]
pub fn classify_error(err: Option<&str>) -> ClassifiedError {
    let Some(err) = err else {
        return ClassifiedError {
            reason: TransportFailureReason::Unknown,
            retryable: false,
        };
    };
    let msg = err.to_lowercase();
    if contains_any(&msg, &["timeout", "etimedout", "econnreset"]) {
        return ClassifiedError {
            reason: TransportFailureReason::Timeout,
            retryable: true,
        };
    }
    if contains_any(&msg, &["econnrefused", "unreachable", "enetunreach"]) {
        return ClassifiedError {
            reason: TransportFailureReason::Unreachable,
            retryable: true,
        };
    }
    if contains_any(&msg, &["401", "403", "auth", "unauthorized", "forbidden"]) {
        return ClassifiedError {
            reason: TransportFailureReason::Auth,
            retryable: false,
        };
    }
    if msg.contains("429") || msg.contains("too many") || rate_limit_like(&msg) {
        return ClassifiedError {
            reason: TransportFailureReason::RateLimit,
            retryable: true,
        };
    }
    if contains_any(&msg, &["400", "reject", "denied"]) {
        return ClassifiedError {
            reason: TransportFailureReason::Rejected,
            retryable: false,
        };
    }
    if contains_any(&msg, &["parse", "json", "syntax"]) {
        return ClassifiedError {
            reason: TransportFailureReason::ParseError,
            retryable: false,
        };
    }
    ClassifiedError {
        reason: TransportFailureReason::Unknown,
        retryable: false,
    }
}

/// Result of a routed send attempt
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct TransportResult {
    pub ok: bool,
    pub via: String,
    pub reason: Option<TransportFailureReason>,
    pub retryable: bool,
}

impl TransportResult {
    #[must_use]
    pub fn success(via: impl Into<String>) -> Self {
        Self {
            ok: true,
            via: via.into(),
            reason: None,
            retryable: false,
        }
    }

    #[must_use]
    pub fn failure(
        via: impl Into<String>,
        reason: TransportFailureReason,
        retryable: bool,
    ) -> Self {
        Self {
            ok: false,
            via: via.into(),
            reason: Some(reason),
            retryable,
        }
    }
}
```

---

## Example Command Implementation: Session

**File**: `/crates/maw-cli/src/core_impl/session.rs`

Shows the standard pattern for implementing a command:

```rust
const DISPATCH_75: &[DispatcherEntry] = &[DispatcherEntry {
    command: "session",
    handler: Handler::Sync(session_run_command),
}];

#[derive(Debug, Clone, Copy, Default, PartialEq, Eq)]
struct SessionOptions {
    short: bool,
    json: bool,
}

/// Trait for injecting tmux IO
trait SessionTmux {
    fn session_display_message(&mut self, format: &str) -> Result<String, String>;
}

struct SessionSystemTmux;

impl SessionTmux for SessionSystemTmux {
    fn session_display_message(&mut self, format: &str) -> Result<String, String> {
        session_validate_format(format)?;
        let args = vec!["-p".to_owned(), format.to_owned()];
        let mut runner = maw_tmux::CommandTmuxRunner::new();
        maw_tmux::TmuxRunner::run(&mut runner, "display-message", &args)
            .map_err(|error| format!("session: tmux display-message failed: {error}"))
    }
}

fn session_run_command(argv: &[String]) -> CliOutput {
    session_run_command_with(
        argv,
        std::env::var_os("TMUX").is_some(),
        &mut SessionSystemTmux,
    )
}

fn session_run_command_with(
    argv: &[String],
    in_tmux: bool,
    runner: &mut impl SessionTmux,
) -> CliOutput {
    match session_run(argv, in_tmux, runner) {
        Ok(stdout) => CliOutput {
            code: 0,
            stdout,
            stderr: String::new(),
        },
        Err(message) => CliOutput {
            code: 1,
            stdout: String::new(),
            stderr: format!("{message}\n"),
        },
    }
}

fn session_run(
    argv: &[String],
    in_tmux: bool,
    runner: &mut impl SessionTmux,
) -> Result<String, String> {
    let options = session_parse_args(argv)?;
    if !in_tmux {
        return Err("maw session requires an active tmux session".to_owned());
    }
    if options.short {
        return runner
            .session_display_message(SESSION_SHORT_FORMAT)
            .map(|raw| format!("{}\n", raw.trim()));
    }
    let raw = runner.session_display_message(SESSION_ADDRESS_FORMAT)?;
    let address = session_parse_address(raw.trim());
    Ok(if options.json {
        session_render_json(&address)
    } else {
        session_render_human(&address)
    })
}

#[cfg(test)]
mod session_tests {
    use super::*;

    #[derive(Default)]
    struct SessionFakeTmux {
        responses: Vec<String>,
        calls: Vec<(String, Vec<String>)>,
        error: Option<String>,
    }

    impl SessionTmux for SessionFakeTmux {
        fn session_display_message(&mut self, format: &str) -> Result<String, String> {
            session_validate_format(format)?;
            self.calls.push(("display-message".to_owned(), vec!["-p".to_owned(), format.to_owned()]));
            if let Some(error) = &self.error {
                return Err(format!("session: tmux display-message failed: {error}"));
            }
            Ok(self.responses.pop().unwrap_or_default())
        }
    }

    #[test]
    fn session_short_uses_constant_tmux_format_only() {
        let mut tmux = SessionFakeTmux {
            responses: vec!["13-nova\n".to_owned()],
            ..Default::default()
        };
        let output = session_run_command_with(&["--short".to_owned()], true, &mut tmux);
        assert_eq!(output.code, 0);
        assert_eq!(output.stdout, "13-nova\n");
    }
}
```

---

## Error Handling Patterns

### Result Types & Error Conversion

Rust's `Result<T, E>` with string errors for CLI context:

```rust
fn session_run(
    argv: &[String],
    in_tmux: bool,
    runner: &mut impl SessionTmux,
) -> Result<String, String> {
    // Errors returned as String; converted to exit code + stderr by caller
    if !in_tmux {
        return Err("maw session requires an active tmux session".to_owned());
    }
    // ...
    Ok(output_string)
}
```

### Error Classification (Transport)

Portable error classification with retry guidance:

```rust
pub fn classify_error(err: Option<&str>) -> ClassifiedError {
    // Pattern match on error message substrings
    let Some(err) = err else { /* ... */ };
    let msg = err.to_lowercase();
    
    if contains_any(&msg, &["timeout", "etimedout", "econnreset"]) {
        return ClassifiedError {
            reason: TransportFailureReason::Timeout,
            retryable: true,  // Can retry on timeout
        };
    }
    if contains_any(&msg, &["401", "403", "auth"]) {
        return ClassifiedError {
            reason: TransportFailureReason::Auth,
            retryable: false, // Do NOT retry on auth failures
        };
    }
    // ...
}
```

### Display Trait for User-Facing Errors

```rust
impl std::fmt::Display for PeerPubkeyMismatchError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(
            f,
            "peer pubkey changed for {}: {}… → {}…; manually `maw peers forget {}` to re-TOFU",
            self.alias,
            prefix16(&self.cached),
            prefix16(&self.observed),
            self.alias
        )
    }
}

impl Error for PeerPubkeyMismatchError {}
```

---

## Async Patterns

### Async Handler Functions

```rust
fn run_discord_async(args: Vec<String>) -> Pin<Box<dyn Future<Output = CliOutput> + Send>> {
    Box::pin(async move {
        let output = run_discord_command(args).await;
        CliOutput {
            code: output.code,
            stdout: output.stdout,
            stderr: output.stderr,
        }
    })
}
```

### Async/Sync Runtime Boundary

```rust
pub async fn run_cli_async(argv: &[String]) -> CliOutput {
    let Some(command) = argv.first().map(String::as_str) else {
        return usage_ok();
    };

    match dispatcher_target(command) {
        DispatchTarget::Native(handler) => {
            // Sync handler: just call it
            cli_dispatch_log_command(command, &argv[1..]);
            native_or_plugin_fallback(argv, || handler(&argv[1..]))
        }
        DispatchTarget::AsyncNative(handler) => {
            // Async handler: we're inside tokio, await directly
            plugin_fallback_for_native_miss(argv, handler(argv[1..].to_vec()).await)
        }
        DispatchTarget::UnknownCommand => dispatch_cli_plugin_or_unknown(argv, command),
    }
}
```

### Detecting Runtime Context

```rust
fn run_async_handler_blocking(handler: AsyncHandler, args: &[String]) -> CliOutput {
    // Guard against nested block_on (which panics)
    if tokio::runtime::Handle::try_current().is_ok() {
        return CliOutput {
            code: 1,
            stdout: String::new(),
            stderr: "cannot block_on inside runtime; call run_cli_async for async commands\n".to_owned(),
        };
    }

    // Create a new runtime for this blocking call
    let runtime = match tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
    {
        Ok(runtime) => runtime,
        Err(error) => {
            return CliOutput {
                code: 1,
                stdout: String::new(),
                stderr: format!("failed to start tokio runtime: {error}\n"),
            };
        }
    };
    runtime.block_on(handler(args.to_vec()))
}
```

---

## Dependency Management & Workspace Setup

**File**: `/Cargo.toml`

```toml
[workspace]
members = [
    "crates/maw-matcher",
    "crates/maw-worktree",
    "crates/maw-transport",
    "crates/maw-schedule",
    "crates/maw-tmux",
    "crates/maw-peer",
    "crates/maw-auth",
    "crates/maw-xdg",
    "crates/maw-plugin-manifest",
    "crates/maw-cli",
    "crates/maw-discord",
]
resolver = "2"

[workspace.package]
edition = "2021"
license = "BUSL-1.1"

[workspace.dependencies]
tokio = { version = "1", features = ["rt-multi-thread", "macros", "sync", "time", "signal"] }

[workspace.lints.rust]
unsafe_code = "forbid"  # No unsafe code in the entire workspace

[workspace.lints.clippy]
pedantic = { level = "warn", priority = -1 }
```

### maw-cli Dependencies

```toml
[dependencies]
tokio.workspace = true
tokio-tungstenite = { version = "0.24", features = ["rustls-tls-webpki-roots"] }
reqwest = { version = "0.12", features = ["rustls-tls", "json"] }
axum = { version = "0.7", features = ["http1", "json", "tokio", "query", "ws"] }
serde_json = "1"
serde = { version = "1", features = ["derive"] }

# Local crates
maw-auth = { path = "../maw-auth" }
maw-tmux = { path = "../maw-tmux" }
maw-peer = { path = "../maw-peer" }
maw-transport = { path = "../maw-transport" }
maw-xdg = { path = "../maw-xdg" }

# External (shared revision)
maw-auto-wake = { git = "https://github.com/Soul-Brews-Studio/maw-crates", rev = "062afc6..." }
maw-calver = { git = "https://github.com/Soul-Brews-Studio/maw-calver", rev = "c74522b..." }
```

---

## Notable Rust Idioms & Patterns

### 1. Option Chaining with `and_then` & `map`

```rust
// From dispatcher.rs
let Some(command) = argv.first().map(String::as_str) else {
    return usage_ok();
};
```

### 2. Unsized Trait Objects for Handlers

```rust
type AsyncHandler = fn(Vec<String>) -> Pin<Box<dyn Future<Output = CliOutput> + Send>>;

fn run_discord_async(args: Vec<String>) -> Pin<Box<dyn Future<Output = CliOutput> + Send>> {
    Box::pin(async move { /* ... */ })
}
```

### 3. Injected Traits for Testability

```rust
trait SessionTmux {
    fn session_display_message(&mut self, format: &str) -> Result<String, String>;
}

struct SessionSystemTmux;  // Real implementation
struct SessionFakeTmux { /* ... */ }  // Test mock
```

### 4. const Functions for Zero-Cost Abstractions

```rust
impl TransportFailureReason {
    #[must_use]
    pub const fn as_str(self) -> &'static str {
        match self {
            Self::Timeout => "timeout",
            Self::Unreachable => "unreachable",
            // ...
        }
    }
}
```

### 5. RAII for Environment Restoration (Tests)

```rust
#[cfg(test)]
struct EnvVarRestore {
    key: &'static str,
    value: Option<std::ffi::OsString>,
}

#[cfg(test)]
impl Drop for EnvVarRestore {
    fn drop(&mut self) {
        if let Some(value) = self.value.take() {
            std::env::set_var(self.key, value);
        } else {
            std::env::remove_var(self.key);
        }
    }
}
```

### 6. Poison Error Recovery in Mutex Locks (Tests)

```rust
#[cfg(test)]
fn env_test_lock() -> std::sync::MutexGuard<'static, ()> {
    static LOCK: std::sync::Mutex<()> = std::sync::Mutex::new(());
    // Recover from poison: a panicking test leaves the lock's state
    // but we don't want that to cascade to sibling tests
    LOCK.lock().unwrap_or_else(std::sync::PoisonError::into_inner)
}
```

### 7. String Interpolation with `writeln!`

```rust
use std::fmt::Write as _;

let mut out = String::new();
let _ = writeln!(out, "serve: {}", if serve.alive() { "running" } else { "stopped" });
```

### 8. BTreeMap/BTreeSet for Sorted, Unique Collections

```rust
let alive = alive_sessions.iter().cloned().collect::<BTreeSet<_>>();
let mut seen = BTreeSet::new();
for command in &commands {
    assert!(seen.insert(*command), "duplicate dispatcher command: {command}");
}
```

---

## Build-Time Code Generation

**File**: `/crates/maw-cli/build.rs`

The build script:

1. Scans `src/core_impl/*.rs` for command implementations
2. Reads header comments (`//maw:order`, `//maw:noauto`) to control ordering
3. Extracts `DISPATCH_NN` array indices
4. Generates:
   - `parts_includes.rs`: Include statements for all files
   - `dispatch_fragments.rs`: `DISPATCHER_FRAGMENTS` array
   - `tmux_sub_fragments.rs`: Tmux subcommand arrays

```rust
// Scan for order markers in header
fn header_order(contents: &str) -> Option<u32> {
    contents.lines().take(HEADER_SCAN_LINES).find_map(|line| {
        let rest = line.trim().strip_prefix("//maw:order")?.trim();
        if rest.is_empty() { return None; }
        rest.parse().ok()
    })
}

// Skip auto-include if marked
fn has_noauto_header(contents: &str) -> bool {
    contents
        .lines()
        .take(HEADER_SCAN_LINES)
        .any(|line| line.trim() == "//maw:noauto")
}
```

Each command file can declare a dispatch order to control position in the dispatcher array:

```rust
// Hypothetical maw-cli/src/core_impl/my_command.rs
// //maw:order 50
// const DISPATCH_50: &[DispatcherEntry] = &[ /* ... */ ];
```

---

## Test Structure

### Concurrent Audit Test (Catch Race Conditions)

```rust
#[test]
fn concurrent_dispatch_audit_appends_remain_parseable_jsonl() {
    let _guard = env_test_lock();
    let (_state_root, _restores) = cli_dispatch_test_env();
    let workers = 32;
    let rows_per_worker = 32;
    let barrier = std::sync::Arc::new(std::sync::Barrier::new(workers));

    std::thread::scope(|scope| {
        for worker in 0..workers {
            let barrier = std::sync::Arc::clone(&barrier);
            scope.spawn(move || {
                barrier.wait();  // Synchronize thread start
                for row in 0..rows_per_worker {
                    cli_dispatch_log_command("audit-concurrency-test", &[/* ... */]);
                }
            });
        }
    });

    // Verify all rows are valid JSON
    let path = super::audit_jsonl_path(&current_xdg_env());
    let text = fs::read_to_string(path).expect("audit log");
    let rows: Vec<_> = text.lines().collect();
    assert_eq!(rows.len(), workers * rows_per_worker);
    for (line, row) in rows.iter().enumerate() {
        serde_json::from_str::<serde_json::Value>(row)
            .unwrap_or_else(|error| panic!("audit line {line} is corrupt: {error}"));
    }
}
```

### Mock Trait Testing

```rust
#[derive(Default)]
struct SessionFakeTmux {
    responses: Vec<String>,
    calls: Vec<(String, Vec<String>)>,
    error: Option<String>,
}

impl SessionTmux for SessionFakeTmux {
    fn session_display_message(&mut self, format: &str) -> Result<String, String> {
        self.calls.push(("display-message".to_owned(), vec!["-p".to_owned(), format.to_owned()]));
        if let Some(error) = &self.error {
            return Err(format!("session: {error}"));
        }
        Ok(self.responses.pop().unwrap_or_default())
    }
}
```

---

## Summary

**maw-rs** demonstrates production Rust patterns:

- **Modular Architecture**: Separate crates with clear responsibilities
- **Testability**: Injected traits, mock implementations, isolated test environments
- **Build-Time Codegen**: Declarative command registry without manual synchronization
- **Error Handling**: Portable error classification for retry decisions
- **Concurrency Safety**: RAII guards, poison recovery, atomic operations
- **No Unsafe Code**: Workspace-wide forbid attribute
- **Async/Sync Boundary Management**: Explicit handler types, runtime context detection
- **Observability**: Audit trails, comprehensive logging

The codebase prioritizes correctness, testability, and maintainability over micro-optimizations, making it a strong reference for multi-crate Rust systems.
