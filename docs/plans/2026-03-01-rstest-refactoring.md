# rstest Test Refactoring Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Refactor 229 existing tests to use `rstest` fixtures and parameterization, reducing to ~130 test functions while preserving 100% of test case coverage.

**Architecture:** Add `rstest` as a dev-dependency. Convert repetitive test groups into `#[rstest] #[case]` parameterized tests. Replace manual `test_app()`/`test_db()` helpers with `#[fixture]` functions. Add `PartialEq` to `InputAction` to enable `assert_eq!` in parameterized command tests.

**Tech Stack:** Rust, rstest 0.25, cargo test

**Design doc:** `docs/plans/2026-03-01-rstest-refactoring-design.md`

---

### Task 1: Add rstest dependency and derive PartialEq on InputAction

**Files:**
- Modify: `Cargo.toml`
- Modify: `src/input.rs:29`

**Step 1: Add rstest dev-dependency to Cargo.toml**

At the end of `Cargo.toml`, add:

```toml
[dev-dependencies]
rstest = "0.25"
```

**Step 2: Add PartialEq derive to InputAction**

In `src/input.rs:29`, change:

```rust
#[derive(Debug)]
pub enum InputAction {
```

to:

```rust
#[derive(Debug, PartialEq)]
pub enum InputAction {
```

**Step 3: Run tests to verify nothing breaks**

Run: `cargo test`
Expected: All 229 tests pass. No compilation errors.

**Step 4: Commit**

```bash
git add Cargo.toml src/input.rs
git commit -m "chore: add rstest dev-dependency, derive PartialEq on InputAction"
```

---

### Task 2: Refactor input.rs tests

**Files:**
- Modify: `src/input.rs:172-407` (the `#[cfg(test)] mod tests` block)

**Step 1: Rewrite the entire test module**

