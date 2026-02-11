AWS CDK (Cloud Development Kit) is an Infrastructure as Code (IaC) framework that lets you define AWS infrastructure using real programming languages instead of raw JSON/YAML.

Built and maintained by Amazon Web Services.

👉 You write TypeScript / JavaScript / Python / Java / C#
👉 CDK synthesizes CloudFormation
👉 CloudFormation creates AWS resources

Why CDK exists (the problem it solves)
Traditional CloudFormation

Huge YAML files

Hard to reuse

No loops, conditions are painful

Error-prone

CDK advantages

Real code (loops, functions, classes)

Strong typing

Reusable constructs

Safer refactoring

Better for large systems (BFSI)

“CDK brings software engineering principles to infrastructure provisioning.”

CDK App (TypeScript)
   ↓ cdk synth
CloudFormation Template
   ↓ cdk deploy
CloudFormation Stack
   ↓
AWS Resources

Important:

CDK does NOT bypass CloudFormation

CloudFormation is always the execution engine


Core CDK Concepts (MUST KNOW)
1️⃣ App

Top-level container

const app = new cdk.App();

2️⃣ Stack

A deployable unit (maps to CF stack)

class PaymentsStack extends cdk.Stack {}

3️⃣ Construct

Reusable infrastructure building block

Levels:

L1: Low-level (CF resources)

L2: Service abstractions

L3: Patterns (best practices)

new lambda.Function(this, "Fn", {...});

| Tool    | Purpose                |
| ------- | ---------------------- |
| AWS CLI | Ops, automation        |
| AWS SDK | Runtime code           |
| AWS CDK | Infrastructure as Code |

“SDK runs at runtime; CDK runs at deploy time.”


BFSI-Grade CDK Architecture (Realistic)::

CDK Stack
 ├─ API Gateway
 ├─ Lambda (Node.js)
 ├─ DynamoDB (Ledger)
 ├─ SQS (Async)
 ├─ IAM Roles (Least Privilege)
 ├─ CloudWatch + Alarms

All provisioned once via CDK
All business logic uses AWS SDK v3

Example: BFSI Serverless Stack (CDK + Node.js)
CDK Stack (TypeScript)
import * as cdk from "aws-cdk-lib";
import * as lambda from "aws-cdk-lib/aws-lambda";
import * as dynamodb from "aws-cdk-lib/aws-dynamodb";

export class LedgerStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string) {
    super(scope, id);

    const table = new dynamodb.Table(this, "Ledger", {
      partitionKey: { name: "pk", type: dynamodb.AttributeType.STRING },
      sortKey: { name: "sk", type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST
    });

    const fn = new lambda.Function(this, "LedgerFn", {
      runtime: lambda.Runtime.NODEJS_18_X,
      handler: "index.handler",
      code: lambda.Code.fromAsset("lambda")
    });

    table.grantWriteData(fn); // least privilege
  }
}


Lambda Runtime (AWS SDK v3)
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";


👉 CDK provisions
👉 SDK executes

IAM & Security (BFSI CRITICAL)
How CDK enforces least privilege
table.grantWriteData(fn);


CDK auto-generates:
IAM role
Scoped policy
No wildcard permissions

Interview line:
“CDK helps enforce least privilege by generating scoped IAM policies automatically.”

CDK Environments (Dev / QA / Prod)
cdk deploy --profile dev
cdk deploy --profile prod


or

new LedgerStack(app, "LedgerProd", {
  env: { account: "123", region: "ap-south-1" }
});


CDK Best Practices (Interview Checklist)

✅ One stack per bounded context
✅ Separate infra & runtime code
✅ No secrets in CDK
✅ Use SSM / Secrets Manager
✅ Enable termination protection
✅ Use constructs for reuse
✅ Version control CDK code

| Aspect         | CDK            | Terraform     |
| -------------- | -------------- | ------------- |
| Language       | Real languages | HCL           |
| AWS-native     | Yes            | No            |
| State mgmt     | CloudFormation | TF State      |
| Learning curve | Lower for devs | Infra-focused |
| BFSI usage     | Very common    | Also common   |


When NOT to use CDK::
❌ Runtime logic
❌ One-off scripts
❌ Quick manual testing
❌ Non-AWS infra


Q: Does CDK replace CloudFormation?
No. CDK generates CloudFormation.

Q: Can CDK be used with SDK?
Yes. CDK provisions infra; SDK is used inside apps.

Q: How does CDK help security?
By enforcing IAM least privilege and avoiding manual policies.

Q: CDK vs SAM?
SAM is YAML-focused; CDK is code-first and more flexible.

CDK → defines infrastructure
CloudFormation → provisions infrastructure
SDK → runs business logic
CLI → deploys & operates

“AWS CDK is a code-first Infrastructure as Code framework that synthesizes CloudFormation, enabling scalable, secure, and reusable AWS infrastructure using familiar programming languages—ideal for BFSI-grade, serverless architectures.”

BFSI Security Pitfalls in AWS CDK (and How to Avoid Them)::

