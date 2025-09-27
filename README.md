# Cedar v0.1.1 — **AI‑Augmented**, Compiler‑Lens Foresight, Edit‑Resilient, Batch‑First

> **Positioning:** Cedar is a **Rust fork / distribution** that ships a Dual‑Engine analyzer integrated into the toolchain. **T‑fast** is deterministic and compiler‑like; **T‑smart AI** runs in the same batch to produce **witnesses, explanations, and concrete patch diffs**. Output is **atomic SARIF**; editors show diagnostics and guarded code actions. **No daemon.**

---

## 0) Objectives & non‑objectives

**Objectives**

* Deliver **T‑fast** findings in ≤2 s (warm, ~30 k LOC) with **content‑addressed, edit‑resilient IDs (SFI)**.
* In the **same run**, execute **T‑smart AI** to add witnesses, explanations, and AI‑refined patch diffs in **≤12 s** on medium repos.
* Emit **atomic SARIF** with **journal + recovery** (no partial/garbage output).
* Provide **safe patch rebasing** (hash + anchors) and honest “stale” UX.
* Cover panic/footguns: unwrap/expect/panic/assert, **indexing**, **division by zero**, **integer overflow (debug)**, UTF‑8 slicing, Serde guards, async blocking/locks.

**Non‑objectives (v0.1.1)**

* Full export‑graph/bin reachability via rustc internals (kept minimal/heuristic).
* Coverage‑sliced testing & function‑scoped Miri (hooks exist; optional later).
* Global macro expansion / full rustc type queries (documented limits; optional future `--expand`).

---

## 1) Workspace & execution model

**Crates**

* **`cedar-types`** — Finding/Patch models; SFI hashing helpers (SFH/SRH/ANCH/ESH).
* **`cedar-sarif`** — SARIF writer + **journal & recovery** (atomic handoff).
* **`cedar-core`** — Two‑phase scan (surface→AST), single visitor, **T‑fast detectors**.
* **`cedar-intel`** — **T‑smart AI (ON)**: context graph v‑lite, seeders, witnesses, CEGR patchers, explain.
* **`cedar-cli`** — `foresee | explain | stage | doctor`; flags `--parallel`, `--profile` (file, not SARIF), `--eager-ast`; startup recovery.
* **`cedar-ra-ext`** — VS Code: diagnostics, **hash→anchor rebase**, “stale” badge, patch preview.
* **`examples/web_demo`** — Seeded faults for demos and E2E tests.

**Batch run (single process)**

```
cargo cedar foresee
 ├─ FAST phase: T‑fast detects + writes findings.part.sarif → atomic rename findings.sarif
 └─ SMART phase: T‑smart AI upgrades same IDs with witnesses/explanations/patch diffs
                 writes smart.part.sarif → atomic rename smart.sarif
Editor loads latest (smart.sarif if present) and merges with fast results if needed.
```

---

## 2) Stable Finding Identity (SFI) — content‑addressed IDs

**ID format (always present)**

```
<rule>::<relpath>::sfh=<16hex>::srh=<16hex>::anch=<16hex>[::dup=<16hex>]
```

**Components & algorithms**

* **SFH (Stable Function Hash, 64‑bit):** from function signature (visibility, `async`, name, generics, **param types only**, return type); tokens normalized via `proc_macro2`; SHA‑256 → 64b.
* **SRH (Stable Region Hash, 64‑bit):** **token‑region** (syn‑neutral):

  * Choose the **smallest complete expression** containing the finding that is **a statement, a block item, an arm body, or has ≥20 tokens**.
  * Expand out with balanced `()[]{}`; normalize tokens; SHA‑256 → 64b.
  * Example edge:

    ```rust
    let x = if cond { value.unwrap() } else { other };
    ```

    Region prefers the **if‑expression** (arm body), not the whole `let` (unless <20 tokens).
* **ANCH (Anchor Hash, 64‑bit):** first 8 tokens of the offending expression + 3 tokens left/right context, normalized; SHA‑256 → 64b.
* **DUP (optional, 64‑bit):** **ESH** of the **entire expression subtree** (normalized tokens), only when (SFH, SRH, ANCH) collide.

