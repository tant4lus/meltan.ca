---
title: Giving Staging the Same CloudFront Setup as Production
date: 2026-07-30
categories:
  - Tech
tags:
  - devops
  - aws
  - cloudfront
  - terraform
---

`stage.meltan.ca` has been quietly different from production since the day I set it up: a plain S3 bucket in static-website mode, publicly readable, served straight off the S3 website endpoint. Production, meanwhile, has had a CloudFront distribution in front of a private bucket for years. Two environments, two different serving stacks, both labeled "the site."

That gap is exactly what bit me two posts ago — a 403 that only showed up in production because staging's S3-website endpoint resolves directory paths on its own, and production's CloudFront-plus-private-bucket setup doesn't, unless something tells it to. I fixed that one CloudFront Function at a time. The actual root cause was to stop letting staging and production be two different systems that happen to look similar, so staging now gets a CloudFront distribution too, built from the same Terraform module production uses.

**I built a module because the two CDNs should look the same.** 
 — same cache policy
 - same price class
 - same function logic
 - same copy of the function.

**On the cost side, this should be close to free — and production's own bill backs that up.** I had always assumed it was going to cost something. I admit that I didn't validate it. When I pulled Cost Explorer data for the account's CloudFront usage from January through today, every line item — data transfer, HTTP and HTTPS requests across all three price-class regions, CloudFront Function executions, even the 14 cache invalidations I've triggered along the way — comes back at *$0.00*. Not "rounds to zero," actually zero, across tens of thousands of requests a month. That's production's bill specifically. I have no reason to think that staging would cost more than production would.

- CloudFront's free tier — 1 TB of data transfer and 10 million HTTP/HTTPS requests a month — stopped being a new-account promo back in 2021 and is just a standing part of the service now.
- Data transferred from an S3 origin into CloudFront doesn't incur S3 data-transfer charges at all — that hop is free. 
- Route 53 alias records pointed at AWS resources like a CloudFront distribution aren't billed per query, unlike ordinary records — and I was already paying for the `meltan.ca` hosted zone itself either way.
- The module defaults to `PriceClass_100`, which limits the distribution to North America and Europe edge locations — the cheapest price class CloudFront offers, and plenty for a personal blog with no real global audience.

It goes to show you that one should really look at the data and facts before deciding that staging doesn't deserve the same setup as production.
