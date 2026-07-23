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

| Trigger | Bucket |
|---|---|
| Pull request → `main` | `meltan.ca-staging` (deploy comment posted on the PR) |
| Push to `main` | `meltan.ca` (production) |

Both jobs run under the `AWS` GitHub environment, which needs:

| Name | Kind | Description |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | Variable | AWS access key |
| `AWS_REGION` | Variable | AWS region (`us-west-2`) |
| `AWS_SECRET_ACCESS_KEY` | Secret | AWS secret key |

---

## License

Personal project — not intended as a reusable template.
