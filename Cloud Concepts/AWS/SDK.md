AWS SDK (Software Development Kit) is a programming library that allows applications to call AWS services directly from code.

You don’t click the AWS Console.
You don’t run terminal commands.
👉 Your code talks to AWS APIs.

| Aspect      | AWS SDK                       | AWS CLI          |
| ----------- | ----------------------------- | ---------------- |
| Used in     | Application code              | Terminal / CI    |
| Runs        | Runtime (Node.js, Java, etc.) | Shell            |
| Purpose     | Business logic                | Ops & automation |
| Credentials | IAM role / env                | IAM / SSO        |
| Example     | Lambda accessing DynamoDB     | Deploying to S3  |

👉 Apps use SDK, humans & pipelines use CLI

AWS SDK for JavaScript (IMPORTANT)
Current version

✅ AWS SDK for JavaScript v3
❌ v2 is legacy (maintenance mode)

Why v3 exists
Modular (import only what you need)
Smaller bundle size
Better performance
Tree-shaking support
Middleware-based design

Your Node.js Code
   ↓
AWS SDK Client
   ↓
SigV4 Signed HTTPS Request
   ↓
AWS Public API
   ↓
JSON Response

npm install @aws-sdk/client-s3
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

const s3 = new S3Client({ region: "us-east-1" });

await s3.send(new PutObjectCommand({
  Bucket: "my-bucket",
  Key: "file.txt",
  Body: "Hello AWS"
}));

SDK automatically finds credentials in this order:

1️⃣ IAM Role (Lambda / EC2 / ECS)
2️⃣ Environment variables
3️⃣ ~/.aws/credentials
4️⃣ Explicit config (discouraged)

Interview line

“AWS SDK follows a credential provider chain, preferring IAM roles over static keys.”

Using AWS SDK in Lambda (BEST PRACTICE)
Lambda code
const client = new DynamoDBClient({});


No keys.
No config.
No secrets.

👉 Lambda injects credentials via IAM Execution Role

AWS SDK with Environment Variables
export AWS_ACCESS_KEY_ID=xxx
export AWS_SECRET_ACCESS_KEY=yyy
export AWS_REGION=us-east-1


SDK auto-detects these.

import { DynamoDBClient, PutItemCommand } from "@aws-sdk/client-dynamodb";

const client = new DynamoDBClient({});

await client.send(new PutItemCommand({
  TableName: "orders",
  Item: {
    orderId: { S: "101" },
    amount: { N: "500" }
  }
}));


import { SQSClient, SendMessageCommand } from "@aws-sdk/client-sqs";

const sqs = new SQSClient({});

await sqs.send(new SendMessageCommand({
  QueueUrl: process.env.QUEUE_URL,
  MessageBody: JSON.stringify({ orderId: 1 })
}));


AWS SDK Middleware (Advanced, interview-worthy)
SDK v3 uses middleware stack:

Serialize → Sign → Retry → Send → Deserialize

Use cases:
Logging
Retry logic
Custom headers
Metrics

client.middlewareStack.add(
  (next) => async (args) => {
    console.log("Request", args);
    return next(args);
  },
  { step: "build" }
);


Performance Best Practices (IMPORTANT)

✅ Reuse SDK clients (don’t create per request)
✅ Prefer async/await
✅ Use pagination wisely
✅ Enable retries (default is good)
❌ Don’t bundle full SDK
❌ Don’t hardcode credentials

Security Best Practices
✅ Use IAM Roles
✅ Least privilege policies
❌ No access keys in code
❌ No credentials in git
✅ Secrets Manager for secrets

AWS SDK in Real Architectures
Typical backend
Node.js API
  ├── AWS SDK
  ├── DynamoDB
  ├── S3
  └── SQS

Event-driven
API → SQS → Lambda → DynamoDB
All communication via AWS SDK

Q1: What is AWS SDK?
Library to interact with AWS services programmatically.

Q2: SDK vs CLI?
SDK → application runtime
CLI → automation & ops

Q3: Latest AWS SDK for JS?
AWS SDK v3 (modular).

Q4: How does SDK authenticate?
Using IAM credentials + SigV4.

Q5: Where are credentials stored?
IAM roles, env vars, shared config.

Q6: Should we hardcode keys?
No. Always use IAM roles.

Q7: SDK in Lambda?
Yes. Lambda provides credentials automatically.

Q8: How retries work?
Built-in exponential backoff.

