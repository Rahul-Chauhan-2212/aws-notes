---
layout: default
title: Aurora
parent: Notes
nav_order: 12
---

# 🌌 Aurora

* [Intro](#intro)
* [Endpoints](#endpoints)
* [High Availability & Read Scaling](#high-availability--read-scaling)
* [Encryption & Network Security](#encryption--network-security)
* [Aurora Serverless](#aurora-serverless)
* [Aurora Multi-Master](#aurora-multi-master)
* [Aurora Global Database](#aurora-global-database)
* [Aurora Events](#aurora-events)
* [Security](#security)
* [Misc](#misc)

---

## Intro

- Regional Service (supports global databases)
- Supports Multi AZ
- AWS managed Relational DB cluster
- Preferred over RDS
- Auto-scaling (max 128TB)
- Up to 15 read replicas
- **Asynchronous Replication** (milliseconds)
- **Supports only MySQL & PostgreSQL**
- Cloud-optimized (5x performance improvement over MySQL on RDS, over 3x the performance of PostgreSQL on RDS)
- **Backtrack**: restore data at any point of time without taking backups

## Endpoints

- **Writer Endpoint** (Cluster Endpoint)
    - Always points to the master (can be used for read/write)
    - Each Aurora DB cluster has one cluster endpoint
- **Reader Endpoint**
    - Provides load-balancing for read replicas only (used to read only)
    - If the cluster has no read replica, it points to master (can be used to read/write)
    - Each Aurora DB cluster has one reader endpoint
- **Custom Endpoint**:
    - Load balance to a subset of replicas
    - Provides load-balanced based on criteria other than the read-only or read-write capability of the DB instances like instance class (ex, direct internal users to low-capacity instances and direct production traffic to high-capacity instances)

## High Availability & Read Scaling

- Self healing (if some data is corrupted, it will be automatically healed)
- Storage is striped across 100s of volumes (more resilient)
- **Automated failover**
    - A read replica is promoted as the new master in less than 30 seconds
    - Aurora flips the **CNAME** record for your DB Instance to point at the healthy replica
    - In case **no replica** is available, Aurora will attempt to **create a new DB Instance** in the **same AZ** as the original instance. This replacement of the original instance is done on a **best-effort basis** and may not succeed.
- Support for **Cross Region Replication**
- Aurora maintains 6 copies of your data across 3 AZ:
    - 4 copies out of 6 needed for writes (can still write if 1 AZ completely fails)
    - 3 copies out of 6 needed for reads

<aside>
💡 Each Read Replica is associated with a priority tier (0-15). In the event of a failover, Amazon Aurora will promote the Read Replica that has the highest priority (lowest tier). If two or more Aurora Replicas share the same tier, then Aurora promotes the replica that is largest in size. If two or more Aurora Replicas share the same priority and size, then Aurora promotes an arbitrary replica in the same promotion tier.

</aside>

## Encryption & Network Security

- Encryption at rest using KMS (same as RDS)
- Encryption in flight using SSL (same as RDS)
- You can’t SSH into Aurora instances (same as RDS)
- Network Security is managed using Security Groups (same as RDS)
- EC2 instances should access the DB using **IAM DB Auth** but they can also do it using credentials fetched from the parameter store (same as RDS)

## Aurora Serverless

- Optional
- Automated database instantiation and auto scaling based on usage
- Good for unpredictable workloads
- No capacity planning needed
- Pay per second

## Aurora Multi-Master

- Optional
- Every node (replica) in the cluster can read and write
- **Immediate failover for writes** (high availability in terms of write). If disabled and the master node fails, need to promote a Read Replica as the new master (will take some time).
- **Client needs to have multiple DB connections for failover**

## Aurora Global Database

- Entire database is replicated across regions to recover from region failure
- Designed for **globally distributed applications** with **low latency local reads** in each region
- **1 Primary Region** (read / write)
- **Up to 5 secondary (read-only) regions** (replication lag < 1 second)
- Up to 16 Read Replicas per secondary region
- Helps for decreasing latency for clients in other geographical locations
- **RTO of less than 1 minute** (to promote another region as primary)

## Aurora Events

- Invoke a **Lambda** function from an **Aurora MySQL-compatible DB cluster** with a **native function** or a **stored procedure** (same as RDS events)
- Used to capture data changes whenever a row is modified

## Security

### At-rest Encryption

- The master and replicas can be encrypted as rest using KMS, but encryption must be enabled at launch time.
- If master is not encrypted, read replicas cannot be encrypted.
- To encrypt an unencrypted database, create a snapshot and restore the database as encrypted using a key from KMS.

### In-flight Encryption

Client needs to use TLS root certificates provided by AWS to establish TLS (in-flight encryption) with RDS or Aurora.

### DB Authentication

- Username:Password based authentication (available on both RDS and Aurora)
- **IAM DB Auth** - use IAM roles to authenticate to DB instead of configuring users on DB (recommended as it allows us to manage all the authentication in IAM)

### Network Security

- Security Groups control network access to the underlying EC2 instances
- No SSH access available as the underlying instances are AWS managed (except on RDS custom)

### Audit Logs

- Audit logs can be enabled - store logs locally for a short period of time
- Send audit logs to CloudWatch for log retention

## Misc

- Aurora Replicas are created in the same DB cluster within a Region. With **Aurora MySQL** you can also enable **binlog replication** to another Aurora DB cluster which can be in the **same or different region**.
- Important parameters:
    - `max_connections` - max number of simultaneous connections Aurora allows
    - `max_user_connections` - max number of simultaneous connections Aurora allows for a single user
