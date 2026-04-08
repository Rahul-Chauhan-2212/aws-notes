---
layout: default
title: Lambda
parent: Notes
nav_order: 2
---

# ƛ Lambda

* [Intro](#intro)
* [Performance & Limits](#performance--limits)
* [Cold Start](#cold-start)
* [Access Control](#access-control)
* [Synchronous Invocation](#synchronous-invocation)
* [Asynchronous Invocation](#asynchronous-invocation)
* [Event Source Mapping](#event-source-mapping)
* [Streams](#streams)
* [Queues](#queues)
* [Integration with ALB](#integration-with-alb)
* [Event and Context Objects](#event-and-context-objects)
* [Destinations](#destinations)
* [Environment Variables](#environment-variables)
* [Observability](#observability)
* [Lambda@Edge](#lambdaedge)
* [Networking](#networking)
* [Execution Context](#execution-context)
* [Storage Options](#storage-options)
* [Concurrency](#concurrency)
* [Deployment](#deployment)
* [Lambda Container Images](#lambda-container-images)
* [Versions & Aliases](#versions--aliases)
* [Function URL](#function-url)

---

## Intro

- Function as a Service (FaaS)
- Serverless
- Auto-scaling
- Pay per request (number of invocations) and compute time
- Free tier: 1,000,000 requests and 400,000 GBs of compute time
- Supported runtime
    - NodeJS
    - Python
    - Java
    - C#
    - Golang
    - Ruby
    - Any other language using Custom Runtime API (example Rust)
    - Docker containers

## Performance & Limits

- RAM: 128MB (default) - 10 GB (1 MB increments)
- Disk capacity (`/tmp`): 512 MB (free) - 10 GB
- Cannot configure vCPU count directly, increase RAM to increase vCPU count
    - RAM = 1,792 MB ⇒ vCPU = 1
    - For vCPU > 1, need to use multi-threading to take advantage of multiple cores
- Timeout: default 3s, max 15 mins (900s)
- Environment variables: max 4 KB
- Deployment
    - Compressed: max 50 MB
    - Uncompressed: max 250 MB (for more use `/tmp`)
 
## Cold Start

- To minimize partial cold start time, minimize the time taken to execute the code outside the lambda handler by:
    - Reducing the deployment package size
    - Increasing computing power (by increasing the allocated memory)
      <img width="1194" height="434" alt="image" src="https://github.com/user-attachments/assets/0bfc6481-a35d-44c9-831c-ef3f59151ed0" />
## Access Control

- IAM permissions to control API access to Lambda
- Resource-based policies to allow other services to invoke the lambda function

## Synchronous Invocation

- The caller invokes the lambda function and waits for it to return the result
- Error handling (retries, exponential backoff, etc) must be done by the caller
- Invoking a lambda function using CLI or SDK results in synchronous invocations by default. To invoke explicitly, set `invocation-type` to `RequestResponse`
- Some services that invoke the lambda function synchronously
    - Invoked by user
        - ALB
        - API Gateway
        - CloudFront (Lambda@Edge)
        - S3 Batch
    - Invoked by service
        - Cognito
        - Step Functions
        - **SQS** (through event source mapping)
