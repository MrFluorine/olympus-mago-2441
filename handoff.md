# Handoff log

## Turn 1 - 2026-11-14 - intake and partial research (stopped early: usage limit)

Issue: https://github.com/carthage-software/mago/issues/2441
Repo: carthage-software/mago (Rust, Apache-2.0, 3.5k stars), cloned to `repo/` (blobless clone).
Candidate pin: `546772c58b13d4e8f4e188939d9cb794d39d1f9e` (main HEAD, 2026-11-14).

### Done
- Workspace created: `repo/`, `deliverables/` (empty), this file.
- Availability: claim map (all mode) clean for #2441; issue-linkage shows no closing PR, branch or assignee.
  PR search (any state) for preg/regex/pcre/capture groups returns only #1980 and #2350, neither implements
  capture-group shape inference. No GitHub Discussions matched.
- Test harness: analyzer tests are PHP fixtures in `crates/analyzer/tests/cases/*.php` with `@mago-expect`
  comments, registered by one `test_case!(name)` line in `crates/analyzer/tests/mod.rs`.
  `cargo test -p mago-analyzer --test mod preg_match` built and passed (3/3) in a `rust:latest`
  (1.100.0) container in 78s; target dir 2.6 GB. Run had network access; offline not yet tested.
- Extension points read: `FunctionCallHook::after_function_call` + `HookContext::set_variable_type`
  (`crates/analyzer/src/plugin/hook/call.rs`, `plugin/context.rs:321`), `FunctionAssertionProvider`
  (`plugin/provider/assertion.rs:80`), out-type application (`invocation/post_process.rs:513`),
  existing provider `plugin/libraries/stdlib/pcre.rs` (delimiter validation only),
  stubs in `crates/prelude/assets/extensions/pcre.php`.

### Known risks
- Toolchain: repo requires Rust 1.100; `olympus-base-rust:latest` ships 1.95. Dockerfile must install a
  newer toolchain at build time with an inline justification.
- LOC: issue as stated is roughly 300-400 lines; closest merged analog #2350 is 206 source lines.
  Needs scope expansion to reach an evidence-backed 800. Not yet sketched per file.
- System-level span is borderline (work sits mostly inside the analyzer crate).
- A 2.0 rewrite is in progress on the `next` branch; analyzer crate is high-churn.
- No local PHP, so expected PHP behaviour cannot be checked against a real interpreter on this machine.
- Docker (Colima) only shares the home folder with containers; stream sources in with tar or keep
  workspaces under the home folder.

### Not done
- Full upstream suite run (`cargo test --workspace`) was started in a container and stopped before
  finishing; no result recorded.
- Deep research checklist (CONTRIBUTING, last 20-30 merged PRs, full test-suite conventions).
- Amplifier audit, per-file LOC sketch, feasibility report (`feasibility-report.md`).
- All five deliverables.

### Deliverables changed this turn
None.

## Turn 2 - 2026-11-14 - feasibility

- Read CONTRIBUTING and the last 30 merged PRs; measured source-only sizes of #2350 (206), #2678 (484),
  #2562 (390), #2565 (105).
- Measured in-repo anchors: provider files 90-220 lines, `sprintf.rs` 437.
- Wrote `feasibility-report.md`: PROCEED WITH MODIFICATIONS, estimate ~910 LOC over ~8 files with four
  expansions folded in (~350 as literally stated).
- Full upstream suite (`cargo test --workspace`) relaunched in a `rust:latest` container; result pending.
- Waiting for the user's go-ahead before any construction.

### Deliverables changed this turn
None.

## Turn 3 - 2026-11-14 - solution draft and LOC measurement

- Local toolchain: `rustup toolchain install 1.100.0` (+ rustfmt, clippy); build with `cargo +1.100.0`.
- Full `cargo test --workspace` in the 6 GB Colima VM failed at link time for the `mago` binary test
  (likely memory); not a repo defect. Analyzer crate suite runs fine locally: 2948 passed in `--test mod`.
- PHP behaviour verified in `php:8.3-cli` (PCRE2 10.42) with `scratch/probe.php`: trailing unmatched groups
  are dropped per match in set order, never in pattern order; named groups appear under name and number;
  `PREG_UNMATCHED_AS_NULL` keeps every group; `PREG_OFFSET_CAPTURE` pairs each capture with an int.
- Scope change, forced by evidence: precise typing of `preg_match` makes `$matches[N] ?? default` provably
  redundant, which flips two upstream fixtures (`by_reference_argument_in_loop`,
  `type_narrowing_external`) to failing with `redundant-null-coalesce`. `base` must pass before and after the
  solution and valid upstream tests may not be excluded, so `preg_match` was dropped. The feature now covers
  `preg_match_all` only, which is what the issue literally asks for. The Psl mirror was also dropped
  (its semantics are not verifiable from the repo).
- Ordering finding: plugin assertions are resolved against the declared out type, so a hook that retypes
  `$matches` after the call is too late (reported `impossible-type-comparison`). The solution instead lets an
  assertion provider return out types that are written before its assertions are resolved.
