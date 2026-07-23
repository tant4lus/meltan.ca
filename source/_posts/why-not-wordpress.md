---
title: Why This Blog Isn't on WordPress
date: 2026-07-23
tags:
  - devops
---

When I decided to build this site, the obvious default answer was WordPress. It's what most people reach for, and it works. I get it, I used to have a hosted Wordpress site but the management was time consuming. For a personal blog, it felt like way more than I needed — so I went with Hexo, a static site generator, deployed straight to S3 instead.

**Cost was the first thing I looked at.** Since this was coming out of my bank account, this matters a lot to me. WordPress needs somewhere to run PHP and a database, which in practice means an EC2 instance humming along 24/7 — plus storage, backups, and usually a managed database. A static site is just files. S3 charges for the storage and requests it actually serves, which for a low-traffic personal blog is pennies a month. There's no server sitting idle waiting for someone to read a post.

**No server also means no patching treadmill.** WordPress core, PHP, plugins, themes, the underlying OS — all of it needs regular updates, and falling behind is how sites get compromised. A static site has no database to inject, no plugin ecosystem to audit, no admin login for bots to hammer. It's a fully serverless setup — S3, CloudFront, and GitHub Actions runners are all managed compute I don't own or patch. The attack surface mostly isn't there.

**"Why not just write HTML," then?** A friend asked me this, and it's a fair question. The answer is theme portability. My posts are markdown — content only, no presentation baked in. Hexo (with the Hiker theme) handles rendering, layout, tagging, and pagination on top of that. If I want a different look next year, I swap the theme and every post updates automatically. If I'd hand-written HTML, changing the design would mean touching every single page.

**The deploy pipeline is the part I'm most pleased with.** Every PR gets a real staging preview, and merging to main is what ships to production. It's a GitOps workflow — git is the source of truth, and nothing goes live without a PR:

![Deploy pipeline: on every PR, GitHub Actions builds and syncs to a staging S3 bucket and comments the preview URL; on merge to main, it builds again, syncs to the production bucket, and invalidates CloudFront](/images/why-not-wordpress-pipeline.svg)

Staging and production run as separate GitHub Actions environments with their own scoped secrets, so a staging deploy has no path to production credentials. Production also sits behind CloudFront, serving pages from an edge location near the reader instead of a single S3 region — free CDN performance a comparable WordPress setup would need Varnish or a paid CDN in front of it to match.

Getting the WordPress equivalent — a second server, a second database, some way to preview changes safely — would roughly double the cost and the maintenance.

None of this required learning new tools for me. It's the same GitHub Actions, S3, and CloudFront I already work with, pointed at a personal project instead of a company's infrastructure. That's really the point: the "advantage" isn't just that it's cheaper, it's that it's cheaper *because* it plays to skills I already have, instead of asking me to become a WordPress admin on the side.

Is it missing things WordPress gives you out of the box, like comments or search? Sure — but those are a Giscus embed or an Algolia index away, not a reason to run a database for a blog that gets a few hundred visits a month. And honestly, I'm not losing sleep over skipping comments — if anything, having them would cost me more sleep. They're mostly just where the internet says things to strangers they'd never say to someone's face.