1️⃣ Over-permissive IAM policies (MOST COMMON)

Pitfall

fn.addToRolePolicy(new iam.PolicyStatement({
  actions: ["dynamodb:*"],
  resources: ["*"]
}));


Why BFSI hates this

Violates least privilege

Large blast radius

Audit red flag (PCI-DSS, SOX, RBI)

Correct CDK pattern

table.grantReadWriteData(fn);


✅ Auto-scoped
✅ Resource-level
✅ Auditable

Interview line

“In BFSI, IAM wildcards are considered a critical security flaw.”

2️⃣ Secrets embedded in CDK code

Pitfall

environment: {
  DB_PASSWORD: "plain-text"
}


Why dangerous

Secrets land in:

Git

CloudFormation

CI logs

Correct pattern

AWS Secrets Manager

Or SSM Parameter Store

const secret = secretsmanager.Secret.fromSecretNameV2(
  this, "DbSecret", "prod/db"
);

fn.addEnvironment("DB_SECRET_ARN", secret.secretArn);
secret.grantRead(fn);

3️⃣ Public networking by default

Pitfall

Lambda not in VPC

DynamoDB via public endpoint

Internet-exposed data plane

BFSI-grade setup

Lambda in private subnets

VPC endpoints for:

DynamoDB

SQS

S3

vpc.addGatewayEndpoint("DynamoEndpoint", {
  service: ec2.GatewayVpcEndpointAwsService.DYNAMODB
});

4️⃣ Missing encryption & retention controls

Pitfall

Default CloudWatch logs

No retention

No KMS CMK

Correct

new logs.LogGroup(this, "AuditLogs", {
  retention: logs.RetentionDays.ONE_YEAR,
  encryptionKey: kmsKey
});

5️⃣ No termination protection

Pitfall

cdk destroy


💥 Ledger gone.

Fix

new cdk.Stack(app, "ProdStack", {
  terminationProtection: true
});

6️⃣ Mixing infra and business logic

Pitfall

AWS SDK calls inside CDK constructs

❌ This breaks separation of concerns
❌ Hard to audit
❌ Dangerous in BFSI

Golden Rule (MEMORIZE)
CDK = provision time
SDK = runtime

| Layer          | Tool   | What it does               |
| -------------- | ------ | -------------------------- |
| Infrastructure | CDK    | IAM, VPC, Lambda, DynamoDB |
| Runtime logic  | SDK v3 | CRUD, events, transactions |
| Ops            | CLI    | Deploy, inspect            |

Example: BFSI Ledger System
CDK (Infrastructure)
const table = new dynamodb.Table(this, "Ledger", {
  encryption: dynamodb.TableEncryption.AWS_MANAGED,
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST
});

const fn = new lambda.Function(this, "LedgerFn", {
  runtime: lambda.Runtime.NODEJS_18_X,
  code: lambda.Code.fromAsset("lambda"),
  handler: "index.handler"
});

table.grantWriteData(fn);

SDK v3 (Runtime – Lambda)
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { PutCommand } from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});

await client.send(new PutCommand({
  TableName: process.env.TABLE_NAME,
  Item: { pk, sk, amount }
}));

✅ No credentials
✅ IAM role injected
✅ Audit-friendly


| Aspect           | CDK                        | SAM            |
| ---------------- | -------------------------- | -------------- |
| Language         | TypeScript / Python / Java | YAML           |
| Abstraction      | High (constructs)          | Medium         |
| Flexibility      | Very high                  | Limited        |
| Learning curve   | Easier for devs            | Easier for ops |
| BFSI suitability | ⭐⭐⭐⭐⭐                      | ⭐⭐⭐⭐           |


SAM – Where It Shines

✅ Simple serverless apps
✅ Quick PoCs
✅ Minimal infra logic

Resources:
  Fn:
    Type: AWS::Serverless::Function

CDK – Why BFSI prefers it

✅ Strong typing
✅ Reusable constructs
✅ Enforced security patterns
✅ Complex networking & IAM
✅ Multi-account setups

“SAM is great for simple serverless apps, but BFSI systems usually require CDK due to complex IAM, VPC, and compliance needs.”

When to Choose What
Choose CDK when:

Multi-account architecture

Strict IAM & security controls

Reusable infra libraries

BFSI / healthcare / gov tech

Choose SAM when:

Simple Lambda + API Gateway

Small teams

Rapid prototyping

BFSI Interview Rapid-Fire Q&A

Q: Biggest CDK security risk?
Over-permissive IAM policies.

Q: Can SDK be used inside CDK?
No. SDK is runtime; CDK is deploy-time.

Q: How does CDK help least privilege?
Through resource-scoped grant APIs.

Q: CDK vs SAM for payments platform?
CDK — due to security, reuse, and scale.

“In BFSI systems, AWS CDK is used to provision secure, least-privilege infrastructure, while AWS SDK v3 is used at runtime for business logic. Security pitfalls like wildcard IAM policies, embedded secrets, and public networking must be avoided. Compared to SAM, CDK provides the flexibility and control required for regulated, large-scale financial platforms.”