Q9: SDK vs CDK?
SDK → runtime calls
CDK → infrastructure provisioning


1️⃣ Serialize & Deserialize in AWS SDK v3 (CORE INTERNALS)
What does serialization mean?
Convert JavaScript objects → HTTP request

What does deserialization mean?
Convert HTTP response → JavaScript objects

SDK v3 request lifecycle
Input (JS Object)
  ↓ Serialize
HTTP Request
  ↓ Sign (SigV4)
Network Call
  ↓ Deserialize
Output (JS Object)

await s3.send(new PutObjectCommand({
  Bucket: "my-bucket",
  Key: "file.txt",
  Body: "hello"
}));


Internally:
Serializer converts this to REST/XML or JSON
Deserializer converts AWS response back to JS

📌 You never write serialize/deserialize logic — SDK generates it.

2️⃣ Tree Shaking — WHY SDK v3 EXISTS
SDK v2 problem
const AWS = require("aws-sdk"); // HUGE


Bundles entire AWS SDK
Bad for Lambda cold starts
Bad for frontend

SDK v3 solution
import { S3Client } from "@aws-sdk/client-s3";


✔ Only required service included
✔ Tree-shakable
✔ Smaller bundles
✔ Faster cold starts

Interview line
“AWS SDK v3 supports tree-shaking through fully modular packages, reducing bundle size and improving performance.”

| Feature       | SDK v2     | SDK v3         |
| ------------- | ---------- | -------------- |
| Architecture  | Monolithic | Modular        |
| Import size   | Huge       | Minimal        |
| Middleware    | Implicit   | Explicit stack |
| Tree shaking  | ❌          | ✅              |
| Retry control | Limited    | Fine-grained   |
| Streams       | Weak       | Native         |
| Abort support | ❌          | ✅              |
| Maintenance   | Legacy     | Active         |

📌 v2 is NOT deprecated, but v3 is recommended

4️⃣ New Features in AWS SDK v3
🔹 Modularized Packages
@aws-sdk/client-s3
@aws-sdk/client-dynamodb
@aws-sdk/lib-dynamodb

🔹 API Consistency Changes
Commands instead of methods

client.send(new GetItemCommand())


✔ Same pattern for all services
🔹 Configuration (Unified)
const client = new S3Client({
  region: "us-east-1",
  maxAttempts: 3
});

🔹 Middleware Stack (BIG DEAL)
Every request passes through a pipeline.

Steps:
initialize
serialize
build
finalize
deserialize


Example logging middleware:

client.middlewareStack.add(
  (next) => async (args) => {
    console.log("Request:", args);
    return next(args);
  },
  { step: "build" }
);

5️⃣ High-Level Concepts in SDK v3
🔸 Generated Packages
Auto-generated from AWS API models
Always in sync with AWS services

🔸 Streams
SDK v3 uses native Node.js streams

const response = await s3.send(cmd);
response.Body.pipe(fs.createWriteStream("file.txt"));

🔸 Paginators
import { paginateListObjectsV2 } from "@aws-sdk/client-s3";

for await (const page of paginateListObjectsV2({ client }, { Bucket })) {
  console.log(page.Contents);
}

🔸 Abort Controller (Timeouts)
const controller = new AbortController();
setTimeout(() => controller.abort(), 3000);
await client.send(cmd, { abortSignal: controller.signal });

🔸 Middleware Stack (Again — CORE)
Used for:
Logging
Retry
Metrics
Security headers
Tracing (X-Ray)

6️⃣ Install from Source
git clone https://github.com/aws/aws-sdk-js-v3
pnpm install
pnpm build


Used mainly by contributors or advanced debugging.
7️⃣ Release Cadence
Weekly to bi-weekly
Auto-generated updates
Security patches fast

8️⃣ Node.js Version Support
Node.js 18+ recommended
Node.js 16 supported

Older versions ❌

9️⃣ Stability of Modular Packages
✔ Each package is versioned
✔ Independent updates
✔ Backward compatibility within minor versions

🔟 Known Issues (Real World)
Cold start overhead if client created per request
Incorrect region config causes silent retries
Misconfigured IAM → AccessDenied (common)

1️⃣1️⃣ AWS Common Runtime (CRT) — IMPORTANT
What is CRT?
Low-level C-based runtime for:
Faster S3
HTTP/2
TLS
Event-streams

When needed?
High-throughput S3
Streaming uploads/downloads
npm install @aws-sdk/client-s3-crt

