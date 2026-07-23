# meltan.ca

Personal blog powered by [Hexo](https://hexo.io/), using the [Hiker](https://github.com/iTimeTraveler/hexo-theme-hiker) theme, deployed to AWS S3 via GitHub Actions.

- Markdown-based blogging, posts live in `source/_posts/`
- Tag support (e.g. #travel, #devops, #hiking, #diving)
- Continuous deployment: pull requests deploy to a staging bucket, merges to `main` deploy to production

---

## Local development

```bash
git clone https://github.com/tant4lus/meltan.ca.git
cd meltan.ca
npm install
npx hexo server
```

Visit http://localhost:4000.

To build the static site without serving it:

```bash
npx hexo generate
```

Output goes to `public/`.

---

## Theme

The site uses [Hiker](https://github.com/iTimeTraveler/hexo-theme-hiker) (`hexo-theme-hiker`), installed directly from GitHub since it isn't published to npm. Theme configuration lives in `_config.hiker.yml` at the repo root — see the [theme's README](https://github.com/iTimeTraveler/hexo-theme-hiker#readme) for the full list of options (homepage background, sidebar widgets, code highlight style, comment systems, etc).

---

## Deployment (GitHub Actions → S3)

`.github/workflows/deploy.yml` runs on every PR and push to `main`, region `us-west-2`:

| Trigger | Environment | Bucket |
|---|---|---|
| Pull request → `main` | `stage` | `meltan.ca-staging` (deploy comment posted on the PR) |
| Push to `main` | `prod` | `meltan.ca` (production) |

Bucket names are stored as the `S3_BUCKET` var per environment, not hardcoded in the workflow. Each environment needs:

| Name | Kind | `stage` | `prod` |
|---|---|---|---|
| `AWS_ACCESS_KEY_ID` | Variable | ✓ | ✓ |
| `AWS_REGION` | Variable | ✓ | ✓ |
| `S3_BUCKET` | Variable | ✓ | ✓ |
| `AWS_SECRET_ACCESS_KEY` | Secret | ✓ | ✓ |
| `CLOUDFRONT_DISTRIBUTION_ID` | Variable | | ✓ |

Resource identifiers should always be stored as vars, never hardcoded in workflows or code — see [#20](https://github.com/tant4lus/meltan.ca/issues/20).

---

## License

Personal project — not intended as a reusable template.
