# AWS Cloud Engineer Journey

# Week 1 — Cloud Fundamentals & Account Setup

**Root user** is the identity created with the AWS account itself. It has
unrestricted access to everything, including billing and account closure, and
its permissions cannot be limited by policy. Because of that, root should only
be used for the handful of tasks that require it (enabling MFA, setting up
billing) and never for daily work.

**IAM users** are identities created inside the account. Each one only has the
permissions explicitly attached to it through policies, so a compromised IAM
user is a bounded problem, while a compromised root user is a total one. For
this program I created an admin IAM user for all console and CLI work and
authenticate the CLI with that user's access key, not root credentials.

The **Shared Responsibility Model** splits security between AWS and the
customer. AWS is responsible for security of the cloud: the physical data
centers, hardware, hypervisor, and managed-service infrastructure. The customer
is responsible for security in the cloud: IAM configuration, security groups,
data encryption, patching their own instances, and not leaking credentials.
Everything I misconfigure in this program is on my side of that line.

# Week 2 — IAM & Security Foundations

## Lab 2.1 — Least-privilege S3 uploader policy

**Policy:** [`policies/S3UploaderOnly-beihao.json`](policies/S3UploaderOnly-beihao.json)

**What it allows:** `s3:PutObject` and `s3:GetObject`, and only on objects
inside `my-training-bucket-beihao`. Nothing else — no listing buckets, no
deleting objects, no access to any other bucket or service.

**Why it is scoped this way:** the only job of `s3-test-user` is to upload
and read back files in one bucket. Granting it `s3:*` or `AmazonS3FullAccess`
would let a leaked key list, read and delete every bucket in the account.
Scoping the policy to exactly the two actions and the one resource it needs
means a compromised key is a bounded problem.

**Why the ARN ends in `/*`:** `arn:aws:s3:::bucket` and
`arn:aws:s3:::bucket/*` are two different resources. The first is the bucket
itself (used by `s3:ListBucket`); the second is the objects inside it (used by
`s3:GetObject` / `s3:PutObject`). Object-level actions on the bucket ARN
without `/*` would be denied — this is the root cause walked through in Debug
Lab 2.3.

**Verification:** `aws s3 ls --profile s3test` returns `AccessDenied`
(see `access-denied.png`). This is the expected result, not a bug: `ListAllMyBuckets`
is not in the policy, so IAM falls through to the implicit deny.

## Lab 2.2 — IAM Access Analyzer

The first attempt to create an analyzer failed with:

> `access-analyzer:CreateAnalyzer` ... *a service control policy explicitly
> denies the action* (see `access-analyzer-scp-denied.png`)

This account was created through AWS's new sign-up experience, which places
the account under an AWS-managed Organization with SCPs that restrict some
services. The session role (`AccountFullAccessRole`) has administrator-level
identity-based permissions, yet the request was still denied — a live example
of the evaluation order: **an explicit Deny at any layer (SCP, resource
policy, identity policy) always wins over any Allow.**

After activating advanced features (taking ownership of the management
account), the SCP no longer applied and the analyzer was created successfully.
It reports **0 active findings**, which is expected: the account contains no
resources shared with external principals.

## Concepts

- **Users / groups / roles.** A user is a long-lived identity with its own
  credentials; a group is a container that attaches policies to many users at
  once; a role is an identity with no permanent credentials that is *assumed*
  temporarily by a user, service or external account.
- **Identity-based vs resource-based policies.** Identity-based policies
  attach to a user/group/role and say what *it* can do. Resource-based
  policies (bucket policies, trust policies) attach to the resource and say
  *who* can act on it.
- **Managed vs inline.** Managed policies are standalone objects that can be
  attached to many identities and versioned; inline policies live inside one
  identity and die with it. `S3UploaderOnly-beihao` is a customer-managed
  policy.
- **Evaluation logic.** Default is implicit deny. An explicit Allow in an
  applicable policy grants access — unless any explicit Deny applies anywhere
  in the chain (SCP → resource policy → permissions boundary → identity
  policy), in which case the request is denied regardless.