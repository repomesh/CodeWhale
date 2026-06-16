# CodeWhale — "What Does Not Spark Joy" Audit

A read-only inventory of code smells: over/under-abstraction, hardcoding that
shouldn't be, duplication, god-objects, dead weight, and repo-hygiene gaps.
Organized by severity. Recorded here as the working reference for the cleanup
pass. **All load-bearing claims were re-verified on `hunter/0.8.62-glm-subagents`
at commit `e90bb93f`** (dead crates have zero dependents; tracked build output
confirmed; cache dirs confirmed un-ignored).

Use this as a backlog, not a mandate. Each item is independently shippable;
prefer small, reviewable, well-tested commits over a big-bang refactor. Some
items are genuinely fine as-is and are marked so.

---

## TL;DR — the 12 worst things

1. **Provider/model identity is the same data, hand-maintained in 8+ files.**
   Adding one provider touches `config/lib.rs` + `config/provider.rs` +
   `agent/lib.rs` + `cli/lib.rs` + `tui/config.rs` + `tui/models.rs` +
   `tui/model_routing.rs` + 3 tui "registry" modules + the web `labelMap`
   (twice) + `config.example.toml`. This is the single biggest structural problem.
2. **`crates/tui` is a 315k-line super-crate** with a 208-field `App` god-object
   (`tui/app.rs:1358`), a 3,333-line event loop (`tui/ui.rs:1333`), a
   12,182-line `config.rs`, and **372 `#![allow(dead_code)]` markers** silencing
   the compiler.
3. **Two orphan crates nobody imports: `tui-core` (230 impl lines) and
   `whaleflow` (5,716 lines).** Verified zero consumers. Either dead or
   parked-future code shipping as production.
4. **Build output tracked in git:** `extensions/vscode/out/*` (6 files) and
   `web/lib/facts.generated.ts`.
5. **`.venv-bench/`, `.uv-cache/`, `.uv-tools/` are not gitignored**
   (~900MB exposed to accidental commit).
6. **Rust toolchain drift with no `rust-toolchain.toml`:** CNB/Docker pin `1.88`
   (4 sites), GHA/Nix float `@stable` (9 sites).
7. **The config crate is an 8,222-line monolith** (`config/lib.rs`) with 4×
   ~140-arm string-key `match` statements (get/set/unset/list) dispatching over a
   typed struct — stringly-typed CRUD over a typed object.
8. **`merge_project_overrides` silently drops 7 of 25 providers**
   (`config/lib.rs:1375`) — repo-local `[providers.anthropic]` etc. is accepted,
   parsed, and discarded.
9. **No i18n layer in `web/`:** bilingual content is hand-duplicated across
   ~3,636 lines / 8 pages via inline `isZh ? "中" : "en"` ternaries and full
   duplicated JSX blocks.
10. **`~/.codewhale` path hardcoded as a string literal ~123 times** in `tui/`
    even though a `codewhale_home()` helper exists.
11. **Localization implemented as 7 giant `match` blocks** over a 441-variant
    enum (`tui/localization.rs`) — ~2,500 lines of structurally identical
    scaffolding that should be a data table.
12. **`.mailmap` collapses every AI-bot commit into the human maintainer**,
    contradicting the repo's own stated credit-preservation policy.

---

## Part A — The Rust codebase

### A1. Provider/model identity is smeared everywhere (the core leak)

The question "what model does provider X serve, and what are its aliases?" is
answered independently in 8+ places. Model-id literal hits per file:

| File | Relevant hits | Lines |
|---|---|---|
| `crates/tui/src/config.rs` | **319** | 12,182 |
| `crates/config/src/lib.rs` | 205 | 8,222 |
| `crates/agent/src/lib.rs` | 190 | 1,648 |
| `crates/tui/src/models.rs` | 54 | 918 |
| `crates/config/src/provider.rs` | 41 | 685 |
| `crates/tui/src/model_routing.rs` | 41 | 1,187 |
| `crates/tui/src/model_registry.rs` | 36 | 412 |
| `crates/cli/src/lib.rs` | 15 | 4,468 |
| `crates/tui/src/model_inventory.rs` | 6 | 310 |
| `crates/tui/src/model_catalog.rs` | 1 | 497 |

**Three separate model registries inside `tui` alone** (`model_catalog`,
`model_registry`, `model_inventory`), and `model_registry.rs` documents that the
duplication is "acknowledged and unfinished." Plus:

- `config/lib.rs:22-126` — ~105 hardcoded `DEFAULT_*_MODEL` / `DEFAULT_*_BASE_URL`
  string constants.
- `agent/lib.rs:69-830` — 82 hand-written `ModelInfo { … }` entries, same model
  re-listed under 6+ providers with inconsistent casing.
- `config/lib.rs:2634` — `normalize_model_for_provider`, a ~140-line per-provider
  match; the same alias list copy-pasted across ~10 arms.