Replace the test module at `src/input.rs:172-407` with:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use rstest::rstest;

    // --- No-arg commands: 18 cases → 1 parameterized test ---

    #[rstest]
    #[case("/part", InputAction::Part)]
    #[case("/p", InputAction::Part)]
    #[case("/quit", InputAction::Quit)]
    #[case("/q", InputAction::Quit)]
    #[case("/sidebar", InputAction::ToggleSidebar)]
    #[case("/sb", InputAction::ToggleSidebar)]
    #[case("/mute", InputAction::ToggleMute)]
    #[case("/settings", InputAction::Settings)]
    #[case("/attach", InputAction::Attach)]
    #[case("/a", InputAction::Attach)]
    #[case("/contacts", InputAction::Contacts)]
    #[case("/c", InputAction::Contacts)]
    #[case("/help", InputAction::Help)]
    #[case("/h", InputAction::Help)]
    #[case("/block", InputAction::Block)]
    #[case("/unblock", InputAction::Unblock)]
    #[case("/group", InputAction::Group)]
    #[case("/g", InputAction::Group)]
    #[case("/bell", InputAction::ToggleBell(None))]
    fn command_returns_expected_action(#[case] input: &str, #[case] expected: InputAction) {
        assert_eq!(parse_input(input), expected);
    }

    // --- Commands with arguments: extract and check the inner string ---

    #[rstest]
    #[case("/join Alice", InputAction::Join("Alice".to_string()))]
    #[case("/j +1234567890", InputAction::Join("+1234567890".to_string()))]
    #[case("/search hello", InputAction::Search("hello".to_string()))]
    #[case("/s world", InputAction::Search("world".to_string()))]
    #[case("/disappearing 30s", InputAction::SetDisappearing("30s".to_string()))]
    #[case("/dm off", InputAction::SetDisappearing("off".to_string()))]
    #[case("/bell direct", InputAction::ToggleBell(Some("direct".to_string())))]
    #[case("/notify group", InputAction::ToggleBell(Some("group".to_string())))]
    fn command_with_argument(#[case] input: &str, #[case] expected: InputAction) {
        assert_eq!(parse_input(input), expected);
    }

    // --- Commands that require an argument but didn't get one ---

    #[rstest]
    #[case("/join")]
    #[case("/search")]
    #[case("/disappearing")]
    fn command_without_required_arg_returns_unknown(#[case] input: &str) {
        let InputAction::Unknown(s) = parse_input(input) else {
            panic!("expected Unknown for {input}");
        };
        assert!(s.contains("requires"), "error for {input} should mention 'requires': {s}");
    }

    // --- SendText variants ---

    #[rstest]
    #[case("hello world", "hello world")]
    #[case("", "")]
    #[case("   ", "")]
    #[case("  hello  ", "hello")]
    fn send_text_variants(#[case] input: &str, #[case] expected: &str) {
        let InputAction::SendText(s) = parse_input(input) else {
            panic!("expected SendText for {input:?}");
        };
        assert_eq!(s, expected);
    }

    // --- Unknown command ---

    #[test]
    fn unknown_command() {
        let InputAction::Unknown(s) = parse_input("/foo") else { panic!("expected Unknown") };
        assert!(s.contains("/foo"));
    }

    // --- Duration parser: valid cases ---

    #[rstest]
    #[case("off", 0)]
    #[case("0", 0)]
    #[case("30s", 30)]
    #[case("5m", 300)]
    #[case("1h", 3600)]
    #[case("8h", 28800)]
    #[case("1d", 86400)]
    #[case("1w", 604800)]
    #[case("4w", 2419200)]
    fn duration_parser_valid(#[case] input: &str, #[case] expected: u64) {
        assert_eq!(parse_duration_to_seconds(input).unwrap(), expected);
    }

    // --- Duration parser: invalid cases ---

    #[rstest]
    #[case("abc")]
    #[case("")]
    #[case("0s")]
    #[case("-1h")]
    fn duration_parser_invalid(#[case] input: &str) {
        assert!(parse_duration_to_seconds(input).is_err(), "expected error for {input:?}");
    }
}
```

**Step 2: Run tests to verify all cases still pass**

Run: `cargo test input::tests -- --show-output`
Expected: All parameterized cases pass. The test output shows individual `case_N` entries.

**Step 3: Run clippy**

Run: `cargo clippy --tests -- -D warnings`
Expected: No warnings.

**Step 4: Commit**

```bash
git add src/input.rs
git commit -m "refactor: convert input.rs tests to rstest parameterization (42 -> 7 functions)"
```

---

### Task 3: Refactor theme.rs tests

**Files:**
- Modify: `src/theme.rs:625-684` (the `#[cfg(test)] mod tests` block)

**Step 1: Rewrite the test module**

Replace the test module at `src/theme.rs:625-684` with:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use rstest::rstest;

    #[test]
    fn default_theme_has_correct_name() {
        assert_eq!(default_theme().name, "Default");
    }

    #[test]
    fn all_builtin_themes_have_unique_names() {
        let themes = all_themes();
        let mut names: Vec<&str> = themes.iter().map(|t| t.name.as_str()).collect();
        let len = names.len();
        names.sort();
        names.dedup();
        assert_eq!(names.len(), len, "duplicate theme names found");
    }

    #[test]
    fn find_theme_returns_default_for_unknown() {
        let t = find_theme("nonexistent");
        assert_eq!(t.name, "Default");
    }

    #[rstest]
    #[case(Color::Rgb(205, 214, 244), "#cdd6f4")]
    #[case(Color::Cyan, "cyan")]
    #[case(Color::Indexed(236), "indexed(236)")]
    fn color_serde_roundtrip(#[case] color: Color, #[case] expected_str: &str) {
        let s = color_to_string(&color);
        assert_eq!(s, expected_str);
        let c = string_to_color(&s).unwrap();
        assert_eq!(c, color);
    }

    #[test]
    fn theme_toml_roundtrip() {
        let theme = default_theme();
        let toml_str = toml::to_string_pretty(&theme).unwrap();
        let parsed: Theme = toml::from_str(&toml_str).unwrap();
        assert_eq!(parsed.name, theme.name);
        assert_eq!(parsed.bg, theme.bg);
        assert_eq!(parsed.sender_palette, theme.sender_palette);
        assert_eq!(parsed.receipt_viewed, theme.receipt_viewed);
    }
}
```

**Step 2: Run tests**

Run: `cargo test theme::tests`
Expected: All pass.

**Step 3: Commit**

```bash
git add src/theme.rs
git commit -m "refactor: convert theme.rs color roundtrip tests to rstest (7 -> 5 functions)"
```

---

### Task 4: Refactor db.rs tests — fixture + round-trip parameterization

**Files:**
- Modify: `src/db.rs:714-1019` (the `#[cfg(test)] mod tests` block)