**Versioning & syn stability**

* **IDs never change** for the same logical issue.
* Track detector evolution via `properties.rule_version` (semver) and `properties.detector_rev`.
* **Pin syn exactly** (e.g., `=2.0.68`); for rare transitions, do **not** pollute SARIF:

  * Write `target/cedar/id_migrations.json` mapping `old_id → new_id` for tools that care.

---

## 3) Two‑phase scan & performance

**Surface pass (gate)** — precompiled multiline regex/substrings:

* Public fn: `(?s)pub\s*(\([^)]*\))?\s*fn` (covers `pub`, `pub(crate)`, split‑lines)
* Async fn: `(?s)async\s*fn`
* Serde derive: `derive\s*\([^)]*Deserialize`
* Panic hints: `\.unwrap\b|\.\s*expect\b|\bpanic!\b|\bassert(?:_eq|_ne)?!\b`
* Structure/arithmetic: `\[|\.\.` and `[\/\+\-\*]`
* Any match → parse to AST; **`--eager-ast`** forces AST for all files.

**AST pass** — single `syn::visit::Visit` per file, feeding **all** detectors.

**Parallelism**

* Default sequential (deterministic).
* `--parallel` (file‑level) with deterministic output ordering by path/bytepos.

**Performance (warm, ~30 k LOC)**

* Read ≈300 ms; AST 55–70% files ≈720 ms; visit ≈630 ms; SARIF ≈50 ms → **~1.7 s** sequential; **0.4–0.6 s** with 8 cores.

**Profiling (no SARIF noise)**

* Write **`target/cedar/profile-<run_id>.json`**; maintain `profile-latest.json` (atomic rename).

---

## 4) T‑fast detectors (deterministic, shipped)

All emit SFI and `rule_version`.

1. **`panic_public_api` v1.1.0** — `.unwrap`, `.expect`, `panic!`, `assert*` in public `fn`.
   *Patch:* comment nudging to `ok_or_else(...)?` / explicit error (minimal edit).

2. **`utf8_raw_slice` v1.0.0** — `&str[..]` without boundary guard.
   *Patch:* comment with `char_indices()` safe slicing pattern.