- `tui/models.rs:267` — `known_context_window_for_model`, a giant substring match.
- `agent/lib.rs:978` — `model_family`, a substring chain classifying models that
  are already in the registry one function away.
- **Provider identity itself is represented three ways**: an `ApiProvider` enum
  (26 variants), `&str` provider-name literals dispatched by `match`
  (`config.rs:7804`), and substring-sniffed base URLs (`config.rs:2377`,
  `:4905`). The same provider can be referred to by all three.

This is the canonical case of "data that should live in one place, hand-copied
into source."

### A2. God-objects and god-functions

**`App` god-object (`tui/app.rs:1358`)** — a 208-field struct. `App::new`
(`app.rs:2030`) is 467 lines just to initialize fields. `reset_token_breakdown`
(`app.rs:1329`) is 613 lines. This struct is threaded mutably through nearly
every function in `ui.rs`/`history.rs`.

**God-functions (non-test, verified by line count):**

| Function | Location | Lines |
|---|---|---|
| `run_event_loop` | `tui/ui.rs:1333` | **3,333** |
| `monitor_turn` | `tui/runtime_threads.rs:2241` | **748** |
| `run_doctor` | `tui/main.rs:2835` | **945** |
| `apply_command_result` | `tui/ui.rs:6702` | 483 |
| `apply_env_overrides` | `tui/config.rs:3822` | 671 (66 inline env-var literals) |
| `run_subagent` | `tui/tools/subagent/mod.rs:3663` | 525 |
| `run_exec_agent` | `tui/main.rs:6419` | 514 |
| `install_rustls_crypto_provider` | `tui/main.rs:132` | 425 (suspiciously large) |
| `render` | `tui/ui.rs:7855` | 429 |
| `Runtime` impl | `core/lib.rs:806-1664` | ~860 |

### A3. Stringly-typed dispatch over typed data

