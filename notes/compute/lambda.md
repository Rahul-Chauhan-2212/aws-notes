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

## Event and Context Objects
<img width="1754" height="536" alt="image" src="https://github.com/user-attachments/assets/b64e5ef5-7794-4bd6-82d1-dad67ceaa31b" />
<img width="1302" height="488" alt="image" src="https://github.com/user-attachments/assets/b2f35778-761d-4849-ab75-463d08a778d6" />

**Event Object**

- JSON formatted event (data for the function to process)
- Lambda runtime converts the event to an object based on the language (e.g. dict type in Python)

**Context Object**

- JSON document available to the function at runtime which provides metadata about the function, invocation and runtime environment
- Contains the name of log stream

## Destinations

- Configure the lambda function to send information regarding the invocations and results of **asynchronous invocations** or **synchronous invocations through event source mapping** (successful or failed) to a destination.
- Successful invocations are sent to destination immediately but failed invocations are retried before sending to the destination.
- Can send successful and failed invocations to different destinations.
- **Destination is recommended over DLQ** (as it supports more destinations)
- Destination for asynchronous invocations of Lambda function:
    - SQS
    - SNS
    - Lambda
    - Event bus
<img width="518" height="340" alt="image" src="https://github.com/user-attachments/assets/358931e9-df09-4020-9e99-aebb5b181c6b" />
<img width="518" height="354" alt="image" src="https://github.com/user-attachments/assets/fe7d1534-6184-4c1a-b87c-f0f90f3a4f54" />


- Destination for discarded event batches in **Stream type Event Source Mapping**:
    - SQS
    - SNS

 ## Access Control

### Lambda Execution Role

- IAM role for Lambda functions
- Best practice: one execution role per function
- When using event source mapping, the Lambda function needs IAM permissions in the execution role to poll another service for events.
- Commonly used managed policies for Lambda:
    - `AWSLambdaBasicExecutionRole` - Send logs to CloudWatch.
    - `AWSLambdaKinesisExecutionRole` - Read from Kinesis
    - `AWSLambdaDynamoDBExecutionRole` - Read from DynamoDB Streams
    - `AWSLambdaSQSQueueExecutionRole` - Read from SQS queues
    - `AWSLambdaVPCAccessExecutionRole` - Deploy Lambda function in VPC
    - `AWSXRayDaemonWriteAccess` - Upload trace data to X-Ray

### Resource-based Policy

- Used to allow other services to invoke the lambda function
- When a service like S3 triggers a lambda function, the resource-based policy for that lambda function gives it access.
- An IAM principal can access Lambda if the IAM policy attached to the principal authorizes it or the resource-based policy authorizes.

### Cross Account Access

You can give a Lambda function created in one account ("account A") permissions to assume a role from another account ("account B") to access resources such as DynamoDB or S3 bucket. You need to create an execution role in Account A. Then you need to create a role in account B that the Lambda function in account A assumes to gain access to the cross-account DynamoDB table. Make sure that you modify the trust policy of the role in Account B to allow the execution role of Lambda to assume this role. Finally, update the Lambda function code to add the `AssumeRole` API call.

## Environment Variables

- Environment variables (key-value pairs) for Lambda
- **No limit on the number of environment variables**
- **Max total size of all environment variables = 4 KB**
- Lambda adds some system environment variables by default
- Can be used to store parameters (config) and secrets (encrypt using Lambda key or CMK in KMS)
- **Lambda encrypts environment variables at rest by default.**
- **Encryption helpers:** encrypt environment variables client side, before sending them to Lambda. This enhances security further by preventing secrets from being displayed unencrypted in the Lambda console, or in function configuration.

## Observability

### Logging & Monitoring

- Lambda automatically sends logs to CW using `AWSLambdaBasicExecutionRole` policy
- Lambda automatically sends metrics to CW
    - Invocations, duration, concurrent execution
    - Error count, success rate, throttles
    - Async delivery failures
    - Iterator age (how much the lambda is lagging when reading from a stream)

