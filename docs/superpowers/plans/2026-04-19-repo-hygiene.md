# Repo Hygiene Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the three hygiene gaps surfaced by the April 11-19 CI outage audit: pin the Rust toolchain, enforce `cargo fmt` in CI, and require CI to pass before merge.

**Architecture:** Three independent PRs shipped in order. Each is revertible without affecting the others. Tasks 1 and 2 are code changes. Task 3 is a GitHub repo settings change documented here but executed in the UI or via `gh api`.

**Tech Stack:** Rust 1.95.0, `dtolnay/rust-toolchain@stable` CI action (respects `rust-toolchain.toml`), GitHub repo rulesets.

**Spec:** `docs/superpowers/specs/2026-04-19-repo-hygiene-design.md`

---

### Task 1: Pin the Rust toolchain

**Files:**
- Create: `rust-toolchain.toml`
- Modify: `CLAUDE.md` (add "Toolchain updates" section)

- [ ] **Step 1: Confirm current stable Rust version**

Run: `rustc --version`
Expected: `rustc 1.95.0 (59807616e 2026-04-14)` (or similar — whatever the current stable is when the plan is executed). Record the exact version; it goes in the toolchain file.

- [ ] **Step 2: Create feature branch**

```bash
git checkout master
git pull origin master
git checkout -b chore/pin-rust-toolchain
```

- [ ] **Step 3: Create `rust-toolchain.toml`**

Write to repo root as `rust-toolchain.toml`:

```toml
[toolchain]
channel = "1.95.0"
components = ["clippy", "rustfmt"]
```

Substitute the actual version from Step 1 if different.

- [ ] **Step 4: Verify local build uses pinned toolchain**

Run: `cargo clippy --tests -- -D warnings && cargo test`
Expected: clippy green, 472 tests pass. The `rust-toolchain.toml` file is automatically picked up by cargo.

- [ ] **Step 5: Add "Toolchain updates" section to CLAUDE.md**

Modify `CLAUDE.md`. Find the "## Releases" section heading. Insert the following new section *before* it:

```markdown
## Toolchain

The Rust toolchain is pinned in `rust-toolchain.toml` to keep CI deterministic — a new stable Rust release cannot break CI without a PR.

### Updating the pin

1. Read the [Rust release blog](https://blog.rust-lang.org/) for the new version.
2. Bump `channel` in `rust-toolchain.toml`.
3. Run `cargo clippy --tests -- -D warnings` locally; fix any new lints.
4. Open a PR. CI will run against the new pin.

```

Keep the trailing blank line so the following `## Releases` heading has whitespace before it.

- [ ] **Step 6: Commit**

```bash
git add rust-toolchain.toml CLAUDE.md
git commit -m "chore: pin Rust toolchain to 1.95.0

Pins the stable channel so new clippy lints cannot break CI without a
deliberate PR. See docs/superpowers/specs/2026-04-19-repo-hygiene-design.md.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 7: Push and open PR**

```bash
git push -u origin chore/pin-rust-toolchain
gh pr create --title "chore: pin Rust toolchain to 1.95.0" --body "$(cat <<'EOF'
## Summary

- Adds \`rust-toolchain.toml\` pinning stable to 1.95.0.
- Adds a "Toolchain" section to CLAUDE.md describing the bump workflow.
- Prevents a repeat of the April 11-19 outage where Rust 1.95's new clippy lints broke CI without any code change.

See \`docs/superpowers/specs/2026-04-19-repo-hygiene-design.md\`.

## Test plan

- [x] \`cargo clippy --tests -- -D warnings\` passes locally against pinned toolchain.
- [x] \`cargo test\` passes (472 tests).
- [ ] CI green.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- [ ] **Step 8: Wait for CI, confirm pinned version in logs**

Run: `gh pr checks <PR_NUMBER> --watch`
Expected: `Lint & Test pass`. Open the CI log and confirm the `Install toolchain` step reports `rust-toolchain.toml found` or similar — this proves the pin took effect.

- [ ] **Step 9: Squash merge**

```bash
gh pr merge <PR_NUMBER> --squash --delete-branch
```

Then sync local master:

```bash
git checkout master
git pull origin master
```

---

### Task 2: One-off `cargo fmt` pass + CI enforcement

**Files:**
- Modify: many (whatever `cargo fmt` touches — expect ~50 files)
- Modify: `.github/workflows/ci.yml` (add format check step)

This task has two distinct commits. Keep them separate so a reviewer can see the fmt churn in isolation from the CI config change.

- [ ] **Step 1: Create feature branch from updated master**

```bash
git checkout master
git pull origin master
git checkout -b chore/enforce-cargo-fmt
```

- [ ] **Step 2: Run cargo fmt across the codebase**

Run: `cargo fmt`
Expected: no output (success). Several files modified. Verify with `git status`.

- [ ] **Step 3: Verify tests still pass after fmt**

Run: `cargo clippy --tests -- -D warnings && cargo test`
Expected: clippy green, 472 tests pass. The fmt pass should never change behavior.

- [ ] **Step 4: Commit the fmt pass alone**

```bash
git add -A
git commit -m "style: one-off cargo fmt pass

Mechanical reformatting to match default rustfmt. No logic changes.
Enables \`cargo fmt --check\` enforcement in the next commit.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 5: Add fmt-check step to ci.yml**

Modify `.github/workflows/ci.yml`. Find the `- name: Clippy` step. Insert a new step *before* it:

```yaml
      - name: Format check
        run: cargo fmt --check
