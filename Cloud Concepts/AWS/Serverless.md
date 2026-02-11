What is Serverless Framework?

Serverless Framework is a third-party, cloud-agnostic framework to build, deploy, and operate serverless applications using a simple config file (serverless.yml).

Built by Serverless Inc

Supports AWS, Azure, GCP

On AWS, it generates CloudFormation

Interview one-liner:
“Serverless Framework is a multi-cloud abstraction layer for deploying serverless applications.”

How Serverless Framework Works (High Level)::

serverless.yml
   ↓ serverless deploy
Framework Engine
   ↓
CloudFormation (AWS)
   ↓
Lambda / API Gateway / SQS / DynamoDB


serverless.yml — Core Building Blocks
This is the heart of the framework.

1️⃣ service

Defines the application name (logical boundary).

service: payments-ledger


Used in stack names

Used in logs

Used in dashboards

2️⃣ provider

Defines cloud provider + runtime settings.

provider:
  name: aws
  runtime: nodejs18.x
  region: ap-south-1
  stage: dev


Includes:

Runtime

Region

IAM

Environment variables

3️⃣ functions

Defines Lambda functions.

functions:
  debit:
    handler: handler.debit


Each function maps to:

One Lambda

One IAM role (by default shared)

4️⃣ events

Defines triggers for functions.

HTTP (API Gateway)
events:
  - http:
      path: /debit
      method: post

SQS
events:
  - sqs:
      arn: arn:aws:sqs:ap-south-1:123:payments

Schedule
events:
  - schedule: rate(5 minutes)

5️⃣ resources

For raw CloudFormation when Serverless abstractions are not enough.

resources:
  Resources:
    LedgerTable:
      Type: AWS::DynamoDB::Table


Used when:

Fine-grained IAM

Custom networking

BFSI compliance controls

6️⃣ plugins

Extend functionality (VERY powerful).

plugins:
  - serverless-offline
  - serverless-dotenv-plugin


Common plugins:

serverless-offline → local dev

serverless-prune-plugin → clean old versions

serverless-webpack → bundling

⚠️ BFSI caution: plugins = third-party code.

7️⃣ stages

Multiple environments: dev / qa / prod.

serverless deploy --stage prod

provider:
  stage: ${opt:stage, 'dev'}


Used for:

Env isolation

Separate stacks

Controlled rollouts

8️⃣ variables (Very Important)

Serverless has powerful variable resolution.

bucket: ${self:service}-${sls:stage}


Sources:

self

opt

env

file()

ssm

Example (secure):

DB_PASSWORD: ${ssm:/prod/db/password}

9️⃣ Packaging

Controls what gets deployed.

