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

## Asynchronous Invocation

- The caller just invokes the lambda function and does not wait for the result. Can send many events and let lambda process them.
- The event is put in an event queue from which the lambda function reads
- Invoking a lambda function asynchronously returns **status code 202 (lambda function invoked)**. We don’t know if the processing was successful or not.
- For asynchronous invocation, set `invocation-type` to `Event`

<img width="768" height="620" alt="image" src="https://github.com/user-attachments/assets/6de3f6ca-3bc6-48f4-a284-de2bf2e175bf" />

- Error handling is done by the lambda function
    - Retries: 0 - 2
    - Exponential Backoff: 1 min (first retry), 2 min (second retry)
    - Can setup DLQ (SQS queue or an SNS topic) **on the lambda function** to send failed events after retries. Need to provide IAM permissions to the lambda to write to DLQ.
    - Retries lead to duplicate logs in CW
    - Make sure the event processing is **idempotent** (not affected by retries)
- Some services that invoke the lambda function asynchronously
    - S3 (S3 notifications)
    - **SNS**
    - EventBridge (events in event buses)
    - CodeCommit (Code Commit Trigger: new branch, new tag, new push)
    - CodePipeline (invoke a Lambda function during the pipeline)
 
## Event Source Mapping

- **Lambda needs to poll the service to get records for processing** (required when the following services are configured as the trigger for lambda function)
    - Kinesis Data Streams (KDS)
    - DynamoDB Streams
    - SQS queues (Standard and FIFO)
- When we configure the lambda function to work with the above services, an event source mapping is created internally which repeatedly polls the service to get a batch of data and **invoke the lambda function synchronously with an event batch**.
- Need to attach IAM permissions to the Lambda role to read from the appropriate service.

<img width="714" height="676" alt="image" src="https://github.com/user-attachments/assets/e6f330f3-464d-4ea6-8c60-7aa3df91517e" />

<aside>
💡 **Event Source Mapping is a process internal to Lambda.** It is managed by the Lambda service but is not directly run inside the Lambda function.

</aside>

There are two kinds of event source mappers:

### **Streams**

- Used for KDS and DynamoDB streams as Lambda trigger
- The event source mapping creates an iterator for each shard. Items within a shard will be processed in order but items across shards might not be processed in order.
- The iterator can be configured where to read from:
    - Trim Horizon (beginning of the shard)
    - At timestamp
    - Latest
- **Processed records aren’t removed from the stream** and can be viewed by other consumers.
- For low traffic streams, process the records in batches for each shard using batch window (time to wait for the batch to fill up)
    - 1 lambda per shard
    - In order processing for each shard
- For high traffic streams, process multiple batches (partitioned using partition key) in parallel at the shard level
    - Max 10 batches per shard (10 lambdas per shard)
    - In order processing for each partition key
      <img width="980" height="298" alt="image" src="https://github.com/user-attachments/assets/9bbca22b-1b11-4ff1-9e2b-17eff18b35df" />
- Error Handling
    - **If an error occurs while processing a batch, the entire batch is reprocessed until the processing succeeds or items in the batch expire.**
    - Processing for the affected shard is paused until the error is resolved (to ensure that batches are processed in order).
    - Event source mapping can be configured to:
        - Discard old events (send to a **destination**)
        - Restrict the number of retries
        - Split the batch on error (process the batch partially) (work around Lambda timeout issue)

### Queues

- Used for SQS as Lambda trigger
- Event source mapping uses **long polling** to poll the queue for messages.
- **Batch size**: 1 - 10 messages
- Recommended to set message visibility timeout to 6x lambda function timeout
- FIFO Queues
    - In-order processing
    - Lambda scales to the number of group IDs
- Standard Queues
    - Processing not necessarily in order
    - Lambda scales up to process items in the queue as quickly as possible (adds 60 more instances per minute)
    - Up to 1000 batches of messages processed simultaneously

<img width="534" height="674" alt="image" src="https://github.com/user-attachments/assets/28b205c7-4fe6-4c31-9045-d41912ee70ea" />


- **When an error occurs, items within the batch are returned to the queue** and might be processed in a different grouping than the original batch.
- The event source mapping might receive the same item multiple times, even if no function error occurred. So the processing should be **idempotent**.
- **Lambda deletes messages from the queue after they're processed successfully.**
- **DLQ needs to be set on the queue, not the Lambda function** because DLQ for Lambda is only for asynchronous invocations.
- Lambda destination can be used for failures

## Integration with ALB

<img width="1024" height="298" alt="image" src="https://github.com/user-attachments/assets/306019b2-9f4a-46ce-a6b2-f65034b54aa5" />


- ALB is used to expose a lambda function as an HTTP/HTTPS endpoint
- Lambda function must be registered in a target group
- ALB converts the HTTP request into a JSON event to invoke the lambda synchronously. It also converts the response from Lambda back into HTTP response to return to the client.
- Multi-value headers support in ALB
    
    If **multi-value header setting in the target group** is enabled, **HTTP headers** and **query string parameters** with the same key but different values are converted to arrays in the JSON event.
    
    <img width="778" height="736" alt="image" src="https://github.com/user-attachments/assets/c0f8af40-4fff-44c8-8ca5-7677a164c8a8" />

