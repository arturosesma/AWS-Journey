# IAM Primitives

## Users

An **IAM user** is an identity you create inside an AWS account to represent a **single person or application** that needs to interact with AWS. It is one of the four IAM primitives (users, groups, roles, policies).

### Key characteristics

- **Long-term credentials**: a user can have a console password and/or up to two **access keys** (access key ID + secret access key) for the CLI/SDK/API. These do not expire until rotated or deleted.
- **No permissions by default**: a new user can do nothing (implicit deny) until you attach policies, either directly or through a group.
- **Identified by an ARN and a unique ID**: `arn:aws:iam::123456789012:user/alice`.
- **Scoped to one account**: an IAM user exists only in the account where it was created. Users are global to the account (not tied to a Region).
- **MFA can be enabled** per user (virtual MFA, FIDO2 security key, hardware token).
- **Limit**: 5,000 IAM users per account.

### Users vs. root user

| | Root user | IAM user |
|---|---|---|
| Created | Automatically with the account | Manually, by an admin |
| Permissions | Unrestricted (full access) | Only what policies grant |
| Use | Only for the few root-only tasks | Daily operations (if you use users at all) |

### Permissions for users

1. **Identity-based policies** attached directly to the user (works, but doesn't scale).
2. **Group membership**: the recommended way to manage permissions for many users (a user can belong to multiple groups; groups cannot be nested).
3. **Permissions boundary**: an optional cap on the maximum permissions the user can ever have.

### Exam takeaways (SAA-C03)

- IAM users are the **legacy / last-resort** option. For **humans**, the preferred answer is **IAM Identity Center** (SSO, federated, temporary credentials). For **applications and AWS services** (EC2, Lambda, ECS), the answer is **IAM roles**, never users with hard-coded access keys.

- Use an IAM user only when the scenario truly needs long-term credentials (e.g., a third-party tool or on-prem system that cannot assume a role and cannot use IAM Roles Anywhere).

- Best practices when you do use them: enforce **MFA**, apply **least privilege**, put users in **groups**, **rotate** access keys, delete unused credentials (check the **credential report** and **Access Advisor**).

- Anti-patterns: sharing one user between people, embedding access keys in code or environment variables, using the root user for daily work.

## Groups

An **IAM group** is a **collection of IAM users**. Its purpose is to manage permissions for many users at once: attach a policy to the group and every member gets those permissions.


### Key characteristics

- **Only contains users**: a group cannot contain roles, other groups or the root user. **Groups cannot be nested.**
- **Not a principal**: a group cannot be referenced as a `Principal` in a resource-based policy (e.g., an S3 bucket policy) and cannot authenticate or sign in. It only carries permissions for its members.
- **A user can belong to multiple groups** (up to 10 groups per user), and permissions from all of them are combined (union).
- **A user does not have to belong to any group**, although that is discouraged.
- **No credentials of its own**: users keep their own passwords and access keys.
- **Permissions**: you attach **identity-based policies** (AWS managed, customer managed or inline) to the group.
- **Scoped to one account**, and global (not tied to a Region).
- **Limit**: 300 groups per account (default quota).

### How permissions are evaluated

A user's effective permissions are the combination of all policies attached to the user **and** to every group the user belongs to. Standard evaluation rules still apply: an **explicit Deny** in any of them overrides any Allow, and anything not allowed is an **implicit deny**.

### Typical example

```
Group: Developers   → policy: AmazonEC2FullAccess
Group: Auditors     → policy: SecurityAudit
User alice          → member of Developers
User bob            → member of Developers + Auditors  (gets both sets of permissions)
```

When alice moves to another team, you change her group membership; you don't edit any policy.

### Exam takeaways (SAA-C03)

- Groups are the **recommended way to assign permissions to IAM users** instead of attaching policies to each user (easier to scale and audit).
- Group by **job function** (Developers, Admins, Auditors), not by individual.
- Groups **cannot be nested** and **cannot be a Principal** in a resource-based policy; to grant access to a set of identities there, list the users/roles or use conditions such as `aws:PrincipalOrgID`.
- Groups only apply to **IAM users**. For roles and federated users, use **IAM Identity Center** groups (permission sets) or role-based access.
- To grant temporary access, prefer a **role**, not a new group.

## Roles

An **IAM role** is an identity with permissions that can be **assumed by anyone or anything that needs it**, instead of being tied to one person. It has **no password and no long-term access keys**: when a principal assumes the role, AWS STS issues **temporary credentials** (access key, secret key and session token).

### Key characteristics

- **Assumed, not signed into**: a principal calls `sts:AssumeRole` (or a variant) and receives temporary credentials that expire (default 1 hour; configurable from 15 minutes up to 12 hours).
- **Two kinds of policies**:
  - **Trust policy** (a resource-based policy on the role): defines **who can assume** the role (the `Principal`).
  - **Permission policies** (identity-based): define **what the role can do** once assumed.
- **Who can assume a role**:
  - AWS services (EC2, Lambda, ECS tasks, Glue...)
  - IAM users or roles in the **same account**
  - IAM users or roles in **another account** (cross-account access)
  - **Federated identities**: SAML 2.0, OIDC (GitHub Actions, EKS IRSA), Cognito identity pools, IAM Identity Center
  - On-prem workloads via **IAM Roles Anywhere**
- **Scoped to one account**, and global (not tied to a Region).
- **A role is both an identity and a resource**: it has permissions (identity) and a trust policy controlling access to it (resource).

### How assuming a role works

```
Principal (EC2, user, federated identity...)
   │  1. sts:AssumeRole  (allowed by the role's trust policy)
   ▼
AWS STS
   │  2. returns temporary credentials
   ▼
Principal acts with the ROLE's permissions (not its own)
   │  3. calls S3, DynamoDB, etc.
   ▼
AWS resource
```

### Common role types

| Type | Purpose |
|---|---|
| **Service role** (e.g., Lambda execution role, EC2 instance profile) | Lets an AWS service act on your behalf |
| **Service-linked role** | Predefined by an AWS service; you can't edit its permissions |
| **Cross-account role** | Grants access to resources in another account (use an **External ID** for third parties) |
| **Federated role** | Gives externally authenticated users (SSO, SAML, OIDC) AWS access |

### Users vs. roles

| | IAM user | IAM role |
|---|---|---|
| Credentials | Long-term (password, access keys) | Temporary (via STS) |
| Tied to | One person/app | Whoever assumes it |
| Best for | Last resort | Applications, services, cross-account, federation |

### Exam takeaways (SAA-C03)

- **Roles are almost always the right answer** for applications and AWS services: EC2 accessing S3, Lambda accessing DynamoDB. Never store access keys on an instance or in code.
- For EC2, the role is delivered through an **instance profile**; use **IMDSv2** to protect the credentials from SSRF.
- **Cross-account access**: create a role in the target account whose trust policy allows the other account, and the caller needs permission for `sts:AssumeRole`.
- **Trust policy = who can assume; permission policy = what they can do.** Questions often test which one is missing.
- Third-party access uses **External ID** to prevent the confused deputy problem.
- **Role chaining** (assuming a role from a role) limits the session to a maximum of **1 hour**.
- Giving a service a role requires `iam:PassRole`, which is why that permission must be tightly scoped.

## Policies

An **IAM policy** is a **JSON document that defines permissions**: which **actions** are allowed or denied, on which **resources**, and under which **conditions**. A policy does nothing until it is **attached** to an identity (user, group, role) or a resource, and AWS evaluates it on every request.

### Policy types (overview)

| Type | Attached to | Grants permissions? |
|---|---|---|
| **Identity-based** | User, group or role | Yes |
| **Resource-based** (bucket policy, trust policy, KMS key policy) | A resource | Yes |
| **Permissions boundary** | User or role | No, sets the maximum |
| **SCP / RCP** (Organizations) | Account or OU | No, sets the maximum |
| **Session policy** | A temporary session | No, limits the session |
| **ACL** | A resource (legacy) | Yes (cross-account, legacy) |

Identity-based policies can be **AWS managed** (created by AWS), **customer managed** (yours, reusable) or **inline** (embedded in one identity, 1:1).

### Anatomy of a policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadOnMyBucket",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111122223333:role/AppRole" },
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"],
      "Condition": {
        "Bool": { "aws:SecureTransport": "true" }
      }
    }
  ]
}
```

| Element | Required | Meaning |
|---|---|---|
| **`Version`** | Yes | Policy language version. Always use `"2012-10-17"` (the older `2008-10-17` lacks policy variables). |
| **`Statement`** | Yes | One statement or an array of them. Each statement is an independent permission rule. |
| **`Sid`** | No | Statement ID: a free-text label to describe the statement. Must be unique within the policy. |
| **`Effect`** | Yes | `Allow` or `Deny`. Everything is denied by default (implicit deny); an explicit `Deny` always wins. |
| **`Principal`** | Only in resource-based policies | **Who** the statement applies to (account, user, role, service). Not used in identity-based policies because the principal is whoever it is attached to. |
| **`Action`** | Yes (or `NotAction`) | **What** can be done, as `service:Operation`, e.g., `s3:GetObject`. Wildcards allowed (`s3:*`, `ec2:Describe*`). |
| **`NotAction`** | Alternative | Matches everything **except** the listed actions. Typically used with `Deny` (e.g., region whitelists). |
| **`Resource`** | Yes (or `NotResource`) | **On which** resources, by ARN. Wildcards allowed (`arn:aws:s3:::my-bucket/*`). Some actions only support `"*"`. |
| **`NotResource`** | Alternative | Matches every resource **except** the listed ones. |
| **`Condition`** | No | **When** the statement applies: `Operator: { ConditionKey: Value }`, e.g., `StringEquals`, `IpAddress`, `Bool`. Uses keys such as `aws:SourceIp`, `aws:MultiFactorAuthPresent`, `aws:RequestedRegion`. |

Notes:
- Multiple conditions inside a `Condition` block are combined with **AND**; multiple values for one key are combined with **OR**.
- In S3, bucket-level actions (`s3:ListBucket`) use the bucket ARN, while object-level actions (`s3:GetObject`) use `bucket/*`. Mixing them up is a classic mistake.
- **Policy variables** such as `${aws:username}` let one policy adapt per user.

### Example 1: S3 (identity-based, read-only on one bucket)

Attach to a user, group or role. Allows listing the bucket and reading its objects, but only over HTTPS.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-bucket"
    },
    {
      "Sid": "ReadObjects",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetObjectVersion"],
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "Bool": { "aws:SecureTransport": "true" }
      }
    }
  ]
}
```

### Example 2: EC2 (start/stop only instances tagged `Env=dev`)

Uses a **resource tag condition** (ABAC). `Describe*` actions don't support resource-level permissions, so they need `"*"`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DescribeEverything",
      "Effect": "Allow",
      "Action": "ec2:Describe*",
      "Resource": "*"
    },
    {
      "Sid": "StartStopDevInstancesOnly",
      "Effect": "Allow",
      "Action": ["ec2:StartInstances", "ec2:StopInstances", "ec2:RebootInstances"],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": { "aws:ResourceTag/Env": "dev" }
      }
    }
  ]
}
```

### Example 3: VPC (manage networking, but only in one Region)

VPC actions live under the `ec2:` namespace. This lets a network engineer manage VPC resources and denies everything outside `us-east-1`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ManageVpcNetworking",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVpc", "ec2:DeleteVpc",
        "ec2:CreateSubnet", "ec2:DeleteSubnet",
        "ec2:CreateRouteTable", "ec2:CreateRoute",
        "ec2:CreateInternetGateway", "ec2:AttachInternetGateway",
        "ec2:CreateSecurityGroup", "ec2:AuthorizeSecurityGroupIngress",
        "ec2:CreateTags", "ec2:Describe*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyOutsideUsEast1",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": { "aws:RequestedRegion": "us-east-1" }
      }
    }
  ]
}
```

### Exam takeaways (SAA-C03)

- **Evaluation order**: explicit `Deny` > Allow > implicit deny. One `Deny` overrides any number of `Allow` statements.
- **Identity-based** policies have no `Principal`; **resource-based** policies require it.
- **Cross-account** access needs permission on both sides (identity policy in account A + resource policy in account B), except when using a role.
- **Least privilege**: avoid `"Action": "*"` with `"Resource": "*"`; start narrow and use **IAM Access Analyzer** to generate policies from CloudTrail activity.
- Prefer **customer managed** policies over inline ones for reuse and versioning.
- You should be able to read a policy and say whether it **allows or denies** a given action.

## When NOT to use IAM users

IAM users carry **long-term credentials** (passwords and access keys) that can leak, be shared, or be forgotten. In most scenarios there is a better option.

| Scenario | Don't use | Use instead |
|---|---|---|
| Application on EC2, Lambda, ECS/EKS | User with access keys in code, env vars or config files | **IAM role** (instance profile, execution role, task role, IRSA) |
| Employees accessing one or many AWS accounts | One IAM user per person per account | **IAM Identity Center** (SSO, temporary credentials) |
| Users who already exist in a corporate directory (AD, Okta, Entra ID) | Duplicating them as IAM users | **Federation** (IAM Identity Center or SAML 2.0) |
| Access to another AWS account | Creating a user there and sharing credentials | **Cross-account role** (`sts:AssumeRole`) |
| CI/CD pipelines (GitHub Actions, GitLab) | Access keys stored as pipeline secrets | **OIDC federation** to an IAM role |
| On-prem servers or workloads | Long-lived access keys on the server | **IAM Roles Anywhere** |
| Mobile/web app end users | IAM users for customers | **Amazon Cognito** (identity pools give temporary AWS credentials) |
| Third-party vendor access | Sharing your credentials | **Role with External ID** |
| Daily administration | The **root user** | An admin identity via Identity Center; keep root for the few root-only tasks, with MFA and no access keys |

**When an IAM user is still acceptable**: a legacy or third-party tool that cannot assume a role or use federation, a break-glass emergency account, or a small standalone account where setting up Identity Center is not justified. Even then: enforce **MFA**, least privilege, rotate keys, and review the **credential report**.

**Exam shortcut**: if an answer offers IAM users with access keys and another offers a role, Identity Center or temporary credentials, **pick the latter**.

## IAM Identity Center

**AWS IAM Identity Center** (formerly **AWS SSO**) is the recommended service for **centralized workforce (human) access** to **multiple AWS accounts** and business applications, using a single sign-on and **temporary credentials**.

### What it does

- **Single sign-on portal**: users sign in once and pick the account and role they need, from the console or the CLI (`aws sso login`).
- **Multi-account access**: integrates with **AWS Organizations**, so you assign access to many accounts from one place instead of creating IAM users in each.
- **Temporary credentials**: behind the scenes it creates **roles** in each target account and issues short-lived credentials; no long-term access keys.
- **Bring your own identities**: use its built-in directory, **Active Directory** (AWS Managed Microsoft AD or AD Connector), or an external identity provider such as **Okta, Microsoft Entra ID or Google Workspace** via SAML 2.0 (with **SCIM** to sync users and groups automatically).
- **MFA** support for sign-in.
- **Free** to use.

### Core concepts

| Concept | Meaning |
|---|---|
| **Identity source** | Where users and groups live: Identity Center directory, Active Directory or an external IdP |
| **Users and groups** | The people you grant access to (manage access through groups) |
| **Permission set** | A reusable template of permissions (AWS managed and/or custom policies, session duration). Identity Center turns it into an IAM role in each assigned account |
| **Assignment** | Links **user/group + permission set + AWS account** |
| **Access portal** | The sign-in page where users choose the account and role |

### How it works

```
User ──▶ Access portal (SSO sign-in, MFA)
            │  authenticated via Identity source (built-in / AD / Okta / Entra)
            ▼
   Assignment: group "Developers" + permission set "PowerUser" + account "Prod"
            ▼
   Identity Center assumes the generated IAM role in the Prod account
            ▼
   User works with temporary credentials (no access keys)
```

### Identity Center vs. IAM users vs. Cognito

| | IAM users | IAM Identity Center | Cognito |
|---|---|---|---|
| For | Legacy / last resort | **Employees** (workforce) | **Customers** of your app |
| Credentials | Long-term | Temporary | Temporary (identity pools) |
| Multi-account | No (per account) | **Yes** | No |

### Exam takeaways (SAA-C03)

- "Give employees access to **multiple accounts** with **SSO**" or "integrate with the company's **Active Directory / external IdP**" → **IAM Identity Center**.
- It works with **AWS Organizations** and uses **permission sets** to define access per account.
- Workforce (employees) → Identity Center; customers/app users → **Cognito**.
- Identity Center replaces the need for individual IAM users and long-term access keys for humans.

## Root account (root user)

The **root user** is the identity created automatically with an AWS account, tied to the **email address** used to sign up. It has **complete, unrestricted access** to everything in the account, and its permissions **cannot be limited by IAM policies** (only by **SCPs** from AWS Organizations, in member accounts).

### Tasks that only the root user can perform

- **Close the AWS account.**
- **Change the account settings**: account name, root email address, root password.
- **Change or cancel the AWS Support plan.**
- **Restore IAM user permissions**: fix the situation where the last IAM admin was locked out or removed their own access.
- **Fix a misconfigured S3 bucket policy** that denies everyone (including admins) access to the bucket, by editing or deleting that policy. The same applies to an **SQS queue policy** that denies all principals.
- **Sign up for GovCloud**, register as a seller in the Reserved Instance Marketplace, and change certain billing settings (e.g., activate IAM access to the Billing console).
- **Enable MFA Delete** on an S3 bucket (via the CLI/API, using root credentials).
- **View certain tax invoices** and manage payment-related settings (depending on the account setup).

> For the exam, remember the core list: **close the account, change the support plan, change account name/email/root password, and recover from a bucket policy that locks everyone out.** The full list is in the AWS docs ("Tasks that require root user credentials").

### Mandatory: MFA and zero access keys

- **MFA is mandatory on the root user.** Because root has unrestricted access, a stolen password alone must never be enough. Use a **FIDO2 security key or hardware/virtual MFA device** (AWS lets you register multiple MFA devices for root as a backup).
- **Zero access keys for root.** Root access keys give programmatic, unrestricted, non-expiring access and cannot be limited by IAM policies. **Delete any that exist** and never create new ones; use roles or Identity Center for CLI/SDK access.
- Verify both with **AWS Config / Security Hub** rules (e.g., `root-account-mfa-enabled`, `iam-root-access-key-check`) and the **credential report**.

### Best practices
- **Never use root for daily work**: create an admin identity (IAM Identity Center or an IAM admin role/user) instead.
- Use a **strong unique password**, store it securely, and use a **shared mailbox/alias** for the root email so it isn't tied to one person.
- **Monitor root usage** with CloudTrail and alarms (e.g., EventBridge/CloudWatch alert on root sign-in).
- In an **AWS Organization**, use **centralized root access management** to remove root credentials from member accounts and perform privileged root tasks temporarily when needed.
- Apply an **SCP** that denies actions by the root user in member accounts where appropriate.

### Exam takeaways (SAA-C03)

- The answer to "who can close the account / change the support plan?" is the **root user**.
- Root should have **MFA enabled and zero access keys**, and be used only for the few tasks that require it.
- The root user is **not** the same as an **IAM user with AdministratorAccess**: an admin user can still be restricted by policies, boundaries and SCPs; root cannot be restricted by IAM.
- Anti-pattern: using root for routine operations or sharing its credentials.
