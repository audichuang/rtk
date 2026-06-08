# Upstream Sync Ledger

This is a **fork** of [rtk-ai/rtk](https://github.com/rtk-ai/rtk). We diverge from
upstream (most notably we carry the richer PR #1089-based `mvn`/`mvnd` module, which
upstream **rejected in favour of its own PR #1956**). Because we are not identical to
upstream, every upstream commit must be **explicitly triaged** — absorbed, cherry-picked,
or skipped-with-reason — and recorded here so future syncs never re-evaluate the same
commit twice or silently miss one.

> Per-commit disposition is mandatory for this fork. When you sync from upstream, append
> the new commits to the ledger below with a disposition and a one-line reason.

## Sync state

| Field | Value |
|-------|-------|
| Upstream remote | `upstream` → `https://github.com/rtk-ai/rtk.git` |
| Origin (our fork) | `origin` → `https://github.com/audichuang/rtk.git` |
| Default branch | `develop` (no `main`; a remote `master` exists) |
| Last sync triage | **2026-06-08** |
| Sync base (merge-base) | `0a630fe` — `Merge pull request #2289 … strip-output-decorators` (2026-06-05) |
| Upstream tip triaged | `047f454` — `Merge pull request #1956 … feat/mvn-rust-module` (2026-06-08) |

## How to re-sync (next time)

```bash
git fetch upstream --tags
# What's new since the last triaged tip:
git log --format='%h %ci %s' 047f454..upstream/develop -- .          # all
git log --oneline 047f454..upstream/develop -- src/cmds/jvm/         # mvn (compare, don't blind-merge)
# Cherry-pick clean, unrelated fixes (verify each touches only its own files):
git cherry-pick -x <hash>
# NEVER blind-merge upstream/develop: it carries upstream's competing mvn module
# (PR #1956) which conflicts with our richer #1089 module. Triage file-by-file.
```

After any change: `cargo fmt --all && cargo clippy --all-targets && cargo test --all` must be green.

## Decision: the `mvn`/`mvnd` module

Upstream merged **PR #1956 (vdufloth/feat/mvn-rust-module)** on 2026-06-08 — a *different*,
more minimal mvn module than the PR #1089 we absorbed. **We keep ours.** A deep 11-dimension
comparison (this fork's module vs upstream's single 2112-line `mvn_cmd.rs`) found **ours is a
functional superset**: it adds `mvnd` daemon, `checkstyle`, `dependency:tree`, `clean`, and
Surefire/Failsafe **XML report enrichment** — none of which upstream's has. We are
equal-or-better on 8 of 11 dimensions; only **3 upstream wins were absorbed** (commit `bcab98f`).

### Absorbed from upstream mvn module → commit `bcab98f`

| # | What | Upstream origin | Our target |
|---|------|-----------------|------------|
| 1 | Single-goal `install`/`package`/`deploy`/`integration-test` filtering (was 0% Passthrough). Routed via `GoalRouting::TestLike`, **literal goal preserved** so `.m2` install / deploy side-effects still run (NOT rerouted to `verify`). | `6c4950e` | `route_goal`, `dispatch`, `run_tests_like` |
| 2 | `has_english_footer` guard — non-English locale (French `BUILD ÉCHEC`) / no-POM builds pass through instead of false "nothing to clean". Single-goal wrapper layer only. | `6c4950e` | `filter_mvn_clean`, `run_tests_like` closure |
| 3 | Surefire 2.x single-dash ` - in ` close-line counting (gate `-- in` → `- in `, covers 2.x+3.x). | `cc152cd` | `filter_mvn_tests_with_goal` |

### Examined but deliberately NOT absorbed (would regress our richer design)

- `97dbf98` `-q` quiet-mode console parser — we strip `-q` pre-invoke and run the full pipeline + XML enrichment.
- `4609102` keep verbatim Reactor Summary rows — we intentionally collapse (≥90% savings).
- `5459c6d` shared `SurefireBlock` state machine — we use a single summarizing accumulator parser; no duplicate machine to extract.
- `df76528` post-failure boilerplate stripping — our `is_maven_boilerplate` is already a superset.
- `1050cfe` re-arm failure trail — our accumulator (`FAILURE_HEADER_RE` per subline) is structurally immune.
- `97bd2a7` compile-error continuation — our `filter_mvn_compile` catch-all already preserves it.
- `be6c812`/`3c0ee94` Failures-summary cap — ours re-renders a capped section (cap=10), no second uncapped site.
- `92a9218`/`77e28d0` idiom polish / duration normalise — ours already borrows / uses `strip_prefix`; never had a normalise to drop. Runtime CRLF: our per-line `.trim()` is more robust than upstream's `$`-anchored regexes.

## Per-commit ledger — `0a630fe..047f454` (triaged 2026-06-08)

Disposition: **✅ absorbed** · **⏭️ skip** (reason) · **➖ n/a** (merge/chore) · **🔁 partial**

| Upstream | Summary | Disposition | Our commit / reason |
|----------|---------|-------------|---------------------|
| `35273c2` (PR #2181) / `a2a63e1` | fix(curl): passthrough binary downloads | ✅ absorbed | cherry-pick `fd185e3` |
| `9574007` (PR #2135) / `ad2bfd3` | fix(aws): preserve JSON values for unsupported subcommands | ✅ absorbed | cherry-pick `1058899` |
| `63a76de` (PR #1645) / `6b30fdd` | fix(filters): remove helm `max_lines` cap | ✅ absorbed | cherry-pick `5180298` |
| `6c4950e` | feat(mvn)!: Rust module replacing TOML filter | 🔁 partial | kept ours; behaviors → absorb #1, #2 (`bcab98f`). `discover/registry.rs`, `discover/rules.rs`, `core/toml_filter.rs` in this commit ⏭️ skipped (mvn-coupled, tie to upstream's module) |
| `cc152cd` | fix(mvn): Surefire 3.x close lines + failure trail | 🔁 partial | 2.x counting → absorb #3 (`bcab98f`); 3.x already equivalent |
| `047f454` | Merge PR #1956 (mvn-rust-module) | ➖ n/a | merge commit; see mvn decision above |
| `f026cfd` | Merge develop into feat/mvn-rust-module | ➖ n/a | merge commit |
| `f58333c` | chore(mvn): drop Cargo.lock churn | ➖ n/a | chore |
| `f8bc856` | style(mvn): canonical `… +N more` overflow tail | ⏭️ skip | cosmetic; ours equivalent |
| `3c0ee94` | refactor(mvn): bind cap to CAP_WARNINGS | ⏭️ skip | ours cap=10 equivalent |
| `df76528` | fix(mvn): strip post-failure help boilerplate | ⏭️ skip | our `is_maven_boilerplate` is a superset |
| `be28a51` | fix(ci): pin fixture line endings, CRLF tests | ⏭️ skip | ours `.trim()` more robust; root `.gitattributes` hygiene → backlog |
| `1050cfe` | fix(mvn): re-arm failure trail on per-test sublines | ⏭️ skip | our accumulator immune |
| `92a9218` | refactor(mvn): strip_prefix, borrow buffers, CRLF | ⏭️ skip | ours already idiomatic |
| `be6c812` | feat(mvn): cap failing-class + Failures entries | ⏭️ skip | ours re-renders capped, cap=10 |
| `4609102` | fix(mvn): keep multi-module Reactor Summary rows | ⏭️ skip | ours intentionally collapses |
| `5459c6d` | refactor(mvn): extract shared SurefireBlock | ⏭️ skip | ours single parser; nothing to extract |
| `97bd2a7` | fix(mvn): preserve compile-error continuation | ⏭️ skip | ours equivalent |
| `97dbf98` | feat(mvn): filter `mvn -q` quiet-mode output | ⏭️ skip | ours strips `-q`, runs full pipeline + XML |
| `77e28d0` | refactor(mvn): drop duration normalisation | ➖ n/a | we never had it |

## Backlog (optional, low-priority)

- Port upstream's CRLF regression tests + add root `.gitattributes` (`tests/fixtures/** -text`)
  for Windows-CI hygiene — **test-only, no logic change** (`be28a51`).
- If we ever want headline savings numbers for the new `install`/`package` path, add a full
  (non-sliced) install fixture comparable to upstream's `mvn_install_full_raw.txt.gz`.