**Step 1: Convert test_db to fixture and parameterize round-trips**

This task modifies only the *infrastructure* and *round-trip* tests. All other tests remain, just updated to use the fixture.

Changes:
1. Add `use rstest::{rstest, fixture};` to imports
2. Convert `fn test_db() -> Database` to `#[fixture] fn db() -> Database`
3. Add `#[rstest]` to every existing test and change `let db = test_db();` to parameter `db: Database`
4. Collapse `mute_round_trip`, `blocked_round_trip`, `update_accepted_round_trip` into a single parameterized test

For the round-trip collapse, the three tests follow the same pattern:
- Create conversation
- Set a boolean flag
- Load and verify it's set
- Reverse the flag
- Load and verify it's reversed

The parameterized version needs to use function pointers since each test calls different DB methods. Use a helper approach:

```rust
#[fixture]
fn db() -> Database {
    Database::open_in_memory().unwrap()
}
```

For every existing `#[test]` function, prepend `#[rstest]` and change the first line from `let db = test_db();` to take `db: Database` as a parameter. Example:

```rust
// Before:
#[test]
fn migration_creates_tables() {
    let db = test_db();
    let count: i64 = db.conn.query_row(...

// After:
#[rstest]
fn migration_creates_tables(db: Database) {
    let count: i64 = db.conn.query_row(...
```

For the three round-trip tests, collapse into:

```rust
#[rstest]
#[case("muted",
    |db: &Database, id: &str, val: bool| db.set_muted(id, val).unwrap(),
    |db: &Database| db.load_muted().unwrap()
)]
#[case("blocked",
    |db: &Database, id: &str, val: bool| db.set_blocked(id, val).unwrap(),
    |db: &Database| db.load_blocked().unwrap()
)]
fn boolean_flag_round_trip(
    db: Database,
    #[case] _label: &str,
    #[case] setter: fn(&Database, &str, bool),
    #[case] loader: fn(&Database) -> std::collections::HashSet<String>,
) {
    db.upsert_conversation("+1", "Alice", false).unwrap();
    db.upsert_conversation("+2", "Bob", false).unwrap();

    setter(&db, "+1", true);
    let set = loader(&db);
    assert!(set.contains("+1"));
    assert!(!set.contains("+2"));

    setter(&db, "+1", false);
    let set = loader(&db);
    assert!(!set.contains("+1"));
}
```

**Note:** `update_accepted_round_trip` uses a different pattern (checks `convs[0].accepted` rather than a HashSet), so keep it as a standalone `#[rstest]` test with the fixture.

**Step 2: Run tests**

Run: `cargo test db::tests`
Expected: All pass.

**Step 3: Run clippy**

Run: `cargo clippy --tests -- -D warnings`
Expected: No warnings.

**Step 4: Commit**

```bash
git add src/db.rs
git commit -m "refactor: convert db.rs tests to rstest fixtures + parameterize round-trips"
```

---

### Task 5: Refactor signal/client.rs tests — parameterize JSON parsing groups

**Files:**
- Modify: `src/signal/client.rs:1968-3053` (the `#[cfg(test)] mod tests` block)

This is the second-highest-value refactoring. The JSON parsing tests follow repetitive patterns.

**Step 1: Add rstest import**

Add to the test module's imports:

```rust
use rstest::rstest;
```

**Step 2: Parameterize call message tests**

Collapse `parse_call_message_voice` and `parse_call_message_video` into:

