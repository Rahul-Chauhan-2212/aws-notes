---
layout: default
title: VPC
parent: Notes
nav_order: 1
---

# 🌐 VPC (Virtual Private Cloud)

* [Intro](#intro)
* [Subnets](#subnets)
* [Internet Gateway (IGW)](#internet-gateway-igw)
* [NAT Instance](#network-address-translation-nat-instance)
* [NAT Gateway](#nat-gateway)
* [NACL](#network-access-control-list-nacl)
* [NACL vs Security Group](#nacl-vs-security-group)
* [VPC Peering](#vpc-peering)
* [VPC Endpoints](#vpc-endpoints)
* [VPC Flow Logs](#vpc-flow-logs)
* [Connecting to On-Premise](#connecting-to-on-premise)

---

## Intro

- **Regional resource**
- Soft limit of 5 VPCs per region
- Only the Private IPv4 ranges are allowed

## Subnets

- Sub-ranges of IP addresses within the VPC
- **Each subnet is bound to an AZ**
- Subnets in a VPC cannot have overlapping CIDRs
- **Default VPC only has public subnets** (1 public subnet per AZ, no private subnet)
- **AWS reserves 5 IP addresses (first 4 & last 1) in each subnet**. These 5 IP addresses are not available for use.
Example: if CIDR block 10.0.0.0/24, then reserved IP addresses are 10.0.0.0, 10.0.0.1, 10.0.0.2, 10.0.0.3 & 10.0.0.255

<aside>
💡 To make the EC2 instances running in private subnets accessible on the internet, place them behind an internet-facing (running in public subnets) Elastic Load Balancer.

</aside>

<aside>
💡 There is no concept of Public and Private subnets. Public subnets are subnets that have:

- “Auto-assign public IPv4 address” set to “Yes”
- The subnet route table has an attached Internet Gateway

This allows the resources within the subnet to make requests that go to the public internet. **A subnet is private by default.**

Since the resources in a private subnet don't have public IPs, they need a NAT gateway for address translation to be able to make requests that go to the public internet. NAT gateway also prevents these private resources from being accessed from the internet.

</aside>


## Internet Gateway (IGW)

<img width="1534" height="766" alt="image" src="https://github.com/user-attachments/assets/cb5d0052-9fb8-4417-8209-c9f4df9c9e9e" />


- Allows resources in a VPC to connect to the Internet
- **Attached to the VPC** (not subnets)
- Should be used to **connect public resources to the internet** (use NAT gateway for private resources since they need network address translation)
- Route table of the public subnets must be edited to allow requests destined outside the VPC to be routed to the IGW
    
    <img width="1862" height="328" alt="image" src="https://github.com/user-attachments/assets/1d6b1ff4-ec49-40bd-8bf4-4cb2e4157589" />

<aside>
💡 IGW performs network address translation (NAT) for a public EC2 instance
</aside>