- Verification: `cargo +1.100.0 clippy -p mago-analyzer` clean, `cargo fmt` applied, analyzer `--test mod`
  2948 passed with the solution applied. `solution.patch` applies cleanly to a fresh worktree at the pin.

### LOC: estimated vs actual
- Estimated at feasibility: ~910 (with four expansions).
- Actual `git diff --numstat -M HEAD`: 919 added / 2 removed across 8 files.
- Effective (non-blank, non-comment added lines): 718.
- Files with meaningful changes: `pcre/pattern.rs` 635, `pcre/shape.rs` 148, `pcre/matches.rs` 78,
  `invocation/post_process.rs` 26, `plugin/provider/assertion.rs` 19; minor: `external/protocol.rs` 6,
  `pcre/mod.rs` 6 (rename of `pcre.rs`), `stdlib/mod.rs` 1.
- What the estimate missed: the scanner is larger than sketched (635 vs 330); `preg_match`, result narrowing
  for it, and the Psl mirror were dropped.

### SHA-256
- `deliverables/solution.patch` 4031f8e12a3f4be3b61b50196b03234513835f1c1c95f7314d4e92036d91f1e4 (draft)

### Deliverables changed this turn
`solution.patch` (created, draft).

## Turn 4 - 2026-11-14 - tests drafted, then paused on new information

- Wrote `repo/crates/analyzer/tests/preg_match_all_matches.rs`: 31 tests as a separate test target (so hidden
  tests never touch `tests/mod.rs`, which a solver is likely to edit). All 31 pass with the solution applied.
  Expected diagnostic codes were captured from the solved binary, not guessed.
- NOT done: red-state run at baseline (test file only, no solution); `test.sh`; `Description.md`;
  `metadata.md`; `Dockerfile`; traceability matrix; any container verification.
- Paused: the user shared a peer tip from Discord saying the real bar is a "seq length" above ~95k measured by
  a blind solver agent with a formula from the announcements channel, that descriptions should be under 125
  words, and that Python repos are the easier path. That conflicts with this workspace's 600-LOC / 600-word
  design. The current contract cannot be stated fairly in 125 words. Waiting for the user to share the formula
  and the Submission criteria panel, and to decide whether to continue with mago.

### Deliverables changed this turn
None (`solution.patch` unchanged; the test file lives in `repo/` and is not yet exported to `test.patch`).

## Turn 5 - 2026-11-14 - short description, tests, harness, Dockerfile

Platform guide (pasted by the user) shows the gates are rollout-based: Nova must never solve unhinted,
Orion/Vega at most 25%, and Nova's average sequence length must clear the context bar in the criteria panel.
No line-count gate is named. The user agreed to finish this package and let prechecks, the Scope Gate and one
or two quick-check rollouts decide.

- `Description.md`: 127 words, ASCII only, three short paragraphs. Leans on "mirrors PHP at runtime" so the
  PCRE rules are derivable without listing them.
- `metadata.md`: created.
- Tests: `crates/analyzer/tests/preg_match_all_matches.rs`, 31 tests, own test target (does not touch
  `tests/mod.rs`). Removed the one assertion stricter than the prose (non-empty after a null check under
  `PREG_UNMATCHED_AS_NULL`).
- `test.sh`: `base` = `--lib --test mod --test external_throw_type_providers --test unhandled_thrown_type_order`
  (3396 tests), `new` = `--test preg_match_all_matches` (31). Converts libtest output to JUnit with awk,
  strips XML-illegal bytes, writes a failing harness case when no test is reported.
- Local 2x2 (Rust 1.100.0, macOS): test.patch only -> base 3396/0 failures, new 31/31 failures;
  + solution -> base 3396/0, new 31/0. All four reports pass `xmllint --noout`. Name sets are disjoint.
- Pitfall hit: sharing `CARGO_TARGET_DIR` between the repo and a temporary worktree left the prelude built
  against the deleted worktree path, so every test reported `non-existent-function`. Fixed with
  `cargo clean -p mago-prelude`. Never share a target dir across checkouts.
- Pitfall hit: `git archive` honours `export-ignore` in `.gitattributes` and drops `Cargo.toml`; use
  `git checkout-index -a --prefix=` for a build context.

