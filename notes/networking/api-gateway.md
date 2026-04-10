---
layout: default
title: API Gateway
parent: Notes
nav_order: 7
---

# 🚪 API Gateway

* [Intro](#intro)
* [Integration Types](#integration-types)
* [Mapping Templates](#mapping-templates)
* [Endpoint Types](#endpoint-types)
* [Deployments & Stages](#deployments)
* [Stage Variables](#stage-variables)
* [Canary Deployment](#canary-deployment)
* [OpenAPI Spec](#openapi-spec)
* [Caching & Invalidation](#caching-api-responses)
* [Usage Plans & API Keys](#usage-plans)
* [Observability (Logging, Tracing, Metrics)](#observability)
* [Performance & Throttling](#performance)
* [CORS](#cross-origin-resource-sharing-cors)
* [Security & Auth](#user-authentication)
* [API Types (REST, HTTP, WebSocket)](#api-types)

---


## Intro

- Serverless REST APIs with **TLS termination** (all of the APIs created with API Gateway expose **HTTPS** endpoints only)
- API versioning
- Multiple environment (dev, test, prod)
- Request throttling
- Authentication and Authorization
- Supports API keys
- Supports **WebSocket**
- Cache API responses
- Generate SDK and API specifications
- Swagger / OpenAPI config supported (import or export)
- Transform and validate requests and responses
- Firewall can be implemented using Web Application Firewall (WAF)

## Integration

### Integration Types

- `MOCK` - API gateway returns a mock (hardcoded) response (useful in development)
- `AWS_PROXY` (Lambda Proxy)
    - Incoming request from the client is **proxied to the lambda function** as a JSON event (request specific params like status code is automatically handled by API gateway)
    - Used to create serverless REST APIs
    - **Cannot modify the request & response in API gateway** (no mapping template)
    - The entire processing of request happens in the Lambda function
    
    <img width="1024" height="174" alt="image" src="https://github.com/user-attachments/assets/a32d8190-5031-4102-bd32-183a2662e7f1" />

    
- `AWS`
    - To integrate with AWS resources or a Lambda function (custom integration)
    - Can modify integration requests and responses using **mapping templates**
- `HTTP_PROXY`
    - Incoming request is proxied to any HTTP backend
    - **Request & response cannot be modified** (no mapping template)
    - Option to add HTTP headers in the request (eg. API key)
    
    <img width="1024" height="230" alt="image" src="https://github.com/user-attachments/assets/47b8cede-55d6-418e-ba16-49dd9fb86449" />

    
- `HTTP`
    - To integrate with any HTTP endpoint
    - Can modify integration requests and responses using **mapping templates**
 
    <img width="1024" height="235" alt="image" src="https://github.com/user-attachments/assets/b8a57df2-02f4-4f9b-8534-bc0255408891" />


### Mapping Templates

- Used to modify requests and responses
    - Modify query string parameters
    - Modify body content
    - Add headers
- Uses **Velocity Template Language (VTL)**
- `Content-Type` must be `application/json` or `application/xml`
- Example: Exposing a SOAP backend as a REST API
    
    Use mapping template to convert requests and responses between JSON and XML.
    
    <img width="1024" height="195" alt="image" src="https://github.com/user-attachments/assets/0cf78df5-d940-4cb5-87d7-f6af81c98340" />


## Endpoint Types

- **Edge-Optimized** (default)
    - For global clients
    - Requests are routed through the CloudFront edge locations (improves latency)
    - The API Gateway lives in only one region but it is accessible efficiently through edge locations
- **Regional**
    - For clients within the same region
    - Could manually combine with your own CloudFront distribution for global deployment (this way you will have more control over the caching strategies and the distribution)
- **Private**
    - Can only be accessed within your VPC using an **Interface VPC endpoint** (ENI)
    - Use resource policy to define access
 
## Endpoint Types

- **Edge-Optimized** (default)
    - For global clients
    - Requests are routed through the CloudFront edge locations (improves latency)
    - The API Gateway lives in only one region but it is accessible efficiently through edge locations
- **Regional**
    - For clients within the same region
    - Could manually combine with your own CloudFront distribution for global deployment (this way you will have more control over the caching strategies and the distribution)
- **Private**
    - Can only be accessed within your VPC using an **Interface VPC endpoint** (ENI)
    - Use resource policy to define access

## Deployments

- Deploy the API to get the API Gateway endpoint
- If you make changes to the API, a new version is internally created. You need to deploy the API for the changes to take effect.
- **Versions (changes) are deployed to stages** (no limit on the number of stages)
- Metrics and logs are separate for each stage
- Each stage has independent configuration and can be rolled back to any version (the whole history of deployments to a stage is kept)
- Handling breaking changes using multiple stages
    
    We want to create a new version of the application which involves breaking changes at the API level. In this case, we can deploy a new stage (v2) and build our new application there. Since multiple stages can co-exist, we can have both the versions working and once all the users have migrated to v2, we can bring down v1. 
    
    <img width="1436" height="496" alt="image" src="https://github.com/user-attachments/assets/e23faebe-ac89-4bf1-9f7c-ddb3efff1831" />

### Stage Variables

- Environment variables for API gateway
- Can be changed without redeploying the API
- Stage variables are passed to **context** object in lambda functions
- Format to access stage variables in API gateway - `${stageVariables.variableName}`
- Example: Stage variables to point to Lambda Aliases
    
    Stage variables can be used to point to different Lambda aliases. Each stage points to a different lambda alias depending on the value of a stage variable. To shift traffic, we can modify the alias weights without making changes to the API gateway.
    
    <img width="1804" height="626" alt="image" src="https://github.com/user-attachments/assets/294af7c6-3134-4f0a-9b61-0af36d8d10bd" />

    
    The lambda function in API gateway - `<function-name>:${stageVariables.variableName}`
    
    <img width="958" height="56" alt="image" src="https://github.com/user-attachments/assets/aac7d956-5e20-4132-973e-b5574980c888" />

    
    This will require adding resource-based policies on all the aliases to allow the API gateway to invoke them.
    
    <img width="1108" height="464" alt="image" src="https://github.com/user-attachments/assets/9d6beb7d-616e-4557-9dc4-020af8c6f1ea" />

    
    For every stage, create a stage variable with the value as name of the alias it should point to. 
    
    <img width="1560" height="454" alt="image" src="https://github.com/user-attachments/assets/9c574980-1d1f-48b5-80f1-b72f26f72dba" />

    

### Canary Deployment

- Use a **canary stage** with the new application version
- Can override stage variables for Canary deployment

<img width="1024" height="296" alt="image" src="https://github.com/user-attachments/assets/1e71b971-f785-4112-8af6-b11b74f04aa5" />

