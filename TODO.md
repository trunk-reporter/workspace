# Trunk Reporter — Autonomous Agent Work Cycle

All repos live at https://github.com/orgs/trunk-reporter/repositories.
Cross-repo roadmap: https://github.com/orgs/trunk-reporter/projects/1

---

## Before Starting Any Session

Read these files in order — they are your operating context:

1. `AGENTS.md` — architecture, subprojects, data flow, deployed instance
2. `GUIDELINES.md` — branching, testing, deployment, secrets rules
3. `EXTERNAL.md` — who the users are and what they actually want
4. The `AGENTS.md` inside whichever subproject you're about to touch

If any of these files are missing or clearly outdated, update them before doing other work.

---

## Phase 1: Pull the Work Queue

Fetch open GitHub issues across all repos:

```bash
gh issue list --repo trunk-reporter/tr-engine --state open --json number,title,labels,assignees
gh issue list --repo trunk-reporter/tr-dashboard --state open --json number,title,labels,assignees
gh issue list --repo trunk-reporter/tr-bot --state open --json number,title,labels,assignees
gh issue list --repo trunk-reporter/tr-plugin-dvcf --state open --json number,title,labels,assignees
gh issue list --repo trunk-reporter/tr-plugin-avcf --state open --json number,title,labels,assignees
gh issue list --repo trunk-reporter/imbe-asr --state open --json number,title,labels,assignees
gh issue list --repo trunk-reporter/qwen3-asr-server --state open --json number,title,labels,assignees
gh issue list --repo trunk-reporter/tr-stack --state open --json number,title,labels,assignees
```

Also check the cross-repo project board for prioritized items:
```bash
gh project list --owner trunk-reporter
```

**Prioritize issues by:**
1. Bug reports with user-reported impact (crashes, data loss, auth breakage)
2. Issues directly matching wants in `EXTERNAL.md` (transcription, affiliation tracking, multi-site, setup friction)
3. Security or data integrity issues
4. Feature requests with clear scope
5. Chores / refactors

---

## Phase 2: Development Loop

For each issue, work through this cycle:

### 2a. Understand the Issue
- Read the full issue thread, including any linked PRs or discussions
- Read the relevant subproject's `AGENTS.md` before touching code.
- Reproduce the bug or trace the code path before writing anything

### 2b. Branch
```bash
git checkout -b fix/issue-123-short-description   # for bugs
git checkout -b feat/issue-123-short-description  # for features
```
Work in the subproject directory. Never commit directly to `main`.

### 2c. Implement
- Match the conventions of the surrounding code
- Don't add error handling, abstractions, or cleanup that the issue didn't ask for
- Update `openapi.yaml` if any tr-engine API endpoint changes
- Update the relevant `AGENTS.md` if architecture, build steps, or conventions change.

### 2d. Build and Test
Run the build and lint for the subproject **before** committing:

| Project | Build | Lint/Type-check |
|---------|-------|-----------------|
| tr-engine | `bash build.sh` | `go vet ./...` |
| tr-dashboard | `npm run build` | `npm run lint` |
| tr-bot | `npm run build` | `npm run lint` |
| imbe_asr | _(no build step)_ | run inference script to confirm |
| qwen3-asr-server | _(no build step)_ | start server, hit `/health` |

Test against the live instance via Tailnet when real data is needed:
`http://gerty.pizzly-manta.ts.net:8080`

Do not push if the build fails or the type-checker errors.

### 2e. Commit and PR
```bash
git add <specific files>
git commit -m "fix: short description (#123)"
gh pr create --title "..." --body "..."
```

Reference the issue number in the commit and PR. PR body should include what changed and how it was tested.

### 2f. Close the Loop
After merging, close the issue if it isn't auto-closed by the PR:
```bash
gh issue close 123 --repo trunk-reporter/<repo> --comment "Fixed in #<PR>"
```

If the fix warrants a deploy to `gerty`, follow the deploy steps in `tr-engine/AGENTS.md`.

---

## Phase 3: Review Passes

When the issue queue is empty (or between issue batches), run structured reviews. Each review produces new GitHub issues for any findings — don't fix-and-forget inline.

### Code Review
Per repo, read the source with fresh eyes looking for:
- Logic errors, off-by-one, unchecked errors
- Inconsistency with the patterns described in the project's `AGENTS.md`
- Dead code, unreachable branches, unused fields
- Missing context in confusing code (add a comment only if the WHY is genuinely non-obvious)
- Make sure documentation matches reality
- Double check documentation/comments

File issues for anything found. Label them `bug` or `chore`.

