# Hexo Blog Starter

This repository is a starter template for a static blog powered by [Hexo](https://hexo.io/) and deployed to an AWS S3 bucket using GitHub Actions. It features:

- ✍️ Markdown-based blogging
- 🖼️ Minimalist photo-friendly theme (`hexo-theme-cactus`)
- 🏷️ Tag support for topics like #travel, #devops, #hiking, and #diving
- ☁️ Continuous deployment to S3 on every push to `main`

---

## 🚀 How It Works

- Posts are written in Markdown inside `source/_posts/`
- Tags are automatically parsed from post front-matter
- GitHub Actions installs Hexo, builds the static site, and syncs `public/` to your S3 bucket
- No server or backend required

---

## 🧪 Local Development

To preview your blog locally before pushing:

```bash
git clone https://github.com/your-username/your-blog.git
cd your-blog
npm install
npx hexo server
```

Visit http://localhost:4000 in your browser.

---

## 🚚 Deployment to S3 (via GitHub Actions)

Add these GitHub Secrets to your repository:

| Secret Name               | Description                        |
|---------------------------|------------------------------------|
| `AWS_ACCESS_KEY_ID`       | Your AWS access key                |
| `AWS_SECRET_ACCESS_KEY`   | Your AWS secret key                |
| `AWS_REGION`              | (optional) AWS region (default: `us-west-2`) |

Update the bucket name in `.github/workflows/deploy.yml`:
```yaml
aws s3 sync public/ s3://YOUR_BUCKET_NAME --delete
```

---

## 📄 License

This template is open-source and provided for personal or commercial use.
