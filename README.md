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

# Week 3 — Runbook: Deploying PropertyLite to EC2

Region: us-east-2 (account is pinned here; handbook uses us-east-1).

## 1. Launch the instance

EC2 → Launch instance, with:

- **Name:** `property-api-01`
- **AMI:** Amazon Linux 2023 (free-tier eligible)
- **Instance type:** `t3.micro`
- **Key pair:** created `training-key` (RSA, `.pem`), moved to `~/.ssh/` and
  `chmod 400` so SSH will accept it. AWS keeps only the public key; the `.pem`
  is the private half and cannot be re-downloaded.
- **Security group:** two inbound rules —
  - SSH, port 22, source = My IP
  - Custom TCP, port 8080, source = 0.0.0.0/0 (PropertyLite listens on 8080,
    not 80)
- **User data:** the bootstrap script in `user-data.sh` (see below)

## 2. Bootstrap script (user data)

EC2 runs user data once, as root, on first boot. The script installs Python
and Flask, writes `app.py` and the sample CSV to `/opt/property-api/`, and
starts the app in the background with `nohup`, logging to
`/var/log/property-api.log`. Nothing is installed by hand — the instance is
self-contained from the moment it boots, which is what makes the same script
reusable in a launch template (Week 6).

## 3. Verify from the laptop

Waited ~3 minutes for status checks to pass, then from my machine:

```bash
curl http://<public-ip>:8080/health
# {"status":"ok"}

curl http://<public-ip>:8080/properties/R100234
# {"L_City":"Sacramento","L_Keyword2":"3", ...}
```

Screenshot: `curl-responses.png`.

Why this works: the app binds to `0.0.0.0:8080`, the security group allows
8080 from anywhere, and the instance has a public IPv4. Remove any one of the
three and the request times out.

## 4. SSH in

```bash
ssh -i ~/.ssh/training-key.pem ec2-user@<public-ip>
cat /var/log/property-api.log
```

Confirmed Flask was serving on 8080 and the two curl requests appeared in the
log.

## 5. EBS snapshot (Lab 3.2)

EC2 → Volumes → selected the instance's 8 GB root volume → Create snapshot.
Status reached **completed** within a couple of minutes
(`snapshot-completed.png`).

A snapshot is a point-in-time copy of the volume stored in S3. Creating a new
volume from it would give a disk identical to this one at the moment the
snapshot was taken — the same mechanism used for backup/restore in Week 12.

## 6. Cleanup

- Terminated `property-api-01` (Terminate, not Stop — a stopped instance still
  bills for its volume)
- Confirmed the root volume was deleted with the instance
- Deleted the snapshot

## Terms

- **EC2** — a virtual machine.
- **EBS volume** — a network-attached disk; the instance's root disk is one.
- **Snapshot** — a backup of a volume at one point in time.
- **Security group** — the instance's stateful firewall (inbound rules only
  needed; replies are allowed automatically).
- **User data** — a script EC2 runs as root on first boot.


# Week 4 — Storage & Databases

Region: us-east-2.

## Lab 4.1 — S3 + lifecycle
Created `my-training-bucket-beihao`, uploaded a file, added a rule moving
all objects to Glacier Flexible Retrieval after 30 days
(`lifecycle-rule.png`).

Also used this bucket to re-test the Week 2 least-privilege user. The first
`aws s3 cp --profile s3test` failed with AccessDenied: the policy's Resource
still contained the literal placeholder `my-training-bucket-<beihao>/*`
from copying the handbook template. Fixed the ARN, and PutObject succeeded
while ListBuckets is still denied — the policy now behaves exactly as
intended. A single wrong character in an ARN is a silent full deny.

## Lab 4.2 — RDS MySQL
Created `training-db` (MySQL 8.0, free tier, not publicly accessible).
Because the instance has no public endpoint, I launched a `db-client` EC2
instance in the same VPC, added an inbound rule on the RDS security group
allowing 3306 from the EC2 instance's security group (the Debug 4.4 fix),
and connected from there with the mysql client.

`rets_property.sql` was not distributed yet, so I created the
`rets_property` table by hand with the Week 0 sample rows and ran the
aggregate query (`rds-query.png`). The PII-drop and ID-remap steps will be
applied when the real export is provided. Deleted the instance after the
lab.

## Lab 4.3 — DynamoDB
Created the `Properties` table with `L_ListingID` as the partition key
(on-demand billing), put one item, and read it back with `get-item`
(`dynamodb-get-item.png`). Deleted the table after the lab.

## Decision guide
See [`decision-guide.md`](decision-guide.md).