# BP-001: Least-privilege IAM role for the transaction-processor Lambda

**Date:** 2026-09-15
**Priority:** High
**Sprint:** 1
**Type:** Feature
**Tags:** #aws #iam #terraform #least-privilege #pci

## Context

Blanco Pata is standing up a new event-driven microservice, **`bp-txn-processor`**,
a Lambda function that ingests raw card-transaction batches, decrypts them, does
some light enrichment, and writes the processed output back to S3. Because this
service touches cardholder data, it falls inside the **PCI DSS** scope — which
means the principle of least privilege (PCI DSS Req. 7) is not a nice-to-have,
it's an audit line item.

Right now there is no IAM role for this function. A teammate's first instinct
was "just attach `AmazonS3FullAccess` and `AWSLambdaBasicExecutionRole` and move
on." We are **not** doing that. Your job is to author the role and its permissions
properly, from scratch, in Terraform, so that this becomes the reference example
other engineers copy.

### Infrastructure context (the real ARNs you're working against)

- **Region:** `mx-central-1`
- **Account ID:** `210987654321`
- **Function name:** `bp-txn-processor`
- **Raw input bucket:** `bp-txn-raw-prod`
  - The function reads objects **only** from the `incoming/` prefix.
- **Processed output bucket:** `bp-txn-processed-prod`
  - The function writes objects **only** to the `enriched/` prefix.
- **KMS key** used to encrypt/decrypt objects in both buckets:
  `arn:aws:kms:mx-central-1:210987654321:key/8b0c1e42-9f7a-4d3e-a1b2-6f5c4d3e2a10`
- The function logs to CloudWatch Logs in the standard location for a Lambda
  of this name.

Assume the buckets and the KMS key already exist and are managed by another
Terraform stack — you are **not** creating them, only referencing them.

## Task

Write the Terraform that defines:

1. An **IAM role** the Lambda can assume (correct trust/assume-role policy).
2. A **permissions policy** attached to that role granting the function exactly
   what it needs to do its job — and nothing more:
   - read the raw transaction objects it processes,
   - write the enriched output objects,
   - use the KMS key for the crypto operations those S3 reads/writes require,
   - write its own execution logs to CloudWatch.

Do not use any AWS-managed policies. Every permission must be one you wrote and
can justify.

## Expected result

Committed under `tickets/BP-001-iam-lambda-least-privilege/`:

- One or more `.tf` files defining the role and policy (name them sensibly, e.g.
  `iam.tf`, plus `variables.tf` / `providers.tf` if you choose to parameterise).
- **`SOLUTION.md`** — your report, in your own words:
  - a short statement per policy statement explaining *why* those actions and
    *why* that resource scope,
  - which specific actions you deliberately left out and why,
  - the output of an IaC scan of your code (see criteria) pasted in, with a note
    on anything it flagged and how you responded.

## Acceptance criteria

- [ ] The role's trust policy allows **only** the Lambda service to assume it —
      no broader principal.
- [ ] No AWS-managed policies are attached; all permissions are customer-authored.
- [ ] No `Action: "*"` and no `Action: "s3:*"` / `kms:*` — actions are enumerated
      to the specific operations required.
- [ ] `Resource` is scoped to the exact ARNs involved. In particular, the S3
      object-level permissions are scoped down to the correct **prefixes**
      (`incoming/*` for read, `enriched/*` for write), not the whole bucket, and
      you can articulate why a bucket-level ARN and an object-level ARN are two
      different things here.
- [ ] Read permissions and write permissions are not collapsed into one
      permissive statement covering both buckets — separate the concerns.
- [ ] KMS permissions grant only what an S3 client actually needs for
      encrypt/decrypt of these objects, scoped to the one key ARN — not `kms:*`,
      not all keys.
- [ ] CloudWatch Logs permissions are scoped to this function's log group ARN,
      not `*`.
- [ ] The Terraform is valid: `terraform validate` passes, and you've run
      `terraform fmt`.
- [ ] You've scanned the code with **Checkov** *or* **tfsec** and included the
      result in `SOLUTION.md`. A clean run is not required — a flagged finding
      you can explain and defend is fine, an unexplained one is not.

---

### Notes / guard-rails for this one

- You may hardcode the ARNs above directly, or parameterise them via variables —
  either is acceptable at this stage; just be consistent.
- If you find yourself unsure whether a particular S3 API call needs a
  bucket-level ARN or an object-level ARN, that uncertainty is exactly the thing
  worth resolving before you submit — the AWS docs on S3 actions and resource
  types are the source of truth, not guesswork.
- Don't over-build. No modules, no remote state, no CI wiring yet — that's later
  sprints. Just clean, correct, defensible IAM.
