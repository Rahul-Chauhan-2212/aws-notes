---
layout: default
title: Route 53
parent: Notes
nav_order: 6
---

# 🗺️ Route 53

* [Intro](#intro)
* [Hosted Zone](#hosted-zone)
* [Record Types](#record-types)
* [Routing Policies](#routing-policies)
* [Health Checks](#health-checks)
* [GoDaddy with Route 53](#godaddy-with-route-53)
* [DNS Resolution in Hybrid Cloud](#dns-resolution-in-hybrid-cloud)
* [Misc](#misc)

---

## Intro

- Global Service
- AWS managed **Authoritative DNS** (customer can update the DNS records and have full control over the DNS)
- Also a Domain Registrar (for registering domain names)
- Only AWS service which provides 100% availability SLA
- Affected by client's DNS caching (not suitable for Blue-Green Deployment if the client caches DNS queries)

<aside>
💡

It is recommended to use DNS names or URLs instead of IPs wherever possible

</aside>

## Hosted Zone

- A container for DNS records that define how to route traffic to a domain and its subdomains.
- Hosted zone is queried to get the IP address from the hostname
- Two types:
    - **Public Hosted Zone**
        - resolves public domain names
        - can be queried by anyone on the internet
    - **Private Hosted Zone**
        - resolves private domain names
        - can only be queried from within the VPC

## Record Types

- **A** - maps a hostname to IPv4
- **AAAA** - maps a hostname to IPv6
- **CNAME** - maps a hostname to another hostname
    - The target is a domain name which must have an A or AAAA record
    - **Cannot point to root domains (Zone Apex)**
    Ex: you can’t create a CNAME record for `example.com`, but you can create for `something.example.com`
- **NS** (Name Servers) - controls how traffic is routed for a domain
- **Alias** - maps a hostname to an AWS resource
    - **AWS proprietary**
    - Can point to root (zone apex) and non-root domains
    - **Alias Record is of type A or AAAA** (IPv4 / IPv6)
    - Targets can be
        - Elastic Load Balancers
        - CloudFront Distributions
        - API Gateway
        - Elastic Beanstalk environments
        - S3 Websites
        - VPC Interface Endpoints
        - Global Accelerator
        - Route 53 record in the same hosted zone
    - **Target cannot be EC2**

## Routing Policies

### Simple

- Route to one or more resources
- If multiple values are returned, client chooses one at random (client-side load balancing)
    
    <img width="740" height="842" alt="image" src="https://github.com/user-attachments/assets/e108fcbb-ad2b-4100-8d18-12b12e091ee4" />

    
- **No health check** (if returning multiple resources, some of them might be unhealthy)

### Weighted

- Route a fraction of request to multiple resources
- Health checks
- Use case: testing a new application version by sending a small amount of traffic
- Can be used for **Active-Active failover** strategy

### Latency-based

- Redirect to the resource that has the lowest latency
- Health checks
- Can be used for **Active-Active failover** strategy

### Failover

- Primary & Secondary Records (if the primary application is down, route to secondary application)
- Health check must be associated with the primary record
- Used for **Active-Passive failover** strategy

### Geolocation

- Routing based on the client's location
- Should create a “Default” record (in case there’s no match on location)
- Use cases: restrict content distribution & language preference

### Geoproximity

- Route traffic to your resources based on the proximity of clients to the resources
- Ability to shift more traffic to resources based on the defined bias.
    - To expand (bias: 1 to 99) → more traffic to the resource
    - To shrink (bias: -1 to-99) → less traffic to the resource
- Resources can be:
    - AWS resources (specify AWS region)
    - Non-AWS resources (specify Latitude and Longitude)
- Uses **Route 53 Traffic Flow**

### Multi-value

- Route traffic to multiple resources (max 8)
- Health Checks (only healthy resources will be returned)

## Health Checks

- HTTP Health Checks are only for public resources
- Allows for Automated DNS Failover
- Three types:
    - **Monitor an endpoint** (application or other AWS resource)
        - Multiple **global health checkers** check the endpoint health
        - Must configure the application firewall to allow incoming requests from the IPs of Route 53 Health Checkers
        - Supported protocols: HTTP, HTTPS and TCP
    - **Monitor other health checks** (Calculated Health Checks)
        - Combine the results of multiple Health Checks into one (AND, OR, NOT)
        - Specify how many of the health checks need to pass to make the parent pass
        - Usage: perform maintenance to your website without causing all health checks to fail
    - **Monitor CloudWatch Alarms** (to perform health check on private resources)
        - Route 53 health checkers are outside the VPC. They can’t access private endpoints  in VPC or on-premises resources.
        - Create a CloudWatch Metric and associate a CloudWatch Alarm to it, then create a Health Check that checks the CW alarm.

## GoDaddy with Route 53

**Use GoDaddy as registrar and Route 53 as DNS**: Once we register a hostname at GoDaddy, we need to update the name servers (NS) of GoDaddy to match the name servers of a public hosted zone created in Route 53. This way, GoDaddy will use Route 53’s DNS.

## DNS Resolution in Hybrid Cloud

- To resolve DNS queries for resources in the VPC from the on-premises network, create an inbound endpoint on Route 53 Resolver and then DNS resolvers on the on-premises network can forward DNS queries to Route 53 Resolver via this endpoint.
- To resolve DNS queries for resources in the on-premises network from the VPC, create an outbound endpoint on Route 53 Resolver and then Route 53 Resolver can conditionally forward queries to resolvers on the on-premises network via this endpoint. To conditionally forward queries, create Resolver rules that specify the domain names for the DNS queries that you want to forward (such as [example.com](http://example.com/)) and the IP addresses of the DNS resolvers on the on-premises network that you want to forward the queries to.

## Misc

- **Failure Threshold** is the parameter used by Route 53 health checks to determine if an endpoint is healthy. A failure occurs if an endpoint does not respond to a request.
- Route 53 can be used as a DNS to register a domain name, route the internet traffic, and perform health checks on resources. If being used for all three tasks, the order of setup must be sequential as above.