```

Final workflow should have the steps in this order: checkout → rust-toolchain → rust-cache → Format check → Clippy → Test.

- [ ] **Step 6: Verify format check locally**

Run: `cargo fmt --check`
Expected: no output, exit 0. Confirms the commit in Step 4 actually satisfied rustfmt.

- [ ] **Step 7: Commit the CI change**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: enforce cargo fmt --check

Adds a format check step to the Lint & Test job. Prevents future fmt
drift like the ~50 files that needed reformatting in the prior commit.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 8: Push and open PR**

```bash
git push -u origin chore/enforce-cargo-fmt
gh pr create --title "chore: enforce cargo fmt in CI" --body "$(cat <<'EOF'
## Summary

Two-commit PR:

1. One-off \`cargo fmt\` pass across the codebase (mechanical, no logic changes).
2. Adds \`cargo fmt --check\` step to CI.

After this lands, contributor PRs that run \`cargo fmt\` will no longer produce massive formatting diffs (see #319 for an example of what this prevents).

No \`rustfmt.toml\` — default rustfmt is the source of truth.

See \`docs/superpowers/specs/2026-04-19-repo-hygiene-design.md\`.

## Test plan

- [x] \`cargo fmt --check\` passes locally.
- [x] \`cargo clippy --tests -- -D warnings\` passes.
- [x] \`cargo test\` passes (472 tests).
- [ ] CI green (will run the new format check step).

## Review tip

Review commit 1 for logic changes (there should be none — it's pure fmt). Review commit 2 for the CI step placement.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- [ ] **Step 9: Wait for CI**

Run: `gh pr checks <PR_NUMBER> --watch`
Expected: `Lint & Test pass`. Confirm the `Format check` step appears in the log and is green.

- [ ] **Step 10: Squash merge**

```bash
gh pr merge <PR_NUMBER> --squash --delete-branch
git checkout master
git pull origin master
```

---

### Task 3: Require CI to pass before merge

This task is a GitHub repo settings change. It cannot be done via a PR. Document the change here and apply it in the UI or via `gh api`.

**Target state:** The existing `no-delete-forcepush` ruleset (ID 13341653) gains a `required_status_checks` rule requiring the `Lint & Test` CI job to pass.

- [ ] **Step 1: Confirm current ruleset state**

Run: `gh api repos/johnsideserf/siggy/rulesets/13341653`
Expected: JSON with `"rules":[{"type":"deletion"},{"type":"non_fast_forward"}]`. No `required_status_checks` rule yet.

- [ ] **Step 2: Apply the change**

**Option A (UI, recommended):** Visit https://github.com/johnsideserf/siggy/rules/13341653 → edit → under "Rules," add "Require status checks to pass" → add the `Lint & Test` check → "Require branches to be up to date before merging" → leave **unchecked** (this matches the current non-rebase squash-merge flow) → save.

**Option B (API):** Run:

```bash
gh api -X PUT repos/johnsideserf/siggy/rulesets/13341653 --input - <<'EOF'
{
  "name": "no-delete-forcepush",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "exclude": [],
      "include": ["~DEFAULT_BRANCH"]
    }
  },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    {
      "type": "required_status_checks",
      "parameters": {
        "required_status_checks": [
          { "context": "Lint & Test" }
        ],
        "strict_required_status_checks_policy": false
      }
    }
  ]
}
EOF
```

- [ ] **Step 3: Verify the ruleset now requires the check**

Run: `gh api repos/johnsideserf/siggy/rulesets/13341653`
Expected: the `rules` array now includes a `required_status_checks` entry with `Lint & Test` as a required context.

- [ ] **Step 4: Smoke test with a throwaway PR**

Create a deliberately-failing throwaway to confirm the gate works:

```bash
git checkout master
git pull origin master
git checkout -b test/required-checks-smoke
# Introduce a compile error so CI fails deterministically
printf '\ncompile_error!("smoke test");\n' >> src/main.rs
git add src/main.rs
git commit -m "test: smoke test required status check (DO NOT MERGE)"
git push -u origin test/required-checks-smoke
gh pr create --title "smoke test required status check" --body "Deliberate failure to verify branch protection. Close without merging." --draft
```

- [ ] **Step 5: Verify the merge is blocked**

Wait for CI to fail. Run: `gh pr view <PR_NUMBER> --json mergeable,mergeStateStatus`
Expected: `"mergeStateStatus":"BLOCKED"` (or similar non-mergeable state). The GitHub UI should show "Required status check 'Lint & Test' is expected to succeed."

- [ ] **Step 6: Clean up the smoke test**

```bash
gh pr close <PR_NUMBER> --delete-branch
```

Then delete any local branch:

```bash
git checkout master
git branch -D test/required-checks-smoke
```

- [ ] **Step 7: File follow-up issue for edition 2024 bump**

```bash
gh issue create --title "chore: upgrade Cargo edition 2021 → 2024" --label enhancement --body "$(cat <<'EOF'
Rust 2024 edition has been stable since Rust 1.85 (Feb 2025). The project is currently on edition 2021. Migration is usually mechanical via \`cargo fix --edition\`.

Not urgent — filed for tracking per the repo hygiene audit.

Migration guide: https://doc.rust-lang.org/edition-guide/rust-2024/index.html

## Steps

- [ ] Run \`cargo fix --edition\`
- [ ] Bump \`edition = \"2024\"\` in \`Cargo.toml\`
- [ ] \`cargo clippy --tests -- -D warnings && cargo test\`
- [ ] PR
EOF
)"
```

---

## Execution notes

- Tasks 1 → 2 → 3 in order. Task 2's fmt-check step works best against the pinned toolchain from Task 1. Task 3's required check depends on Task 1's CI having a stable name (`Lint & Test`).
- Never combine these into a single PR — keeping them separate means any one can be reverted cleanly.
- If Task 2's fmt pass touches a file that another in-flight PR (#312, #319) is modifying, warn the author that they'll need to rebase + re-run fmt.