- **`ConfigToml` key dispatch** (`config/lib.rs`) — four near-identical giant
  `match key { … }` blocks for `get_value` (:1452), `get_display_value` (:1594),
  `set_value` (:1604), `unset_value` (:1877), `list_values` (:2007), each ~140
  string arms. **365 string-literal comparisons** in 1366–2557. Adding a config
  key means editing all four. `set_value` also hides business rules inline
  (DeepSeek write-through mirrors to top-level fields, `:1633-1652`; others don't).
- **`Option<String>` fields that should be enums**: `approval_policy` (`:634`),
  `sandbox_mode` (`:635`), `auth_mode` (`:629`), `mode` — validated by scattered
  free functions instead of types. Invalid strings are accepted at parse time.
- **`cli/lib.rs:910,1055`** — `provider_slot`/`provider_env_vars` re-implement by
  hand exactly what `config::provider::Provider` already exposes as a trait.
- **`PROVIDER_LIST: [ProviderKind; 25]`** (`cli/lib.rs:941`) hand-duplicates
  `ProviderKind::ALL`. `provider_is_supported_by_tui` (`:969`) is a 23-variant
  `matches!()` listing. Both drift on every provider add.
- **`cli::run`** (`lib.rs:642-778`) — a ~135-line `match command` where ~18 arms
  are identical `delegate_to_tui(...)` calls. A table wearing a match.
- **`app-server::dispatch_stdio_request`** (`lib.rs:554-901`) — 347-line match on
  JSON-RPC method-name strings.
- **`mcp::default_rpc_methods`** (`lib.rs:679`) returns a hardcoded `Vec<&str>`
  hand-matched elsewhere — list and match maintained in parallel.

### A4. Dead weight

- **`crates/tui-core`** — 230 lines of impl, **zero consumers** (verified: no
  `Cargo.toml` references it, no file outside the crate mentions its types). The
  real TUI state machine lives in `crates/tui/src/core/engine/*`.
- **`crates/whaleflow`** — 5,716 lines, **zero consumers** (verified). Either
  abandoned or speculative-generality. `MockWorkflowExecutor` (~300 lines) lives
  in `lib.rs` non-test code despite no consumer.
- **372 `#![allow(dead_code)]` markers in `crates/tui/src`**, including whole
  modules: `provider_adapter.rs`, `provider_readiness.rs`, `features.rs`,
  `goal_loop.rs`, all of `tui/tab/*`, much of `fleet/*`.
- **Dormant knobs**: `MAX_SPAWN_DEPTH_CEILING == DEFAULT_SPAWN_DEPTH == 3`
  (`config/lib.rs:1118,1123`) — the ceiling is inert.
- **`Extras` catch-all** (`config/lib.rs:679`, `#[serde(flatten)]`) silently
  swallows unknown keys — combined with A5, misspelled keys vanish silently.

### A5. Silent config drops (a correctness bug dressed as a smell)

`merge_project_overrides` (`config/lib.rs:1375`) forwards **only 18 of 25**
`ProvidersToml` fields. Missing: `anthropic`, `deepinfra`, `minimax`,
`openai_codex`, `stepfun`, `together`, `zai`. A repo-local
`.codewhale/config.toml` with `[providers.anthropic]` is parsed and silently
discarded.

### A6. Hardcoding that shouldn't be

- **`~/.codewhale` as a string literal ~123 times** in `tui/` even though
  `codewhale_home()` exists (`main.rs:2935`).
- **Backwards-compat typo hostname enshrined forever**: `config.rs:162`
  permanently recognizes `api.deepseeki.com`.
- **Magic timeouts inline**: 15 bare `Duration::from_secs/millis(<literal>)` in
  `runtime_api.rs`.
- **Token format buried in a fn body**: `runtime_api.rs:156`
  `format!("cwrt_{}{}", uuid, uuid)`.
- **`execpolicy/src/bash_arity.rs:44`** — `BASH_ARITY_TABLE`, 215 hardcoded
  `(prefix, arity)` tuples. Data-that-should-be-a-file.
- **Legacy-name sprawl in env vars**: every knob in `release/lib.rs` is tripled
  (`CODEWHALE_*` / `DEEPSEEK_TUI_*` / `DEEPSEEK_*`).
- **`app-server/lib.rs:235,559`** — `"service": "deepseek-app-server"`, a legacy
  name in a "codewhale" product.

### A7. Duplication, not abstraction

- **`tui/history.rs`** — 18 near-identical `render_*` functions sharing
  5-parameter signatures. `render_tool_output_mode`/`render_exec_output_mode`/
  `render_preserved_output_mode` are strong copy-paste.
- **`core/lib.rs:1664-2040`** — dozens of hand-rolled serde adapters — inverse
  pairs kept in sync by hand between `protocol`, `state`, and ad-hoc
  `serde_json::Value`.
- **`state/lib.rs:1602-1670`** — four pairs of `*_to_str`/`*_from_str`
  duplicating `protocol`'s enums.
- **`state/lib.rs`** — 23 inline SQL strings with hand-listed columns.
- **`tools/subagent/mod.rs`** — 4-way manually-synced alias vocabulary
  (`VALID_SUBAGENT_TYPES`, `VALID_ROLE_ALIASES`, `SUBAGENT_TYPE_DESCRIPTION`,
  `SUBAGENT_ROLE_DESCRIPTION`) + two match statements that must all agree.
- **`secrets/lib.rs`** — `FileKeyringStore` is named "Keyring" but stores
  **plaintext JSON** to `~/.codewhale/secrets/secrets.json`. Leaky abstraction.
- **`tui/src/config.rs:4770-4820`** — near-identical predicate functions with one
  hostname swapped.

### A8. What is *not* a problem (credit where due)

- **Panic discipline is strong.** ~4,840 `unwrap`/`expect` in `tui/src`, but
  virtually all inside `#[cfg(test)]`. Only non-test panics are `config.rs:4382`
  (`unreachable!`) and two `expect`s in `whaleflow/starlark_authoring.rs`.
- **Zero `TODO`/`FIXME`/`HACK`/`XXX`/`todo!()`/`unimplemented!()`** in Rust.
  (Cuts both ways — obvious debt is unacknowledged.)

---

## Part B — Cargo workspace

- **`crates/tui` bypasses `[workspace.dependencies]`** and re-declares ~15 deps
  inline (62 inline version pins). Already drifted: `chrono` is `0.4.43` in the
  workspace, bare `0.4` in tui.
- **`tempfile` skew**: `3.16` in the workspace, `3.27` inline in 4 crates.
- **`reqwest` feature fragmentation**: each crate enables a different slice.
- **Version literal duplicated ~30×**: every internal path dep hardcodes
  `version = "0.8.61"`. A bump touches every manifest.
- **Suspicious layering**: `config` (a leaf schema crate) depends **upward** on
  `execpolicy` + `secrets`.
- **`default-members`** = only `cli`, `app-server`, `tui`.

---

## Part C — The `web/` app, `npm/` installer, VS Code extension

### C1. Tracked build output (the clearest "should not be in git")
- **`extensions/vscode/out/`** — 6 files committed; not in `.gitignore`.
- **`web/lib/facts.generated.ts`** — tracked; guarantees a noisy generated diff
  every release.

### C2. Hardcoded values that should be derived
- **`Hmbown/CodeWhale` slug in ~85 places** despite a `GITHUB_REPO` env var.
- **`0.8.61` version hand-literal** alongside the `facts.version` system.
- **"25 providers" hardcoded** instead of derived from `facts.providers.length`.
- **~60 GitHub handles** baked into `page.tsx:23-138`.
- **Forked client config**: `community-agent.ts` vs `deepseek.ts` — two configs
  for the same LLM client, divergent.

### C3. No i18n — bilingual content hand-duplicated
- 62 inline `isZh ? "中" : "en"` ternaries in `page.tsx`; `faq.tsx`/`contribute.tsx`
  write full separate EN-then-ZH JSX blocks. ~3,636 lines total, no extraction.
- Section "chrome" hand-written on every section instead of a `<Section>`.
- `buildPageMetadata` SEO helper used by only 2 of 8 pages.

### C4. The `facts` drift machinery is over-engineered
- `facts-drift.ts` re-implements a mini Rust parser via regex at runtime.
- **Provider `labelMap` duplicated in THREE places**: `derive-facts.mjs:67`,
  `facts-drift.ts:78`, the Rust enum.

### C5. Other web/npm smells
- Duplicate env-shape interfaces across 3-4 files.
- Same GitHub-fetch headers block copy-pasted 6× despite a `headers(token)`
  helper.
- Hand-rolled HTML parser in `content-watch.ts:150-217`.
- Dual env namespace: `DEEPSEEK_*` in 9 npm files, `CODEWHALE_*` in 6.
- **What's clean**: zero `any`/`@ts-ignore`/`eslint-disable`; consistent
  `unknown`-narrowing.

---

## Part D — Scripts, CI, repo hygiene

### D1. Build/CI drift
- **Rust toolchain**: no `rust-toolchain.toml`. CNB/Docker pin `1.88` (4 sites);
  GHA/Nix float `@stable` (9 sites).
- **Node**: `web.yml`/`.cnb.yml` use Node 22; a benchmark installs Node 20; no
  `.nvmrc`.
- **Duplicated CI steps**: `.cnb.yml` has two near-identical ~30-line Rust-gate
  blocks. `apt-get update` retry loop copy-pasted 6×.
- **Parity tests only run at release**: `release.yml` runs parity tests that
  `ci.yml`/`.cnb.yml` don't. Release-only checks not enforced on PRs.
- **Secrets risk (concrete)**: `.github/workflows/web.yml:49` hardcodes
  `CLOUDFLARE_ACCOUNT_ID: cf50f793171d7cb3b2ce23368b69cdcb` in plaintext.

### D2. Repo hygiene gaps
- **Three tool/cache dirs not gitignored** (~900MB exposed): `.venv-bench/`
  (268M), `.uv-cache/` (643M), `.uv-tools/`. Verified `git check-ignore` = NOT
  IGNORED for all three.
- `.gitignore` globally ignores `*.sh`/`*.cmd` then re-`!`-includes `scripts/**`
  — a stray `.sh` anywhere else silently vanishes.

### D3. Doc sprawl
- `CLAUDE.md` is a 100% paraphrase of `AGENTS.md`.
- Version-pinned evergreen docs: both reference the `v0.8.59` queue (now updated
  to v0.8.62 in this branch).
- `README.ja-JP.md` is ~37% shorter than its siblings; no sync check.
- `config.example.toml` is 1,007 lines, 83% comments — the de-facto config
  manual, mixing two crates' keys and documenting keys config ignores.

### D4. `.mailmap` — doing work that contradicts policy
`.mailmap` maps **15 distinct bot/agent identities** all to
`Hunter Bown <hmbown@gmail.com>`. Displays every AI-tool commit as authored by
the human — the *opposite* of the credit-preservation philosophy in
`AGENTS.md`/`CONTRIBUTING.md`.

---

## Suggested execution order (lowest risk → highest)

These are independent, reviewable units. Pick one, finish it, commit, move on.

1. **Quick repo-hygiene wins (no behavior change):**
   - Add `.venv-bench/`, `.uv-cache/`, `.uv-tools/` to `.gitignore` (D2).
   - Untrack `extensions/vscode/out/*` and add to `.gitignore` (C1).
   - Add a `rust-toolchain.toml` pinning `1.88` (D1).
   - Move `CLOUDFLARE_ACCOUNT_ID` to a repo variable/secret (D1).
2. **Concrete correctness bug:** fix `merge_project_overrides` to forward all 25
   providers, or warn on the dropped 7 (A5).
3. **Dead-code removal:** delete `crates/tui-core` and `crates/whaleflow` (after
   confirming with Hunter they're not parked-future), or document why they stay
   (A4).
4. **Mechanical dedup:** consolidate the three tui model registries into one
   (A1). Large but high-value; do behind the existing trait.
5. **The big structural ones (each its own epic):** `App` god-object teardown,
   `config/lib.rs` stringly-typed dispatch, provider/model single-source-of-truth.

---

*Audit performed as a read-only pass; nothing here has been changed. Verify each
claim against the current `hunter/0.8.62-glm-subagents` HEAD before acting — line
numbers drift as the tree evolves.*
