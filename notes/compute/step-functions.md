---
layout: default
title: Step Functions
parent: Notes
nav_order: 3
---

# 🪜 Step Functions

* [Intro](#intro)
* [Task](#task)
* [States](#states)
* [Error Handling](#error-handling)
* [Retry](#retry)
* [Catch](#catch)
* [Wait for Task Token](#wait-for-task-token)
* [Activity Tasks](#activity-tasks)
* [Workflow Types](#workflow-types)
    * [Standard Workflows](#standard-workflows-default)
    * [Express Workflows](#express-workflows)
 

---

<img width="1800" height="574" alt="image" src="https://github.com/user-attachments/assets/7c37f71e-a273-47e1-9ce3-cfa3ca86141f" />

## Intro

- **Serverless** workflow orchestration
- Model workflows as state machines (1 state machine per workflow)
- Written in JSON
- **Visual workflow execution**
- Execution event history shows the input and output of each step
- Step function has **amazing error handling capabilities** (can be offloaded from the application)

## Task

<img width="776" height="594" alt="image" src="https://github.com/user-attachments/assets/60c754c2-304d-49a0-b168-fc886c22fd0a" />

- Each task in the workflow could perform an action on an AWS service or launch another step function workflow.
- Task definition to invoke a Lambda function
<img width="1024" height="507" alt="image" src="https://github.com/user-attachments/assets/3d7278e0-ff3a-4273-9744-67a324f6404d" />

## States

- **Choice State** - Test for a condition to send to a branch (or default branch)
- **Fail or Succeed State** - Stop execution with failure or success
- **Pass State** - Simply pass its input to its output or inject some fixed data
- **Wait State** - Provide a delay for a certain amount of time or until a specified time or date
- **Map State** - Dynamically iterate steps
- **Parallel State** - Begin parallel branches of execution (**asynchronous** execution)
- **Task State** - Run some code **synchronously**

## Error Handling

- Handle errors in the state machine instead of the application code. This makes the application logic simpler. Also, step functions provide execution history.
- Predefined error codes:
    - `States.ALL` - any error
    - `States.Timeout` -  task ran longer than `TimeoutSeconds` or no heartbeat received
    - `States.TaskFailed` - task execution failure
    - `States.Permissions` - insufficient privileges to execute code
- The state can also throw custom errors that can be caught in the step function (eg. a Lambda function throwing a custom error)

## Retry
<img width="820" height="692" alt="image" src="https://github.com/user-attachments/assets/8040b4bd-1c2b-42f1-b078-ff4aeffe6638" />

- Retry failed state
- `Retry` block is evaluated top to bottom
- `BackoffRate` - what factor the `IntervalSeconds` should be multiplied with at each retry
- When `MaxAttempts` are reached, the `Catch` block kicks in


### **Catch**

<img width="944" height="768" alt="image" src="https://github.com/user-attachments/assets/bce0b0f4-e757-4134-a85f-15c666ed2325" />


- Transition to failed path
- `Catch` block is evaluated top to bottom
- After all the retries have been exhausted, the state function goes into `Catch`
- If the error is of X type, go to the next state Y.
- `ResultPath` - a path that determines what input is sent to the state specified in the `Next` field. Example: it can be used to send the error to the `Next` state.

<img width="1746" height="568" alt="image" src="https://github.com/user-attachments/assets/b26cc541-111d-4a04-bee3-c63fdce08bad" />

## Wait for Task Token

- Used to integrate the workflow with an external task where the external task must finish the job before the workflow proceeds further.
- **Push-based** (the task pushes the work to the external application or worker which after completion invokes a callback API)
- The task is paused until it receives the callback (API call) with the `TaskToken`
- Append `.waitForTaskToken` to the `Resource` field to tell Step Functions to wait for the Task Token to be returned.
Example: `"Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken"`
- **Working**: The step function is paused during the `Check Credit` task execution, where we pass the `TaskToken` to the external application (push to SQS). After the external application is done processing, it makes a `SendTaskSuccess` API call with the result of processing and the passed `TaskToken`. This means the external task was executed successfully and the step function can continue execution. If the external application fails to process, it makes a `SendTaskFailure` API call which is treated as task failed.

<img width="1698" height="840" alt="image" src="https://github.com/user-attachments/assets/52e4e420-f2dc-4e57-8cfa-962c9b576dcc" />


## Activity Tasks

- **Activity Workers** (running on any compute resource) poll the workflow for tasks using `GetActivityTask` API.
- If an activity worker gets a task, it will complete it and send the response of success or failure as `SendTaskSuccess` or `SendTaskFailure` API call.
- **Pull-based** (tasks are pulled by the Activity Workers)
- `TaskToken` is used to identify which task got completed (same way as in Wait for Task Token)
- To keep the Task active:
    - Configure `TimeoutSeconds` on the step function - how long the task will wait for the activity worker to complete (max 1 year)
    - Send heartbeats periodically from the activity worker to the task using `SendTaskHeartBeat` at an interval less than `HeartBeatSeconds` parameter (set in the step function).

<img width="514" height="836" alt="image" src="https://github.com/user-attachments/assets/fa787347-70bd-41c9-aeb4-4c79d59a40ae" />

## Workflow Types

<img width="820" height="610" alt="image" src="https://github.com/user-attachments/assets/de284033-2531-4598-904f-8a53aac9d447" />


### Standard Workflows (default)

- Max duration: 1 year
- Execution model: **Exactly-once Execution**
- Execution rate: Over 2,000 workflow executions per sec
- Execution history: **90 days in the console** (send to CloudWatch to retain for longer)
- Pricing: based on the number of state transitions
- Use cases: **Non-idempotent actions** (eg. payment processing)

### Express Workflows

- Max duration: 5 min
- Execution rate: Over 100,000 workflow executions per sec
- Execution history: Not available in the console (must use CloudWatch)
- Use cases: IoT data ingestion, streaming data, backend for apps, etc.
- Types:
    - **Asynchronous** Express Workflows
        - Execution model: **At-least Once Execution** (the operation must be idempotent because there could be retries)
        - When the workflow is invoked, it just starts and doesn’t return the result of computation.
        - Use cases: where we don’t need immediate response (eg. messaging services)
    - **Synchronous** Express Workflows
        - Execution model: **At-most Once Execution**
        - When the workflow is invoked, it completes it and returns the response.
        - Can be invoked from **API Gateway** or **Lambda function**