### Tracing

- Enable **Active Tracing** in Lambda configuration (runs X-Ray daemon)
- Instrument your code with X-Ray SDK
- Ensure Lambda function has `AWSXRayDaemonWriteAccess` policy (automatically added when Active Tracing is enabled)
- Environment variables used by the X-Ray SDK (automatically configured)
    - `AWS_XRAY_DAEMON_ADDRESS`: the X-Ray Daemon ip-address:port
    - `_X_AMZN_TRACE_ID`: contains the tracing header
    - `AWS_XRAY_CONTEXT_MISSING`: by default, LOG_ERROR
 
## Lambda@Edge

- Run lambda function, written in **NodeJS or Python**, at edge locations by attaching them to CloudFront distributions
- Minimizes latency (globally available)
- Throughput: **thousands of requests/second**
- Can be used to modify Viewer and Origin Request and Response
    
    <img width="998" height="236" alt="image" src="https://github.com/user-attachments/assets/cca4e722-49e9-42ec-af43-5c8fb6ee2bd6" />

    
- Create functions in `us-east-1` ⇒ CloudFront replicates to the edge locations

### Lambda@Edge vs CloudFront Functions
<img width="1638" height="702" alt="image" src="https://github.com/user-attachments/assets/c7cd7e91-d2de-43dc-b019-ededcea2f261" />


## Networking

- By default, Lambda functions are launched outside your VPC (in an AWS owned VPC). So, it cannot access resources within your VPC.

- To deploy a lambda in your VPC
    - Configure the VPC ID and Subnets
    - Create a **security group for the lambda function**
- Deploying a Lambda function within a VPC internally creates ENIs within the subnets configured, with the lambda security group attached to them. To create ENIs, Lambda function needs `AWSLambdaENIManagementAccess` policy (need to configure manually).
- A lambda function deployed in a public subnet of a VPC does not have internet access or a public IP.

<img width="512" height="708" alt="image" src="https://github.com/user-attachments/assets/f0f0ad63-ceba-4a1a-940a-86290a342485" />


- To have internet access, deploy the lambda in a private subnet with a NAT gateway (in a public subnet). In this setting, AWS services outside the VPC can be accessed by the lambda function without traversing the public internet using VPC endpoints.

<aside>
💡 Irrespective of how Lambda is deployed, it will always be able to send logs to CloudWatch without any additional configuration.

</aside>

<img width="866" height="782" alt="image" src="https://github.com/user-attachments/assets/a673c74d-3f56-44bd-95a4-0737854d3df2" />


## Execution Context
<img width="1714" height="618" alt="image" src="https://github.com/user-attachments/assets/077b04b9-08c5-4db9-b850-bb9049e87d2e" />

- Temporary runtime environment that runs the code outside of the Lambda handler
- Used to initialize external dependencies of the lambda handler
    - Create DB connections
    - Start web servers
    - Initialize SDKs
- Execution context is maintained as long as that instance of lambda function lives in anticipation of another invocation. If the same lambda instance (if it is still alive) is invoked again, it will reuse the execution context to save time.
- `/tmp` directory is included in the execution context
    - can be used as a transient cache across invocations
    - free up to 512 MB, max 10 GB
- **Best practice**: anything that takes time should be moved outside of the event handler (into execution context)

## Storage Options

<img width="1618" height="780" alt="image" src="https://github.com/user-attachments/assets/adf6501e-e7b0-40ad-9e07-2294043e1568" />

### EFS Mounting

- **Lambda functions must be running within the VPC**
- Configure Lambda to mount EFS to a local directory during initialization
- **Must leverage EFS Access Points**
- Limitations: each lambda instance creates a connection with EFS (watch out for connection limits)
<img width="512" height="603" alt="image" src="https://github.com/user-attachments/assets/42122a65-f2af-460a-84bf-cd8133004592" />


## Concurrency