```rust
#[rstest]
#[case("AUDIO_CALL", "Missed voice call")]
#[case("VIDEO_CALL", "Missed video call")]
fn parse_call_message_type(#[case] offer_type: &str, #[case] expected_body: &str) {
    // Build JSON with the offer_type, parse, assert body matches expected_body
    // Copy the JSON structure from the existing parse_call_message_voice test,
    // replacing the hardcoded "AUDIO_CALL" with the offer_type parameter
}
```

Read the existing test bodies to get the exact JSON structure. The key is that the two tests differ only in the `"type"` field value and the expected body string.

**Step 3: Parameterize receipt parsing tests**

Collapse `parse_receipt_event_extracts_type_and_timestamps` and `parse_receipt_event_read` into:

```rust
#[rstest]
#[case(true, false, false, "DELIVERY")]
#[case(false, true, false, "READ")]
fn parse_receipt_variants(
    #[case] is_delivery: bool,
    #[case] is_read: bool,
    #[case] is_viewed: bool,
    #[case] expected_type: &str,
) {
    // Build JSON with the boolean flags, parse, assert receipt_type matches
}
```

**Step 4: Parameterize expiration tests**

Collapse `parse_expiration_update` and `parse_expiration_disabled` into:

```rust
#[rstest]
#[case(604800, "Disappearing messages set to 1 week")]
#[case(0, "Disappearing messages disabled")]
fn parse_expiration_variants(#[case] seconds: i64, #[case] expected_body: &str) {
    // Build JSON with expiresInSeconds, parse, assert body
}
```

**Step 5: Parameterize sticker tests**

Collapse `parse_sticker_message_with_emoji`, `parse_sticker_message_without_emoji`, `parse_sticker_sync` into:

```rust
#[rstest]
#[case(Some("😂"), false, "[Sticker: 😂]")]
#[case(None, false, "[Sticker]")]
#[case(Some("😂"), true, "[Sticker: 😂]")]
fn parse_sticker_variants(
    #[case] emoji: Option<&str>,
    #[case] is_sync: bool,
    #[case] expected_body: &str,
) {
    // Build JSON with/without emoji field, wrap in syncMessage if is_sync
}
```

**Step 6: Parameterize view-once tests**

Collapse `parse_view_once_message`, `parse_view_once_false_passes_through`, `parse_view_once_sync` into:

```rust
#[rstest]
#[case(true, false, "[View-once message]")]
#[case(false, false, "visible text")]  // viewOnce: false passes through normally
#[case(true, true, "[View-once message]")]
fn parse_view_once_variants(
    #[case] view_once: bool,
    #[case] is_sync: bool,
    #[case] expected_body: &str,
) {
    // Build JSON, parse, assert body
}
```

**Step 7: Parameterize text style tests**

Collapse `parse_text_styles_basic` and `parse_text_styles_empty_or_missing` into:

```rust
#[rstest]
#[case(true, 1)]   // has styles → 1 style parsed
#[case(false, 0)]  // no styles → empty
fn parse_text_styles(#[case] has_styles: bool, #[case] expected_count: usize) {
    // Build JSON with or without textStyles array, parse, check count
}
```

**Step 8: Add `#[rstest]` annotation to all remaining standalone tests**

Every remaining `#[test]` that isn't parameterized should just keep `#[test]` — no need to convert standalone tests to `#[rstest]` unless they use a fixture.

**Step 9: Run tests**

Run: `cargo test signal::client::tests`
Expected: All pass.

**Step 10: Run clippy**

Run: `cargo clippy --tests -- -D warnings`
Expected: No warnings.

**Step 11: Commit**

```bash
git add src/signal/client.rs
git commit -m "refactor: convert signal/client.rs tests to rstest parameterization (39 -> ~15 functions)"
```

---

### Task 6: Refactor app.rs tests — fixture + parameterize all identified groups

**Files:**
- Modify: `src/app.rs:4745-7115` (the `#[cfg(test)] mod tests` block)

This is the largest task. It touches ~119 tests but most changes are mechanical.

**Step 1: Add rstest imports and convert test_app to fixture**

