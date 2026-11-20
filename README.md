# AWS EC2 Start/Stop Automation with DLQ and Failure Notifications
Automated EC2 start/stop scheduling with Lambda, EventBridge Scheduler, SQS Dead Letter Queue, SNS notifications, and CloudWatch monitoring.


## Project Overview

This project automates the starting and stopping of an Amazon EC2 instance using **Amazon EventBridge Scheduler and AWS Lambda**.

It also implements a failure-handling workflow using an **Amazon SQS Dead Letter Queue (DLQ)**, a notification Lambda function, and Amazon SNS email notifications. Amazon CloudWatch is used for logging and monitoring.

The goal is to automate EC2 instance operations, reduce unnecessary running costs, and provide visibility into scheduling delivery failures.

## Architecture

![AWS EC2 Start Stop Automation Architecture](architecture/architecture.png)

### Architecture Workflow

#### Success Flow

1. Amazon EventBridge Scheduler triggers at the configured start or stop time.
2. Scheduler invokes the controller Lambda function with an action payload.
3. The Lambda function reads the requested action (`start` or `stop`).
4. The function calls the appropriate EC2 API.
5. The target EC2 instance is started or stopped.

#### Failure Flow

1. If Scheduler fails to deliver the invocation to its target after configured retries, the failure message is sent to the SQS Dead Letter Queue.
2. The SQS queue triggers the notification Lambda function.
3. The notification Lambda processes the failure message.
4. Failure details are published to an SNS topic.
5. SNS sends an email notification to subscribed recipients.

#### Monitoring

- CloudWatch Logs are used for Lambda troubleshooting.
- A CloudWatch alarm monitors the number of visible messages in the DLQ.
- The alarm threshold is configured to detect when at least one message is present in the queue.

---

## AWS Services Used

| AWS Service | Purpose |
|---|---|
| Amazon EC2 | Target instance to be started or stopped |
| Amazon EventBridge Scheduler | Scheduled start and stop triggers |
| AWS Lambda | EC2 controller and failure notification processing |
| Amazon SQS | Dead Letter Queue for failed Scheduler deliveries |
| Amazon SNS | Email notifications for failures |
| Amazon CloudWatch | Lambda logging and DLQ monitoring |
| AWS IAM | Roles and least-privilege permissions |

---

## Resources Created

| Resource | Name |
|---|---|
| EC2 Instance | `poc-ec2-start-stop` |
| Controller Lambda | `poc-ec2-start-stop-lambda` |
| Controller IAM Role | `poc-ec2-lambda-execution-role` |
| Controller IAM Policy | `poc-ec2-start-stop-policy` |
| Start Schedule | `poc-ec2-start-schedule` |
| Stop Schedule | `poc-ec2-stop-schedule` |
| Scheduler IAM Role | `poc-ec2-scheduler-execution-role` |
| Scheduler IAM Policy | `poc-ec2-scheduler-permissions-policy` |
| SQS DLQ | `poc-ec2-start-stop-dlq` |
| Notification Lambda | `poc-ec2-failure-notifier` |
| Notification Lambda IAM Role | `poc-ec2-notifier-execution-role` |
| SNS Topic | `poc-ec2-failure-notifications` |
| CloudWatch Alarm | `poc-ec2-dlq-message-alarm` |

---

## Implementation

### 1. Launch EC2 Instance

- Launched an Amazon Linux EC2 instance.
- Configured the instance for the POC.
- Verified that the instance was running and passed its status checks.
- Used the instance ID in the Lambda function and IAM policy.

### 2. Configure IAM Permissions

Created separate IAM roles for the controller Lambda and EventBridge Scheduler.

The controller Lambda role was configured with permissions to:

- Start the target EC2 instance.
- Stop the target EC2 instance.
- Write logs to CloudWatch.

The Scheduler role was configured with permissions to:

- Invoke the controller Lambda.
- Send failed target deliveries to the SQS DLQ.

Permissions were scoped to the relevant resources wherever applicable.

### 3. Create Controller Lambda

Created a Python Lambda function to process scheduled actions.

