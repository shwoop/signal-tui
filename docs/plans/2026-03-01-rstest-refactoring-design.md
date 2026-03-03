# rstest Test Refactoring Design

## Goal

Refactor the existing 229 tests to use `rstest` for parameterization and fixtures, reducing boilerplate and improving maintainability. No new module coverage is added.

## Dependency

Add `rstest = "0.25"` as a dev-dependency. No other new dependencies.

## Fixtures

Two `#[fixture]` functions replace existing test helpers:

- **`app()`** in `app.rs` — replaces `test_app()`. Creates `App` with in-memory DB and connected state. Tests declare `fn my_test(mut app: App)` to receive it.
- **`db()`** in `db.rs` — replaces `test_db()`. Creates in-memory `Database`. Tests declare `fn my_test(db: Database)`.

Existing non-fixture helpers (`msg_from`, `mouse_down`, `mouse_scroll_up`, `mouse_scroll_down`) remain as plain functions since they take parameters.

## Module-by-Module Plan

### input.rs (42 -> ~6 tests)

| Parameterized Test | Cases | What It Replaces |
|---|---|---|
| `command_returns_expected_action` | ~18 | All no-arg command + alias tests |
| `command_with_argument` | ~6 | `/join Alice`, `/search hello`, `/bell on`, `/notify off`, `/disappearing 30s` |
| `join_and_search_without_arg` | ~2 | Missing arg returns base action |
| `send_text_variants` | ~4 | plain text, empty, whitespace, trimmed |
| `duration_parser_valid` | ~7 | off, 30s, 5m, 1h, 1d, 1w, 4w |
| `duration_parser_invalid` | ~4 | abc, empty, 0s, -1h |

**Note**: If `InputAction` doesn't derive `PartialEq`, we'll need either to add it or use match-based assertion helpers.

### signal/client.rs (39 -> ~15 tests)

| Parameterized Test | Cases | What It Replaces |
|---|---|---|
| `parse_call_message_type` | 2 | voice + video |
| `parse_sticker_variants` | 3 | with emoji, without, sync |
| `parse_view_once_variants` | 3 | incoming, false passthrough, sync |
| `parse_reaction_variants` | 4 | incoming, remove, group, sync |
| `parse_list_contacts_variants` | 5 | basic, empty, empty name, skips no number, with uuid |
| `parse_list_groups_variants` | 3 | basic, empty, skips no id |
| `parse_receipt_variants` | 2 | delivery + read |
| `parse_expiration_variants` | 2 | enabled + disabled |
| `parse_text_styles` | 2 | basic + empty |
| Standalone tests | ~13 | Unique structure (mentions, read sync, send result, etc.) |

### app.rs (119 -> ~85 tests)

| Parameterized Test | Cases | What It Replaces |
|---|---|---|
| `autocomplete_visibility` | 5 | slash prefix, prefix filtering, non-slash hidden, space hides, no match |
| `input_edit_operations` | 6 | char insert, backspace, delete, left/right, home/end, unhandled |
| `resolve_mentions_variants` | 4 | basic, empty, multiple, unknown fallback |
| `text_style_ranges_variants` | 4 | empty, byte offsets, multibyte, with mentions |
| `bell_skipped_for_filtered` | 2 | blocked + unaccepted |
| `read_receipts_not_sent` | 2 | blocked + unaccepted |
| `block_no_active_conversation` | 2 | /block + /unblock with no active conv |
| `block_already_in_state` | 2 | already blocked + not blocked |
| `group_menu_items_context` | 3 | in group, not group, no conv |
| `mouse_scroll_behavior` | 3 | up increases, down decreases, saturates at zero |
| `has_overlay_flags` | 11 | Each overlay boolean as a case |
| `group_action_produces_send_request` | 3 | leave, create, rename |
| `message_request_key_handling` | 3 | accept, delete, esc |
| `conversation_acceptance` | 3 | unknown unaccepted, outgoing accepted, known contact accepted |
| `clears_attachment_on_navigation` | 2 | next_conversation + /part |
| Standalone tests | ~92 | Complex multi-step tests with unique setup |

### db.rs (22 -> ~19 tests)

| Parameterized Test | Cases | What It Replaces |
|---|---|---|
| `boolean_flag_round_trip` | 3 | mute, blocked, accepted |
| `migration_variants` | 3-4 | base, v4 reactions, v8 accepted, v9 blocked |
| Standalone tests | ~16 | Search, unread counts, etc. |

### theme.rs (7 -> 5 tests)

| Parameterized Test | Cases | What It Replaces |
|---|---|---|
| `color_serde_roundtrip` | 3 | hex, named, indexed |
| Standalone tests | 4 | default theme, unique names, unknown fallback, TOML roundtrip |

## Summary

| Module | Before | After | Reduction |
|---|---|---|---|
| input.rs | 42 | ~6 | -36 |
| client.rs | 39 | ~15 | -24 |
| app.rs | 119 | ~85 | -34 |
| db.rs | 22 | ~19 | -3 |
| theme.rs | 7 | 5 | -2 |
| **Total** | **229** | **~130** | **-99 (~43%)** |

## Constraints

- All existing test cases must remain exercised (same coverage, fewer test functions).
- No behavioral changes to production code.
- `PartialEq` may need to be derived on `InputAction` and potentially `SendRequest` for clean assertions in parameterized tests. If this is undesirable, match-based helpers will be used instead.
- Each `#[case]` produces a separate test binary entry, so `cargo test` output will still show individual case results.
