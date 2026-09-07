# Authentication vs Authorization in AWS IAM
## Production-Focused Guide for Cloud & DevOps Engineers

<img width="965" height="881" alt="image" src="https://github.com/user-attachments/assets/c717da85-e990-4d16-aba5-ab27c65709de" />

---

# Table of Contents

1. Overview
2. Authentication vs Authorization
3. Why the Difference Matters in Production
4. AWS IAM Authentication Methods
5. AWS IAM Authorization Model
6. End-to-End Request Flow in AWS
7. Production Example: Accessing an Amazon S3 Bucket
8. Understanding `AccessDenied`
9. IAM Policy Evaluation Logic
10. IAM Policies Used in Production
11. Identity-Based vs Resource-Based Policies
12. Roles vs IAM Users
13. Cross-Account Access
14. Temporary Credentials and STS
15. MFA in Production
16. Service Accounts and Workload Authentication
17. Least Privilege
18. Permission Boundaries
19. Service Control Policies (SCPs)
20. Common Production Troubleshooting Scenarios
21. Security Best Practices
22. Monitoring and Auditing
23. Incident Troubleshooting Checklist
24. Architecture Flow
25. Interview Questions and Answers
26. Key Takeaways

---

# 1. Overview

Authentication and Authorization are two fundamental security concepts used in every production system.

Although they are often discussed together, they answer two completely different questions:

- **Authentication:** Who are you?
- **Authorization:** What are you allowed to do?

In AWS, these concepts are heavily used by:

- IAM Users
- IAM Roles
- AWS services
- Applications
- CI/CD pipelines
- Kubernetes workloads
- Lambda functions
- EC2 instances
- Cross-account access
- External identity providers

Understanding the difference is especially important when troubleshooting errors such as:

```text
AccessDenied
AccessDeniedException
UnauthorizedOperation
403 Forbidden
```

---

# 2. Authentication vs Authorization

## Authentication

Authentication verifies the identity of a user, application, service, or workload.

The main question is:

> **Who are you?**

Examples:

- Username and password
- Access key and secret access key
- Multi-Factor Authentication (MFA)
- SAML authentication
- OpenID Connect (OIDC)
- Temporary credentials
- IAM role credentials

If authentication succeeds, AWS knows the identity making the request.

---

## Authorization

Authorization determines what an authenticated identity is allowed to access or perform.

The main question is:

> **What are you allowed to do?**

Examples:

- Can the user read an S3 object?
- Can the application create an EC2 instance?
- Can a DevOps engineer update a Kubernetes cluster?
- Can a CI/CD pipeline deploy to production?
- Can an IAM role delete an RDS database?

Authorization is primarily controlled through:

- IAM policies
- Resource-based policies
- Service Control Policies (SCPs)
- Permission boundaries
- Session policies
- Explicit deny statements

---

# 3. Why the Difference Matters in Production

In production environments, successful authentication does not mean the user or workload has permission to perform every action.

For example:

A DevOps engineer successfully logs into AWS.

Authentication:

```text
AWS confirms the identity.
```

The engineer then tries to delete a production S3 bucket.

Authorization:

```text
AWS evaluates whether the engineer has permission to delete the bucket.
```

The engineer may be authenticated successfully but still receive:

```text
AccessDenied
```

This means:

```text
Authentication = Successful
Authorization = Failed
```

This distinction is extremely important during production troubleshooting.

---

# 4. AWS IAM Authentication Methods

AWS supports multiple authentication mechanisms depending on the type of identity.

## 4.1 IAM User Login

An IAM user can authenticate using:

- Username
- Password
- MFA

Typical use case:

```text
Human user → AWS Console → IAM authentication
```

For production environments, IAM users should generally not be the primary long-term access method for workforce access when federation or IAM Identity Center is available.

---

## 4.2 Access Keys

Applications or automation can authenticate using:

```text
Access Key ID
Secret Access Key
```

However, long-lived access keys create security risks.

Production best practice:

```text
Avoid long-lived access keys whenever possible.
Use IAM roles and temporary credentials.
```

---

## 4.3 IAM Roles

IAM roles are commonly used in production.

Examples:

- EC2 instance role
- Lambda execution role
- ECS task role
- EKS workload role
- CI/CD deployment role
- Cross-account role

IAM roles provide temporary credentials.

This reduces the need to store secrets inside:

- Source code
- Environment variables
- Configuration files
- CI/CD pipelines

---

## 4.4 Federated Authentication

Organizations commonly integrate AWS with an identity provider.

Examples:

- Microsoft Entra ID
- Okta
- Google Workspace
- Enterprise identity providers

Flow:

```text
Employee
   ↓
Corporate Identity Provider
   ↓
Authentication
   ↓
AWS IAM Identity Center / Federation
   ↓
Temporary AWS Credentials
```

This is preferred for enterprise production environments.

---

# 5. AWS IAM Authorization Model

AWS evaluates permissions when an authenticated identity makes an API request.

Example:

```text
User → AWS API → Request Authorization Check → Allow or Deny
```

A request may involve multiple authorization layers.

These include:

- Identity-based policies
- Resource-based policies
- Permission boundaries
- Organizations SCPs
- Session policies
- Service-specific authorization rules

The final decision depends on AWS policy evaluation logic.

---

# 6. End-to-End Request Flow in AWS

Consider the following request:

```text
DevOps Engineer
      ↓
Authentication
      ↓
Authenticated AWS Identity
      ↓
Request: GetObject from S3
      ↓
IAM Policy Evaluation
      ↓
Resource Policy Evaluation
      ↓
SCP / Permission Boundary Evaluation
      ↓
Explicit Deny?
      ↓
ALLOW or DENY
```

The important point is:

> Authentication happens first. Authorization determines whether the requested action is permitted.

---

# 7. Production Example: Accessing an Amazon S3 Bucket

Suppose a DevOps engineer wants to download a file from:

```text
s3://production-config-bucket/application/config.yaml
```

## Step 1: Authentication

The engineer authenticates through the corporate identity provider.

AWS verifies:

```text
Who is making this request?
```

Authentication succeeds.

---

## Step 2: Authorization

The engineer requests:

```text
s3:GetObject
```

AWS evaluates:

- IAM role permissions
- S3 bucket policy
- SCP
- Permission boundary
- Explicit deny rules