Example input:

**Start EC2**

```json
{
  "action": "start"
}
```

**Stop EC2**

```json
{
  "action": "stop"
}
```

The Lambda function processes the action and calls the corresponding EC2 API.

### 4. Configure EventBridge Scheduler

Created separate schedules for starting and stopping the EC2 instance.

- Start schedule invokes Lambda with `{"action":"start"}`.
- Stop schedule invokes Lambda with `{"action":"stop"}`.
- Configured retry handling.
- Configured the SQS DLQ for failed target deliveries.

### 5. Create SQS Dead Letter Queue

Created a Standard SQS queue to store Scheduler target-delivery failures.

The DLQ helps retain failure information for troubleshooting and further processing.

> The Scheduler DLQ is intended for failures to deliver the scheduled invocation to its target after retry handling. It does not automatically capture every exception occurring inside the Lambda function after invocation.

### 6. Configure SNS Notifications

Created an SNS topic and configured an email subscription.

The email subscription was confirmed to allow notification delivery.

### 7. Create Failure Notification Lambda

Created a second Lambda function to:

- Receive messages from the SQS DLQ.
- Process failure details.
- Publish notification messages to SNS.

Configured an SQS trigger for the notification Lambda.

### 8. Configure CloudWatch Monitoring

Created a CloudWatch alarm based on the SQS metric:

`ApproximateNumberOfMessagesVisible`

Alarm configuration:

- Statistic: Maximum
- Period: 1 minute
- Threshold: Greater than or equal to 1
- Evaluation: 1 out of 1 datapoints

The alarm monitors the presence of messages in the DLQ.

---

## Testing and Validation

### EC2 Start/Stop Testing

- Tested the controller Lambda with the `start` action.
- Verified that the EC2 instance transitioned to Running.
- Tested the controller Lambda with the `stop` action.
- Verified that the EC2 instance transitioned to Stopped.
- Tested scheduled start and stop operations.

### SQS-to-SNS Notification Testing

- Sent a test message to the SQS DLQ.
- Verified that the SQS trigger invoked the notification Lambda.
- Checked the notification Lambda CloudWatch logs.
- Verified that the SNS publishing operation succeeded.
- Confirmed email delivery to the subscribed recipient.

### Scheduler Failure Testing

A Scheduler test produced an `AccessDeniedException`.

The failure behavior was investigated using Scheduler execution details and the SQS DLQ.

**Testing note:** The manual SQS message test verified the SQS → Lambda → SNS → Email notification path. A complete end-to-end test demonstrating an actual Scheduler delivery failure reaching the DLQ and triggering the final email requires separate validation.

---

## Security Considerations

- Used separate IAM roles for different AWS services.
- Applied resource-scoped permissions for EC2 operations.
- Avoided granting unnecessary administrative permissions.
- Configured Scheduler permissions separately from Lambda execution permissions.
- No AWS credentials or secrets should be committed to the repository.

---

## Key Learnings

- Automating EC2 lifecycle management using EventBridge Scheduler.
- Using Lambda to perform AWS API operations.
- Configuring IAM roles, trust policies and permission policies.
- Implementing SQS Dead Letter Queues for failure handling.
- Integrating SQS, Lambda and SNS for notification workflows.
- Monitoring AWS resources using CloudWatch.
- Troubleshooting AWS service permissions and delivery failures.
- Understanding the distinction between target-delivery failures and errors occurring inside Lambda execution.

---

## Cleanup

After testing, AWS resources were removed to avoid unnecessary costs.

Cleanup included:

- EventBridge schedules
- Lambda functions
- SQS queue
- SNS topic and subscription
- CloudWatch alarm and log resources as applicable
- IAM roles and custom policies
- EC2 instance

---

## Repository Structure

```text
aws-ec2-start-stop-automation/
│
├── README.md
├── architecture/
│   └── architecture.png
│
└── docs/
    └── implementation-runbook.pdf
```

---

## Project Objective

This project demonstrates practical implementation of AWS automation, serverless architecture, IAM security, failure handling, notification workflows and cloud monitoring.
