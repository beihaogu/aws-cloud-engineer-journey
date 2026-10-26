# Week 4 — Storage Decision Guide

| Service | What it is | Use this when… |
|---|---|---|
| **S3** | Object storage (files + metadata behind an HTTP API) | You have files — images, logs, backups, static sites, data exports. Not for anything that needs to be mounted as a disk or queried like a table. |
| **EBS** | Block storage — a network disk attached to one EC2 instance | The instance needs a real filesystem: OS root volume, database data files, anything that expects a disk. One instance at a time. |
| **EFS** | Managed NFS — a shared filesystem many instances mount at once | Several instances need the same files simultaneously (shared uploads, home directories, CMS content). Costs more per GB than EBS. |
| **RDS** | Managed relational database (MySQL/Postgres/etc.) on a VM you size | Data is structured with relations, you need JOINs, transactions, ad-hoc SQL. You accept managing instance size, connections, and VPC placement. |
| **DynamoDB** | Managed NoSQL key-value/document store, serverless | Access pattern is known up front and is mostly "get/put by key" at any scale, with no servers to manage. Pick this for Lambda-backed APIs; avoid when you need flexible queries or JOINs. |

**PropertyLite's path:** reads a CSV today → DynamoDB for the single-listing
lookup behind API Gateway/Lambda (Week 7) → RDS for the full dataset in the
capstone (Week 12).