```rust
use rstest::{rstest, fixture};

#[fixture]
fn app() -> App {
    let db = Database::open_in_memory().unwrap();
    let mut app = App::new("+10000000000".to_string(), db);
    app.set_connected();
    app
}
```

**Step 2: Update ALL existing tests to use the fixture**

For every test that calls `let mut app = test_app();`, change to:
- Add `#[rstest]` above the function
- Change signature to take `mut app: App`
- Remove the `let mut app = test_app();` line

For tests that use `let app = test_app();` (immutable), use `app: App`.

Example:

```rust
// Before:
#[test]
fn contact_list_does_not_create_conversations() {
    let mut app = test_app();
    assert!(app.conversations.is_empty());
    ...
}

// After:
#[rstest]
fn contact_list_does_not_create_conversations(mut app: App) {
    assert!(app.conversations.is_empty());
    ...
}
```

**Step 3: Parameterize autocomplete_visibility (5 tests → 1)**

```rust
#[rstest]
#[case("/", true, None)]           // slash prefix: visible, non-empty
#[case("/jo", true, Some(1))]      // prefix filtering: visible, 1 match
#[case("hello", false, Some(0))]   // non-slash: hidden, empty
#[case("/join ", false, None)]      // space hides: hidden
#[case("/zzz", false, Some(0))]    // no match: hidden, empty
fn autocomplete_visibility(
    mut app: App,
    #[case] input: &str,
    #[case] expected_visible: bool,
    #[case] expected_count: Option<usize>,
) {
    app.input_buffer = input.to_string();
    app.update_autocomplete();
    assert_eq!(app.autocomplete_visible, expected_visible, "visibility for {input:?}");
    if let Some(count) = expected_count {
        assert_eq!(app.autocomplete_candidates.len(), count, "count for {input:?}");
    }
}
```

Remove: `autocomplete_slash_prefix`, `autocomplete_prefix_filtering`, `autocomplete_non_slash_hidden`, `autocomplete_space_hides`, `autocomplete_no_match`

**Step 4: Parameterize resolve_mentions (4 tests → 1)**

```rust
#[rstest]
#[case(
    &[("uuid-alice", "Alice")],
    "\u{FFFC} check this out",
    &[Mention { start: 0, length: 1, uuid: "uuid-alice".to_string() }],
    "@Alice check this out",
    1
)]
#[case(
    &[],
    "no mentions here",
    &[],
    "no mentions here",
    0
)]
#[case(
    &[("uuid-alice", "Alice"), ("uuid-bob", "Bob")],
    "\u{FFFC} and \u{FFFC} should join",
    &[
        Mention { start: 0, length: 1, uuid: "uuid-alice".to_string() },
        Mention { start: 6, length: 1, uuid: "uuid-bob".to_string() },
    ],
    "@Alice and @Bob should join",
    2
)]
#[case(
    &[],
    "\u{FFFC} said hi",
    &[Mention { start: 0, length: 1, uuid: "abcdef12-3456".to_string() }],
    "@abcdef12 said hi",
    1
)]
fn resolve_mentions_variants(
    mut app: App,
    #[case] uuid_names: &[(&str, &str)],
    #[case] body: &str,
    #[case] mentions: &[Mention],
    #[case] expected_body: &str,
    #[case] expected_range_count: usize,
) {
    for (uuid, name) in uuid_names {
        app.uuid_to_name.insert(uuid.to_string(), name.to_string());
    }
    let (resolved, ranges) = app.resolve_mentions(body, mentions);
    assert_eq!(resolved, expected_body);
    assert_eq!(ranges.len(), expected_range_count);
}
```

Remove: `resolve_mentions_basic`, `resolve_mentions_multiple`, `resolve_mentions_unknown_uuid_fallback`, `resolve_mentions_empty`

**Step 5: Parameterize bell and read receipt filtering (4 tests → 2)**

