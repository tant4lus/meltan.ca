# meltan.ca

Personal blog built with [Hexo](https://hexo.io/), theme [Hiker](https://github.com/iTimeTraveler/hexo-theme-hiker), deployed to S3.

## Stack

- Hexo 7, theme `hexo-theme-hiker` (installed from GitHub via `github:iTimeTraveler/hexo-theme-hiker` in package.json — it isn't published to npm)
- Theme settings live in `_config.hiker.yml` at the repo root (not in `node_modules/hexo-theme-hiker/`, since that isn't tracked)
- Posts: `source/_posts/<year>/*.md` — one folder per year, since keeping everything flat in `_posts/` doesn't scale. If a year folder ever gets unwieldy, split further into `<year>/<month>/` then. Static pages (e.g. About): `source/<slug>/index.md`

### Categories and tags

Every post should set one category (broad section) and one or more tags (specific topics):

- **Categories**: `Tech` (DevOps/infrastructure/cloud), `Adventures` (backpacking, bike touring, skiing, kayaking), `Crafting` (sewing, watercolour)
- **Tags**: freeform within a category, but reuse existing ones where they fit — e.g. `devops`, `aws`, `ci-cd`, `terraform`, `github-actions` for Tech; `backpacking`, `bike-touring`, `skiing`, `kayaking`, `csia` for Adventures; `sewing`, `watercolour` for Crafting. Check `/tags/` and `/categories/` for what already exists before inventing a new one.

### Permalinks and post filing

Permalinks (`_config.yml`'s `permalink: :year/:month/:day/:title/`) are date- and slug-driven, not path-driven — but the `:title` token isn't read from a `slug` front-matter field. This Hexo version ignores `slug:` in front matter entirely; it unconditionally recomputes the slug from the post's source path relative to `_posts/`, parsed against `new_post_name`. That's why `new_post_name` is set to `:year/:title.md` — it matches the `<year>/<title>.md` layout above, so the year folder gets parsed out instead of folded into the slug. If a post's path doesn't match that pattern (not inside a year folder, or nested one level deeper than expected), the leftover path segments leak into the slug and produce a wrong or doubled URL — verify with `npx hexo server` and check the actual rendered link, don't assume front matter alone controls it. `hexo new` already respects `new_post_name`, so posts created that way land in the right year folder automatically.

### Non-npm dependencies

Everything in `package.json` is handled by `npm install`/`npm ci` — no need to document those individually. What's not npm-managed:

- **Node 24** — matches the version CI uses (`.github/workflows/deploy.yml`). There's no `.nvmrc` or `engines` field enforcing this locally, so mismatches are possible.

## Deployment

GitHub Actions (`.github/workflows/deploy.yml`), region `us-west-2`:
- PR against `main` → builds and syncs to the staging bucket (`meltan.ca-staging`), comments the staging URL on the PR
- Push to `main` → builds and syncs to the production bucket (`meltan.ca`)
- Bucket names are read from the `S3_BUCKET` var (not hardcoded in the workflow) — `meltan.ca-staging` in the `stage` environment, `meltan.ca` in `prod`
- `AWS_ACCESS_KEY_ID` and `AWS_REGION` are GitHub Actions **vars**; `AWS_SECRET_ACCESS_KEY` is a **secret** — `deploy-staging` reads these from the `stage` environment, `deploy-production` from the `prod` environment
- `deploy-production` also invalidates CloudFront after the S3 sync (distribution ID read from the `CLOUDFRONT_DISTRIBUTION_ID` var in the `prod` environment, not hardcoded in the repo), since production sits behind CloudFront and staging doesn't

Resource identifiers (bucket names, distribution IDs, etc.) belong in GitHub Actions vars, not hardcoded in workflows or code — see #20.

Local preview: `npx hexo server` (http://localhost:4000). Build: `npx hexo generate` (outputs to `public/`).

## Writing posts

- After drafting or editing post content, let Melissa review the wording before committing it — don't commit prose on her behalf until she's confirmed it's ready. This is distinct from code/config changes, which don't need this extra review gate.
- Proofread drafts for typos/spelling before committing, even when the wording is hers verbatim — flag anything that looks like an error and confirm before changing it, rather than silently rewriting her voice.
- After drafting, run `npx hexo generate` and `npx hexo server`, then open the post's local preview URL (http://localhost:4000/...) in her actual browser (e.g. via `open <url>` on macOS, or navigating a real Chrome tab) so she can see it rendered before it goes to a staging PR.

## Commit messages

One commit per unrelated concern — don't fold two unrelated changes (e.g. a theme change and a docs update) into a single commit just because they happened in the same session. If a commit message needs "and" to describe it, it's probably two commits.

Format: `type: short summary`, colon-separated, imperative mood. Types: `feat`, `fix`, `chore`, `docs`, `refactor`.

```
feat: switch theme from landscape to Hiker
fix: read AWS_ACCESS_KEY_ID from vars not secrets
docs: update README for Hiker theme
```

## Pull requests

- Branch off `main`, open a PR back to `main` — this is what triggers the staging deploy, so it's the way to preview changes before they go live. Never commit directly to `main`.
- PR description: a `## Summary` of what changed and why, plus a `## Test plan` checklist (what was verified locally, e.g. `hexo generate` succeeds, pages checked in browser)
- Check the bot-posted staging URL comment before merging
- The repo has "Automatically delete head branches" enabled, so merged branches are cleaned up by GitHub automatically — no manual `git branch -d`/`git push origin --delete` needed after merging
