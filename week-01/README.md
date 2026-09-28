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