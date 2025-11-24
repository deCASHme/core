# CLAUDE.md - AI Assistant Guide for Delta Chat Core

> **Version:** 2.27.0
> **Last Updated:** 2025-11-24
> **Purpose:** Comprehensive guide for AI assistants working on the Delta Chat Core codebase

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture and Design Principles](#architecture-and-design-principles)
3. [Codebase Structure](#codebase-structure)
4. [Technology Stack](#technology-stack)
5. [Development Workflow](#development-workflow)
6. [Coding Conventions](#coding-conventions)
7. [Testing Strategy](#testing-strategy)
8. [Common Tasks](#common-tasks)
9. [Important Constraints](#important-constraints)
10. [API Migration Strategy](#api-migration-strategy)
11. [Resources](#resources)

---

## Project Overview

**Delta Chat Core** is a low-level Rust library implementing secure, privacy-focused instant messaging over standard email protocols. It serves as the foundation for Delta Chat applications on Android, iOS, Desktop, and various other platforms.

### Key Features

- **Email-Based Messaging**: Uses existing email infrastructure (IMAP/SMTP)
- **End-to-End Encryption**: Implements Autocrypt and OpenPGP standards
- **Decentralized**: No central servers required for messaging
- **Cross-Platform**: Rust core with bindings for C, Python, JavaScript, Go, Java, Swift
- **P2P Capabilities**: Iroh-based peer-to-peer networking for multi-device sync
- **WebXDC Apps**: Embeddable web applications in messages

### Quick Facts

- **Language**: Rust (Edition 2024, MSRV 1.85.0)
- **License**: MPL-2.0
- **Repository**: https://github.com/chatmail/core
- **Version**: 2.27.0
- **Lines of Code**: ~117 Rust files in `/src/`, ~100k+ lines total
- **Build System**: Cargo (primary), CMake (C library), Nix (reproducible builds)

---

## Architecture and Design Principles

### Core Design Principles

1. **Email-First Architecture**
   - All messaging works over standard email protocols
   - Compatible with existing email servers worldwide
   - No proprietary protocols required

2. **Encryption by Default**
   - Autocrypt for automatic key exchange
   - OpenPGP (via rPGP library) for encryption/decryption
   - Secure Join protocol for verified group setup

3. **Privacy by Protocol**
   - No central tracking or analytics servers
   - End-to-end encryption for message content
   - Minimal metadata exposure

4. **Offline-First**
   - Messages queue when offline
   - Works with delayed delivery
   - Local SQLite database for all data

5. **Avoid Panics**
   - No `.unwrap()` or `.expect()` in library code
   - Use `anyhow::Result` for error handling
   - Panics are difficult to debug on user devices

6. **Deterministic Behavior**
   - Use `BTreeMap` instead of `HashMap`
   - Avoid non-deterministic code
   - Ensures consistent behavior across devices

### System Architecture

```
┌─────────────────────────────────────────────────┐
│           Client Applications                    │
│  (Android, iOS, Desktop, Bots, CLIs)            │
└──────────────┬──────────────────────────┬───────┘
               │                          │
       ┌───────▼─────────┐       ┌───────▼──────────┐
       │  deltachat-ffi  │       │ JSON-RPC Clients │
       │  (Deprecated)   │       │  (Python/JS/Go)  │
       └───────┬─────────┘       └────────┬─────────┘
               │                          │
               │                  ┌───────▼──────────────┐
               │                  │ deltachat-rpc-server │
               │                  │   (stdio JSON-RPC)   │
               │                  └───────┬──────────────┘
               │                          │
               └──────────┬───────────────┘
                          │
                  ┌───────▼────────────┐
                  │ deltachat-jsonrpc  │
                  │   (RPC API Layer)  │
                  └───────┬────────────┘
                          │
        ┌─────────────────▼───────────────────┐
        │      deltachat (Core Library)        │
        │   - Email protocols (IMAP/SMTP)      │
        │   - Encryption (Autocrypt/PGP)       │
        │   - Chat logic & database            │
        │   - P2P networking (Iroh)            │
        └─────┬────────────────────────┬───────┘
              │                        │
    ┌─────────▼──────┐     ┌──────────▼─────────┐
    │ Helper Crates  │     │  Protocol Support  │
    │ - time         │     │  - format-flowed   │
    │ - ratelimit    │     │  - contact-tools   │
    │ - derive       │     │                    │
    └────────────────┘     └────────────────────┘
```

### Key Components

- **Context** (`src/context.rs`): Main API entry point, manages database and events
- **IMAP** (`src/imap.rs`): Email receiving, IDLE push, folder management
- **SMTP** (`src/smtp.rs`): Email sending with connection pooling
- **Chat** (`src/chat.rs`): Chat management (largest module, 4,954 lines)
- **Message** (`src/message.rs`): Message creation and handling
- **Encryption** (`src/pgp.rs`, `src/e2ee.rs`): PGP encryption and Autocrypt
- **Database** (`src/sql.rs`): SQLite with SQLCipher encryption
- **P2P** (Iroh integration): Multi-device sync and WebXDC realtime

---

## Codebase Structure

### Directory Layout

```
/home/user/core/
├── src/                      # Core library (117 .rs files)
│   ├── blob/                 # Binary storage
│   ├── calls/                # Voice/video calls
│   ├── chat/                 # Chat management (4,954 lines)
│   ├── config/               # Configuration
│   ├── configure/            # Email auto-config
│   ├── contact/              # Contact management
│   ├── context/              # Main Context API
│   ├── ephemeral/            # Disappearing messages
│   ├── events/               # Event system
│   ├── imap/                 # IMAP protocol (2,638 lines)
│   ├── imex/                 # Import/Export
│   ├── message/              # Message handling
│   ├── mimefactory/          # MIME creation
│   ├── mimeparser/           # MIME parsing
│   ├── net/                  # Network layer
│   ├── pgp/                  # Encryption
│   ├── qr/                   # QR codes
│   ├── receive_imf/          # Message reception (4,061 lines)
│   ├── scheduler/            # Job scheduling
│   ├── securejoin/           # Secure group protocol
│   ├── smtp/                 # SMTP protocol
│   ├── sql/                  # Database operations
│   ├── tests/                # Integration tests
│   └── webxdc/               # WebXDC apps
│
├── deltachat-ffi/            # C FFI bindings (deprecated)
├── deltachat-jsonrpc/        # JSON-RPC API layer
│   └── typescript/           # TypeScript client
├── deltachat-rpc-server/     # RPC server binary
├── deltachat-rpc-client/     # Python RPC client
├── deltachat-repl/           # CLI REPL client
├── deltachat-contact-tools/  # Contact utilities
├── deltachat-time/           # Time utilities
├── deltachat-ratelimit/      # Rate limiting
├── deltachat_derive/         # Proc macros
├── format-flowed/            # RFC 3676 support
│
├── python/                   # Legacy CFFI bindings (deprecated)
├── benches/                  # Performance benchmarks (10 files)
├── fuzz/                     # Fuzzing targets
├── test-data/                # Test fixtures
├── scripts/                  # CI/CD and dev tools
├── .github/workflows/        # GitHub Actions CI
│
├── Cargo.toml                # Workspace manifest
├── CMakeLists.txt            # CMake build
├── flake.nix                 # Nix reproducible builds
├── CONTRIBUTING.md           # Contribution guide
├── STYLE.md                  # Coding conventions
└── CHANGELOG.md              # Version history (252KB!)
```

### Workspace Members

The repository uses a Cargo workspace with these crates:

1. **deltachat** (root) - Core library
2. **deltachat-ffi** - C bindings (being deprecated)
3. **deltachat-jsonrpc** - JSON-RPC API (preferred)
4. **deltachat-rpc-server** - Standalone RPC server
5. **deltachat-repl** - CLI client
6. **deltachat_derive** - Procedural macros
7. **deltachat-time** - Time utilities
8. **deltachat-ratelimit** - Rate limiting
9. **deltachat-contact-tools** - Contact parsing
10. **format-flowed** - Text formatting
11. **fuzz** - Fuzzing targets

### Important Files

- `src/lib.rs` - Core library entry point
- `src/context.rs` - Main Context API (1,335 lines)
- `src/chat.rs` - Chat logic (4,954 lines, largest module)
- `src/receive_imf.rs` - Message reception (4,061 lines)
- `Cargo.toml` - Dependencies and workspace config
- `deny.toml` - Dependency auditing rules
- `cliff.toml` - Changelog generation config

---

## Technology Stack

### Rust Dependencies (Key)

**Protocols & Networking**
- `async-imap` 0.11.1 - IMAP protocol
- `async-smtp` 0.10.2 - SMTP protocol
- `tokio` 1.x - Async runtime
- `hyper` 1.x - HTTP client
- `fast-socks5` 0.10 - SOCKS5 proxy
- `shadowsocks` 1.23.1 - Shadowsocks proxy

**Cryptography**
- `pgp` 0.17.0 (rPGP) - OpenPGP
- `blake3` 1.8.2 - Hashing
- `tokio-rustls` 0.26.2 - TLS

**Email & MIME**
- `mailparse` 0.16.1 - Email parsing
- `mail-builder` 0.4.4 - Email construction
- `quick-xml` 0.38 - XML parsing

**Database**
- `rusqlite` 0.36 - SQLite
- Feature: `sqlcipher` for encryption

**P2P**
- `iroh` 0.35 - P2P networking
- `iroh-gossip` 0.35 - Gossip protocol

**Serialization**
- `serde` 1.0, `serde_json` 1.0
- `yerpc` 0.6.4 - JSON-RPC framework

**Error Handling**
- `anyhow` 1.x - Error handling (primary)
- `thiserror` 2.x - Custom errors

**Utilities**
- `chrono` 0.4.42 - Date/time
- `regex` 1.10 - Regular expressions
- `uuid` 1.x - UUID generation
- `image` 0.25.6 - Image processing
- `qrcodegen` 1.7.0 - QR codes

### Build Tools

- **Cargo** - Primary build system
- **CMake** - C library installation
- **Nix** - Reproducible builds
- **NPM** - TypeScript client
- **setuptools** - Python packages

### Testing Frameworks

- **cargo nextest** - Rust test runner
- **pytest** - Python testing
- **Mocha + Chai** - JavaScript testing
- **proptest** - Property-based testing
- **cargo-bolero** - Fuzzing

---

## Development Workflow

### Setting Up Development Environment

```bash
# Install Rust
curl https://sh.rustup.rs -sSf | sh

# Clone repository
git clone https://github.com/chatmail/core.git
cd core

# Build the project
cargo build --locked

# Run tests
cargo test --all

# Run CLI client
cargo run --locked -p deltachat-repl -- ~/test-db

# Optional: Install CLI
cargo install --locked --path deltachat-repl/
```

### Branch Naming Convention

When creating branches for PRs:
- Pattern: `<username>/<feature>`
- Example: `alice/fix-race-condition`
- Makes it clear who is responsible for the branch

### Commit Message Format

Follow [Conventional Commits](https://www.conventionalcommits.org/):

**Prefixes:**
- `feat`: New features
- `fix`: Bug fixes
- `api`: API changes
- `refactor`: Code refactoring
- `perf`: Performance improvements
- `test`: Test changes
- `build`: Build system changes
- `ci`: CI configuration
- `docs`: Documentation
- `chore`: Miscellaneous tasks

**Breaking Changes:**
- Use `!` after prefix: `api!: Remove dc_chat_can_send`
- Or add `BREAKING CHANGE:` in commit body

**Examples:**
```
feat: Pause IO for BackupProvider
fix: delete smtp rows when message sending is canceled
api(rust): add get_msg_read_receipts(context, msg_id)
refactor: iterate over msg_ids without .iter()
perf: improve SQLite performance with PRAGMA synchronous=normal
```

**Check Your Commit:**
```bash
git cliff --unreleased
```

### Pull Request Workflow

1. **Select an Issue**: Assign to yourself or comment
2. **Write the Code**: Follow `STYLE.md` conventions
3. **Commit**: Use conventional commit format
4. **Open PR**: Reference the issue
5. **CI Checks**: Ensure all pass
6. **Request Review**: Use GitHub review feature
7. **Merge**: Author merges if they have access

**Merge Strategies:**
- **Squash merge**: Single logical change
- **Rebase merge**: Multiple coherent commits
- Ensure PR title follows conventional commits if squashing

### Pre-Commit Checks

```bash
# Format code
cargo fmt

# Run linter
./scripts/clippy.sh

# Run tests
cargo test --all

# Check changelog preview
git cliff --unreleased

# Run spell check
./scripts/codespell.sh

# Check dependencies
./scripts/deny.sh
```

### CI Pipeline

GitHub Actions runs on every push:
- **Formatting**: `cargo fmt --check`
- **Linting**: `scripts/clippy.sh`
- **Tests**: Rust, Python, TypeScript
- **MSRV Check**: Ensures Rust 1.85.0 compatibility
- **Documentation**: Builds all docs
- **Dependencies**: cargo-deny audit
- **Cross-Platform**: Linux, macOS, Windows

---

## Coding Conventions

### General Style

**Formatting:**
```bash
cargo fmt  # Always run before committing
```

**Linting:**
```bash
./scripts/clippy.sh  # Fix all warnings
```

### SQL Conventions

**Multi-line SQL:**
```rust
// ✅ CORRECT
sql.execute(
    "CREATE TABLE messages (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        text TEXT DEFAULT '' NOT NULL,  -- message text
        timestamp INTEGER NOT NULL
    ) STRICT",
)
.await?;

// ❌ WRONG - Don't escape newlines
sql.execute(
    "CREATE TABLE messages ( \
        id INTEGER PRIMARY KEY AUTOINCREMENT, \
        text TEXT DEFAULT '' NOT NULL \
    ) STRICT",
)
.await?;
```

**SQL Requirements:**
- Use `STRICT` for new tables
- Use `AUTOINCREMENT` for primary keys
- All columns `NOT NULL` with `DEFAULT` values
- Use `IFNULL()` for existing nullable columns
- Never change column types or semantics (breaks downgrades)
- Don't delete columns too early (old versions need them)

### Error Handling

**Use `anyhow` for most errors:**
```rust
// ✅ CORRECT
use anyhow::{Context, Result};

fn process_message(msg_id: MsgId) -> Result<()> {
    let msg = get_message(msg_id)
        .with_context(|| format!("Unable to load message {msg_id}"))?;
    // Process...
    Ok(())
}
```

**Context formatting:**
- Capitalize first word
- No full stop (contexts chain with `:`)

**Never panic in library code:**
```rust
// ❌ WRONG - Don't use in library
value.unwrap()
value.expect("message")

// ✅ CORRECT
value?  // Propagate error
value.context("Unable to...")?
if let Err(err) = value {
    warn!(context, "Failed: {err:#}.");
}
```

**In tests, `.expect()` is acceptable:**
```rust
#[tokio::test]
async fn test_something() -> Result<()> {
    let context = TestContext::new().await.expect("Failed to create context");
    // ...
}
```

**Async streams:**
```rust
// ✅ CORRECT - Use try_next()
while let Some(event) = stream.try_next().await? {
    process(event)?;
}

// ❌ WRONG - Don't use next() for Result streams
while let Some(event_res) = stream.next().await {
    let event = event_res?;  // Easy to forget!
}
```

### Collections

**Prefer BTreeMap/BTreeSet:**
```rust
// ✅ CORRECT - Deterministic iteration
use std::collections::BTreeMap;
let mut map = BTreeMap::new();

// ❌ WRONG - Non-deterministic
use std::collections::HashMap;
let mut map = HashMap::new();
```

**Reason:** Deterministic behavior prevents:
- Flaky tests
- Different behavior across devices
- Difficult-to-reproduce bugs

### Logging

**Log message format:**
```rust
// Capitalize first word, full stop at end
info!(context, "Ignoring addition of {added_addr:?} to {chat_id}.");
warn!(context, "Failed to connect: {err:#}.");
error!(context, "Database error: {err:#}.");

// Format anyhow errors with {:#} for full context chain
error!(context, "Failed to set selfavatar timestamp: {err:#}.");
```

### Documentation

**All public items must be documented:**
```rust
/// Sends a message to a chat.
///
/// Returns the message ID of the sent message.
///
/// # Arguments
///
/// * `chat_id` - The chat to send to
/// * `text` - The message text
///
/// # Examples
///
/// ```
/// let msg_id = chat.send_text_msg("Hello!").await?;
/// ```
pub async fn send_text_msg(&self, text: &str) -> Result<MsgId> {
    // ...
}
```

**Private items should be documented too** (helps with `cargo doc --document-private-items`)

**Follow RFC 1574:**
- Capitalize first word
- Full sentences
- Code examples
- Link to related items

---

## Testing Strategy

### Test Organization

**Unit Tests** - In source files:
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_something() {
        assert_eq!(add(2, 2), 4);
    }

    #[tokio::test]
    async fn test_async_something() -> Result<()> {
        // Async test
        Ok(())
    }
}
```

**Integration Tests** - In `src/tests/`:
- Located in `/src/tests/` directory
- Test cross-module functionality
- Use `TestContext` helper

**Doc Tests** - In documentation:
```rust
/// Adds two numbers.
///
/// ```
/// use deltachat::tools::add;
/// assert_eq!(add(2, 2), 4);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

**Benchmarks** - In `benches/`:
- Use Criterion.rs
- Run with `cargo bench`

### Running Tests

```bash
# All tests
cargo test --all

# Specific test
cargo test test_name

# With logging
RUST_LOG=info cargo test test_name

# Expensive tests (marked with #[ignore])
cargo test -- --ignored

# Online tests (require CHATMAIL_DOMAIN)
export CHATMAIL_DOMAIN=ci-chatmail.testrun.org
cargo test

# Using nextest (faster, parallel)
cargo nextest run --workspace --locked

# Coverage
./scripts/coverage.sh

# Python tests
./scripts/run-python-test.sh

# RPC tests
./scripts/run-rpc-test.sh
```

### Test Helpers

**TestContext:**
```rust
use crate::test_utils::TestContext;

#[tokio::test]
async fn test_send_message() -> Result<()> {
    let t = TestContext::new().await;
    let chat_id = t.create_chat_with_contact("alice@example.org").await;
    let msg_id = send_text_msg(&t, chat_id, "Hello").await?;
    assert!(msg_id.is_valid());
    Ok(())
}
```

### Test Categories

**Offline Tests:**
- Run without network
- Use test data from `test-data/`
- Fast and reliable

**Online Tests:**
- Require test email server
- Set `CHATMAIL_DOMAIN` environment variable
- Test real email sending/receiving

**Expensive Tests:**
- Marked with `#[ignore]`
- Long-running or resource-intensive
- Run separately in CI

### Debugging Tests

```bash
# With backtrace
RUST_BACKTRACE=1 cargo test test_name

# With full backtrace and line numbers
RUST_BACKTRACE=full cargo test test_name

# With MIME debugging
DCC_MIME_DEBUG=1 cargo test test_name

# With protocol tracing
RUST_LOG=async_imap=trace,async_smtp=trace cargo test test_name
```

---

## Common Tasks

### Adding a New Feature

1. **Update Core Library** (`src/`)
   ```rust
   // Add to appropriate module
   pub async fn new_feature(&self) -> Result<()> {
       // Implementation
   }
   ```

2. **Add to JSON-RPC API** (`deltachat-jsonrpc/src/`)
   ```rust
   #[rpc_method]
   async fn new_feature(&self, account_id: u32) -> RpcResult<()> {
       let ctx = self.get_context(account_id).await?;
       ctx.new_feature().await?;
       Ok(())
   }
   ```

3. **Update TypeScript Types** (auto-generated from schema)

4. **Add Tests**
   ```rust
   #[tokio::test]
   async fn test_new_feature() -> Result<()> {
       let t = TestContext::new().await;
       t.new_feature().await?;
       Ok(())
   }
   ```

5. **Update Documentation**
   - Add rustdoc comments
   - Update CHANGELOG.md
   - Update README.md if user-facing

6. **Commit**
   ```bash
   git add .
   git commit -m "feat: Add new feature for X"
   ```

### Fixing a Bug

1. **Write Failing Test First**
   ```rust
   #[tokio::test]
   async fn test_bug_reproduction() -> Result<()> {
       // This should fail before the fix
       let t = TestContext::new().await;
       // Reproduce bug
       Ok(())
   }
   ```

2. **Fix the Bug**
   ```rust
   // Make minimal changes to fix
   ```

3. **Verify Test Passes**
   ```bash
   cargo test test_bug_reproduction
   ```

4. **Commit**
   ```bash
   git commit -m "fix: Fix race condition in message sending"
   ```

### Adding a Database Column

```rust
// In migration code (src/sql.rs)
sql.execute(
    "ALTER TABLE messages
     ADD COLUMN new_field TEXT DEFAULT '' NOT NULL",
    []
).await?;

// Update schema version
const CURRENT_SCHEMA_VERSION: i32 = 123;  // Increment
```

**Important:**
- Always use `NOT NULL` with `DEFAULT`
- Never change existing column types
- Never change column semantics
- Don't delete columns too early

### Adding a Configuration Option

```rust
// In src/config.rs
pub enum Config {
    // ...

    /// Description of new config option.
    ///
    /// Default: "default_value"
    NewConfigOption,
}

impl Config {
    pub fn to_sql(self) -> &'static str {
        match self {
            // ...
            Self::NewConfigOption => "new_config_option",
        }
    }
}

// In src/configure/mod.rs
// Add default value if needed
```

### Updating Dependencies

```bash
# Update Cargo.lock
cargo update

# Update specific dependency
cargo update -p dependency_name

# Test after updating
cargo test --all

# Check for security advisories
./scripts/deny.sh

# Update provider database
./scripts/update-provider-database.sh
```

### Running Benchmarks

```bash
# Run all benchmarks
cargo bench

# Run specific benchmark
cargo bench --bench create_account

# Compare with baseline
cargo bench -- --save-baseline main
git checkout feature-branch
cargo bench -- --baseline main
```

### Fuzzing

```bash
# Install cargo-bolero
cargo install cargo-bolero

# Run fuzzer
cd fuzz
cargo bolero test fuzz_mailparse -s NONE

# Add corpus from test data
cp ../test-data/message/*.eml fuzz_targets/corpus/fuzz_mailparse/
```

---

## Important Constraints

### Critical Rules

1. **Never Panic in Library Code**
   - No `.unwrap()` or `.expect()`
   - Always use `Result` and `?`
   - Panics are hard to debug on devices

2. **Never Change Database Schema Carelessly**
   - Don't change column types
   - Don't change column semantics
   - Don't delete columns too early (old versions need them)
   - Always use migrations

3. **Maintain Deterministic Behavior**
   - Use `BTreeMap` not `HashMap`
   - Use `BTreeSet` not `HashSet`
   - Avoid timing-dependent code
   - Ensures consistent behavior across devices

4. **Preserve Backwards Compatibility**
   - Old versions must read new databases (within reason)
   - Don't break import/export format
   - Mark breaking changes with `!` in commits

5. **Never Skip Security**
   - All external input must be validated
   - SQL: Use parameterized queries
   - Email: Sanitize headers
   - HTML: Escape properly
   - Check OWASP Top 10

### Common Pitfalls

**SQL NULL Values:**
```rust
// ❌ WRONG - Nullable columns are problematic
"CREATE TABLE foo (bar TEXT)"

// ✅ CORRECT - NOT NULL with DEFAULT
"CREATE TABLE foo (bar TEXT DEFAULT '' NOT NULL)"
```

**HashMap Non-Determinism:**
```rust
// ❌ WRONG - Order changes between runs
use std::collections::HashMap;
for (k, v) in hashmap.iter() {
    process(k, v);  // Order unpredictable!
}

// ✅ CORRECT - Deterministic order
use std::collections::BTreeMap;
for (k, v) in btreemap.iter() {
    process(k, v);  // Always same order
}
```

**Unwrap in Library:**
```rust
// ❌ WRONG - Will panic on None/Err
let value = option.unwrap();
let result = fallible_fn().expect("failed");

// ✅ CORRECT - Handle errors
let value = option.context("Missing value")?;
let result = fallible_fn().context("Operation failed")?;
```

**Breaking Schema:**
```rust
// ❌ WRONG - Changes column type
"ALTER TABLE messages ALTER COLUMN timestamp TYPE BIGINT"

// ✅ CORRECT - Add new column instead
"ALTER TABLE messages ADD COLUMN timestamp_v2 BIGINT"
```

### Performance Considerations

- **Database**: Use indexes, avoid N+1 queries
- **Async**: Don't block async executor with sync I/O
- **Memory**: Be mindful of large message batches
- **Network**: Implement timeouts and retries
- **Encryption**: PGP operations are CPU-intensive

### Security Considerations

- **SQL Injection**: Always use parameterized queries
- **Path Traversal**: Validate file paths
- **Email Headers**: Sanitize and validate
- **HTML/XSS**: Escape user content
- **Command Injection**: Never use raw commands with user input
- **Timing Attacks**: Use constant-time comparisons for secrets

---

## API Migration Strategy

### Current State

**Two API Approaches:**

1. **C FFI (deltachat-ffi)** - **DEPRECATED**
   - Used by: Android, iOS, Ubuntu Touch
   - Harder to maintain
   - Being phased out

2. **JSON-RPC (deltachat-jsonrpc)** - **PREFERRED**
   - Used by: Python, JavaScript, Go
   - Auto-generated bindings
   - Language-agnostic
   - Easier maintenance

### For New Development

**Always use JSON-RPC API:**
```rust
// Add to deltachat-jsonrpc/src/api/
#[rpc_method]
async fn your_new_method(&self, account_id: u32, param: String) -> RpcResult<ReturnType> {
    let ctx = self.get_context(account_id).await?;
    let result = ctx.your_implementation(param).await?;
    Ok(result)
}
```

**TypeScript bindings auto-generated:**
```typescript
// Automatically available after build
await deltachat.yourNewMethod(accountId, param);
```

**Python bindings:**
```python
# Available via RPC client
result = await rpc.your_new_method(account_id, param)
```

### Migration Plan

- **Short-term**: Maintain both FFI and JSON-RPC
- **Medium-term**: Encourage clients to migrate to JSON-RPC
- **Long-term**: Deprecate and remove FFI completely

**When adding features:**
1. Implement in core library (`src/`)
2. Expose via JSON-RPC API
3. (Optional) Add to FFI if needed for compatibility
4. Update JSON-RPC clients (Python, JS, etc.)

---

## Resources

### Documentation

- **Main README**: `/README.md`
- **Contributing Guide**: `/CONTRIBUTING.md`
- **Coding Style**: `/STYLE.md`
- **Release Process**: `/RELEASE.md`
- **Changelog**: `/CHANGELOG.md`
- **Protocol Spec**: `/spec.md`
- **Standards Used**: `/standards.md`
- **WebXDC Guide**: `/webxdc.md`

### Generated Documentation

- **Rust API**: `cargo doc --open` or https://docs.rs/deltachat
- **C API**: https://c.delta.chat
- **Python API**: https://py.delta.chat
- **JavaScript API**: https://js.jsonrpc.delta.chat
- **OpenRPC Spec**: `deltachat-rpc-server --openrpc`

### Online Resources

- **Website**: https://delta.chat
- **Forum**: https://support.delta.chat
- **GitHub**: https://github.com/chatmail/core
- **Issue Tracker**: https://github.com/chatmail/core/issues
- **Standards**: https://securejoin.rtfd.io (Autocrypt/SecureJoin)

### Key Scripts

```bash
# Testing
./scripts/run-rust-test.sh      # Rust tests
./scripts/run-python-test.sh    # Python tests
./scripts/run-rpc-test.sh       # RPC tests
./scripts/coverage.sh           # Code coverage

# Quality
./scripts/clippy.sh             # Linting
./scripts/codespell.sh          # Spell check
./scripts/deny.sh               # Dependency audit

# Documentation
./scripts/run-doxygen.sh        # Generate C docs
cargo doc --open                # Generate Rust docs

# Maintenance
./scripts/update-provider-database.sh
./scripts/set_core_version.py <version>
```

### Environment Variables

```bash
# Debugging
export DCC_MIME_DEBUG=1                    # Print MIME messages
export RUST_LOG=info                       # Set log level
export RUST_LOG=async_imap=trace           # IMAP protocol tracing
export RUST_BACKTRACE=1                    # Enable backtraces

# Testing
export CHATMAIL_DOMAIN=ci-chatmail.testrun.org  # Online test server
export DC_ACCOUNTS_PATH=/path/to/accounts       # Account storage

# Build
export CARGO_PROFILE_RELEASE_LTO=true      # Link-time optimization
```

### Quick Reference

**Minimum Rust Version**: 1.85.0
**Standard Rust Version (CI)**: 1.91.0
**Default Branch**: main
**License**: MPL-2.0
**Code of Conduct**: Follow GitHub community guidelines

---

## For AI Assistants: Best Practices

When working on this codebase:

1. **Always read files before modifying** - Never propose changes to code you haven't read
2. **Follow the style guide** - Run `cargo fmt` and `./scripts/clippy.sh`
3. **Write tests** - Add tests for all new features and bug fixes
4. **Check commits** - Use `git cliff --unreleased` to preview changelog
5. **Handle errors properly** - Use `anyhow::Result` and `?`, never `.unwrap()`
6. **Be deterministic** - Use `BTreeMap`/`BTreeSet` instead of Hash variants
7. **Document everything** - Public items must have rustdoc comments
8. **Think about compatibility** - Don't break database schema or APIs
9. **Consider security** - Validate all external input
10. **Test cross-platform** - Remember Windows, macOS, Android, iOS

**When unsure:**
- Check existing code for patterns
- Read `STYLE.md` and `CONTRIBUTING.md`
- Ask in PR comments or forum
- Look for similar functions in the codebase

**Remember:**
- This is a **privacy and security critical** application
- Bugs can affect users worldwide
- Always prioritize correctness over cleverness
- Simple, readable code is better than "elegant" complexity

---

**Last Updated**: 2025-11-24
**Version**: 2.27.0
**Maintainers**: Delta Chat Core Team

For questions or clarifications, see https://github.com/chatmail/core/contribute
