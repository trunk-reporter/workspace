# Development Guidelines

## Version Control

All work happens through Git. Every change — no matter how small — lives in a commit on a branch in the appropriate GitHub repo under the [trunk-reporter](https://github.com/orgs/trunk-reporter/repositories) org.

- Work on a feature branch, not directly on `main`
- Commit often with descriptive messages; don't batch unrelated changes into one commit
- Open a PR to merge into `main`; squash or rebase before merging to keep history readable
- `main` should always be in a deployable state

## Testing Before Pushing

**Never push untested code.** Before any `git push`:

1. Build the project locally (`bash build.sh` for tr-engine, `npm run build` for Node projects)
2. Run type checks / linters (`npm run lint` for TypeScript projects)
3. Smoke-test the changed behavior — at minimum hit the relevant endpoint or exercise the code path
4. For tr-engine: `curl http://localhost:8080/api/v1/health` and exercise affected endpoints

If CI is configured, a green push is the floor — not the ceiling.

## Local Machine Testing

The local machine (and the live `gerty` instance via Tailnet) is available for real-data testing, but it is **not a staging environment**. Rules:

- Test locally to verify your change works against real radio data
- Do not leave experimental or debug builds running on `gerty` long-term
- Do not make schema or data changes on the production database as a testing shortcut — test migrations locally first
- Restore `gerty` to a clean state (latest `main` image or binary) once you're done with a test session

## Deployment

Deployments go through the deploy scripts — not manual file drops:

- `tr-engine/deploy-dev.sh` — cross-compiles and deploys to `gerty`
- `docker compose up -d` on `gerty` for container-based services
- Always use `docker compose up -d <service>` (not `restart`) when env vars change — `restart` does not re-read `.env`

Only deploy from a clean, tested commit on `main` (or a release tag).

## Secrets and Configuration

- `.env` files are gitignored — never commit them
- Credentials, API keys, and tokens go in `.env` only
- Use `sample.env` (committed) to document all available config fields
- The `AUTH_TOKEN`, `ADMIN_PASSWORD`, and `JWT_SECRET` in production are managed in `/docker/tr-engine/.env` on `gerty`

## Repository Structure

Each subproject lives in its own directory (and its own GitHub repo). When making cross-cutting changes:

- Update `openapi.yaml` in tr-engine whenever the API contract changes — it is the source of truth
- Update the relevant `CLAUDE.md` when architecture, conventions, or build steps change
- Keep the root `CLAUDE.md` and `GUIDELINES.md` current as the org evolves
