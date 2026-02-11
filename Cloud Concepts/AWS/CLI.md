What is AWS CLI actually called?
AWS Command Line Interface (AWS CLI)

Official tool by Amazon Web Services to manage AWS services from your terminal.

Is AWS CLI still used or deprecated?

✅ Very much used
❌ NOT deprecated

In fact:
AWS CLI v2 is the current standard
AWS CLI v1 → maintenance mode only

👉 If anyone asks in interviews:
“AWS CLI is actively used and maintained. AWS CLI v2 is the recommended version.”

Why AWS CLI exists (problem it solves)
Without CLI:
Click-heavy AWS Console
Manual, error-prone
Not automation-friendly

With CLI:
Scriptable
Automatable
CI/CD friendly
Infrastructure as code–friendly

How AWS CLI works (internal flow)
Think of it as terminal → AWS APIs

You (terminal)
   ↓
AWS CLI command
   ↓
AWS SDK (embedded)
   ↓
Signed HTTPS request (SigV4)
   ↓
AWS Service API (S3, EC2, IAM, etc.)
   ↓
JSON response

Key points:
CLI is just a thin wrapper over AWS APIs
Uses IAM credentials
Every command maps to an AWS API call


AWS CLI versions (important)
AWS CLI v1
Python-based
Separate install of Python
Legacy

AWS CLI v2 (current)
Bundled installer
Built-in auto-completion
SSO support
Better performance

👉 Always say “AWS CLI v2” in interviews

Core Features
Manage all AWS services
Works with IAM, SSO, MFA
Output formats: json | table | text
Shell auto-completion
Scriptable (bash, PowerShell, CI/CD)

Installing AWS CLI v2
aws --version
aws-cli/2.x.x Python/3.x

Configuring AWS CLI
aws configure
~/.aws/credentials
~/.aws/config

Using AWS SSO (enterprise standard)
aws configure sso
aws sso login

Most Important AWS CLI Commands (Interview + Real Life)
aws iam list-users
aws iam get-user

aws s3 ls
aws s3 mb s3://my-bucket
aws s3 cp file.txt s3://my-bucket/
aws s3 sync ./build s3://my-bucket

aws ec2 describe-instances
aws ec2 start-instances --instance-ids i-123
aws ec2 stop-instances --instance-ids i-123

aws lambda list-functions
aws lambda invoke --function-name myFn out.json

aws logs describe-log-groups
aws logs tail /aws/lambda/myFn --follow

aws ecr get-login-password \
 | docker login --username AWS --password-stdin <acct>.dkr.ecr.us-east-1.amazonaws.com


Output formatting:
aws ec2 describe-instances --output table
aws s3 ls --output text


Filtering with JMESPath:
aws ec2 describe-instances \
 --query "Reservations[].Instances[].InstanceId"


AWS CLI Use Cases (REAL)::
1️⃣ CI/CD pipelines
Build → upload to S3
Push Docker images to ECR
Trigger deployments

2️⃣ Infrastructure automation
Spin up EC2
Manage IAM roles
Rotate credentials

3️⃣ Debugging & ops
Tail CloudWatch logs
Inspect Lambda configs
Check S3 objects

4️⃣ Bulk operations
Sync files
Delete resources
Batch updates

AWS CLI Best Practices (VERY IMPORTANT)

✅ Use IAM Roles, not access keys
✅ Prefer SSO for enterprises
✅ Never commit ~/.aws/credentials
✅ Use least privilege policies
✅ Use --dry-run where available
✅ Combine with shell scripts
✅ Store secrets in AWS Secrets Manager


| Tool    | Purpose                             |
| ------- | ----------------------------------- |
| AWS CLI | Manual + automation                 |
| AWS SDK | Programmatic (Node.js, Java, etc.)  |
| AWS CDK | Infrastructure as code (high-level) |

👉 CLI = operations
👉 SDK = application code

3️⃣ How credentials work in Node.js
Priority order:
IAM Role (EC2 / Lambda)
Environment variables
Shared credentials (~/.aws)
Explicit config (not recommended)


Q: Does AWS CLI use SDK internally?
A: Yes, it wraps AWS APIs via SDK logic.

Q: Do we use AWS CLI inside Node.js apps?
A: No. Node.js apps use AWS SDK; CLI is for ops and automation.

Q: CLI vs Console?
A: CLI is faster, scriptable, and automation-friendly.


1️⃣ Signed HTTPS request (SigV4) — HOW AWS AUTH REALLY WORKS
SigV4 = AWS Signature Version 4
It proves who you are + what you’re allowed to do.

What is signed?
Every AWS CLI / SDK request signs:
HTTP method (GET, POST…)
URI
Query params
Headers
Request body
Timestamp
Region + service