Example allowed policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::production-config-bucket/application/*"
    }
  ]
}
```

The engineer can read objects under the allowed path.

---

## Step 3: Attempting an Unauthorized Action

The engineer tries:

```text
s3:DeleteObject
```

If the permission does not exist:

```text
Request → DENIED
```

Result:

```text
AccessDenied
```

The identity is valid.

The requested action is not authorized.

---

# 8. Understanding AccessDenied in Production

When troubleshooting `AccessDenied`, do not immediately assume the credentials are incorrect.

First determine:

```text
Did authentication fail?
OR
Did authorization fail?
```

## Authentication Failure Examples

Possible symptoms:

```text
InvalidClientTokenId
SignatureDoesNotMatch
ExpiredToken
InvalidAccessKeyId
```

These may indicate:

- Invalid credentials
- Expired credentials
- Incorrect secret key
- Incorrect request signing
- Invalid token

---

## Authorization Failure Examples

Possible symptoms:

```text
AccessDenied
AccessDeniedException
UnauthorizedOperation
403 Forbidden
```

These usually indicate that:

```text
Identity is authenticated
BUT
Permission is not granted
```

---

# 9. IAM Policy Evaluation Logic

AWS authorization follows an important principle:

## Default Deny

Every request starts as:

```text
DENY
```

Unless a policy explicitly allows the action.

---

## Explicit Allow

A matching allow statement can grant permission.

Example:

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "*"
}
```

---

## Explicit Deny

An explicit deny overrides an allow.

Example:

```json
{
  "Effect": "Deny",
  "Action": "s3:DeleteObject",
  "Resource": "*"
}
```

Even if another policy allows deletion:

```text
Explicit Deny → Final Decision = DENY
```

---

## Simplified Policy Evaluation

```text
Start
  ↓
Explicit Deny?
  ├── Yes → DENY
  └── No
        ↓
Explicit Allow?
  ├── Yes → ALLOW
  └── No → Implicit DENY
```

---

# 10. IAM Policies Used in Production

## 10.1 Identity-Based Policies

Attached to:

- IAM users
- IAM groups
- IAM roles

Example:

```text
IAM Role
    ↓
Policy
    ↓
Allowed Actions
```

---

## 10.2 Resource-Based Policies

Attached directly to resources.

Examples:

- S3 bucket policies
- SQS queue policies
- SNS topic policies
- KMS key policies

Example:

```text
S3 Bucket
   ↓
Bucket Policy
   ↓
Defines who can access the bucket
```

---

# 11. Identity-Based vs Resource-Based Policies

| Identity-Based Policy | Resource-Based Policy |
|---|---|
| Attached to an IAM identity | Attached to a resource |
| Defines what identity can do | Defines who can access resource |
| IAM user or role permissions | S3 bucket, SQS queue, KMS key |
| Common for internal permissions | Common for cross-account access |

In production, both may be evaluated.

---

# 12. IAM Users vs IAM Roles

## IAM Users

IAM users represent long-term identities.

Potential credentials:

- Password
- Access keys

Production concerns:

- Long-lived credentials
- Key rotation requirements
- Higher secret management risk

---

## IAM Roles

IAM roles are preferred for workloads.

Examples:

```text
EC2 → IAM Role
Lambda → Execution Role
EKS Pod → IAM Role
GitHub Actions → OIDC → IAM Role
```

Benefits:

- Temporary credentials
- No hardcoded access keys
- Easier rotation
- Better security posture

Production recommendation:

> Prefer IAM roles and temporary credentials over long-lived IAM user access keys.

---

# 13. Cross-Account Access

Production organizations commonly separate AWS accounts.

Example:

```text
Development Account
Staging Account
Production Account
Security Account
Shared Services Account
```

A DevOps engineer may need temporary access to the production account.

Flow:

```text
User authenticates
      ↓
Assumes IAM Role
      ↓
Receives temporary credentials
      ↓
Accesses Production Account
```

The target role uses a trust policy.

Authorization then determines what actions the assumed role can perform.

---

# 14. Temporary Credentials and AWS STS

AWS Security Token Service (STS) provides temporary credentials.

Temporary credentials include:

- Access Key ID
- Secret Access Key
- Session Token

They expire after a configured duration.

Production benefits:

- Reduced risk from leaked credentials
- Automatic expiration
- Easier short-term access
- Better for CI/CD and workloads

Common STS use cases:

```text
AssumeRole
AssumeRoleWithSAML
AssumeRoleWithWebIdentity
```

---

# 15. MFA in Production

Multi-Factor Authentication adds an additional authentication factor.

Example:

```text
Password
   +
MFA Code
   ↓
Authentication
```

MFA should be strongly enforced for:

- Root account access
- Privileged users
- Administrative operations

For sensitive production actions, organizations may also require MFA-based authorization conditions.

---

# 16. Service Accounts and Workload Authentication

Production applications should not use personal IAM credentials.

Bad design:

```text
Application
   ↓
Hardcoded AWS Access Key
```

Better design:

```text
Application
   ↓
IAM Role
   ↓
Temporary Credentials
```

Examples:

## EC2

```text
Application
   ↓
EC2 Instance Profile
   ↓
IAM Role
```

## Lambda

```text
Lambda Function
   ↓
Execution Role
```

## EKS

```text
Kubernetes Pod
   ↓
OIDC / Pod Identity
   ↓
IAM Role
```

This approach separates application identity from human identity.

---

# 17. Least Privilege

Least privilege means granting only the permissions required.

Bad policy:

```json
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}
```

Better approach:

```text
Allow only required:
- Actions
- Resources
- Conditions
```

Example:

```text
Application Role
    ↓
Read only required S3 bucket
    ↓
Cannot access other production buckets
```

This reduces the blast radius if credentials are compromised.

---

# 18. Permission Boundaries

Permission boundaries define the maximum permissions an IAM identity can receive.

Think of them as:

```text
Maximum Permission Limit
```

Even if an IAM policy allows an action, the permission boundary can restrict it.

Example:

```text
IAM Policy → Allows EC2 creation
Permission Boundary → Does not allow EC2 creation
Final Result → DENY
```

---

# 19. Service Control Policies (SCPs)

SCPs are used with AWS Organizations.

They define permission boundaries at the organizational level.

Example:

```text
Organization
      ↓
SCP
      ↓
AWS Account
      ↓
IAM Roles and Users
```

An SCP can deny an action even when the IAM role allows it.

Example:

```text
IAM Role → Allows s3:DeleteBucket
SCP → Denies s3:DeleteBucket
Final Result → DENY
```

Production troubleshooting must consider SCPs when permissions appear correct.

---

# 20. Common Production Troubleshooting Scenarios

## Scenario 1: User Can Log In but Cannot Access S3

Symptoms:

```text
AWS Console Login Successful
S3 Access → AccessDenied
```

Troubleshooting:

1. Verify identity
2. Check IAM policies
3. Check bucket policy
4. Check explicit denies
5. Check SCP
6. Check permission boundary

---

## Scenario 2: EC2 Application Cannot Access Secrets

Possible causes:

- Missing IAM role
- Incorrect instance profile
- Missing Secrets Manager permission
- KMS permission missing
- Resource policy restriction

Troubleshooting flow:

```text
EC2 Role
   ↓
IAM Policy
   ↓
Secrets Manager Permission
   ↓
KMS Permission
   ↓
Resource Policy
```

---

## Scenario 3: CI/CD Pipeline Gets AccessDenied

Common causes:

- Incorrect IAM role
- Missing `sts:AssumeRole`
- Missing trust policy
- SCP restriction
- Session policy restriction
- Resource policy deny

Important checks:

```text
Who is making the request?
Which role is being assumed?
What action is being performed?
Which policy is denying the request?
```

---

## Scenario 4: Kubernetes Pod Cannot Access AWS Resources

Example:

```text
EKS Pod → S3 → AccessDenied
```

Possible causes:

- Pod has no AWS IAM role
- Incorrect OIDC trust relationship
- Incorrect service account
- Missing IAM permission
- Incorrect resource policy

Troubleshooting flow:

```text
Pod
 ↓
Kubernetes Service Account
 ↓
IAM Role Association
 ↓
Temporary Credentials
 ↓
IAM Policy
 ↓
AWS Resource
```

---

# 21. Production Security Best Practices

## Human Access

Prefer:

```text
Corporate Identity Provider
      ↓
AWS IAM Identity Center
      ↓
IAM Roles
```

Avoid:

```text
Shared IAM Users
Long-lived access keys
Root account usage
```

---

## Workload Access

Prefer:

```text
IAM Roles
Temporary Credentials
OIDC Federation
Workload Identity
```

Avoid:

```text
Hardcoded credentials
Credentials in Git repositories
Credentials in container images
Long-lived production keys
```

---

## Apply Least Privilege

Grant:

```text
Minimum Action
+
Minimum Resource
+
Required Conditions
```

Review permissions regularly.

---

## Separate Production Access

Use:

- Separate AWS accounts
- Separate IAM roles
- Approval workflows
- Short-lived privileged access
- MFA for sensitive operations

---

# 22. Monitoring and Auditing

Production environments should log authentication and authorization events.

Important AWS services:

## AWS CloudTrail

CloudTrail records AWS API activity.

Useful for:

- Who made the request?
- Which role was used?
- Which action was performed?
- When did it happen?
- Which resource was accessed?

---

## IAM Access Analyzer

Useful for identifying:

- Public access
- Cross-account access
- Unintended permissions

---

## CloudWatch

Useful for:

- Application authentication failures
- AccessDenied errors
- Security events
- Operational alerts

---

# 23. Production Incident Troubleshooting Checklist

When you receive an AWS access issue, follow this sequence.

## Step 1: Identify the Requesting Identity

Ask:

```text
Who is making the request?
```

Examples:

- IAM User
- IAM Role
- EC2 Instance Role
- Lambda Execution Role
- EKS Pod Role
- CI/CD Role

---

## Step 2: Confirm Authentication

Check:

```text
Are the credentials valid?
Are the credentials expired?
Is the correct role being used?
```

---

## Step 3: Identify the Exact Action

Example:

```text
s3:GetObject
ec2:DescribeInstances
secretsmanager:GetSecretValue
```

Do not troubleshoot permissions without knowing the exact action.

---

## Step 4: Identify the Target Resource

Example:

```text
Which S3 bucket?
Which secret?
Which KMS key?
Which AWS account?
```

---

## Step 5: Evaluate IAM Permissions

Check:

```text
Identity-based policy
Resource-based policy
Explicit deny
Permission boundary
SCP
Session policy
```

---

## Step 6: Check CloudTrail

Use CloudTrail to determine:

```text
Who made the request?
What action was denied?
Which resource was targeted?
Which role was involved?
```

---

# 24. Production Architecture Flow

A secure production access flow should look like:

```text
                ┌──────────────────────┐
                │   User / Workload    │
                └──────────┬───────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │   Authentication     │
                │  Who are you?        │
                └──────────┬───────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │ Authenticated        │
                │ Identity             │
                └──────────┬───────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │   Authorization      │
                │ What can you do?     │
                └──────────┬───────────┘
                           │
                           ▼
        ┌──────────────────────────────────┐
        │ IAM Policy Evaluation            │
        │ Resource Policy                  │
        │ Permission Boundary              │
        │ SCP                              │
        │ Explicit Deny                    │
        └────────────────┬─────────────────┘
                         │
                ┌────────┴────────┐
                ▼                 ▼
             ALLOW              DENY
                │                 │
                ▼                 ▼
         Resource Access     AccessDenied
```

---

# 25. Interview Questions and Answers

## Q1. What is the difference between Authentication and Authorization?

**Answer:**

Authentication verifies the identity of a user or workload. It answers:

> Who are you?

Authorization determines what an authenticated identity is allowed to access. It answers:

> What are you allowed to do?

In AWS, credentials are used for authentication, while IAM policies and permissions are primarily used for authorization.

---

## Q2. Can Authentication Succeed but Authorization Fail?

Yes.

Example:

```text
User logs in successfully
        ↓
Authentication succeeds
        ↓
User tries to delete production S3 bucket
        ↓
Permission missing
        ↓
Authorization fails
        ↓
AccessDenied
```

---

## Q3. Why Does an IAM User Get AccessDenied Even Though the Policy Allows Access?

Possible reasons include:

- Explicit deny
- SCP restriction
- Permission boundary
- Resource-based policy restriction
- Missing required conditions
- KMS permissions
- Session policy restrictions

An explicit deny always overrides an allow.

---

## Q4. What Is the Difference Between an IAM User and IAM Role?

IAM users usually represent long-term identities.

IAM roles provide temporary credentials and are commonly used for:

- AWS services
- Applications
- CI/CD pipelines
- Cross-account access

In production, IAM roles are generally preferred for workloads.

---

## Q5. How Would You Troubleshoot AccessDenied in Production?

My approach would be:

```text
Identify Identity
      ↓
Verify Authentication
      ↓
Identify Exact Action
      ↓
Identify Target Resource
      ↓
Check IAM Policy
      ↓
Check Resource Policy
      ↓
Check Permission Boundary
      ↓
Check SCP
      ↓
Check Explicit Deny
      ↓
Check CloudTrail
```

---

## Q6. What Is the Principle of Least Privilege?

Least privilege means giving users or workloads only the permissions they actually need.

For example:

Instead of:

```text
s3:*
Resource: *
```

Grant:

```text
Only required S3 actions
Only required bucket
Only required object path
```

This reduces security risks and limits the blast radius.

---

## Q7. Why Are IAM Roles Preferred in Production?

IAM roles provide:

- Temporary credentials
- Automatic credential rotation
- Reduced secret exposure
- No hardcoded credentials
- Better integration with AWS workloads

---

# 26. Key Takeaways

Remember these two questions:

## Authentication

```text
WHO are you?
```

Examples:

- Password
- MFA
- Access keys
- SAML
- OIDC
- Temporary credentials

---

## Authorization

```text
WHAT are you allowed to do?
```

Examples:

- IAM policies
- Resource policies
- SCPs
- Permission boundaries
- Session policies

---

## Production Troubleshooting Rule

When an AWS request fails, always determine:

```text
1. Did authentication fail?
```

or:

```text
2. Did authentication succeed but authorization fail?
```

Then trace the authorization layers systematically.

---

# Final Summary

```text
Authentication
      ↓
WHO ARE YOU?
      ↓
Identity Verified
      ↓
Authorization
      ↓
WHAT CAN YOU DO?
      ↓
Policy Evaluation
      ↓
ALLOW or DENY
```

The most important production lesson is:

> **Successful authentication does not guarantee access. An identity must also be authorized for the requested action.**

This concept is essential for:

- AWS production environments
- DevOps troubleshooting
- CI/CD security
- Kubernetes workload access
- Cross-account access
- IAM design
- Cloud security
- AWS interviews

---

**Author: Manish Patil**  
**ManishCareerTalks**  
**AI & Cloud Engineer | Content Creator | Career Growth Advocate**