- Concurrency soft limit across account: 1000 (can be raised by opening a support ticket)
- **Reserved concurrency**: concurrency hard limit for a lambda function
    - Recommended to reserve (limit) concurrency for every lambda function.
    - With unreserved concurrency, a function can throttle other functions.
    - To throttle a function, set its reserved concurrency to 0
- Each request over the concurrency limit will throttle
    - Synchronous invocation → return **ThrottleError 429**
    - Asynchronous invocation → retry (up to 6 hours) with exponential backoff (1 sec - 5 min)
- **Provisioned concurrency**:  keep some lambda functions warm to prevent cold start
    - Specified number of lambda functions will be ready to serve requests at all times
    - **Application Auto Scaling** can control the provisioned concurrency on a schedule or based on a target metric.
- Required Concurrency = `invocations_per_sec` x `avg_execution_time`
- If the concurrency requirements are above the limit for the account, contact AWS to increase the quota.
- **Unreserved concurrency cannot go below 100**, so that functions that do not have reserved concurrency limits can still process requests. So, if your total account limit is 1000, you are limited to allocating 900 reserved concurrency to individual functions combined.

## Deployment

### Packaging Dependencies

- Code and dependencies need to be packaged into a zip file and uploaded directly to Lambda if less than 50 MB, otherwise upload to S3 and download to `/tmp`.
- Native libraries need to be compiled on Amazon Linux
- AWS SDK is installed by default on Lambda (no need to package it)


### Lambda Layers

<img width="1024" height="436" alt="image" src="https://github.com/user-attachments/assets/0a4a9e25-d564-42ba-96d1-84387c5f690d" />


- Used to **separate dependencies from the application code** to reuse them
- Allow the application package to be small
- Lambda layers can be referenced by multiple functions
- Additional feature: **support custom runtimes** (allows Lambda functions to run any programming language)
- Up to 5 layers per function (max 250 MB total)


### Deployment with CloudFormation

- Simple code (Python or NodeJS) **without dependencies** can be written directly in the CloudFormation template
    
    <img width="794" height="570" alt="image" src="https://github.com/user-attachments/assets/041ccbdd-0527-4529-a0eb-9bca83a5df7b" />

    
- Reference the zip file stored in S3
    - `S3ObjectVersion` is required only if the bucket is versioned (recommended)
    - CloudFormation will only update (redeploy) the function if either `S3Bucket` , `S3Key` or `S3ObjectVersion` changes in the template.
    
    <img width="794" height="494" alt="image" src="https://github.com/user-attachments/assets/5fad8f44-c156-4026-8a63-1fcb5e3d19d3" />

    
- Deploy a lambda function in multiple AWS accounts
    
    The zipped lambda code is in an S3 bucket in account 1. Bucket policy will be used in account 1 to allow accounts 2 and 3 to access the zipped lambda code. Also, the CloudFormation templates in account 2 and 3 need to have an execution role to allow reading the contents of the bucket at the desired location.

    <img width="1674" height="728" alt="image" src="https://github.com/user-attachments/assets/6c6601af-0cd5-4c36-8eb7-f1fc389de86f" />


### Traffic-shifting with CodeDeploy

- Automatically increases the percentage of traffic (x) going to the new version until 100%
- Integrated within the SAM framework
- **Pre & Post Traffic Hooks** to check the health of Lambda function after traffic shift. If something fails, CodeDeploy will rollback the deployment to the old version.

<img width="792" height="556" alt="image" src="https://github.com/user-attachments/assets/3783dac2-5bf5-416a-be0f-a987dc9587ff" />


- Traffic-shifting strategies:
    - **AllAtOnce** - change to 100% immediately
    - **Linear** - grow traffic at regular intervals
        - `Linear10PercentEvery3Minutes` - increase traffic by 10% every 3 mins
    - **Canary** - try a small percentage, then 100%
        - `Canary10Percent5Minutes` - try 10% for 5 minutes