package:
  individually: true
  exclude:
    - node_modules/**


Why important:

Smaller Lambda size

Faster cold starts

Lower attack surface

🔟 Deployment
Deploy everything
serverless deploy

Deploy single function
serverless deploy function -f debit


Uses:
CloudFormation
Change sets
Rollbacks on failure

1️⃣1️⃣ Logs & Monitoring
View logs
serverless logs -f debit --tail


Under the hood:
CloudWatch Logs
Per-function log groups
Can integrate with:
X-Ray
Datadog
New Relic

1️⃣2️⃣ Serverless Dashboard
Optional SaaS by Serverless Inc.
Features:
Deploy tracking
Error monitoring
Performance metrics
Team collaboration

⚠️ BFSI note:
Many banks disable Dashboard due to data residency & audit concerns.

Serverless Framework — BFSI Perspective
✅ Strengths
Fast development
Multi-cloud support
Excellent local dev
Simple YAML

⚠️ Risks
Third-party dependency
Plugin security
Harder audits
Less IAM control than CDK

Interview line:
“Serverless Framework is productive, but BFSI teams often prefer AWS-native tools like SAM or CDK for compliance and auditability.”

| Aspect      | Serverless FW | SAM    | CDK   |
| ----------- | ------------- | ------ | ----- |
| Owner       | Third-party   | AWS    | AWS   |
| Cloud       | Multi         | AWS    | AWS   |
| Language    | YAML          | YAML   | Code  |
| IAM control | Medium        | Medium | High  |
| BFSI fit    | ⭐⭐⭐           | ⭐⭐⭐⭐   | ⭐⭐⭐⭐⭐ |


Real Production Pattern (Correct Usage)
serverless.yml  → Infrastructure
AWS SDK v3      → Runtime logic
IAM Roles       → Security
CloudWatch      → Audit


Never:
Hardcode credentials
Overuse plugins
Use wildcards in IAM

“Serverless Framework is a third-party, multi-cloud serverless deployment framework that uses serverless.yml to define services, functions, events, and resources. While it excels in developer productivity and local development, regulated environments like BFSI often prefer AWS-native alternatives such as SAM or CDK for stronger security controls and audit compliance.”

Serverless Framework – Core Commands

Built by Serverless Inc

1️⃣ Project Initialization
Create a new service
serverless create --template aws-nodejs --path payments-ledger


Common templates:

aws-nodejs

aws-nodejs-typescript

aws-python

aws-java-maven

📌 Used at project start

2️⃣ Configuration & Validation
Validate serverless.yml
serverless print


Shows:

Fully resolved variables

Actual config sent to CloudFormation

✅ Very important in BFSI audits

3️⃣ Deployment Commands
Deploy full stack
serverless deploy

Deploy to specific stage
serverless deploy --stage prod

Deploy single function (fast iteration)
serverless deploy function -f debit


📌 Uses CloudFormation change sets

4️⃣ Remove / Destroy Stack
serverless remove


⚠️ BFSI warning

This deletes ALL resources

Often disabled in prod via IAM

5️⃣ Local Development
Start local API Gateway
serverless offline


(Plugin: serverless-offline)

Invoke function locally
serverless invoke local -f debit


Uses:

Local Node.js runtime

Mocked AWS context

6️⃣ Function Invocation
Invoke deployed Lambda
serverless invoke -f debit

Invoke with payload
serverless invoke -f debit --data '{"amount":100}'


Used for:

Smoke tests

Debugging

7️⃣ Logs & Monitoring
View logs
serverless logs -f debit

Tail logs (live)
serverless logs -f debit --tail


Under the hood:

CloudWatch Logs

📌 Mandatory for production debugging

8️⃣ Info & Status
Show deployed endpoints
serverless info


Shows:

API Gateway URLs

Lambda ARNs

Region & stage

9️⃣ IAM & Credentials
Configure AWS credentials
serverless config credentials \
  --provider aws \
  --key AKIA... \
  --secret xxxx


⚠️ BFSI BEST PRACTICE

Avoid static keys

Use IAM roles / SSO instead

🔟 Packaging
Package without deploying
serverless package


Generates:

.serverless/ folder

CloudFormation template

Used in:

CI validation

Security reviews

1️⃣1️⃣ Plugins Management
Install plugin
serverless plugin install -n serverless-offline

List plugins
serverless plugin list


⚠️ BFSI:

Plugins = third-party code

Must be security-reviewed

1️⃣2️⃣ Variables & Stages
Pass stage dynamically
serverless deploy --stage qa

Pass custom variables
serverless deploy --param="tableName=ledger"

1️⃣3️⃣ Debugging & Troubleshooting
Verbose output
SLS_DEBUG=* serverless deploy


Very useful when:

CloudFormation fails

IAM permission issues

1️⃣4️⃣ CI/CD Friendly Commands (Common)
serverless package
serverless deploy --stage prod
serverless info


Often combined with:

GitHub Actions

Jenkins

GitLab CI

| Command                | Purpose         |
| ---------------------- | --------------- |
| `serverless create`    | Create service  |
| `serverless deploy`    | Deploy stack    |
| `serverless deploy -f` | Deploy function |
| `serverless remove`    | Delete stack    |
| `serverless offline`   | Local API       |
| `serverless invoke`    | Run Lambda      |
| `serverless logs`      | View logs       |
| `serverless info`      | Stack info      |
| `serverless print`     | Resolve config  |
| `serverless package`   | Build only      |

BFSI Interview One-Liners (Use These)

Deploy:
“We use serverless deploy backed by CloudFormation for controlled rollouts.”

Debug:
“serverless print helps verify resolved variables before deployment.”

Security:
“In BFSI, stack deletion via serverless remove is restricted in production.”

“Serverless Framework provides a rich CLI to create, deploy, test, and monitor serverless applications. While highly productive, in BFSI environments it’s used cautiously with strict IAM controls, minimal plugins, and strong CI/CD governance.”

Basics (Foundational)
Q1. What is Serverless Framework?

Answer:
Serverless Framework is a third-party, cloud-agnostic framework that simplifies building, deploying, and operating serverless applications using a declarative serverless.yml configuration.

Q2. Is Serverless Framework AWS-only?

Answer:
No. It supports multiple cloud providers including AWS, Azure, and GCP. On AWS, it generates CloudFormation templates.

Q3. What is serverless.yml?

Answer:
It’s the main configuration file that defines the service, provider, functions, events, resources, plugins, and deployment settings for a serverless application.

Q4. Does Serverless Framework replace CloudFormation?

Answer:
No. On AWS, it uses CloudFormation under the hood to create and manage resources.

Core Concepts
Q5. What is a service in Serverless Framework?

Answer:
A service is the logical unit of deployment. It maps to one CloudFormation stack and groups related functions and resources.

Q6. What is the role of the provider block?

Answer:
It defines the cloud provider (AWS), runtime, region, stage, IAM settings, and global environment configuration.

Q7. How are Lambda functions defined?

Answer:
Using the functions section, where each function specifies a handler and one or more event triggers.

Q8. What are events?

Answer:
Events define what triggers a function, such as API Gateway, SQS, SNS, schedules, or streams.

Q9. What is the resources section used for?

Answer:
To define raw CloudFormation resources when Serverless abstractions are insufficient or when fine-grained control is needed.

Deployment & Operations
Q10. How do you deploy a Serverless app?

Answer:
Using serverless deploy, which packages the app and applies changes through CloudFormation.

Q11. How do you deploy a single function?

Answer:
serverless deploy function -f functionName

Q12. How do you check deployed endpoints?

Answer:
serverless info

Q13. How do you view logs?

Answer:
Using serverless logs -f functionName (CloudWatch Logs underneath).

Variables & Configuration
Q14. How does variable resolution work?

Answer:
Serverless supports dynamic variables from sources like self, opt, env, file, and ssm.

Q15. How do you manage multiple environments?

Answer:
Using stages (dev, qa, prod) and passing them via --stage during deployment.

Plugins & Extensibility
Q16. What are plugins?

Answer:
Plugins extend Serverless Framework functionality, such as local development, bundling, or pruning old versions.

Q17. Risks of plugins in enterprise/BFSI?

Answer:
They introduce third-party code, increasing security, compliance, and audit risks.

Local Development & Testing
Q18. How do you test locally?

Answer:
Using serverless-offline for APIs or serverless invoke local for functions.

Q19. Is local execution identical to AWS?

Answer:
No. It simulates the environment but doesn’t fully replicate AWS networking, IAM, or service integrations.

Security & IAM (Very Important)
Q20. How is IAM handled in Serverless Framework?

Answer:
IAM permissions are defined at the provider level or via iamRoleStatements. By default, functions share a role.

Q21. What is a common IAM pitfall?

Answer:
Using wildcard permissions (*) or over-privileged roles.

Q22. How do you handle secrets securely?

Answer:
Using AWS Secrets Manager or SSM Parameter Store, not environment variables or hardcoded values.

BFSI / Enterprise Perspective
Q23. Is Serverless Framework suitable for BFSI?

Answer:
Yes for non-core workloads, but AWS-native tools like SAM or CDK are often preferred for compliance and auditability.

Q24. Why do BFSI teams prefer CDK over Serverless Framework?

Answer:
CDK offers stronger typing, reusable constructs, tighter IAM control, and better alignment with enterprise security standards.

Q25. What are audit concerns with Serverless Framework?

Answer:
Third-party dependency, plugin security, abstraction hiding generated IAM policies, and external dashboards.

Advanced / Troubleshooting
Q26. How do you debug failed deployments?

Answer:
Use serverless print, CloudFormation console, and SLS_DEBUG=* serverless deploy.

Q27. What happens during rollback?

Answer:
CloudFormation automatically rolls back to the previous stable stack state.

Q28. How do you reduce Lambda cold starts?

Answer:
Smaller packages, fewer dependencies, individual packaging, and avoiding large plugins.

Comparison Questions (Common)
Q29. Serverless Framework vs AWS SAM?

Answer:
Serverless Framework is multi-cloud and third-party; SAM is AWS-native and CloudFormation-based.

Q30. Serverless Framework vs CDK?

Answer:
Serverless Framework focuses on simplicity and speed; CDK provides more control and is preferred for complex, regulated systems.

“Serverless Framework is a productive, multi-cloud serverless deployment tool built on CloudFormation. While excellent for rapid development and non-core services, enterprise and BFSI systems often prefer AWS-native tools like SAM or CDK for tighter security, IAM control, and audit compliance.”