```rust
#[rstest]
#[case("unaccepted")]
#[case("blocked")]
fn bell_skipped_for_filtered_conversations(mut app: App, #[case] filter_type: &str) {
    match filter_type {
        "unaccepted" => {
            app.handle_signal_event(SignalEvent::MessageReceived(msg_from("+1")));
        }
        "blocked" => {
            app.get_or_create_conversation("+1", "Alice", false);
            if let Some(conv) = app.conversations.get_mut("+1") {
                conv.accepted = true;
            }
            app.blocked_conversations.insert("+1".to_string());
            app.handle_signal_event(SignalEvent::MessageReceived(msg_from("+1")));
        }
        _ => unreachable!(),
    }
    assert!(!app.pending_bell);
}

#[rstest]
#[case("unaccepted")]
#[case("blocked")]
fn read_receipts_not_sent_for_filtered(mut app: App, #[case] filter_type: &str) {
    app.send_read_receipts = true;
    match filter_type {
        "unaccepted" => {
            app.handle_signal_event(SignalEvent::MessageReceived(msg_from("+1")));
        }
        "blocked" => {
            app.get_or_create_conversation("+1", "Alice", false);
            if let Some(conv) = app.conversations.get_mut("+1") {
                conv.accepted = true;
            }
            app.blocked_conversations.insert("+1".to_string());
            app.handle_signal_event(SignalEvent::MessageReceived(msg_from("+1")));
        }
        _ => unreachable!(),
    }
    app.queue_read_receipts_for_conv("+1", 0);
    assert!(app.pending_read_receipts.is_empty());
}
```

Remove: `bell_skipped_for_unaccepted_conversations`, `bell_skipped_for_blocked_conversations`, `read_receipts_not_sent_for_unaccepted_conversations`, `read_receipts_not_sent_for_blocked_conversations`

**Step 6: Parameterize block/unblock no-active-conversation (2 → 1)**

```rust
#[rstest]
#[case("/block", "no active conversation")]
#[case("/unblock", "no active conversation")]
fn block_unblock_no_active_conversation(mut app: App, #[case] cmd: &str, #[case] expected_msg: &str) {
    app.input_buffer = cmd.to_string();
    let req = app.handle_input();
    assert!(req.is_none());
    assert!(app.status_message.contains(expected_msg));
}
```

Remove: `block_no_active_conversation`, `unblock_no_active_conversation`

**Step 7: Parameterize block/unblock already-in-state (2 → 1)**

```rust
#[rstest]
#[case("/block", true, "already blocked")]
#[case("/unblock", false, "not blocked")]
fn block_unblock_already_in_state(
    mut app: App,
    #[case] cmd: &str,
    #[case] pre_blocked: bool,
    #[case] expected_msg: &str,
) {
    app.get_or_create_conversation("+1", "Alice", false);
    app.active_conversation = Some("+1".to_string());
    if pre_blocked {
        app.blocked_conversations.insert("+1".to_string());
    }
    app.input_buffer = cmd.to_string();
    let req = app.handle_input();
    assert!(req.is_none());
    assert!(app.status_message.contains(expected_msg));
}
```

Remove: `block_already_blocked_shows_status`, `unblock_not_blocked_shows_status`

**Step 8: Parameterize mouse scroll (3 → 1)**

```rust
#[rstest]
#[case(0, true, 3)]    // scroll up from 0: offset becomes 3
#[case(10, false, 7)]   // scroll down from 10: offset becomes 7
#[case(1, false, 0)]    // scroll down from 1: saturates at 0
fn mouse_scroll_behavior(
    mut app: App,
    #[case] initial_offset: usize,
    #[case] scroll_up: bool,
    #[case] expected_offset: usize,
) {
    app.mouse_messages_area = Rect::new(0, 0, 80, 20);
    app.scroll_offset = initial_offset;
    let event = if scroll_up {
        mouse_scroll_up(10, 10)
    } else {
        mouse_scroll_down(10, 10)
    };
    app.handle_mouse_event(event);
    assert_eq!(app.scroll_offset, expected_offset);
}
```

Remove: `mouse_scroll_up_increases_offset`, `mouse_scroll_down_decreases_offset`, `mouse_scroll_down_saturates_at_zero`

**Step 9: Parameterize conversation acceptance (3 → 1)**

