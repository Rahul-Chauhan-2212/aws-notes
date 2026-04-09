---
layout: default
title: VPC
parent: Notes
nav_order: 4
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


## Network Address Translation (NAT) Instance

<img width="1528" height="754" alt="image" src="https://github.com/user-attachments/assets/89efecd9-a776-4b7b-a8ae-1c291e91016a" />


- An EC2 instance **launched in the public subnet** which performs network address translation to enable private instances to use the public IP of the NAT instance to access the internet. This is exactly the same as how routers perform NAT. This also prevents the private instances from being accessed from the public internet.
- **Must disable EC2 setting: source / destination IP check on the NAT instance** as the IPs can change.
- **Must have an Elastic IP attached to it**
- Route Tables for private subnets must be configured to route internet-destined traffic to the NAT instance (its elastic IP)
- **Can be used as a Bastion Host**

<img width="634" height="824" alt="image" src="https://github.com/user-attachments/assets/23d918ff-1151-4c00-9525-5d82a2b7157a" />


### **Disadvantages**

- Not highly available or resilient out of the box. Need to create an ASG in multi-AZ + resilient user-data script
- Internet traffic bandwidth depends on EC2 instance type
- You must manage Security Groups & rules:
    - Inbound:
        - Allow HTTP / HTTPS traffic coming from Private Subnets
        - Allow SSH from your home network (access is provided through Internet Gateway)
    - Outbound:
        - Allow HTTP / HTTPS traffic to the Internet

## NAT Gateway

<img width="1530" height="766" alt="image" src="https://github.com/user-attachments/assets/1f4695c1-9ba9-4d91-9b8c-e99e077eb412" />


- AWS managed NAT with **bandwidth autoscaling** (up to 45Gbps)
- Preferred over NAT instances
- **Uses an Elastic IP** and Internet Gateway behind the scenes
- **Created in a public subnet**
- **Bound to an AZ**
- **Cannot be used by EC2 instances in the same subnet** (only from other subnets)
- **Cannot be used as a Bastion Host**
- Route Tables for private subnets must be configured to route internet-destined traffic to the NAT gateway
    
    <img width="1058" height="238" alt="image" src="https://github.com/user-attachments/assets/e50fe244-fe19-4d42-9b1c-335907e76464" />

    
- No Security Groups to manage
- Pay per hour

### High Availability

- Create NAT gateways in public subnets bound to different AZ all routing outbound connections to the IGW (attached to the VPC)
- No cross-AZ failover needed because if an AZ goes down, all of the instances in that AZ also go down.

<img width="890" height="832" alt="image" src="https://github.com/user-attachments/assets/c34e5d89-aeac-4679-8666-68b271e5614c" />


## Network Access Control List (NACL)

- NACL is a firewall at the subnet level
- One NACL per subnet but a NACL can be attached to multiple subnets
- **New subnets are assigned the Default NACL**
- **Default NACL allows all inbound & outbound requests**
    
    <img width="1024" height="437" alt="image" src="https://github.com/user-attachments/assets/635597eb-1948-4401-a15a-cc1ad0a718e6" />

    
- NACL Rules
    - Based only on IP addresses
    - Rules number: 1-32766 (lower number has higher precedence)
    - First rule match will drive the decision
    - The last rule denies the request (only when no previous rule matches)

## NACL vs Security Group
<img width="1748" height="716" alt="image" src="https://github.com/user-attachments/assets/b582bffb-f5fd-466b-91f5-f76795819b3a" />
| Feature    | Security Group                    | NACL                                       |
| ---------- | --------------------------------- | ------------------------------------------ |
| Scope      | Firewall for EC2 (applied to ENI) | Firewall for subnets                       |
| Rules      | Supports only Allow rules         | Supports both Allow and Deny rules         |
| Behavior   | Stateful (only request evaluated) | Stateless (request and response evaluated) |
| Evaluation | All rules are evaluated           | Only the first matched rule is considered  |


## VPC Peering

<img width="1532" height="754" alt="image" src="https://github.com/user-attachments/assets/3e497c2e-be5a-4f55-8128-ea28f3ab39a6" />


- Connect two VPCs (could be in **different regions or accounts**) using the AWS private network
- Participating VPCs must have **non-overlapping CIDR**
- VPC Peering connection is **non-transitive** (A - B, B - C != A - C) because it works based on route-table rules.
- Must update route tables in each VPC’s subnets to ensure requests destined to the peered VPC can be routed through the peering connection
    
    <img width="1570" height="274" alt="image" src="https://github.com/user-attachments/assets/ec5b0d0d-8104-4d63-9bf2-7898497591b3" />

    
    <img width="1556" height="274" alt="image" src="https://github.com/user-attachments/assets/84fca2a7-0b14-46f1-8f6d-313548ca367d" />

    
- You can reference a security group in a peered VPC across account or region. This allows us to use SG instead of CIDR when configuring rules.


## VPC Endpoints

<img width="1652" height="788" alt="image" src="https://github.com/user-attachments/assets/089bc7e4-682f-4c30-ab71-173a48f20777" />


- **Private endpoints** within your VPC that allow AWS services to privately connect to resources within your VPC without traversing the public internet (cheaper)
- Powered by **AWS PrivateLink**
- **Route table is updated automatically**
- **Bound to a region** (do not support inter-region communication)
- Need to create separate endpoints for different resources
- Two types:
    - **Interface Endpoint**
        - Provisions an **ENI** (private IP) as an entry point per subnet
        - Need to **attach a security group to the interface endpoint** to control access
        - Supports most AWS services
    - **Gateway Endpoint**
        - Provisions a gateway
        - Must be used as a target in a route table
        - Supports only **S3** and **DynamoDB**

## VPC Flow Logs

- Captures information about **IP traffic** going into your **interfaces**
- Three levels:
    - **VPC** Flow Logs
    - **Subnet** Flow Logs
    - **ENI** Flow Logs
- Can be configured to show accepted, rejected or all traffic
- Flow logs data can be sent to **S3** (bulk analytics) or **CloudWatch Logs** (near real-time decision making)
- Query VPC flow logs using **Athena** in S3 or **CloudWatch Logs Insights**

## Connecting to On-Premise

- **Site-to-site VPN** - **IPSec encrypted** connection over the public internet
- **Direct Connect (DX)** - Dedicated connection
    - Not encrypted (but private connection)
    - Takes about 30 days to setup
