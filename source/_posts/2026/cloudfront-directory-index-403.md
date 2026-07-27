---
title: The 403 That Only Showed Up in Production
date: 2026-07-24
categories:
  - Tech
tags:
  - devops
  - aws
  - cloudfront
---

I shipped what should have been the most boring change possible: replacing the About page's one-line placeholder with real content. I knew there were going to be problems as I had done the bare minimum testing for this. I only had one post written and was only testing my pipeline, but not the content. Staging looked fine on the PR preview. I merged, production deployed, and `https://meltan.ca/about/` came back with a 403.

**The first clue was that it wasn't universal.** The homepage loaded fine. Every other path — `/about/`, `/archives/`, even a blog post URL — came back `AccessDenied`. Staging, which I'd just checked, had no problem at all. So it wasn't the page content, and it wasn't a general outage. Something about production specifically didn't know how to serve a directory. This goes to show you what happens when your production setup doesn't quite match your staging setup. But it was a cost trade-off I was willing to take for a personal blog, but not a mission-critical application. Since I knew my whole setup and this is not complex, it was easy to troubleshoot quickly.

**Reading the response headers instead of just the status code got me there fast.**

```
HTTP/2 403
content-type: application/xml
server: AmazonS3
x-cache: Error from cloudfront
```

`server: AmazonS3` on a response that came through CloudFront meant CloudFront wasn't rejecting the request itself — it was faithfully passing along a rejection from S3. And the body confirmed it: `<Error><Code>AccessDenied</Code></Error>`, the exact response S3 gives when you ask for an object key that doesn't exist and you don't have list permissions to say otherwise.

**That pointed straight at how the origin was configured.** Pulling the distribution config showed the origin was `<bucket>.s3.<region>.amazonaws.com` — the S3 REST API endpoint, accessed through an Origin Access Identity — with `DefaultRootObject: index.html`. That setting only applies to the literal root path `/`. It doesn't get extended to subdirectories. A request for `/about/` asks S3 for the object key `about/`, which doesn't exist — only `about/index.html` does — so S3 says access denied and CloudFront relays it verbatim. Staging worked because it's served from S3's static-website-hosting endpoint directly, which resolves directory index documents on its own; production never went through that layer at all.

**There were two ways to fix it, and they trade off differently.** Point CloudFront at the S3 website-hosting endpoint instead of the REST endpoint, and directory indexing comes for free — but that endpoint doesn't support origin access identities, so the bucket has to become public-read to work at all. Or keep the REST origin and the private bucket, and add a CloudFront Function that rewrites the request URI before it ever reaches S3. I went with the function — the entire point of putting a static site behind CloudFront with an OAI is that the bucket itself stays private, and I didn't want to give that up to fix a routing quirk.

The function itself is small — it runs at the viewer-request stage, before the cache lookup:

```js
function handler(event) {
    var request = event.request;
    var uri = request.uri;

    if (uri.endsWith('/')) {
        request.uri += 'index.html';
    } else if (!uri.includes('.')) {
        request.uri += '/index.html';
    }

    return request;
}
```

Paths ending in `/` get `index.html` appended. Paths with no file extension (someone requesting `/about` instead of `/about/`) get `/index.html` appended. Anything that looks like an actual file — `.css`, `.js`, `.svg`, `.xml` — is left alone. This is the standard pattern AWS documents for exactly this S3-plus-OAI setup, and it's a handful of lines running at the edge instead of a second copy of the bucket policy to maintain.

If this is a problem you face, then here is your fix. Every value that matters — both ETags, the function ARN — comes from parsing the previous command's own output. Nothing gets typed in by hand, nothing gets fat-fingered from a terminal scrollback, and the whole sequence can be re-run or scripted without a human in the loop copying values between windows.

```bash
# 1. Create the function and capture its ETag + ARN directly from the response
cat > append-index-html.js << 'EOF'
function handler(event) {
    var request = event.request;
    var uri = request.uri;

    if (uri.endsWith('/')) {
        request.uri += 'index.html';
    } else if (!uri.includes('.')) {
        request.uri += '/index.html';
    }

    return request;
}
EOF

CREATE_OUTPUT=$(aws cloudfront create-function \
  --name append-index-html \
  --function-config Comment="Append index.html for directory-style requests",Runtime="cloudfront-js-2.0" \
  --function-code fileb://append-index-html.js)

FUNCTION_ETAG=$(echo "$CREATE_OUTPUT" | jq -r '.ETag')
FUNCTION_ARN=$(echo "$CREATE_OUTPUT" | jq -r '.FunctionSummary.FunctionMetadata.FunctionARN')

# 2. Publish it
aws cloudfront publish-function --name append-index-html --if-match "$FUNCTION_ETAG"

# 3. Pull the distribution config and attach the function to the default
#    cache behavior's viewer-request event, without hand-editing the JSON
DIST_ID="<your-distribution-id>"

aws cloudfront get-distribution-config --id "$DIST_ID" > dist-config.json
DIST_ETAG=$(jq -r '.ETag' dist-config.json)

jq --arg arn "$FUNCTION_ARN" '
  .DistributionConfig
  | .DefaultCacheBehavior.FunctionAssociations = {
      "Quantity": 1,
      "Items": [{"FunctionARN": $arn, "EventType": "viewer-request"}]
    }
' dist-config.json > dist-config-updated.json

# 4. Apply the update and invalidate the cache so old 403s don't linger
aws cloudfront update-distribution --id "$DIST_ID" --if-match "$DIST_ETAG" --distribution-config file://dist-config-updated.json
aws cloudfront create-invalidation --distribution-id "$DIST_ID" --paths "/*"
```

**Fixing that surfaced a second, unrelated problem.** `/tags/` and `/categories/` were also 403ing, and for a second I assumed it was the same CloudFront issue. It wasn't — after the function was live, those two paths still failed, because the pages genuinely didn't exist in the build output. Hexo generates a page per tag (`tags/devops/index.html`) but not a landing page listing all of them, unless you explicitly add one. The fix was two small files — `source/tags/index.md` and `source/categories/index.md`, each with a `layout` front-matter field matching the theme's `tags.ejs`/`categories.ejs` templates — the same kind of gap as the About page stub that started this whole investigation.

Two bugs, same afternoon, same root cause category: testing that had never actually gone past the homepage to check that hexo ran. A future me problem will be to figure out how to add testing into my pipeline for a blog.