1️⃣2️⃣ BFSI-GRADE AWS SDK SECURITY ANSWERS
✔ IAM Role based auth only
No static credentials
No secrets in code

✔ Least privilege IAM
Service + action scoped
Resource-level policies

✔ Secure secrets
AWS Secrets Manager
No env secrets for long-lived creds

✔ Network isolation
VPC endpoints
Private subnets for Lambda

✔ Auditing
CloudTrail
X-Ray
Structured logs

Interview line

“In BFSI systems, AWS SDK access is strictly controlled using IAM roles, VPC endpoints, least-privilege policies, and full audit logging.”

1️⃣3️⃣ REAL PRODUCTION NODE.JS AWS ARCHITECTURE
API Gateway
   ↓
Lambda (Node.js)
   ↓ AWS SDK v3
SQS / DynamoDB / S3
   ↓
Event-driven Lambdas


Patterns used:
Idempotency
Retries
DLQ
Circuit breakers

Structured logging

1️⃣4️⃣ CRUD USING AWS SDK v3 — LAMBDA NODE.JS
DynamoDB CRUD Example
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import {
  DynamoDBDocumentClient,
  PutCommand,
  GetCommand,
  UpdateCommand,
  DeleteCommand
} from "@aws-sdk/lib-dynamodb";

const client = DynamoDBDocumentClient.from(new DynamoDBClient({}));

export const handler = async () => {
  await client.send(new PutCommand({
    TableName: "users",
    Item: { id: "1", name: "Surya" }
  }));

  const user = await client.send(new GetCommand({
    TableName: "users",
    Key: { id: "1" }
  }));

  return user.Item;
};


✔ No marshalling
✔ Clean JSON
✔ Production-ready

1️⃣5️⃣ How to Upgrade v2 → v3 (STEP-BY-STEP)
Step 1

Remove:
npm uninstall aws-sdk

Step 2
Install required clients:

npm install @aws-sdk/client-s3

Step 3
Replace API style

// v2
s3.putObject(params).promise();

// v3
await client.send(new PutObjectCommand(params));

Step 4

Use IAM roles (no keys)

“AWS SDK v3 is a modular, middleware-driven, tree-shakable SDK that provides secure, high-performance access to AWS services using SigV4, IAM roles, and native Node.js constructs. It is designed for modern serverless and enterprise workloads.”


AWS SDK for JavaScript v3 — New Features & Architecture
1️⃣ New Features in SDK v3 (Why v3 exists)
✅ Modular architecture
✅ Consistent API design
✅ Middleware-based request pipeline
✅ Native async/await support
✅ Smaller bundle size (tree-shaking)
✅ Better performance in Lambda & browsers
✅ Fine-grained retries, timeouts, and aborts
2️⃣ Modularized Packages (Core Change)
v2 (Monolithic)
const AWS = require("aws-sdk");


Loads all services

Heavy bundle

Slower cold starts

v3 (Modular)
import { S3Client } from "@aws-sdk/client-s3";


Each service is its own package:

@aws-sdk/client-s3
@aws-sdk/client-dynamodb
@aws-sdk/lib-dynamodb

Benefits
Tree-shaking
Faster cold starts
Smaller Lambda size
Better browser support

📌 Production & interview point

Modularization is the single biggest improvement in SDK v3.

3️⃣ API Consistency Changes
v2 (inconsistent)
s3.putObject(params).promise();
ddb.getItem(params).promise();

v3 (command pattern)
client.send(new PutObjectCommand(params));
client.send(new GetItemCommand(params));

Why this matters
Same mental model for every AWS service
Easier onboarding
Easier automation & code generation

4️⃣ Configuration (Unified & Explicit)
const client = new S3Client({
  region: "us-east-1",
  maxAttempts: 3
});

Key configuration options
region
credentials (usually omitted → IAM role)
maxAttempts
requestHandler
logger

📌 No global AWS config anymore — client-scoped config

5️⃣ Middleware Stack (CORE ARCHITECTURE)
Every request flows through a middleware pipeline:

initialize
 → serialize
 → build
 → finalizeRequest
 → deserialize

What middleware enables
Logging
Retry strategies
Metrics
Security headers
Tracing (X-Ray)

Example
client.middlewareStack.add(
  (next) => async (args) => {
    console.log("Outgoing request");
    return next(args);
  },
  { step: "build" }
);


📌 This replaces hidden internal logic from v2 with explicit control.