Why?
Prevents request tampering
Prevents replay attacks
Ensures request is from trusted IAM identity

Request → Canonical Request
        → String to Sign
        → HMAC with Secret Key
        → Authorization Header
        → AWS API

“SigV4 cryptographically signs every AWS API request using IAM credentials, ensuring integrity, authenticity, and authorization.”

2️⃣ AWS APIs — what CLI & SDK actually call
AWS CLI does NOT do magic.
aws s3 ls

👇 internally calls
GET https://s3.us-east-1.amazonaws.com/

Every CLI command maps 1:1 to AWS public APIs:
IAM → iam.amazonaws.com
S3 → s3.amazonaws.com
EC2 → ec2.amazonaws.com

👉 CLI = API client


3️⃣ Least Privilege Policies (VERY IMPORTANT)
Give minimum permissions required to do the job — nothing more.

❌ BAD (admin)
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

✅ GOOD (least privilege)
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::my-bucket/*"
}

Why interviewers care
Security breaches = over-permissioned IAM
BFSI & healthcare mandate least privilege

Interview line
“IAM policies should follow least privilege to minimize blast radius in case of credential compromise.”

4️⃣ --dry-run — SAFE MODE IN AWS CLI
Used to validate permissions without executing.

Example
aws ec2 run-instances --image-id ami-123 --dry-run

Result:
❌ If permissions missing → UnauthorizedOperation
✅ If allowed → DryRunOperation

Where used
EC2 start/stop
Security group rules
Networking changes

Interview line

“Dry-run helps validate IAM permissions safely before making destructive changes.”


5️⃣ Shared credentials (~/.aws) — WILL THEY BE COMMITTED?
Short answer

❌ NO, if you follow best practices
Where credentials live
~/.aws/credentials
~/.aws/config


Example:
[default]
aws_access_key_id=AKIA...
aws_secret_access_key=...

Are they visible to users?
Only on your local machine
NOT pushed unless you commit them (big mistake)
How we prevent commits
Add to .gitignore:
.aws/

CI/CD?
CI never uses ~/.aws
Uses IAM Role or OIDC

Interview line
“Shared credentials are local-only and must never be committed; CI/CD should use IAM roles or OIDC instead.”


6️⃣ CI/CD PIPELINE EXAMPLE USING AWS CLI
Example: React + S3 + CloudFront
GitHub Actions pipeline

name: Deploy to S3

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123:role/github-deploy
          aws-region: us-east-1

      - run: npm install
      - run: npm run build

      - run: aws s3 sync build/ s3://my-app-bucket --delete

      - run: aws cloudfront create-invalidation \
              --distribution-id ABC123 \
              --paths "/*"

Why AWS CLI here?
Upload artifacts
Invalidate cache
Scriptable & fast

Interview line
“AWS CLI is commonly used in CI/CD for deployment, cache invalidation, and infrastructure orchestration.”

7️⃣ Lambda + Node.js + IAM Role — FULL FLOW (CRITICAL)
Architecture::
Node.js Lambda
   ↓
IAM Execution Role
   ↓
AWS SDK
   ↓
AWS Service (S3 / DynamoDB)

Step 1: IAM Role
{
  "Effect": "Allow",
  "Action": ["dynamodb:PutItem"],
  "Resource": "arn:aws:dynamodb:us-east-1:123:table/orders"
}

Attached to Lambda execution role


Step 2: Node.js Lambda code
import { DynamoDBClient, PutItemCommand } from "@aws-sdk/client-dynamodb";

const client = new DynamoDBClient({});

export const handler = async () => {
  await client.send(new PutItemCommand({
    TableName: "orders",
    Item: {
      id: { S: "123" }
    }
  }));
};

Step 3: Credential resolution
Lambda automatically injects IAM role
No keys
No config
No secrets

“In Lambda, credentials are provided via IAM execution roles, eliminating hardcoded secrets and improving security.”

Q1: What is AWS CLI?

AWS Command Line Interface to manage AWS services via terminal.

Q2: Is AWS CLI deprecated?

No. AWS CLI v2 is actively maintained.

Q3: How does AWS CLI authenticate?

Using IAM credentials and SigV4 signed HTTPS requests.

Q4: AWS CLI vs SDK?

CLI → automation & ops
SDK → application runtime code

Q5: Where are AWS CLI credentials stored?

~/.aws/credentials and ~/.aws/config

Q6: Should we use access keys in production?

No. Use IAM roles.

Q7: What is least privilege?

Granting only minimum required permissions.

Q8: What is --dry-run?

Permission validation without execution.

Q9: Does Node.js app use AWS CLI?

No. It uses AWS SDK.

Q10: AWS CLI in CI/CD?

Yes — deployment, infra ops, cache invalidation.
