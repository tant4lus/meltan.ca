# meltan.ca

Personal blog built with [Hexo](https://hexo.io/), theme [Hiker](https://github.com/iTimeTraveler/hexo-theme-hiker), deployed to S3.

## Stack

- Hexo 7, theme `hexo-theme-hiker` (installed from GitHub via `github:iTimeTraveler/hexo-theme-hiker` in package.json — it isn't published to npm)
- Theme settings live in `_config.hiker.yml` at the repo root (not in `node_modules/hexo-theme-hiker/`, since that isn't tracked)
- Posts: `source/_posts/*.md`. Static pages (e.g. About): `source/<slug>/index.md`

### Non-npm dependencies

Everything in `package.json` is handled by `npm install`/`npm ci` — no need to document those individually. What's not npm-managed:

- **Node 24** — matches the version CI uses (`.github/workflows/deploy.yml`). There's no `.nvmrc` or `engines` field enforcing this locally, so mismatches are possible.

## Deployment

GitHub Actions (`.github/workflows/deploy.yml`), region `us-west-2`:
- PR against `main` → builds and syncs to `s3://meltan.ca-staging`, comments the staging URL on the PR
- Push to `main` → builds and syncs to `s3://meltan.ca`
- `AWS_ACCESS_KEY_ID` and `AWS_REGION` are GitHub Actions **vars**; `AWS_SECRET_ACCESS_KEY` is a **secret** (both under the `AWS` environment)

Local preview: `npx hexo server` (http://localhost:4000). Build: `npx hexo generate` (outputs to `public/`).

## Commit messages

One commit per unrelated concern — don't fold two unrelated changes (e.g. a theme change and a docs update) into a single commit just because they happened in the same session. If a commit message needs "and" to describe it, it's probably two commits.

Format: `type: short summary`, colon-separated, imperative mood. Types: `feat`, `fix`, `chore`, `docs`, `refactor`.

```
feat: switch theme from landscape to Hiker
fix: read AWS_ACCESS_KEY_ID from vars not secrets
docs: update README for Hiker theme
```

## Pull requests

- Branch off `main`, open a PR back to `main` — this is what triggers the staging deploy, so it's the way to preview changes before they go live
- PR description: a `## Summary` of what changed and why, plus a `## Test plan` checklist (what was verified locally, e.g. `hexo generate` succeeds, pages checked in browser)
- Check the bot-posted staging URL comment before merging