6️⃣ How to Upgrade from SDK v2 → v3
Step 1: Remove v2
npm uninstall aws-sdk

Step 2: Install required services
npm install @aws-sdk/client-s3

Step 3: Refactor API usage
// v2
s3.getObject(params).promise();

// v3
await client.send(new GetObjectCommand(params));

Step 4: Use IAM roles (no keys)

No code change needed if roles are already used.

7️⃣ High-Level Concepts in SDK v3
🔹 Generated Packages
Auto-generated from AWS service models
Always aligned with AWS APIs
Minimal human error
Faster service updates

🔹 Streams (Native Node.js Support)
const response = await s3.send(cmd);
response.Body.pipe(fs.createWriteStream("file.txt"));


Used for:
Large S3 objects
Media files
High-throughput data

🔹 Paginators (Async Iteration)
import { paginateScan } from "@aws-sdk/client-dynamodb";

for await (const page of paginateScan({ client }, params)) {
  console.log(page.Items);
}


✔ Memory efficient
✔ Cleaner than manual loops

🔹 Abort Controller (Timeout & Cancellation)
const controller = new AbortController();

setTimeout(() => controller.abort(), 3000);

await client.send(cmd, {
  abortSignal: controller.signal
});


Used for:
Timeouts
Circuit breakers
User cancellations

🔹 Middleware Stack (Revisited)
Middleware is first-class in v3:
Custom retry
Request signing
Observability
Security

8️⃣ Install SDK v3 from Source
Used only by contributors or advanced debugging.

git clone https://github.com/aws/aws-sdk-js-v3
pnpm install
pnpm build

9️⃣ Giving Feedback & Contributing
GitHub issues & PRs
Feature requests
Bug reports
Documentation fixes

Repo:
github.com/aws/aws-sdk-js-v3

🔟 Release Cadence
Weekly / bi-weekly releases
Automated code generation
Fast security patches
Independent package versioning

| Node.js | Support         |
| ------- | --------------- |
| 18+     | ✅ Recommended   |
| 16      | ✅ Supported     |
| <16     | ❌ Not supported |

1️⃣2️⃣ Stability of Modular Packages

Independent versioning
Semantic versioning
Backward compatibility within major versions
Production-ready maturity

📌 AWS uses v3 internally.
1️⃣3️⃣ Known Issues (Real-World)

Creating SDK clients per request → cold start impact
Misconfigured region → silent retries
Over-logging middleware → latency
Incorrect IAM policies → AccessDenied

1️⃣4️⃣ Functionality Requiring AWS Common Runtime (CRT)
What is AWS CRT?

Low-level native runtime written in C.
Used for:
High-performance S3 transfers
HTTP/2
TLS optimization
Event streams

When CRT is required
Large S3 uploads/downloads
Streaming workloads
High throughput

npm install @aws-sdk/client-s3-crt

Interview line
“AWS CRT is used in SDK v3 for performance-critical operations like high-throughput S3 transfers.”

“AWS SDK v3 introduces a modular, middleware-driven architecture with consistent APIs, native streaming, async pagination, abort control, and improved performance. It is designed for modern Node.js, serverless, and enterprise workloads.”

Q: Why SDK v3 for BFSI?
Because of modularity, middleware control, and IAM-based security.

Q: How do you prevent double debit?
Idempotency keys + DynamoDB conditional writes.

Q: How is audit handled?
CloudWatch logs + middleware + CloudTrail.

Q: How do you handle failures?
Retries → DLQ → manual reconciliation.

“A BFSI-grade system using AWS SDK v3 is built on IAM-secured Lambdas, atomic DynamoDB transactions, event-driven SQS processing, middleware-based auditing, and end-to-end observability to ensure security, correctness, and compliance.”


BFSI Use Case (Concrete Scenario)::
Use case:
💳 Real-time Payment Transaction Processing & Ledger System
(Think: card payment, UPI, loan repayment, wallet debit)

Non-negotiable BFSI requirements
🔐 Zero credential leakage
📜 Full audit trail
🔁 Idempotent transactions
⚖️ Strong consistency for ledger
🚨 Failure isolation & DLQ
📊 Observability (logs, metrics, traces)
🧾 Compliance (SOX, PCI-DSS, RBI, GDPR where applicable)

Client (Web / Mobile)
   ↓
API Gateway
   ↓
Lambda (Node.js, SDK v3)
   ↓