- `AppSpec.yml` can be used for deploying a new version of Lambda function
    - `Name` - name of the function
    - `Alias` - name of the alias that needs to be updated
    - `CurrentVersion` - current version
    - `TargetVersion` - new version

<img width="564" height="402" alt="image" src="https://github.com/user-attachments/assets/556ebfa2-5352-4052-badc-898b3f20a070" />


## Lambda Container Images

- Docker images up to **10 GB** in size from ECR can be deployed on Lambda.
- Base image (in `Dockerfile`) must implement the **Lambda Runtime API**. Base images are available for **Python**, **NodeJS**, **Java**, **.NET**, **Go** and **Ruby**.
- We can create custom images, as long as they implement the Lambda Runtime API.
    
    <img width="1304" height="510" alt="image" src="https://github.com/user-attachments/assets/7dbf6c6d-f129-4ef4-8638-47a3841da088" />

    
- Built container images can be tested locally using **Lambda Runtime Interface Emulator**
- Ease of shipping FaaS code as container images
- The lambda function and the ECR repo must be in the same AWS account.
- Best practices for building container images:
    - Use AWS-provided Base Images as they are stable and cached by Lambda service
    - Multi-Stage Builds: copy the built artifact and discard the build stage
    - Build from Stable to Frequently Changing layers as we go down the `Dockerfile`
    - Use a Single Repository for Functions with Large Layers (to take advantage of layer caching in Docker)

## Versions

- When we work on a Lambda function we work on its unpublished version `$LATEST` (mutable) which always has the latest code and configuration.
- When we publish a Lambda function, we create an **immutable version** of the function. We cannot modify the code or configuration of a lambda version. We must create a new incremental version.
- Each version gets a unique ARN and is independents of other immutable versions and the `$LATEST` version.
- Old versions of the Lambda function can also be accessed.

## Aliases

- **Pointers to Lambda function versions** (mutable)
- They provide a **stable endpoint** (does not keep changing with version)
- Multiple aliases can be created (eg. dev, test, prod), each pointing to a different version
- Canary deployment by assigning weights to different lambda versions for an alias
- **Aliases cannot reference other aliases**

<img width="802" height="726" alt="image" src="https://github.com/user-attachments/assets/4783bf45-eaa6-493e-89ec-164b5d298890" />


## Function URL

- **HTTPS endpoints for Lambda functions** (without using an API gateway or ALB)
- Function URL can only be generated for `$LATEST` (unpublished function version) and aliases. **It cannot be generated for published function versions.**
- A unique **static URL** is generated for each alias or `$LATEST` which can only be accessed from public internet.

### Security

- To access the URL from another domain, need to configure CORS
- **Resource-based policy -** control which accounts / CIDR / IAM principals can access the URL
    - Auth type: `NONE` - allow public and unauthenticated access
        
        <aside>
        💡 The function will be publicly accessible without requiring any additional authentication. However, **you still need to write custom authorization logic to verify the signature provided in the HTTP headers and ensure that the request is coming from a valid user.**
        
        </aside>
        
        <img width="1690" height="510" alt="image" src="https://github.com/user-attachments/assets/b9a49c8a-47f1-4667-8537-082bd69ea084" />

        
    - Auth type: `AWS_IAM` - use IAM to authenticate and authorize requests
        - Principal’s identity based policy is also evaluated (Principal should have `lambda:InvokeFunctionUrl` permission)
        - For same account access, either resource based or identity based policy should allow
        - For cross account access, both resource based and identity based policy should allow
        
        <img width="1778" height="406" alt="image" src="https://github.com/user-attachments/assets/ba025c6d-9227-4cb5-ac56-ec00c5df2e96" />

        

## Profiling using CodeGuru

- CodeGuru Profiler is integrated with Lambda to provide runtime performance
- Supported for Java and Python runtimes
- Just enable from the console
    - CodeGuru profiler is added to the function as a Lambda layer
    - Required environment variables are populated
    - `AmazonCodeGuruProfilerAgentAccess` policy is attached to the lambda function



