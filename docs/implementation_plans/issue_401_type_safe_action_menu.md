# Implementation Plan: Type-safe Action Menu (Issue #401)

## Problem Statement

The action menu (the per-message context menu) currently uses string-based key hints (e.g., `"q"`, `"e"`, `"r"`) to link menu items to their corresponding keyboard shortcuts and execution logic. This is "stringly-typed" and prone to typos that can silently break functionality without compiler warnings.

## Proposed Solution

Replace the `key_hint: &'static str` field in the `MenuAction` struct with a strongly-typed enum `ActionMenuHint`. This will ensure that all menu actions are known at compile-time and that the mapping between keys and actions is centralized and type-safe.

## Implementation Steps

### 1. Define `ActionMenuHint` Enum
In `src/app.rs`, define the new enum:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ActionMenuHint {
    Reply, Edit, React, Forward, Copy, Delete, PinToggle, Vote, EndPoll,
    OpenAttachment, OpenLink,
}
```

Implement the following methods for `ActionMenuHint`:
- `from_char(c: char) -> Option<Self>`: Maps a character to its corresponding hint.
- `key_label(self) -> &'static str`: Returns the single-character string representation used in the UI.

### 2. Refactor `MenuAction`
Update the `MenuAction` struct in `src/app.rs`:

```rust
pub struct MenuAction {
    pub label: &'static str,
    pub key_hint: ActionMenuHint, // Changed from &'static str
    pub nerd_icon: &'static str,
}
```

### 3. Update `action_menu_items`
Refactor the `action_menu_items` method in `src/app.rs` to use the new enum variants when constructing `MenuAction` objects (e.g., `key_hint: ActionMenuHint::Reply` instead of `key_hint: "q"`).

### 4. Refactor Action Dispatch Logic
Update the following methods in `src/app.rs`:

- **`handle_action_menu_key`**:
    - In `ListKeyAction::Select`, pass the `ActionMenuHint` directly to `execute_action_by_hint`.
    - In the shortcut key branch (`KeyCode::Char(c)`), use `ActionMenuHint::from_char(c)` to get the hint and check for its existence in the current menu items.
- **`execute_action_by_hint`**:
    - Change the signature from `&str` to `ActionMenuHint`.
    - Change the `match` statement to match on the enum variants.

### 5. Add New Test
Add a test case in `src/app.rs` (or a dedicated test module) that iterates through all `ActionMenuHint` variants and verifies that each one correctly generates a corresponding `MenuAction` in the `action_menu_items()` list for a given message state.

## Verification Plan

1.  **Compile Check**: Ensure the project compiles without errors.
2.  **Lint Check**: Run `cargo clippy --tests -- -D warnings` to ensure no new lints are introduced.
3.  **Unit Tests**: Run `cargo test` to verify existing functionality and the new test case.
4.  **Manual Verification**: (Optional) Run the application and verify that the action menu shortcuts still work as expected.

## Complexity Estimate

- **Effort**: Small (approx. 50-80 LOC).
- **Risk**: Low, as the refactor is localized to the action menu logic and the compiler will guide the changes.