DynamoDB (Ledger – Strongly Consistent)
   ↓
SQS (Async Processing)
   ↓
Settlement / Notification Lambdas

| BFSI Need     | SDK v3 Advantage           |
| ------------- | -------------------------- |
| Security      | IAM role based auth        |
| Performance   | Modular, small cold start  |
| Observability | Middleware stack           |
| Reliability   | Retries + abort            |
| Compliance    | Deterministic API calls    |
| Scale         | Async paginators & streams |

“SDK v3 is ideal for BFSI workloads due to its modular design, explicit middleware control, and secure IAM-based access.”


Authentication & Security (CRITICAL)
🔐 Identity & Access
No access keys
IAM execution roles only
Least privilege policies

{
  "Effect": "Allow",
  "Action": ["dynamodb:PutItem","dynamodb:TransactWriteItems"],
  "Resource": "arn:aws:dynamodb:ap-south-1:*:table/ledger"
}

🔒 Network Security
Lambda inside VPC
VPC Endpoints for DynamoDB, SQS
No public internet for data plane

Transaction Flow (Step-by-Step)
1️⃣ API Gateway
JWT / OAuth validation
Rate limiting
Request validation

2️⃣ Lambda (Node.js + AWS SDK v3)
Idempotency check
Business validation
Ledger write (atomic)

3️⃣ DynamoDB (Ledger)
Strongly consistent reads
Transactional writes
Append-only ledger pattern

4️⃣ SQS (Async)
Settlement
Notifications
Reconciliation

SDK v3 – Core Lambda Code (Ledger Write)::

import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import {
  DynamoDBDocumentClient,
  TransactWriteCommand
} from "@aws-sdk/lib-dynamodb";

const ddb = DynamoDBDocumentClient.from(
  new DynamoDBClient({ region: "ap-south-1" })
);

export const handler = async (event) => {
  const { txnId, userId, amount } = JSON.parse(event.body);

  await ddb.send(new TransactWriteCommand({
    TransactItems: [
      {
        Put: {
          TableName: "ledger",
          Item: {
            pk: `USER#${userId}`,
            sk: `TXN#${txnId}`,
            amount,
            createdAt: Date.now()
          },
          ConditionExpression: "attribute_not_exists(sk)" // idempotency
        }
      }
    ]
  }));

  return { statusCode: 200, body: "SUCCESS" };
};

Why this is BFSI-safe::
✅ Atomic transaction
✅ Idempotent
✅ No double debit
✅ No race conditions

| Layer    | Technique          |
| -------- | ------------------ |
| API      | Idempotency key    |
| DynamoDB | Conditional writes |
| SQS      | De-duplication     |
| Lambda   | Request hash       |

“Idempotency is enforced at the data layer using conditional writes to guarantee exactly-once semantics.”

Middleware Stack (SDK v3 = POWER)::
Add Audit Logging Middleware
ddb.middlewareStack.add(
  (next) => async (args) => {
    console.log("AUDIT", {
      operation: args.input,
      timestamp: Date.now()
    });
    return next(args);
  },
  { step: "finalizeRequest" }
);

BFSI usage::
Audit logs
Request tracing
Compliance evidence

Error Handling & Resilience::
Retry & Abort
const controller = new AbortController();
setTimeout(() => controller.abort(), 2000);

await ddb.send(cmd, { abortSignal: controller.signal });

Failure strategy::
Retry (SDK default exponential backoff)
Send to DLQ
Alert via CloudWatch + PagerDuty

Observability (MANDATORY in BFSI)
| Tool               | Purpose    |
| ------------------ | ---------- |
| CloudWatch Logs    | Audit      |
| CloudWatch Metrics | SLA        |
| X-Ray              | Trace      |
| CloudTrail         | Compliance |


Data Consistency Strategy
| Data Type | Choice            |
| --------- | ----------------- |
| Ledger    | DynamoDB (strong) |
| Events    | SQS               |
| Reports   | Athena            |
| Archival  | S3 (WORM enabled) |


Compliance Checklist (Interview GOLD)::

✔ IAM least privilege
✔ No secrets in code
✔ Encrypted at rest & transit
✔ Append-only ledger
✔ Full audit trail
✔ Deterministic retries
✔ DLQ isolation

BFSI line:
“All financial mutations are append-only, auditable, and immutable.”

Scaling Strategy::
Lambda auto-scales
DynamoDB on-demand
SQS buffers spikes
No shared state