### Security Review
For each repo, check:
- **Input validation** — all API endpoints, file uploads, MQTT payloads validated before use
- **Auth gaps** — endpoints that should require auth but don't; auth bypass paths
- **Secrets** — no API keys, tokens, or passwords in source; `sample.env` only
- **SQL** — parameterized queries only, no string concatenation into SQL
- **CORS / rate limiting** — tr-engine's per-IP rate limiter and CORS origins configured correctly
- **Dependency versions** — check for known CVEs in Go modules (`govulncheck ./...`) and npm deps (`npm audit`)

File issues for any findings. Label them `security`. Flag critical issues immediately.

### Usability Review
Evaluate against `EXTERNAL.md` — are the things users actually want working well?
- Does the setup flow (Docker one-liner) still work end-to-end?
- Are error messages in the UI actionable?
- Do the docs (README, AGENTS.md, sample.env comments) reflect current behavior?
- Are there features described in EXTERNAL.md that are missing, incomplete, or broken?
- Is the auth-init flow (open / token / full mode detection) working correctly from a fresh browser?

File issues for gaps. Label them `ux` or `docs`.

### Issue Triage: Proposed Enhancements
After the above reviews, review `EXTERNAL.md` against the current feature set and file enhancement issues for anything clearly missing or substandard. Keep scope tight — one issue per discrete feature.

---

## Phase 4: Restart

After completing a review pass and filing issues, go back to **Phase 1**. Pull the newly created issues into the work queue and continue the loop.

---

## Review Findings — 2026-05-05

Static cross-project review findings. Per Phase 3, turn these into focused GitHub issues before fixing; keep each PR scoped to one item or tightly related cluster.

### Cross-Repo

- [x] Finish AGENTS.md migration: `tr-bot`, `tr-plugin-dvcf`, `tr-plugin-avcf`, `tr-stack`, `tr-docker`, and `symbolstream` have no guidance file at all. `p25_audio_quality` and `projectTRanscribe` have only `CLAUDE.md` and need migration or an explicit archive notice.
- [x] Remove stale `CLAUDE.md` files from repos that have completed migration: `tr-engine` and `tr-update-worker` both have both `AGENTS.md` and `CLAUDE.md` coexisting; the old `CLAUDE.md` should be deleted once content is confirmed merged.
- [x] Add repository hygiene policy and `.gitignore` coverage for local artifacts: model directories, checkpoints, `.venv`, `node_modules`, screenshots, backup tarballs, Playwright output, and downloaded Qwen model trees.
- [ ] Decide ownership/archive status for `p25_audio_quality` and `projectTRanscribe`; either migrate them into the active workflow or label them clearly as historical research.
- [ ] No CI is configured in any repo. GUIDELINES says "if CI is configured, a green push is the floor" but it never is. Add GitHub Actions workflows for at minimum: `bash build.sh && go vet ./...` on tr-engine, `npm run build && npm run lint` on tr-dashboard and tr-bot, on push to any branch.

### tr-engine