3. **`serde_guard` v1.0.0`(warn)** —`Deserialize`struct missing`#[serde(deny_unknown_fields)]`;   **`serde_secret_hygiene` v1.0.0` (note)** — fields like `password/token/...`;
   **`serde_enum_tryfrom` v1.0.0`(note)** — primitive‑repr enums need`TryFrom`.

4. **`async_blocking_io` v1.1.0`/`async_lock_blocking` v1.1.0`** — blocking std I/O/net / `std::sync::{Mutex,RwLock}` inside `async fn`.
   *Patch:* comment to `tokio::task::spawn_blocking` + timeout; prefer `tokio::sync`.

5. **`panic_indexing_public_api` v1.0.0** — `Expr::Index` on likely slice/array/Vec.
   *Patch:* use `get(i)` and handle `None` / `?`.

6. **`panic_divzero_public_api` v1.0.0** — `/` with literal `0` (warn) or **unguarded** variable RHS within ±5 statements (note).
   *Patch:* guard or use `NonZero*` in signature.

7. **`panic_overflow_debug_public_api` v1.0.0** — integer `+ - *` without `checked|saturating|wrapping`.
   *Patch:* `checked_*` with error propagation or `saturating_*`. (Note: debug‑only panic.)

**Known limits:** Macro‑generated async may be missed (future `--expand` optional).

---

## 5) **T‑smart AI** (on from day one)

**Role:** Immediately after FAST, **AI augments** findings with concrete proof and safer diffs.

**Components (in `cedar-intel`)**

* **Context Graph v‑lite:** functions/types/modules, call/derive/export‑light edges, role/domain scoring (e.g., PublicApi, Web, DB, Async) via rules.
* **Seeders (input generation):**

  * UTF‑8: ASCII + multibyte + combining marks; boundary killers.
  * Panic: strings that force `None/Err` for common unwrap patterns (e.g., `split('@')` with no `@`).
  * Serde: unknown fields, out‑of‑range enums, over‑large arrays/maps.
  * Async: contention/deadlock probes (mutex + awaits), blocking scenarios.
* **Witnesses:** For each SFI, attempt **≥1 failing input + local trace** (panic message/line, error path). Store under `target/cedar/evidence/<finding_id>/`.
* **CEGR Patchers (AI‑refined):**

  * Start from the minimal T‑fast operator intent; synthesize a **concrete before/after diff** that compiles.
  * Validate locally (rebuild and, if configured, run targeted micro‑test).
  * Attach **`validated: true/false`** and scope of change.
* **Explain:** Natural‑language rationale (short, precise), tying role/domain + witness to risk and the patch.

**Inference model**

* **Local‑first:** no network by default; deterministic heuristics + optional tiny local models.
* **Hooks:** `CEDAR_LLM=<provider>` enables external or local LLM; strictly **time‑boxed** with deterministic fallback.
* **Failure handling:** If AI exceeds budget or fails, FAST remains; add a `cedar_ai_error` note with cause (no crash, no partials).

**Budgets (defaults, overridable)**

* Repo ~30 k LOC warm: **≤12 s** total SMART phase.
* Per finding: ≤250 ms seed+probe; ≤350 ms if proposed patch requires compile test.
* Global cap: `--smart-budget-ms <N>` or `CEDAR_SMART_BUDGET_MS`.

**SMART SARIF upgrade (per finding)**

```json
{
  "finding_id": "…(same SFI)…",
  "phase": "smart",
  "properties": {
    "witness": { "seed": "user@café.com", "trace": "thread 'main' panicked at ..." },
    "patch":   { "hunk": "@@ -12,7 +12,11 @@", "validated": true },
    "explain": "Public API should not panic; multibyte input cuts a codepoint…",
    "context": { "role": {"PublicApi":0.86}, "domains":[{"Web":0.82}] }
  }
}
```

---

## 6) Patch anchoring & safe apply (editor)

**Patch payload**

```json
"patch": {
  "file": "src/lib.rs",
  "start_line": 42, "start_col": 17, "end_line": 42, "end_col": 24,
  "new_text": "/* cedar suggestion or diff snippet */",
  "pre_context": "let v = thing",
  "post_context": ");",
  "file_sha256": "<64hex>",
  "span_sha256": "<64hex>",
  "rebase": { "max_lines": "auto", "strategy": "hash_then_anchors" }
}
```

**Rebase window (revised)**

```
window_lines = min(
  max(50, original_line),
  max(300, floor(0.15 * file_lines))
)
```

**Apply algorithm**

1. If `current_file_sha == file_sha256` → apply at coords.
2. Else try exact `span_sha256`; if none, KMP `pre_context → post_context` within `window_lines`.
3. Apply only if **unique**; otherwise show **“Stale — re‑run Cedar”** and disable quick fix.

**UX**

* “(stale)” badge on changed files; **patch preview** command (“Cedar: Show Patch Context”).

---

## 7) SARIF schema (clean, deterministic)

**Run `properties`**

```json
{ "root": "<path>", "workspace_ts": <unix>, "workspace_sha256": "<64hex>", "run_id": "<16hex>", "cedar_version": "0.1.1", "phase": "fast|smart" }
```

**Result `properties`**

```json
{
  "finding_id": "<rule>::<relpath>::sfh=..::srh=..::anch=..[::dup=..]",
  "rule_version": "1.1.0",
  "detector_rev": "r1234abcd",
  "phase": "fast|smart",
  "state": "open|closed",
  "anchors": { "sfh":"..","srh":"..","anch":"..","pre_context":"..","post_context":".." },
  "patch": { ... as above ... },
  "stale_policy": { "allow_fuzzy": true, "max_lines": "auto" },
  "witness": { "... if smart ..." },
  "explain": "… if smart …",
  "context": { "role": {..}, "domains": [..] }
}
```

**No profiling in SARIF** (lives in `profile-<run_id>.json`).
**ID migrations** (if needed) live in `id_migrations.json`.

---

## 8) Journal & crash recovery

* On start: if `.sarif.journal` + `.part` exist:

  * If `.part` parses → rename to final (`findings.sarif` or `smart.sarif`), remove journal.
  * Else → quarantine to `*.aborted.sarif`, remove journal.
* On success: write `.part`, `fsync`, atomic rename, remove journal.
* Extension shows one‑time warning if `*.aborted.sarif` is newer than final.

---

## 9) CLI

* `cargo cedar foresee [--files … | --all] [--json-progress] [--parallel] [--profile] [--eager-ast] [--smart-budget-ms N]`

  * FAST then SMART (AI) in one run; emits `findings.sarif` and `smart.sarif` (if any).
  * Writes `profile-<run_id>.json` when `--profile`. Progress JSONL (no timings) includes `run_id`.

* `cargo cedar explain <symbol|file:line>`

  * Emits `target/cedar/explain.json` (may include AI rationale and witness summary).

* `cargo cedar stage`

  * Writes CI workflow (placeholders for coverage/Miri on patched functions).

* **NEW** `cargo cedar doctor [--corpus <path>] [--edits <path>] [--thresholds <file>]`

  * **SFI stability** across edits & syn pins.
  * **Rebase accuracy** on scripted edits.
  * **Detector consistency** (`rule_version`, `detector_rev`).
  * **Perf sanity** from `profile-<run_id>.json`.
  * Non‑zero exit on violations.

**Environment hooks**

* `CEDAR_LLM` (provider/URL); `CEDAR_SMART_BUDGET_MS`; `CEDAR_PARALLEL=1`.

---

## 10) VS Code extension (`cedar-ra-ext`)

* Loads SARIF (fast and/or smart), publishes diagnostics, merges upgrades by `finding_id`.
* Shows **witness** details and **explain** text when present.
* Guarded quick fix with rebase; preview diff; stale badge when buffer hash differs.
* Settings: `cedar.patchRebase.maxLines` (default “auto”), `cedar.patchRebase.strategy`.

---

## 11) Performance budgets & validation

* **SLOs:** ≤2.0 s (FAST); ≤12 s (SMART) on ~30 k LOC warm; ≤0.6 s with `--parallel` for FAST.
* **Doctor** validates SFI stability, rebase success rate, detector versions, and perf thresholds.

---

## 12) Testing

* Unit tests per detector (positive/negative, guard windows).
* E2E on `examples/web_demo`: FAST findings + SMART upgrades (witness + patch).
* Recovery tests: kill mid‑run; next run finalizes/quarantines `.part` then succeeds.
* Performance sweeps: 10k/30k/60k LOC; capture profiles.

---

## 13) Security & privacy

* **Local‑first**: no network by default; artifacts stay local.
* Optional LLM calls are explicit, time‑boxed, and redact sensitive payloads.
* Secret‑hygiene findings never include real secrets.

---

## 14) Limitations & mitigations

* Macro‑generated async may be missed → document; future `--expand`.
* Surface‑pass misses (e.g., `expect` via var message) → acceptable; `--eager-ast` available.
* Type‑precision is heuristic in T‑fast; T‑smart can optionally use analyzer lenses later.

---

## 15) Acceptance criteria

1. **Identity:** Edits above a finding do not change the **finding_id**; duplicate patterns disambiguate via **DUP=ESH**.
2. **AI from the go:** SMART phase runs in the same batch; each FAST finding attempts a **witness** and, when safe, a **validated patch**.
3. **Async correctness:** Blocking I/O and `std::sync` locks detected anywhere inside `async fn`.
4. **Recovery:** Crashes leave either a finalized SARIF or an `*.aborted.sarif`; next run proceeds cleanly.
5. **Safe patching:** Patches apply or are safely refused with clear “Stale — re‑run Cedar”, never mis‑applied.
6. **Performance:** Meets SLOs; `doctor` passes on corpus.

---

**Bottom line, Bee:** Cedar launches **with AI in the loop** — not bolted on later. T‑fast gives immediate, deterministic signal; **T‑smart AI** adds **proof and patches** right away, using our compiler‑lens identity and atomic pipeline to stay trustworthy. Con calma y fuerza: ambitious, but grounded.
