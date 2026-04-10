---
layout: default
title: EFS
parent: Notes
nav_order: 10
---

# 📂 EFS (Elastic File System)

* [Intro](#intro)
* [Performance Mode](#performance-mode)
* [Throughput Mode](#throughput-mode)
* [Storage Tiers](#storage-tiers)
* [Security](#security)

## Intro

- AWS managed Network File System (NFS)
- Can be mounted to multiple EC2 instances **across AZs**
- **Pay per use** (no capacity provisioning)
- **Auto scaling** (up to PBs)
- Compatible with **Linux** based AMIs (**POSIX** file system)

## Performance Mode

- **General Purpose** (default)
    - latency-sensitive use cases (web server, CMS, etc.)
- **Max I/O**
    - higher latency & throughput (big data, media processing)

## Throughput Mode

- **Bursting** (default)
    - Throughput: 50MB/s per TB
    - Burst of up to 100MB/s.
- **Provisioned**
    - Fixed throughput (provisioned)

## Storage Tiers

- **Standard** - for frequently accessed files
- **Infrequent access (EFS-IA)** - cost to retrieve files, lower price to store
- Lifecycle management feature to move files to **EFS-IA** after N days

## Security

- **Security Groups** to control network traffic
- **POSIX Permissions** to control access from hosts by IAM User or Group
