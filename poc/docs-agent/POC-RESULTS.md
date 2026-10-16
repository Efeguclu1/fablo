# POC: Agentic documentation automation for Fablo

**Question this POC set out to answer:** can we keep Fablo's documentation
correct and up to date automatically, using AI agents driven by
[jaiph](https://jaiph.org) workflows in CI?

**Answer: yes.** Both halves of the problem were built and verified against the
real repository — detecting/fixing doc drift, and authoring new docs from the
codebase. Details and evidence below.

---

## 1. How it works — three layers

| Layer | Role | What plays this part |
|---|---|---|
| **Execution** | *When* to run, on *what*, and what to do with the output (gather context → run agent → capture result → write file / open PR → test → schedule) | **jaiph** — a small workflow language (`.jh` files) that orchestrates coding agents. v0.10.0. |
| **Methodology** | *How* to write good docs | The [`documentation-writer`](https://www.skills.sh/github/awesome-copilot/documentation-writer) skill (Diátaxis framework), embedded verbatim into the agent prompt. |
| **Author** | Actually reads code and writes prose | **Claude** (jaiph's `agent.backend = "claude"`). |

jaiph runs the agent, the skill directs the agent, the agent writes the doc.
GitHub Pages is *not* part of generation — it is only the hosting layer for the
final site, and Fablo already has it wired (`docs/CNAME` → `fablo.io`).

---

## 2. Two tracks, both proven

### Track A — Drift sync (keep existing docs current)

**Goal:** when code changes, catch docs that have gone stale, and fix them.

- [`docs-drift.jh`](docs-drift.jh) — *report-only* detector. Gathers a `git diff`
  of the source surface (`src/`, `fablo.sh`, `docs/schema.json`) since a base
  commit, feeds it plus `SUPPORTED_FEATURES.md` to the agent, and returns a
  **structured verdict**: `{ needs_update: boolean, stale_rows, summary }`.
- [`docs-drift.test.jh`](docs-drift.test.jh) — deterministic unit tests that mock
  the scripts and the agent, so the workflow logic is verified in CI with **no
  LLM call and no cost**.
- Auto-fix stage prototyped separately (`check → if drift, agent rewrites the doc
  → write to disk`; idempotent — a second run makes no change). Kept out of the
  repo since it edits files; it folds into the CI fix step.

**Evidence (live runs against this repo):**

| Scenario | Result | Time |
|---|---|---|
| Real, up-to-date matrix (`HEAD~10..HEAD`) | `needs_update = false` — correctly saw the recent `mysql` CA-DB change is *already* covered; did not false-alarm on version bumps / path fixes | ~16–23 s |
| Doctored matrix (MySQL flipped to unsupported) | `needs_update = true`, pinpointed exactly `CA DB - MySQL` (`✕→✓`, v2 & v3), citing the type-union diff as evidence | ~24 s |
| Auto-fix (demo): export a new function, run sync | Agent added the missing entry in the existing style — a single clean line — then a re-run reported in-sync and wrote nothing (**idempotent**) | ~24 s |
| `jaiph test` (mocked) | 2/2 pass | <1 s |

The detector discriminates *real* drift from noise, and the boolean is reliable
enough to gate a CI job.

### Track B — Generation (author new docs from the codebase)

**Goal:** produce real documentation pages, not just detect drift — the basis for
a proper docs site.

- [`generate.jh`](generate.jh) — a reusable engine `generate_page(out_path, brief, source)`
  that applies the `documentation-writer` skill to any source + brief; `default()`
  wires three real Fablo subjects to it. Add a subject = one gatherer script +
  three lines. Output → `out/` (touches no real docs).
- [`skills/documentation-writer.md`](skills/documentation-writer.md) — the skill,
  fetched verbatim from `github/awesome-copilot`.

**Evidence:** live runs produced **three real pages** spanning different repo
areas and two Diátaxis types — each fact-checked against its source:

| Page | Source | Type | Fact-check |
|---|---|---|---|
| [`out/configuration.md`](out/configuration.md) | `docs/schema.json` + `sample.json` | Reference | Every documented field exists in the schema; enums match (`solo/raft/BFT`, loglevel `debug/info/warn`); caught the `ccaas` conditional; honest that `hooks` is out of scope. **No hallucinated fields.** |
| [`out/cli-commands.md`](out/cli-commands.md) | `fablo.sh` usage + `src/commands/` | Reference | Documents exactly the commands in `fablo.sh` (`init`, `generate`, `up`, `down/start/stop`, `reset`, `prune`, `recreate`, `chaincode(s) …`, `channel`, `snapshot`, `restore`). **No invented commands.** |
| [`out/getting-started.md`](out/getting-started.md) | `fablo.sh` + `init` command | How-to | Concise 3-step recipe (init → up → tear down); correct problem-oriented shape. |

Each page is a single agent call (~30–65 s).

---

## 3. Honest limitations / open questions

- **Two Diátaxis types proven, two not yet.** *Reference* (×2) and *How-to* are
  validated and accurate. *Tutorial* and *Explanation* — longer, more prose, less
  fact-anchored — have not been exercised; that is where the remaining quality
  risk sits, so those pages will need closer human review.
- **The skill is interactive by design.** Its native workflow is "ask questions →
  propose outline → *await approval* → write." For an unattended pipeline we
  pinned type/audience/goal/scope up front and told it to proceed in one pass.
  A real deployment should keep a **human-review gate** — which the PR-based CI
  flow provides naturally.
- **Sandbox / workspace.** Default `jaiph run` executes in Docker and auto-detects
  the workspace as the `.jh`'s folder, which excludes the repo root / `.git`.
  POC runs used `--unsafe --workspace ..`. CI must either run unsafe in the
  runner or mount the repo root.
- **Backend auth in CI.** The `claude` backend needs credentials + a writable
  `CLAUDE_CONFIG_DIR`; local runs use the developer's authenticated CLI. CI needs
  an API key wired as a secret. `codex` is an alternative backend.
- **Cost/latency.** Each live check or generation is one agent call
  (~15–65 s, a few cents). The `needs_update` boolean gate means the expensive
  fix/generation step only runs when there is actually drift.

---

## 4. Recommended path to production

1. **Sync in CI (small, low-risk).** A GitHub Action runs `docs-drift` on
   push/PR (or nightly); when `needs_update = true`, it runs the fix stage and
   opens a **pull request** with the doc change. Ship it opt-in / non-blocking
   until trusted. The mocked tests run on every PR for free.
2. **Generate the site (larger).** Extend `generate.jh` with more subjects to
   author the full set of pages (a Reference set first, then Tutorials/How-to),
   review the output by hand, then point the existing `fablo.io` Pages setup at
   the generated content (optionally via a static-site generator for polish).

Track 1 delivers value immediately with the least risk; Track 2 is the bigger
"real docs site" investment and can follow once Track 1 is trusted.

---

## 5. Reproduce it

```bash
cd poc/docs-agent

# Track A — live drift check against the repo (read-only):
jaiph run --unsafe --workspace .. docs-drift.jh "HEAD~10"

# Track A — deterministic tests (no agent, CI-safe):
jaiph test docs-drift.test.jh

# Track B — generate the three sample pages into out/:
jaiph run --unsafe --workspace .. generate.jh
```

Requirements: `jaiph` v0.10.0+ on PATH, an authenticated `claude` CLI, run from
inside the Fablo git repo.

---

## 6. File inventory

| Path | What it is |
|---|---|
| [`docs-drift.jh`](docs-drift.jh) | Track A — report-only drift detector |
| [`docs-drift.test.jh`](docs-drift.test.jh) | Track A — mocked unit tests |
| [`generate.jh`](generate.jh) | Track B — multi-subject doc generator (reusable engine + 3 wired subjects) |
| [`skills/documentation-writer.md`](skills/documentation-writer.md) | The Diátaxis skill (verbatim from awesome-copilot) |
| [`out/`](out/) | Three generated sample pages — config & CLI reference + getting-started how-to; judge quality here |
| `.jaiph/` | jaiph workspace (`SKILL.md` = language reference; `runs/` are gitignored) |
