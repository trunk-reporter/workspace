# Trunk Reporter — Autonomous Agent Work Cycle

All repos live at https://github.com/orgs/trunk-reporter/repositories.
Cross-repo roadmap: https://github.com/orgs/trunk-reporter/projects/1

---

## Before Starting Any Session

Read these files in order — they are your operating context:

1. `CLAUDE.md` — architecture, subprojects, data flow, deployed instance
2. `GUIDELINES.md` — branching, testing, deployment, secrets rules
3. `EXTERNAL.md` — who the users are and what they actually want
4. The CLAUDE.md inside whichever subproject you're about to touch

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
- Read the relevant subproject's CLAUDE.md before touching code
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
- Update the relevant CLAUDE.md if architecture, build steps, or conventions change

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

If the fix warrants a deploy to `gerty`, follow the deploy steps in `tr-engine/CLAUDE.md`.

---

## Phase 3: Review Passes

When the issue queue is empty (or between issue batches), run structured reviews. Each review produces new GitHub issues for any findings — don't fix-and-forget inline.

### Code Review
Per repo, read the source with fresh eyes looking for:
- Logic errors, off-by-one, unchecked errors
- Inconsistency with the patterns described in the project's CLAUDE.md
- Dead code, unreachable branches, unused fields
- Missing context in confusing code (add a comment only if the WHY is genuinely non-obvious)

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
- Do the docs (README, CLAUDE.md, sample.env comments) reflect current behavior?
- Are there features described in EXTERNAL.md that are missing, incomplete, or broken?
- Is the auth-init flow (open / token / full mode detection) working correctly from a fresh browser?

File issues for gaps. Label them `ux` or `docs`.

### Issue Triage: Proposed Enhancements
After the above reviews, review `EXTERNAL.md` against the current feature set and file enhancement issues for anything clearly missing or substandard. Keep scope tight — one issue per discrete feature.

---

## Phase 4: Restart

After completing a review pass and filing issues, go back to **Phase 1**. Pull the newly created issues into the work queue and continue the loop.

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
- **Update `CLAUDE.md` files when you learn something non-obvious** about a project that future sessions would need to know.
