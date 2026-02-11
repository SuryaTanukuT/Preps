AWS SAM (Serverless Application Model) is a serverless-focused Infrastructure as Code (IaC) framework that lets you define Lambda-centric architectures using simplified YAML, which then gets transformed into CloudFormation.

Built and maintained by Amazon Web Services.

One-liner:
“SAM is a CloudFormation extension optimized for serverless applications.”

template.yaml (SAM)
   ↓ sam build
CloudFormation template
   ↓ sam deploy
CloudFormation stack
   ↓
Lambda / API Gateway / DynamoDB

Key points:
SAM does NOT replace CloudFormation
It simplifies CloudFormation for serverless
Uses Transform: AWS::Serverless-2016-10-31

Core SAM Concepts (Must Know)
1️⃣ Template (template.yaml)
Everything lives here.
Transform: AWS::Serverless-2016-10-31

2️⃣ Serverless Resources
| SAM Type                      | Expands To     |
| ----------------------------- | -------------- |
| AWS::Serverless::Function     | Lambda + IAM   |
| AWS::Serverless::Api          | API Gateway    |
| AWS::Serverless::Table        | DynamoDB       |
| AWS::Serverless::StateMachine | Step Functions |

3️⃣ Policies (Shortcut IAM)
Policies:
  - DynamoDBWritePolicy:
      TableName: Ledger

⚠️ Convenient, but can hide permissions (BFSI caution).

Example: Simple BFSI-Style Ledger API (SAM)
Resources:
  LedgerFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: nodejs18.x
      Handler: index.handler
      CodeUri: ledger/
      Policies:
        - DynamoDBWritePolicy:
            TableName: Ledger
      Events:
        Api:
          Type: Api
          Properties:
            Path: /debit
            Method: post

  Ledger:
    Type: AWS::Serverless::Table
    Properties:
      PrimaryKey:
        Name: pk
        Type: String
      BillingMode: PAY_PER_REQUEST

✔ Very fast to write
✔ Minimal boilerplate
❌ Less control than CDK

SAM CLI (Big Advantage)::
Local development & testing
sam build
sam local invoke
sam local start-api


You can:
Run Lambda locally in Docker
Test API Gateway locally
Debug faster than CDK

Interview line:
“SAM’s biggest strength is its local development and testing experience.”

| Aspect           | SAM      | CDK                 |
| ---------------- | -------- | ------------------- |
| Language         | YAML     | TypeScript / Python |
| Abstraction      | Medium   | High                |
| Flexibility      | Limited  | Very high           |
| Local testing    | ⭐⭐⭐⭐⭐    | ⭐⭐                  |
| IAM control      | Moderate | Precise             |
| BFSI suitability | ⭐⭐⭐⭐     | ⭐⭐⭐⭐⭐               |

When BFSI teams choose SAM

Simple Lambda + API Gateway

Small team

Faster PoC

Minimal networking/IAM complexity

When BFSI teams avoid SAM

Complex IAM boundaries

Multi-account / multi-VPC

Shared infra libraries

Fine-grained security controls

SAM Security Considerations (BFSI Angle)
⚠️ Common Pitfalls

Over-reliance on policy templates

Less visibility into generated IAM

Limited VPC & networking expressiveness

Harder to enforce org-wide standards

✅ Best Practices

Review generated CloudFormation

Avoid wildcard policies

Use explicit IAM where possible

Enable CloudWatch + CloudTrail

Use Secrets Manager (never env secrets)

AM + SDK (Correct Usage)

Rule: SAM provisions → SDK executes

SAM

Creates Lambda

Attaches IAM role

Sets environment variables

SDK v3 (inside Lambda)
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";


✔ IAM role injected automatically
✔ No access keys
✔ BFSI-safe

SAM vs CDK vs CLI (Mental Model)
SAM → simple serverless infra
CDK → complex & reusable infra
SDK → runtime business logic
CLI → ops & automation

Common Interview Questions (With Crisp Answers)

Q: Is SAM replacing CloudFormation?
No. SAM transforms into CloudFormation.

Q: SAM vs CDK — which is better?
Neither universally. CDK for complex systems, SAM for simple serverless apps.

Q: Can SAM be used in BFSI?
Yes, for smaller workloads; CDK is preferred for core systems.

Q: Does SAM support local testing?
Yes, via sam local.


AWS SAM vs Serverless Framework — Are they the same?

❌ Not the same
✅ Both are serverless deployment frameworks

What is AWS SAM?

AWS SAM (Serverless Application Model) is an AWS-native framework to build and deploy serverless applications on AWS only.

Built by Amazon Web Services

Uses CloudFormation under the hood

Best for AWS-only environments

What is Serverless Framework?

Serverless Framework is a third-party, multi-cloud framework to deploy serverless apps across AWS, Azure, GCP, etc.

Built by Serverless Inc

Uses CloudFormation on AWS

Uses its own abstraction layer

Architecture Comparison::
AWS SAM:
template.yaml → CloudFormation → AWS Resources

Serverless Framework:
serverless.yml → Framework Engine → CloudFormation → AWS Resources

| Feature         | AWS SAM        | Serverless Framework |
| --------------- | -------------- | -------------------- |
| Ownership       | AWS            | Third-party          |
| Cloud support   | AWS only       | Multi-cloud          |
| IaC engine      | CloudFormation | CloudFormation (AWS) |
| Language        | YAML           | YAML                 |
| Local testing   | sam local      | serverless offline   |
| Vendor lock-in  | Yes            | Lower                |
| BFSI acceptance | Very high      | Medium               |


Security & BFSI Perspective (VERY IMPORTANT)
Why BFSI prefers SAM

AWS-native

No external dependency

Predictable IAM generation

Easier audits

Long-term support

Interview line:
“BFSI teams prefer AWS-native tools like SAM or CDK over third-party frameworks due to audit and compliance requirements.”

Serverless Framework concerns in BFSI

Third-party abstraction

Paid enterprise features

Plugin risks

Security approvals harder