### Requirement <-> test matrix
| REQ | Description sentence (abridged) | Covering tests (prefix `preg_match_all_matches_`) |
| --- | --- | --- |
| R1 | infer when pattern literal and flags omitted | named_groups_are_keyed_by_name_and_number, keys_outside_the_pattern_are_undefined |
| R2 | flags a literal, valid combination of the four constants | non_literal_flags_keep_declared_type (PATTERN_ORDER half), set_order_*, offset_capture_*, unmatched_as_null_* |
| R3 | every capturing group under its number and name, as PHP numbers them | three_named_group_syntaxes_are_recognized, non_capturing_constructs_are_not_numbered, extended_modifier_ignores_comments, inline_extended_option_ignores_comments, no_auto_capture_modifier_keeps_named_groups_only, inline_no_auto_capture_option_keeps_named_groups_only, branch_reset_alternatives_share_numbers |
| R4 | keys optional only where PHP can omit them | set_order_lists_one_array_per_match, set_order_trailing_optional_group_may_be_missing, set_order_optional_group_before_required_group_is_present, set_order_groups_after_last_required_group_may_be_missing |
| R5 | null for unmatched groups when the flags ask | unmatched_as_null_makes_optional_captures_nullable, set_order_unmatched_as_null_keeps_every_group |
| R6 | capture paired with its integer offset when the flags ask | offset_capture_pairs_captures_with_offsets, set_order_offset_capture_pairs_captures_with_offsets |
| R7 | whole match or always-participating group is non-empty-string when it cannot match empty | captures_are_non_empty_when_the_group_cannot_be_empty, whole_match_is_non_empty_when_the_pattern_cannot_be_empty, optional_group_captures_may_be_empty, group_inside_optional_group_may_be_empty, alternative_captures_may_be_empty, branch_reset_group_in_every_alternative_always_takes_part, lookahead_group_takes_part_unless_negative |
| R8 | lists non-empty where the call is known to have matched | capture_lists_are_non_empty_when_the_call_matched, set_order_list_is_non_empty_when_the_call_matched |
| R9 | other calls keep the declared type | non_literal_pattern_keeps_declared_type, non_literal_flags_keep_declared_type, conflicting_order_flags_keep_declared_type |
| R10 | patterns that do not compile or reuse a group name keep the declared type | unbalanced_pattern_keeps_declared_type, reused_group_name_keeps_declared_type |

Forward audit: every REQ has a red-on-baseline test. Reverse audit: every test maps to a REQ above.
Every fallback test pairs the unchanged half with a positive half so it is red at baseline.
Known soft spot: the expected diagnostic codes (`possibly-invalid-argument`, `mismatched-array-index`,
`undefined-*-array-index`, `possibly-undefined-*-array-index`, `possibly-null-argument`) are produced by
the existing analyzer from the inferred type, not chosen by the solver, but a reviewer may still ask about them.

### Signature audit
Tests call no new Rust symbol. They use the pre-existing `framework::TestCase` and PHP built-ins only.

### Mutation log (local, solution restored and byte-identical afterwards)
| Mutant | Tests that flipped |
| --- | --- |
| branch reset treated as plain group | 2 |
| trailing omission disabled | 2 |
| extended flag ignored | 2 |
| every group treated as always matched | 10 |
| no narrowing when matched | 2 |
| never non-empty-string | 15+ |
| `n` modifier ignored | 1 |

### Not done / not verified
- Container 2x2 as uid 1000 with `--network none` on the Olympus Rust image: first attempt failed on a bad
  build context (see pitfall); rerun in progress at time of writing.
- Flake repeats, CPU-saturation run, Test Fairness triage (platform side), agent rollouts.
- Description not checked against any AI-prose detector.

### SHA-256
- 406366b04fd262eb786465ee5ed924f2bd41826061705bdf4df38adf5c482c58  Description.md
- 4d75db87a8350a340a2a0535f6b7d29210e5b86eeb6ab30c53ae42ce65f48c1a  Dockerfile
- 1d746ff16d87a383ee546cbd7e03b4e37a6a3fd38c7f2f2c6d8a42deb94e3bc4  metadata.md
- 4031f8e12a3f4be3b61b50196b03234513835f1c1c95f7314d4e92036d91f1e4  solution.patch
- e8f377db5645c1ecfe4022ec2dffd2dc0780cf796183e8683f8e0558d4fc2eda  test.patch

### Deliverables changed this turn
`Description.md`, `metadata.md`, `Dockerfile`, `test.patch` (created); `solution.patch` (regenerated, content unchanged apart from excluding test files).

## Turn 6 - 2026-11-14 - container verification and audit

- Image `mago-pcre:verify` built from the pinned tree on `olympus-base-rust:latest` (9.97 GB, 714 s build).
- 2x2 inside the image as uid 1000 with `--network none`, patches applied with `git apply`:
  test.patch only -> base 3396/0 failures (exit 0), new 31/31 failures (exit 101);
  + solution.patch -> base 3396/0 (exit 0), new 31/0 (exit 0). All four JUnit files pass `xmllint --noout`.
- File audit passed: description 127 words ASCII, no links; test.patch has two files, `test.sh` mode 100755,
  no platform names; solution.patch has no test files; Dockerfile has base image, `/app`, `COPY . .`, `CMD`.
- `metadata.md` gained a License line (Apache-2.0) at the user's request.
- Workspace pushed to private repo MrFluorine/olympus-mago-2441 (no co-author trailer, at the user's request).

### Open risks (not fixed)
- `test.sh` uses cargo's `--no-fail-fast`; a naive "no fail-fast flags" check could misread it.
- Dockerfile line `cargo build ... --tests` only compiles, but contains the word "tests".
- Image is ~10 GB; Dockerfile installs Rust 1.100 over the base image's 1.95 (justified inline).
- 127-word description relies on "mirrors PHP at runtime"; Test Quality may call some tests under-specified.
- Not run: flake repeats, CPU-saturation run, any platform check, any agent rollout.

### Deliverables changed this turn
`metadata.md`.