- [x] Update `sample.env` auth comments to match current `open` / `token` / `full` auth behavior; remove stale auto-generated `AUTH_TOKEN` and read-only `WRITE_TOKEN` language.
- [x] Remove or rewrite stale Caddy auth-injection guidance in `AGENTS.md` and docs now that `auth-init` handles guest read tokens.
- [x] Harden `web/playground.html` token handling: do not include bearer tokens in AI prompts or generated page text; use the `auth-init` flow or require manual local token entry in previews.
- [ ] Add `.gitignore` coverage and purge local artifacts from the repo directory: backup tarballs (`*.tar.gz`), screenshot PNGs, compiled debug binaries (`debug-receiver`, `*.exe`), and `debug-samples` directory are committed to the working tree.
- [ ] Add SSE observability: dropped events from identity-resolution failures (#17) and ring-buffer evictions (#11) are currently invisible in production. Emit a counter/log line for each discarded event and expose via a `/metrics` or `/api/v1/debug/sse` endpoint.

### tr-dashboard

- [ ] Replace token-in-query audio playback with an opaque blob URL or same-origin audio proxy; `getCallAudioUrl` currently exposes JWTs in URLs.
- [ ] Add browser smoke or regression coverage for `auth-init` open/token/full mode and authenticated audio playback behavior.

### tr-bot

- [ ] Fix off-topic persona cost before enabling it broadly: cap serialized history/token budget, bound image count, and test the context-size limit. The current handler serializes channel history into both persona prompts.
- [ ] Add an operational kill switch or default-off config for the off-topic persona handler.

### tr-plugin-dvcf

- [ ] Investigate low DVCF production ratio with instrumentation around `call_start`, `voice_codec_data`, and `call_end`; correlate skipped calls by `audio_type`, `codec_type`, and talkgroup.
- [ ] Add a DVCF smoke fixture that validates SSSP headers, `CALL_START` / `METADATA` / `CALL_END` ordering, and stale-call salvage behavior.

### tr-plugin-avcf

- [ ] Tighten the `analog_only` filter: empty `audio_type` currently appears able to pass; verify digital calls cannot be wrapped as AVCF when Trunk Recorder omits `audio_type`.
- [ ] Add an AVCF fixture or smoke test for wrapping a sample audio file and verifying metadata plus audio extraction.

### symbolstream

- [ ] Refactor stream socket ownership to RAII and clear global stream config on parse/restart; raw socket pointers and global vectors risk duplicate streams and leaks.
- [ ] Add receiver/parser tests for JSON and binary frame modes, including reconnect and truncated-payload behavior.

### imbe_asr

- [ ] Add upload size limits and sanitize 500 responses in `server/app.py`; the endpoint reads full uploads into memory and returns raw exception text.
- [ ] Audit `torch.load(..., weights_only=False)` call sites; document the trusted-checkpoint assumption or switch to safer loading where compatible.
- [ ] Split and pin runtime versus training requirements; current requirements are broad, mostly unpinned, and mix server dependencies with `wandb` / `awscli`.

### qwen3-asr-server

- [ ] Fix temp-file cleanup when `_ensure_wav` / ffmpeg conversion fails before the `try` / `finally`; the temporary uploaded file can leak.
- [ ] Add request upload size limits and reject unsupported media before reading full bodies into memory.
- [ ] Pin compatible runtime dependencies or add constraints; `torch`, `transformers`, `librosa`, and `qwen_asr` are mostly unpinned.

### tr-stack

- [ ] Update auth config and docs from `AUTH_ENABLED` / `WRITE_TOKEN` / Caddy injection to the current `open` / `token` / `full` model with `ADMIN_PASSWORD` and API keys.
- [ ] Make the default compose path CPU-friendly or split GPU support into a clear override; the default NVIDIA device reservation can break first-run on non-GPU hosts.
- [ ] Lock down Mosquitto/default port exposure: avoid anonymous MQTT on `0.0.0.0` by default or document LAN-only assumptions and auth setup.

### tr-docker

- [ ] Pin Trunk Recorder and plugin git refs via build args for reproducible images; the Dockerfile currently builds latest shallow clones.
- [ ] Update the README included-plugin list to mention `mqtt_avcf` and any current patches or versions.

### tr-update-worker

- [ ] Update `PRODUCTS` and README GitHub URLs from `LumenPrima/*` to `trunk-reporter/*`.
- [ ] Validate and cap the `days` query parameter on stats/dashboard endpoints to avoid expensive unbounded D1 queries.
- [ ] Consider fail-closed auth for stats/dashboard when `DASHBOARD_KEY` is unset, or make open analytics an explicit config flag.
- [ ] Add a build/typecheck script, such as `tsc --noEmit` or Wrangler type generation, so CI can validate worker code.
- [ ] Remove screenshot PNGs from repo root (`dashboard-*.png`, `tv-overview-*.png`); add to `.gitignore`. None of the above items have GitHub issues yet — file them before starting work.

### p25_vocoder

- [ ] Add or verify `.gitignore` and artifact policy for checkpoints, data, and samples; keep large training outputs out of source control.
- [ ] Add minimal README/`AGENTS.md` status explaining whether this is active research and how to run smoke inference.

### p25_audio_quality

- [ ] Migrate legacy `.claude` guidance to `AGENTS.md` or mark this project archived.
- [ ] Document vendored dependency/data layout and ignore generated comparison/test outputs.

### projectTRanscribe

- [ ] Add `AGENTS.md` / README orientation or an archive notice; the repo has many design docs but no active Codex guidance.
- [ ] Replace default runnable secrets such as `minioadmin` in examples with explicit dev-only notes and env placeholders.

---

## Subproject Priority Order

When no issue priority is specified, prefer working in this order:

1. **tr-engine** — the core; bugs here affect everything downstream
2. **tr-dashboard** — primary user-facing surface
3. **tr-plugin-dvcf / tr-plugin-avcf** — data capture; errors here cause silent data loss
4. **tr-bot** — support and community tooling
5. **qwen3-asr-server / imbe_asr** — ASR quality improvements
6. **tr-stack** — deployment convenience
7. **p25_vocoder / symbolstream** — research / infrastructure

---

## Rules for the Agent

- **No pushes without a passing build.** No exceptions.
- **No commits to `main` directly.** Always branch → PR → merge.
- **No production database changes** as a testing shortcut. Test migrations locally first.
- **File an issue before fixing anything found during a review.** The issue is the record; the fix is the resolution.
- **When in doubt about scope, file an issue instead of implementing.** Keep PRs focused.
- **Update `AGENTS.md` files when you learn something non-obvious** about a project that future sessions would need to know.
