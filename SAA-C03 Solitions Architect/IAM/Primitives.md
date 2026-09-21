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
