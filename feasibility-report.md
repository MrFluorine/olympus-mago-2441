## Feasibility Report

### Issue Summary
mago#2441 asks the analyzer to understand that `preg_match_all` with named subpatterns writes string keys into
the `$matches` array. Today the out-parameter is typed `list<array<string>>`, so `$m['name']` is reported as an
error. The maintainer's reading: a function hook is the easy part, parsing the regex to extract type information
is the hard part.

### Repository Quick Facts
- **Name**: carthage-software/mago
- **Stars**: 3559
- **License**: Apache-2.0 (dual MIT/Apache files in tree; GitHub reports apache-2.0) - allowed
- **Language**: Rust
- **Last Active**: 2026-11-14
- **Commit Hash**: 546772c58b13d4e8f4e188939d9cb794d39d1f9e (main HEAD at intake)

### Estimated Difficulty
Very Hard (with the expansions below; Medium-to-Hard as literally stated).
The core is a hand-written PCRE group scanner plus a key-presence analysis whose rules interact with four
`PREG_*` flags and differ between `preg_match` and `preg_match_all`. The `regex` crate already in the workspace
cannot parse PCRE lookbehind, backreferences or possessive quantifiers, so there is no shortcut. The result has
to flow through the existing plugin hook, assertion and out-parameter machinery without disturbing the current
return-type diagnostics.

### Difficulty Amplifiers Present
- **3 Protocol Extension** - pattern scanner, shape builder and the PHP stub's out-types must agree on numbering,
  naming and nullability.
- **6 Error Handling Completeness** - non-literal patterns or flags, unparseable patterns and unknown modifiers
  must fall back to the declared type instead of guessing.
- **7 Deep Type-Level Programming** - result is a keyed-array type with optional keys, nested lists and
  flag-dependent value types, narrowed by the call's result.
- **10 Mirror Feature with Non-Trivial Adaptation** - `preg_match` omits trailing unmatched groups,
  `preg_match_all` never does; `PREG_SET_ORDER` flips nesting.
- **13 Invariant Enforcement Across Boundaries** - existing `possibly-false-operand` and delimiter diagnostics
  and all current fixtures must keep passing.
- Not activated: 1, 2, 4, 5, 8, 9, 11, 12. Configuration Propagation (4) was considered (a setting to turn the
  inference off) and rejected as padding: no maintainer asked for it.

### Estimated Solution Scope

| File | Change description | Est. LOC | Evidence anchor |
| --- | --- | --- | --- |
| `crates/analyzer/src/plugin/libraries/stdlib/pcre/pattern.rs` (new) | Split delimiter/body/modifiers; scan groups (numbered, three named syntaxes, non-capturing, lookaround, atomic, comments, char classes, escapes, `\Q..\E`, `x` and `n` modifiers, branch reset, duplicate names); per-group "may be unmatched" analysis over quantifiers and alternation | 330 | `sprintf.rs` is 437 lines for a far simpler format grammar; the delimiter scanner alone in `pcre.rs:88-133` is 45 lines |
| `.../stdlib/pcre/shape.rs` (new) | Build the `$matches` type for both functions across `PREG_OFFSET_CAPTURE`, `PREG_UNMATCHED_AS_NULL`, `PREG_SET_ORDER`, `PREG_PATTERN_ORDER`, including the trailing-omission rule | 220 | `parse_url.rs` 218, `array_column.rs` 187, `psl/regex/capture_groups.rs` 123 (one shape, no flags) |
| `.../stdlib/pcre/mod.rs` (existing `pcre.rs` moved) | Keep delimiter validation; add after-call hook that resolves literal pattern and flags and retypes the `$matches` argument | 140 | hook files: `setcookie.rs` 108, `session_set_save_handler.rs` 119, `intdiv.rs` 100 |
| `.../stdlib/pcre/assertion.rs` (new) | When the call result is truthy, narrow `$matches` to the matched shape; otherwise leave the empty/declared shape | 110 | `array_all.rs` 92 (assertion provider) |
| `.../psl/regex/` (new provider file) | Same inference for `Psl\Regex\first_match` / `every_match` when no capture-group type is passed | 90 | `capture_groups.rs` 123 |
| `.../stdlib/mod.rs`, `.../psl/mod.rs` | Registration | 12 | existing registration lines |
| `crates/prelude/assets/extensions/pcre.php` | Widen declared out-types so string keys are legal in the fallback case | 10 | PR #1980 touched the same stubs |
| **Total** | | **~910** | |

- **Files modified**: ~8
- **Lines of code (net new/changed)**: ~910 with expansions; ~350 for the issue as literally stated
- **Analogous merged PRs measured** (source lines only, fixtures stripped): #2350 preg delimiter validation 206;
  #2678 484; #2562 390; #2565 105. Every analog is under 600, so the literal issue will be too. The estimate
  clears 800 only because of the expansions folded in below.
- **Estimated agent messages**: ~150

### Meets Minimum Thresholds?
- [ ] >= 600 lines of real, purposeful code - YES only with expansions (estimate ~910); NO as literally stated (~350)
- [x] >= 5 files modified - YES
- [x] >= 120 agent messages (estimated) - YES
- [x] >= 3 distinct difficulty amplifiers - YES (5)

### Red Flags
- **Existing PR**: none. Claim map and linkage are clean; PR search in any state finds only #1980 and #2350,
  neither of which infers capture groups. No Discussions matched.
- **Maintainer signal**: lukewarm ("not sure how worth it this is"). Not declined.
- **Scope**: too narrow as stated. Depends on expansions that the issue does not ask for (`preg_match`, flags,
  result narrowing, Psl mirror). They are natural, but a reviewer could read them as invented scope.
- **System-level span**: borderline. Work sits in one crate (analyzer) plus the stub. Two entry points
  (PHP stdlib and Psl) and three layers (parser, type construction, flow narrowing) are the argument for it.
- **Toolchain**: repo requires Rust 1.100; `olympus-base-rust:latest` ships 1.95. The Dockerfile must install a
  newer toolchain at build time, which the Dockerfile rules allow only with an inline justification.
- **External knowledge**: correct behaviour is defined by PHP/PCRE semantics that are not in the repo. Every rule
  the tests rely on must be stated in the description, which strains the 600-word budget.
- **Oracle**: no PHP interpreter on the authoring machine; expected shapes must be verified in a PHP container.
- **Churn**: analyzer crate had 408 commits in 60 days and a 2.0 rewrite is under way on `next`.
- **Upstream suite**: full `cargo test --workspace` run in the image is pending at time of writing.

### Scope Expansion Suggestions
Folded into the estimate above:
- Cover `preg_match` as well as `preg_match_all`.
- Honour all four flags and their combinations; fall back when flags are not literal.
- Narrow `$matches` by the call result.
- Mirror the inference for `Psl\Regex\first_match` / `every_match`.
Considered and left out: `preg_replace_callback` callback parameter typing (no confirmed extension point for
closure parameter types), a config switch (padding), new pattern-syntax diagnostics (would pin invented wording).

### Recommendation
PROCEED WITH MODIFICATIONS.
The hard core is real and unclaimed, and the expanded scope gives an evidence-backed ~910 lines across ~8 files.
It is not a clean PROCEED: the literal issue is under the bar, every analogous PR is under 600 source lines, the
span is borderline, and the base image's toolchain is too old. If the solution outline does not hold the margin
at the pre-coding checkpoint, stop and pick a different repo.