```rust
#[rstest]
#[case("unknown_sender", false)]
#[case("known_contact", true)]
fn conversation_acceptance(mut app: App, #[case] scenario: &str, #[case] expected_accepted: bool) {
    match scenario {
        "unknown_sender" => {
            app.handle_signal_event(SignalEvent::MessageReceived(msg_from("+1")));
        }
        "known_contact" => {
            app.contact_names.insert("+1".to_string(), "Alice".to_string());
            app.handle_signal_event(SignalEvent::MessageReceived(msg_from("+1")));
        }
        _ => unreachable!(),
    }
    assert_eq!(app.conversations["+1"].accepted, expected_accepted);
}
```

Keep `outgoing_sync_creates_accepted_conversation` as standalone since it has a very different setup (constructs a full outgoing `SignalMessage`).

Remove: `unknown_sender_creates_unaccepted_conversation`, `known_contact_creates_accepted_conversation`

**Step 10: Parameterize group_menu_items_context (3 → 1)**

```rust
#[rstest]
#[case("in_group", 5, "Members", "Leave")]
#[case("not_group", 1, "Create group", "Create group")]
#[case("no_conv", 1, "Create group", "Create group")]
fn group_menu_items_context(
    mut app: App,
    #[case] context: &str,
    #[case] expected_len: usize,
    #[case] first_label: &str,
    #[case] last_label: &str,
) {
    match context {
        "in_group" => {
            app.get_or_create_conversation("g1", "Family", true);
            app.active_conversation = Some("g1".to_string());
        }
        "not_group" => {
            app.get_or_create_conversation("+1", "Alice", false);
            app.active_conversation = Some("+1".to_string());
        }
        "no_conv" => {}
        _ => unreachable!(),
    }
    let items = app.group_menu_items();
    assert_eq!(items.len(), expected_len);
    assert_eq!(items[0].label, first_label);
    assert_eq!(items[items.len() - 1].label, last_label);
}
```

Remove: `group_menu_items_in_group_context`, `group_menu_items_not_in_group`, `group_menu_items_no_conversation`

**Step 11: Parameterize clears_attachment (2 → 1)**

```rust
#[rstest]
#[case("next_conversation")]
#[case("part_command")]
fn clears_attachment_on_navigation(mut app: App, #[case] method: &str) {
    app.get_or_create_conversation("+1", "Alice", false);
    app.active_conversation = Some("+1".to_string());
    app.pending_attachment = Some(std::path::PathBuf::from("/tmp/photo.jpg"));

    match method {
        "next_conversation" => {
            app.get_or_create_conversation("+2", "Bob", false);
            app.next_conversation();
        }
        "part_command" => {
            app.input_buffer = "/part".to_string();
            app.input_cursor = 5;
            app.handle_input();
        }
        _ => unreachable!(),
    }
    assert!(app.pending_attachment.is_none());
}
```

Remove: `next_conversation_clears_attachment`, `part_clears_attachment`

**Step 12: Remove the old test_app function**

Delete the `fn test_app() -> App` function (replaced by the `#[fixture] fn app()`).

**Step 13: Run full test suite**

Run: `cargo test`
Expected: All tests pass.

**Step 14: Run clippy**

Run: `cargo clippy --tests -- -D warnings`
Expected: No warnings.

**Step 15: Commit**

```bash
git add src/app.rs
git commit -m "refactor: convert app.rs tests to rstest fixtures + parameterization (119 -> ~85 functions)"
```

---

### Task 7: Final verification

**Files:** None (verification only)

**Step 1: Run full test suite**

Run: `cargo test`
Expected: All tests pass. Total test count should be similar to before (rstest `#[case]` entries show as individual tests in output).

**Step 2: Run clippy**

Run: `cargo clippy --tests -- -D warnings`
Expected: No warnings.

**Step 3: Verify test count**

Run: `cargo test 2>&1 | tail -1`
Expected: Something like `test result: ok. N passed; 0 failed; 0 ignored` where N is close to the original 229 (each `#[case]` still counts as a separate test).

**Step 4: Verify no behavioral changes**

Run: `cargo build`
Expected: Clean build. The only production code change is `PartialEq` derive on `InputAction`.
