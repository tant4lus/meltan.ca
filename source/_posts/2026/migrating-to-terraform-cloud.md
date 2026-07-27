---
title: Importing My Hand-Built AWS Setup Into Terraform Without Breaking It
date: 2026-07-27
categories:
  - Tech
tags:
  - devops
  - aws
  - terraform
---

Everything this blog runs on — the S3 buckets, the CloudFront distributions, the ACM certs, the Route 53 records — I had started off by hand in the AWS console back in 2022 when the concept came to me as I was working my AWS Certified Solutions Architect exam and applying it to my own personal tech stack. That was fine for getting the site up quickly and figuring out how I wanted to set things up, but none of it was documented in notes or by IaC. Now that I have some free time, I split it into a companion repo, `meltan.ca-iac`, and spent a few Claude sessions turning all of it into Terraform, wired up to Terraform Cloud.

**Writing Terraform for infrastructure that's already live is a different exercise than writing it for something new.** At the start, AWS was the source of truth instead of the IaC. Get a new resource wrong and `apply` fails, or creates the wrong thing, which you can destroy and reapply. Get an *existing* resource wrong and `apply` can decide the safest thing to do is destroy what's there and recreate it to match your — possibly wrong — HCL. This was a live site and I didn't want to throw away anything I created. So the rule I held myself to was: write the resource block to match reality, `terraform import` it, run `plan`, and don't move on until it says zero changes. If the diff isn't empty, the HCL is wrong, not AWS. After all the work, I know that the IaC and the HCL are the source of truth going forward.

Importing the S3 buckets, `plan` kept turning up differences one attribute at a time — ACLs, versioning, lifecycle rules — none of which I'd written into the config at all. It turns out the AWS provider's base `aws_s3_bucket` resource reads a whole set of legacy sub-attributes on every refresh, whether or not you manage them separately, and if the IAM role doing the reading can't see one of them, Terraform doesn't error — it just concludes that attribute doesn't exist and quietly plans to change it. The sharpest version of this: the bucket-existence check calls `HeadBucket`, which AWS authorizes via `s3:ListBucket` rather than any dedicated action. Without that one permission, a bucket that was sitting there completely unchanged got read back as *deleted*, and Terraform planned to recreate the whole thing. Nothing was wrong with the bucket. The IAM policy just couldn't see it.

I created three environments: stage, prod, and global. I originally didn't expect to create `global` but realized it was needed to hold the account-wide stuff — the Route 53 zone, the mail MX records, the site-verification TXT — that doesn't belong to any one environment.

Importing the DNS zone had its own quiet trap. Route 53 stores TXT record values with the quote marks baked into the string — `"google-site-verification=..."`, literal quote characters and all — but Terraform's state, after import, holds the same value with the quotes stripped. Write the quotes into your HCL to "match" what you saw in the AWS console, and `plan` shows a change that doesn't exist. It's a small thing, but it's exactly the kind of small thing zero-diff-before-apply is built to catch, instead of the alternative — applying and quietly rewriting a DNS record you didn't mean to touch.

**Auth was the one place I upgraded rather than just replicated.** Rather than generate long-lived AWS access keys and paste them into Terraform Cloud, each workspace authenticates over OIDC — Terraform Cloud presents a signed token, AWS trusts it via an identity provider, and the workspace assumes an IAM role scoped to nothing but what it actually manages. The trust policy is scoped down to the workspace by name, so the staging workspace's role can't be assumed by anything claiming to be the production workspace:

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "<oidc-provider-arn>" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": { "app.terraform.io:aud": "aws.workload.identity" },
    "StringLike": {
      "app.terraform.io:sub": "organization:meltan:project:meltan-ca:workspace:meltan-ca-prod:run_phase:*"
    }
  }
}
```

No static credentials sitting in a secrets store waiting to leak — just a workspace proving who it is on every run.

**Put together, a PR against `meltan.ca-iac` triggers a fairly different chain of events than a PR against the blog itself.**

![Terraform Cloud workflow: on every pull request, Terraform Cloud runs a speculative plan by assuming a read-scoped IAM role in AWS over OIDC and posts the result as a PR check; on merge to main, it queues a real run, assumes the workspace's own scoped role over OIDC, applies the change against AWS, and stores the resulting state in Terraform Cloud](/images/terraform-cloud-oidc-flow.svg)

Every plan and apply assumes credentials fresh over OIDC and lets them expire — nothing long-lived is sitting anywhere waiting to be stolen. Next is to convert the GitHub credentials to work with OIDC as well, instead of my traditional access key ID and secret access key.

This blog is being built one step at a time. No huge project plan, just agile development as I discover things along the way. Each step, I am trying to discover what exactly brings me joy in my work. Or in the words of Marie Kondo, "does it spark joy?"