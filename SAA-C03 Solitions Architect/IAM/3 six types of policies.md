# The Six Types of Policies

AWS evaluates several kinds of policies on every request. Some **grant** permissions, others only **limit** the maximum that can be granted. Knowing which is which is what the exam tests.

| # | Type | Attached to | Grants permissions? | Purpose |
|---|---|---|---|---|
| 1 | **Identity-based** | User, group, role | Yes | Say what an identity can do |
| 2 | **Resource-based** | A resource (S3, KMS, SQS, role trust...) | Yes | Say who can access this resource |
| 3 | **Permissions boundary** | User or role | No (limits) | Cap the maximum permissions of an identity |
| 4 | **SCP / RCP** (Organizations) | Org root, OU or account | No (limits) | Guardrails across accounts |
| 5 | **Session policy** | A temporary session | No (limits) | Restrict a single assumed-role/federated session |
| 6 | **ACL** | A resource (S3, VPC NACL... legacy) | Yes (limited) | Legacy cross-account grants |

> **Rule of thumb**: only identity-based, resource-based and ACLs **give** access. Boundaries, SCPs/RCPs and session policies **never give** access; they only **filter** what other policies grant.

---

## 1. Identity-based policies

JSON policies **attached to an IAM user, group or role**. They define what that identity can do.

- Have **no `Principal`** element (the principal is whoever it is attached to).
- Three flavors: **AWS managed** (maintained by AWS), **customer managed** (yours, reusable and versioned) and **inline** (embedded in a single identity, deleted with it).
- Use for: "Developers can start EC2 instances", "this Lambda role can read this DynamoDB table".

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

## 2. Resource-based policies

JSON policies **attached to a resource**. They define **who** (which principals) can access that resource and what they can do.

- **Require a `Principal`** element.
- Examples: **S3 bucket policy**, **KMS key policy**, SQS/SNS policies, Lambda resource policy, Secrets Manager, ECR, and the **trust policy of an IAM role** (a resource-based policy on the role).
- Can grant access to principals in **other accounts** directly.
- Use for: "allow account B to read this bucket", "allow API Gateway to invoke this Lambda", "restrict this bucket to my organization".

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::222233334444:root" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

**Same-account vs. cross-account**
- **Same account**: an Allow in *either* the identity-based *or* the resource-based policy is enough.
- **Cross-account**: **both** sides must allow (identity policy in account A + resource policy in account B). With an **assumed role**, the caller acts as the role in the target account, so the role's policies apply instead.
- **KMS** is special: the **key policy is the final authority**; IAM policies alone are not enough unless the key policy delegates to IAM.

## 3. Permissions boundaries

An **advanced feature** that sets the **maximum permissions** an IAM **user or role** can have. It does not grant anything by itself.

- Effective permissions = **intersection** of the identity-based policy **and** the boundary.
- Uses a managed policy as the boundary.
- Use for: **delegating administration safely**. For example, let developers create roles for their apps, but only if the role has a boundary that prevents privilege escalation.

```
Identity policy:   s3:*, ec2:*, iam:*
Boundary:          s3:*, ec2:*
Effective:         s3:*, ec2:*      (iam:* is NOT allowed)
```

## 4. Service control policies (SCPs) and resource control policies (RCPs)

Policies in **AWS Organizations** that act as **guardrails** for the accounts in an organization.

- **SCP**: limits the maximum permissions of **principals** (users and roles) in the affected member accounts, **including the root user of those accounts**. It does **not** apply to the management account.
- **RCP**: limits the maximum permissions on **resources** in the affected accounts (e.g., block access from outside the organization), regardless of what the resource policies say.
- Neither one **grants** anything: an action needs an explicit Allow in an identity/resource policy **and** must not be blocked by the SCP/RCP.
- Applied at the **organization root, an OU or an account**, and inherited downward.
- Use for: "deny all regions except `eu-west-1`", "nobody can disable CloudTrail", "no one can leave the organization".

```
SCP allows:        ec2:*, s3:*
IAM policy allows: ec2:*, dynamodb:*
Effective:         ec2:*   (dynamodb blocked by SCP, s3 never granted by IAM)
```

## 5. Session policies

Policies passed **as a parameter when creating a temporary session** (`AssumeRole`, `AssumeRoleWithSAML`, `AssumeRoleWithWebIdentity`, `GetFederationToken`), for a role or a federated user.

- They **limit** the session; they never expand permissions.
- Effective permissions = **intersection** of the role's identity-based policy and the session policy (plus boundary/SCP where they apply).
- Use for: giving a **narrower, one-off** set of permissions than the role normally has, for example an app that assumes a broad role but hands each end user a session scoped to their own S3 prefix.

## 6. Access control lists (ACLs)

The **legacy** policy type. ACLs control which **principals in other accounts** can access a resource. They are **not JSON**.

- Attached to **resources**, mainly **S3 buckets and objects** (also supported by a few other services, such as AWS WAF web ACLs and Amazon VPC network ACLs, which work differently from IAM ACLs).
- Very limited: grant only basic read/write style permissions to accounts or predefined groups; **cannot** use conditions and **cannot** grant to IAM users/roles individually.
- **S3 ACLs are deprecated**: new buckets have them **disabled** by default (**Object Ownership = Bucket owner enforced**). Use bucket policies instead.
- Use only for niche legacy cases (e.g., S3 server access log delivery in older setups).

> Don't confuse **S3 ACLs** with **VPC network ACLs (NACLs)**, which are stateless subnet-level firewalls, not IAM policies.

---

## How they combine: evaluation logic

For a request to succeed, **no policy may deny it**, and **enough policies must allow it**:

1. **Explicit Deny** in any policy → **denied** (always wins).
2. **SCP / RCP** must allow (if the account is in an organization).
3. **Permissions boundary** must allow (if set).
4. **Session policy** must allow (if used).
5. An **identity-based** or **resource-based** policy must **Allow** (subject to same-/cross-account rules).
6. Otherwise → **implicit deny**.

```
              ┌──────── Explicit Deny anywhere? ───► DENY
              ▼
   SCP/RCP ─► Boundary ─► Session policy ─► Identity/Resource Allow? ─► ALLOW
   (limits)    (limits)      (limits)             (grants)             else: implicit DENY
```

## Exam takeaways (SAA-C03)

| Scenario | Policy type |
|---|---|
| "Give this role/user/group permissions" | **Identity-based** |
| "Allow another account to use my bucket/key/queue" | **Resource-based** |
| "Let devs create roles but never exceed X permissions" | **Permissions boundary** |
| "Prevent every account in the org from using unapproved regions/services" | **SCP** |
| "Prevent data access from outside the organization on all buckets" | **RCP** (or `aws:PrincipalOrgID` in bucket policies) |
| "Give a temporary session fewer permissions than its role" | **Session policy** |
| "Legacy S3 cross-account grant" | **ACL** (avoid; use bucket policy) |

- **Boundaries, SCPs/RCPs and session policies only restrict**; they never grant permissions.
- SCPs affect **member accounts only** (not the management account) and apply to **all principals, including root** of those accounts.
- **Explicit Deny always wins.**
- Cross-account needs **both sides** to allow, except through an **assumed role**.
