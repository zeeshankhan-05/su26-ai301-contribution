# Contribution 1: Automatically resolve superclass redefinition errors while generating gem RBIs

**Contribution Number:** 1
**Student:** Zeeshan Khan
**Issue:** [https://github.com/Shopify/tapioca/issues/1834](https://github.com/Shopify/tapioca/issues/1834)
**Status:** Phase III Complete

---

## Why I Chose This Issue

I chose this issue because it is open, unassigned, labeled `good-first-issue`, and has a clear expected behavior. The issue is in `Shopify/tapioca`, which is a recognizable open-source project from Shopify, so it gives me a chance to work on a real developer-tooling project while also building a contribution that would be valuable to discuss on my resume.

This issue also seems realistic for a first open-source PR because the problem is specific and connected to an existing part of the codebase: RBI validation. I am interested in learning more about how Tapioca works with Sorbet, how generated RBI files are checked, and how open-source projects handle errors in a way that helps users without hiding important information.

---

## Understanding the Issue

### Problem Description

Tapioca generates RBI files for Ruby projects, then validates those generated files using Sorbet. Some generated RBI files can cause payload superclass redefinition errors (Sorbet error `5012`). Before Phase III, Tapioca did not automatically add the needed Sorbet config suppression for those specific constants, so users had to handle the error manually.

### Expected Behavior

When Tapioca detects a superclass redefinition error during RBI validation, it should update `sorbet/config` with the appropriate suppression line:

`--suppress-payload-superclass-redefinition-for=ConstantName`

However, Tapioca should still inform the user about the mismatch instead of silently hiding the problem.

### Current Behavior

Before Phase III, Tapioca validated generated RBI files and could adjust RBI strictness for some conflicts, but it did not automatically add superclass redefinition suppressions to `sorbet/config`. On branch `fix-issue-1834`, Tapioca now handles Sorbet error `5012` during gem RBI validation when `auto_strictness` is enabled.

### Affected Components

The main area likely involved is:

- `Tapioca::Helpers::RbiFilesHelper#validate_rbi_files`

Other possible areas may include:

- Sorbet config handling
- RBI validation logic
- Tests related to generated RBI validation or strictness changes

---

## Reproduction Process

### Environment Setup

Completed on June 7, 2026.

**Local environment:**

- **OS:** macOS arm64, darwin 25.5.0
- **Local repo path:** `~/Developer/Personal Projects/tapioca` (also documented earlier as `~/Projects/tapioca`)
- **Fork URL:** [https://github.com/zeeshankhan-05/tapioca](https://github.com/zeeshankhan-05/tapioca)
- **Upstream URL:** [https://github.com/Shopify/tapioca](https://github.com/Shopify/tapioca)
- **Ruby:** 4.0.2 through rbenv
- **Bundler:** 4.0.10
- **Homebrew:** 5.1.15

**Setup verification:**

- Dependencies installed successfully with `bundle install`.
- Smoke test passed with `bundle exec exe/tapioca help`.
- Targeted test passed with `bin/test spec/tapioca/cli/help_spec.rb`.
- Targeted test result: 3 tests, 12 assertions, 0 failures.
- Full test suite was not completed during initial Phase II setup.
- No setup errors occurred.

**Repository setup notes:**

- No `CONTRIBUTING.md` file was found in the repo.
- Setup references are `README.md` and `dev.yml`.
- `dev.yml` includes `bin/test`, `bin/typecheck`, and `bin/style`.

### Steps to Reproduce

This is a minimal Sorbet-level reproduction of the superclass redefinition behavior that issue #1834 asks Tapioca to handle.

1. From the Tapioca repo, create a temporary repro directory outside the repo:
  ```bash
   rm -rf /tmp/tapioca-1834-repro
   mkdir -p /tmp/tapioca-1834-repro/gem_rbis /tmp/tapioca-1834-repro/dsl_rbis
  ```
2. Create `/tmp/tapioca-1834-repro/gem_rbis/net_imap_literal.rbi`:
  ```rbi
   # typed: true

   class Net::IMAP::Literal < String
   end
  ```
3. Run Sorbet against the temporary RBI directories:
  ```bash
   bundle exec srb tc --no-config --error-url-base=https://srb.help/ /tmp/tapioca-1834-repro/dsl_rbis /tmp/tapioca-1834-repro/gem_rbis
  ```
4. Confirm that Sorbet reports a payload superclass redefinition error for `Net::IMAP::Literal`.
5. Confirm the suppression flag works:
  ```bash
   bundle exec srb tc --no-config --error-url-base=https://srb.help/ --suppress-payload-superclass-redefinition-for=Net::IMAP::Literal /tmp/tapioca-1834-repro/dsl_rbis /tmp/tapioca-1834-repro/gem_rbis
  ```

### Reproduction Evidence

- **Working branch:** [https://github.com/zeeshankhan-05/tapioca/tree/fix-issue-1834](https://github.com/zeeshankhan-05/tapioca/tree/fix-issue-1834)
- **Reproduction command:**
  ```bash
  bundle exec srb tc --no-config --error-url-base=https://srb.help/ /tmp/tapioca-1834-repro/dsl_rbis /tmp/tapioca-1834-repro/gem_rbis
  ```
- **Observed error excerpt:**
  ```text
  /tmp/tapioca-1834-repro/gem_rbis/net_imap_literal.rbi:3: Parent of class `Net::IMAP::Literal` redefined from `Net::IMAP::CommandData` to `String` https://srb.help/5012
       3 |class Net::IMAP::Literal < String
                                     ^^^^^^
      https://github.com/sorbet/sorbet/tree/bd88920e92609a8965bf8ccce34b5b61d20ff271/rbi/stdlib/net.rbi#L5040: Originally defined here
      5040 |class Net::IMAP::Literal < Net::IMAP::CommandData
            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    Note:
      Pass `--suppress-payload-superclass-redefinition-for=Net::IMAP::Literal` at the command line or in the `sorbet/config` file to silence this error.
  Errors: 1
  ```
- **Suppression verification:** Running the same command with `--suppress-payload-superclass-redefinition-for=Net::IMAP::Literal` returned `No errors! Great job.`
- **Tapioca validation-path note (Phase II):** The original `validate_rbi_files` command used `--stop-after namer`; with that flag alone, this temporary payload conflict did not surface and Sorbet returned `No errors! Great job.`
- **Phase III resolution:** The implementation keeps the namer pass for error `4010` and adds a second full Sorbet pass (when `auto_strictness` is enabled) to detect resolver-level error `5012`.
- **Tapioca CLI note (Phase II):** A controlled `tapioca gem net-imap` run using `/tmp` output completed successfully and changed the generated RBI strictness to `typed: false`; it did not reproduce the payload suppression behavior directly through the CLI before the fix.
- **My findings:** The underlying Sorbet behavior is confirmed. The error code is `5012`, the constant is `Net::IMAP::Literal`, and Sorbet explicitly recommends adding `--suppress-payload-superclass-redefinition-for=Net::IMAP::Literal` to suppress this class of payload superclass mismatch.

---

## Solution Approach

### Analysis

Sorbet reports payload superclass redefinition as error `5012`. In my minimal reproduction, the error output included the exact constant (`Net::IMAP::Literal`) and Sorbet’s suggested suppression flag (`--suppress-payload-superclass-redefinition-for=Net::IMAP::Literal`).

Tapioca validates generated RBI files in `Tapioca::Helpers::RBIFilesHelper#validate_rbi_files`. That method:

- runs Sorbet with `--no-config`, `--error-url-base=...`, and `--stop-after namer` against the DSL and gem RBI directories
- parses Sorbet output with `Spoom::Sorbet::Errors::Parser.parse_string`
- handles parse errors (`code < 4000`) by raising `Tapioca::Error`
- handles validation conflicts through `auto_strictness`, which filters error `4010` and calls `update_gem_rbis_strictnesses` to change conflicting gem RBI files to `typed: false`

Phase III added automatic handling for Sorbet error `5012` on branch `fix-issue-1834`. When `auto_strictness` is enabled, a second full Sorbet pass detects resolver-level payload superclass redefinitions, extracts the affected constant, appends `--suppress-payload-superclass-redefinition-for=ConstantName` to `sorbet/config` if not already present (exact-line match), and prints a user-facing message. The original namer pass is unchanged for error `4010`.

The upstream repo already contains manual suppressions in `sorbet/config` for `Net::IMAP::CommandData` and `Net::IMAP::Literal`, which suggests maintainers have handled this case manually before.

### Proposed Solution

Extend `validate_rbi_files` so that when Sorbet reports error `5012`, Tapioca extracts the constant name, appends the matching `--suppress-payload-superclass-redefinition-for=ConstantName` line to `sorbet/config` if it is not already present, and prints a clear user-facing warning about the superclass mismatch and the suppression that was added.

### Implementation Plan

Using UMPIRE framework:

**Understand:** When Tapioca validates generated gem RBIs, some generated definitions can redefine a Sorbet payload class with a different superclass. Sorbet reports this as error `5012` and recommends a suppression flag. Tapioca should add that suppression to `sorbet/config` automatically, but still tell the user what happened instead of silently hiding the mismatch.

**Match:** Closest existing patterns in the codebase:

- **Sorbet error parsing:** `validate_rbi_files` already parses Sorbet stderr with `Spoom::Sorbet::Errors::Parser` and branches on `error.code`
- **Automatic remediation for validation conflicts:** `auto_strictness` already handles error `4010` by calling `update_gem_rbis_strictnesses`
- **User-facing output style:** `update_gem_rbis_strictnesses` uses `say(..., [:yellow, :bold])`; other commands use `say_error(..., :yellow)` for warnings
- **Config file reading:** `Spoom::Sorbet::Config.parse_file` is already used in `lib/tapioca/static/requires_compiler.rb`
- **Config file creation:** `lib/tapioca/commands/configure.rb` creates `sorbet/config`, but there is no existing helper for appending suppression flags during validation
- **Existing tests:** strictness behavior is covered in `spec/tapioca/cli/gem_spec.rb` and `spec/tapioca/cli/dsl_spec.rb` through CLI integration tests

**Plan:**

1. Inspect how `Spoom::Sorbet::Errors::Error` represents error `5012`, including whether the constant name can be parsed from `message` or `more` lines containing `--suppress-payload-superclass-redefinition-for=...`.
2. Add handling in `validate_rbi_files` for Sorbet error `5012` alongside the existing `4010` auto-strictness path.
3. Extract the constant name from Sorbet’s suggested suppression flag or error output.
4. Add helper logic to update `sorbet/config` with:
  `--suppress-payload-superclass-redefinition-for=ConstantName`
5. Avoid duplicate suppression entries if `sorbet/config` already contains the line.
6. Print a clear user-facing warning explaining:
  - which constant had a payload superclass mismatch
  - what suppression line was added
  - that the user should review the generated RBI and config change
7. Decide whether the current validation command needs adjustment so error `5012` is actually detected during Tapioca validation, since `--stop-after namer` did not surface the reproduced payload conflict.
8. Add or update tests covering:
  - detection of error `5012`
  - writing the suppression line to `sorbet/config`
  - not duplicating an existing suppression line
  - user still gets informed through stdout/warning output
  - existing `4010` auto-strictness behavior still works

**Implement:** [https://github.com/zeeshankhan-05/tapioca/tree/fix-issue-1834](https://github.com/zeeshankhan-05/tapioca/tree/fix-issue-1834)

**Review:** Before opening a PR, check:

- `bin/style`
- `bin/typecheck`
- relevant tests with `bin/test`
- only intended files changed
- no unrelated `Gemfile.lock` setup change is included

**Evaluate:** Phase III verification completed:

- reproduced the Sorbet-level failure from Phase II before making changes
- confirmed the integration test failed for the expected reason before implementing the fix
- ran the scenario after the fix through Tapioca’s validation path via `spec/tapioca/cli/gem_spec.rb`
- confirmed `sorbet/config` receives the expected suppression line
- confirmed the user-facing warning/message appears
- ran targeted tests and project checks (`bin/test spec/tapioca/cli/gem_spec.rb`, `bin/test spec/tapioca/cli/dsl_spec.rb`, `bin/typecheck`, `bin/style`)

---

## Testing Strategy

### Unit Tests

- [ ] Add helper-level coverage for parsing error `5012` and extracting the constant name from Sorbet output.
- [ ] Add helper-level coverage for appending `--suppress-payload-superclass-redefinition-for=ConstantName` to `sorbet/config`.
- [ ] Add helper-level coverage to ensure duplicate suppression lines are not added when the config already contains the constant.

Skipped in favor of a single integration test in `spec/tapioca/cli/gem_spec.rb`, following maintainer guidance.

### Integration Tests

- [x] Add CLI/integration coverage in `spec/tapioca/cli/gem_spec.rb` and/or `spec/tapioca/cli/dsl_spec.rb` for the new suppression behavior.
- [x] Verify the user still receives a warning/message when Tapioca adds a suppression entry.
- [x] Verify existing `4010` auto-strictness behavior still works and is not broken by the new logic.

### Manual Testing

- [x] Re-run the minimal Sorbet-level reproduction from `/tmp/tapioca-1834-repro` to confirm the underlying error still reproduces before the fix.
- [x] After implementation, run the Tapioca validation path or targeted tests and confirm `sorbet/config` is updated as expected.
- [x] Confirm the warning output explains the superclass mismatch and the added suppression line.

### Project Checks

- [x] `bin/test spec/tapioca/cli/gem_spec.rb` — 67 tests, 494 assertions, 0 failures, 0 errors, 1 skip (re-run during Phase III audit)
- [x] `bin/test spec/tapioca/cli/dsl_spec.rb` — 68 tests, 328 assertions, 0 failures, 0 errors, 0 skips (re-run during Phase III audit)
- [x] `bin/typecheck`
- [x] `bin/style`
- [x] Verified the suppression is added to `sorbet/config`
- [x] Verified the user-facing message appears
- [x] Verified a second run does not duplicate the suppression
- [x] Verified a similar suppression for another constant does not block the correct entry

### Full Suite

Not treated as fully passing. One earlier full-suite run completed with **794 tests, 0 failures, 2 errors, and 2 skips**. The two errors were unrelated `RubyLsp::Tapioca::Addon` failures caused by `URI::InvalidURIError` when handling a local file path containing spaces (`Personal Projects`). The affected code was unrelated to issue #1834.

---

## Implementation Notes

### Week 1 Progress

Selected Shopify/tapioca issue #1834 for my CodePath AI301 Open Source Capstone. I commented on the GitHub issue expressing interest, claimed the issue on the CodePath Google Sheet, and started documenting my understanding in this Contribution README.

### Week 2 Progress

Completed Phase II by setting up the local environment, creating the working branch, reproducing the superclass redefinition behavior at the Sorbet level, and writing a UMPIRE-based solution plan.

### Week 3 Progress

Completed Phase III implementation on branch `fix-issue-1834`:

- Followed Shopify engineer guidance to add an integration test in the `describe "strictness"` block of `spec/tapioca/cli/gem_spec.rb`
- Added the test before the production fix and confirmed it failed for the expected missing behavior
- Implemented a dual-pass validation approach: kept the existing `--stop-after namer` pass for error `4010`, and added a second full Sorbet pass when `auto_strictness` is enabled to detect resolver-level error `5012`
- Added automatic handling for Sorbet error `5012`, including constant extraction and `--suppress-payload-superclass-redefinition-for=ConstantName` updates to `sorbet/config`
- Used exact-line idempotency (`config.lines(chomp: true).include?(flag)`) and preserved existing config content when appending
- Added user-facing output for both newly added and already-present suppressions
- Ran `bin/test spec/tapioca/cli/gem_spec.rb`, `bin/test spec/tapioca/cli/dsl_spec.rb`, `bin/typecheck`, and `bin/style`

### Code Changes

- **Files modified:** `lib/tapioca/helpers/rbi_files_helper.rb`, `spec/tapioca/cli/gem_spec.rb`
- **Key commit:** [https://github.com/zeeshankhan-05/tapioca/commit/1b40e73f5da5f8db817482eb9dd72f470908cec3](https://github.com/zeeshankhan-05/tapioca/commit/1b40e73f5da5f8db817482eb9dd72f470908cec3)
- **Working branch:** [https://github.com/zeeshankhan-05/tapioca/tree/fix-issue-1834](https://github.com/zeeshankhan-05/tapioca/tree/fix-issue-1834)
- **Approach decisions:** Preserved the existing namer validation pass for error `4010`. Added a second full Sorbet pass (only when `auto_strictness` is enabled) to detect resolver-level error `5012`. Used exact-line matching to prevent duplicate or false-positive config matches. Appended suppressions without stripping existing config content.

---

## Pull Request

**PR Link:** Not submitted yet.

**PR Description:** Not started yet.

**Maintainer Feedback:**

Guidance received on the GitHub issue from Shopify contributor [@KaanOzkan](https://github.com/KaanOzkan) (June 8, 2026). This was implementation guidance, not a review of the final code:

- Use the `describe "strictness"` block in `spec/tapioca/cli/gem_spec.rb` as the main test model
- Exercise `tapioca gem` and verify RBI validation behavior
- Assert that `sorbet/config` receives `--suppress-payload-superclass-redefinition-for=ConstantName`
- Assert that the user is still informed about the mismatch
- Make the config update idempotent

No maintainer review or approval of the final implementation has been received yet.

**Status:** Not started yet.

---

## Learnings & Reflections

### Technical Skills Gained

Not started yet.

### Challenges Overcome

Not started yet.

### What I'd Do Differently Next Time

Not started yet.

---

## Resources Used

- Shopify/tapioca issue #1834: [https://github.com/Shopify/tapioca/issues/1834](https://github.com/Shopify/tapioca/issues/1834)
- Shopify/tapioca repository: [https://github.com/Shopify/tapioca](https://github.com/Shopify/tapioca)
- CodePath AI301 Phase I instructions
- First Contributions tutorial

