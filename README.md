# AWS Solution Architech Professional

## IAM Policy Evaluation

**Explicit Deny > Allow > Default (Implicit) Deny**

- Every request starts as an implicit deny.
- An explicit `Allow` overrides the implicit deny.
- An explicit `Deny` (`"Effect": "Deny"`) always wins, no matter how many Allows exist.
- 💡 For cross-account access, **both** the identity-based policy (in the caller's account) and the resource-based policy (in the resource's account) must allow it. Within the same account, either one is enough.
- 💡 SCPs, permission boundaries and session policies only **limit** permissions, they never grant them.

## Condition Operators

| Type | Operators / Examples |
|---|---|
| String | `StringEquals`, `StringNotEquals`, `StringLike` |
| Numeric | `NumericEquals`, `NumericNotEquals`, `NumericLessThan`, `NumericGreaterThan` |
| Date | `DateEquals`, `DateNotEquals`, `DateLessThan`, `DateGreaterThan` |
| Boolean | `{"Bool": {"aws:SecureTransport": "true"}}` · `{"Bool": {"aws:MultiFactorAuthPresent": "true"}}` |
| IP | `{"IpAddress": {"aws:SourceIp": "203.0.113.0/24"}}` (also `NotIpAddress`) |
| ARN | `ArnEquals`, `ArnLike` |
| Null | `{"Null": {"aws:TokenIssueTime": "true"}}` → `true` means the key **does not exist** (i.e. long-term IAM user credentials were used, not temporary STS credentials) |

- ⚠️ Fixed typos: `NumericNotEquals`, `ArnEquals`, `aws:SourceIp` (not `aws.SourceIp`), `aws:TokenIssueTime` (no space).
- 💡 `...IfExists` variants (e.g. `BoolIfExists`) evaluate to true when the key is missing. Use `BoolIfExists: {"aws:MultiFactorAuthPresent": "false"}` in a **Deny** to also catch requests with no MFA info at all.
- 💡 Multi-value condition: `ForAllValues` / `ForAnyValue` set operators.

### IAM Policy Variables
- `${aws:username}` → `arn:aws:s3:::myBucket/${aws:username}/*` (each user gets their own "folder")

### AWS-Wide Condition Keys
- `aws:CurrentTime`, `aws:TokenIssueTime`, `aws:PrincipalType`, `aws:SecureTransport`, `aws:SourceIp`, `aws:userid`, `ec2:SourceInstanceARN`
- 💡 `aws:SourceIp` is **not** useful when the request goes through a VPC endpoint (the source is a private IP). Use `aws:SourceVpc`, `aws:SourceVpce` or `aws:VpcSourceIp` instead.
- 💡 `aws:SourceIp` also doesn't apply when an AWS service makes the call on your behalf (e.g. CloudFormation). Use `aws:ViaAWSService` to exempt those.

### Service-Specific Condition Keys
- `s3:prefix`, `s3:max-keys`, `s3:x-amz-acl`, `sns:Endpoint`, `sns:protocol`

### Tag-Based (ABAC)
- `iam:ResourceTag/key-name` (tag on an IAM resource)
- `aws:PrincipalTag/key-name` (tag on the calling user/role)
- 💡 `aws:ResourceTag/key-name`, `ec2:ResourceTag/key-name` (tag on the target resource), `aws:RequestTag/key-name` (tag being passed in the request), `aws:TagKeys`
- 💡 ABAC pattern: allow when `aws:PrincipalTag/Department` equals `ec2:ResourceTag/Department`. This scales without editing policies per project.

## Assume Role vs Resource-Based Policy

- **Assume a role** → the principal **gives up its original permissions** and only has the permissions of the assumed role (for that session).
- **Resource-based policy** → the principal **keeps its own permissions** while also getting access granted by the resource policy. No need to switch roles.
- 💡 This is why resource-based policies are handy for cross-account use cases, such as S3 → DynamoDB → Lambda, where the caller needs to keep working in its own account at the same time.

## Identity Federation

Identity federation comes in several flavors:
1. SAML 2.0
2. Custom Identity Broker
3. Web Identity Federation (with or without Amazon Cognito)
4. IAM Identity Center

### SAML 2.0
- **Security Assertion Markup Language 2.0.**
- Open standard used by many identity providers (e.g. Microsoft Active Directory Federation Services, Okta, Azure AD/Entra ID, or any SAML 2.0-compatible IdP) to integrate with AWS.
- Gives access to the AWS Console, CLI and API using **temporary credentials**.
  - No need to create an IAM user for each employee.
- Trust must be configured **both ways**: AWS IAM trusts the IdP (IAM SAML provider + role trust policy), and the IdP is configured with AWS as a relying party.
- Under the hood it uses the STS API **`AssumeRoleWithSAML`**.
- **Flow (API/CLI):** user logs in at the IdP portal → IdP authenticates against the identity store (e.g. AD) → IdP returns a **SAML assertion** → client calls STS `AssumeRoleWithSAML` → STS verifies the assertion and returns temporary access keys.
- **Flow (Console):** same mechanism, but the browser **POSTs the assertion to `https://signin.aws.amazon.com/saml`** → AWS creates temporary credentials via STS and redirects the user to the console with a sign-in URL.
- ⚠️ A **custom identity broker** is used when the IdP is **not** SAML 2.0-compatible (see below).
- 💡 Role trust policy needs `"Action": "sts:AssumeRoleWithSAML"` with `Principal: Federated: <SAML provider ARN>`.
- 💡 AWS now recommends **IAM Identity Center** over raw SAML federation for new workforce setups.

### Custom Identity Broker
- 💡 Used when the corporate IdP is **not compatible with SAML 2.0**.
- You write the broker application. It authenticates the user itself, decides which IAM policy/role applies, then calls STS **`AssumeRole`** or **`GetFederationToken`**.
- For console access the broker calls the AWS federation endpoint to get a sign-in token (`GetSigninToken`) and builds the console URL.
- The broker needs IAM permissions to call STS. It is a trusted component, so protect it.

### Web Identity Federation
- Lets users log in with a third-party/OIDC identity provider: Google, Facebook, Amazon, or any **OpenID Connect (OIDC)**-compatible IdP.
- Restrict permissions using **IAM policy variables** with the IdP's user ID:
  - `cognito-identity.amazonaws.com:sub`
  - `www.amazon.com:user_id` (⚠️ the policy variable is `${www.amazon.com:user_id}`, not `amazon.com:user_id`)
  - `graph.facebook.com:id`
  - `accounts.google.com:sub`

```json
{
  "Sid": "AccessOnlyOwnObjects",
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
  "Resource": "arn:aws:s3:::my-app-bucket/${www.amazon.com:user_id}/*"
}
```

#### Without Cognito (not recommended)
- Client logs in with a third-party IdP (Google, Facebook, Amazon, OIDC).
- Client calls STS **`AssumeRoleWithWebIdentity`** directly with the IdP token.

#### With Cognito (preferred)
- Create IAM roles for the Cognito identity pool with **least privilege**.
- Trust is built between the OIDC IdP and AWS through Cognito.
- Flow: user authenticates with the third-party IdP → the IdP token is given to **Cognito (not directly to STS)** → Cognito calls STS and returns temporary AWS credentials.
- Supports **anonymous (guest) users**.
- Supports **MFA**.
- Supports **data synchronization** across devices (Cognito Sync / now commonly AWS AppSync).
- 💡 **User Pool vs Identity Pool**:
  - **User Pool** = authentication (sign-up/sign-in, returns JWT tokens, can federate with social/SAML/OIDC IdPs).
  - **Identity Pool (Federated Identities)** = authorization, exchanges a token (from a User Pool or a social IdP) for **temporary AWS credentials**.
- 💡 Identity pools can assign different IAM roles for authenticated vs. guest users.

### IAM Identity Center (successor to AWS SSO)
- 💡 Central place for workforce SSO across multiple AWS accounts (via AWS Organizations) and SAML-enabled business apps.
- 💡 Identity source options: built-in Identity Center directory, **Active Directory** (AWS Managed Microsoft AD or AD Connector), or an external IdP (SAML 2.0 + SCIM).
- 💡 Uses **permission sets** (templates deployed as IAM roles in each target account) assigned to users/groups per account.

## Microsoft Active Directory (AD)

- ⚠️ The original heading said "MS Azure Directory". On-prem AD is **Microsoft Active Directory**; Azure's cloud version is **Microsoft Entra ID** (formerly Azure AD).
- Found on any Windows Server with **AD Domain Services**.
- Database of **objects**: user accounts, computers, printers, file shares, security groups.
- Centralized security management: create accounts, assign permissions.
- Objects are organized in **trees**; a group of trees is called a **forest**.
- 💡 AWS Directory Service options:
  - **AWS Managed Microsoft AD**: real AD in AWS; supports trust relationships with on-prem AD.
  - **AD Connector**: proxy that redirects requests to on-prem AD, nothing cached.
  - **Simple AD**: standalone, Samba-based, basic AD features; no trust with on-prem AD.

## ADFS (AD Federation Services)

- Provides **Single Sign-On** across applications.
- Uses **SAML** across third parties: AWS Console, Dropbox, Office 365, etc.
- 💡 ADFS is the on-prem IdP in the SAML flow described above (AD authenticates, ADFS issues the SAML assertion).

## KMS (Key Management Service)

- ⚠️ KMS is a **regional** service. Keys never leave their region (except multi-region keys, which are replicated copies).
- **Symmetric** keys (AES-256): the only type that all AWS services integrated with KMS use.
- **Asymmetric** keys (RSA, ECC): for encrypt/decrypt or sign/verify; the public key can be downloaded, but the private key never leaves KMS. Used by services/users outside AWS that can't call KMS.
- 💡 Direct `Encrypt` API limit is **4 KB**. For larger data use **envelope encryption** (`GenerateDataKey`).
- Access is controlled by the **key policy** (a resource policy, mandatory) plus IAM policies and grants. Usage is audited in **CloudTrail**.
- 💡 Cross-account: key policy must allow the other account **and** the other account must grant its principals IAM permission.

### Key Types
| Type | Notes |
|---|---|
| **Customer managed key (CMK)** | You create/control it, key policy, can enable rotation (new key material each year by default; older material is kept so old data can still be decrypted; ⚠️ the rotation period is now configurable, 90 to 2560 days). 💡 Costs $1/month per key. |
| **AWS managed key** (`aws/s3`, `aws/ebs`, `aws/redshift`, ...) | Created by AWS in your account for integrated services; **rotated automatically every year**; you can view the key policy and audit usage in CloudTrail but can't change it. |
| **AWS owned key** | Created and managed by AWS, used by some services to protect your resources. Lives in AWS-owned accounts (shared across many customers), **not visible in your account**; rotation is managed by AWS. |

### Key Material Origin
- **KMS** (default): KMS generates the key material.
- **External** (BYOK, bring your own key): you import your own key material. Supports symmetric and asymmetric. ⚠️ Only **manual** rotation; you can set an expiry.
- **CloudHSM custom key store**: key material generated and stored in your CloudHSM cluster. KMS API is used, but crypto happens in the HSM.
- 💡 **External key store (XKS)**: key material lives in an external HSM outside AWS.
- ⚠️ The original note said BYOK "can't be used with CloudHSM". A more accurate summary: imported (BYOK) key material is a different origin from a CloudHSM custom key store, and you choose one origin per key.

### KMS Multi-Region Keys
- Replicates a key (same key ID and key material, prefix `mrk-`) into **multiple regions**. It is **not** one global key. Each replica is managed independently.
- You can **promote** a replica to become the primary key.
- Use cases: global encrypted DynamoDB tables, Aurora Global, client-side encryption/decryption in several regions, disaster recovery.

## Parameter Store (SSM)

- Serverless, scalable, durable, secure configuration and secrets store.
- **Version tracking** of values.
- Security through **IAM** (and path-based policies, e.g. `/prod/app1/*`).
- **KMS encryption is optional** (`SecureString` type).
- Notifications via **EventBridge**.
- Integrates with **CloudFormation** (parameters can be pulled as stack parameters).
- 💡 Hierarchical paths like `/my-department/my-app/dev/db-url` make IAM policies easy.
- Can reference Secrets Manager: `/aws/reference/secretsmanager/<secret_ID_in_Secrets_Manager>`
- Public parameters: `/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2` (⚠️ fixed the typo `amaz2` → `amzn2`, and the exact name needs the suffix). Always fetch the latest AMI ID this way.

### Standard vs Advanced Tiers

| | Standard | Advanced |
|---|---|---|
| Number of parameters allowed | 10,000 | 100,000 |
| Max size of value | 4 KB | 8 KB |
| Parameter policies | ❌ | ✅ |
| Cost | No additional charge | Charges apply |
| Storage pricing | Free | $0.05 per advanced parameter per month |

### Parameter Policies (Advanced Parameters only)
- Assign a **TTL** (expiration date) to a parameter to force updating or deleting sensitive data.
- Multiple policies can be attached to a parameter at once.
- Policy types:
  - `Expiration`: deletes the parameter at the given time.
  - `ExpirationNotification`: sends an EventBridge event before expiry.
  - `NoChangeNotification`: sends an EventBridge event if the parameter hasn't changed in a given time.

## Secrets Manager

- Purpose-built for secrets, with **automatic rotation** (using Lambda).
- **Native integration** with RDS, Redshift, DocumentDB (rotation functions are provided for them).
- Access is controlled with **IAM and resource-based policies**.
- **KMS encryption is mandatory** (⚠️ vs. optional in Parameter Store).
- 💡 Can **replicate secrets across regions** (multi-region secrets), useful for DR and global apps.
- 💡 Costs per secret per month (about $0.40) vs. Parameter Store standard being free. This is the usual exam deciding factor: need rotation/cross-region replication → Secrets Manager; need cheap config → Parameter Store.

## RDS Security

- **KMS encryption at rest** (underlying EBS volumes, snapshots, automated backups, read replicas).
- **TDE (Transparent Data Encryption)** for Oracle and SQL Server.
- **SSL/TLS encryption in flight** is possible for all RDS DBs. 💡 Enforce with `rds.force_ssl` (PostgreSQL) or `REQUIRE SSL` (MySQL).
- **IAM authentication** for MySQL, PostgreSQL and MariaDB (uses a short-lived 15-minute auth token instead of a password).
- ⚠️ Authorization still happens **within the DB** (SQL grants), not in IAM.
- You **can't directly encrypt an existing unencrypted DB**. Instead: snapshot → **copy snapshot with encryption enabled** → restore from the encrypted snapshot.
- **CloudTrail cannot track queries** inside RDS (it only sees RDS API calls). 💡 Use DB audit logs, or Database Activity Streams (Aurora/Oracle/SQL Server) for query-level auditing.
- 💡 Network: security groups, private subnets, no public access.

## SSL / TLS

- **SSL** (Secure Sockets Layer): the old protocol, deprecated. **TLS** (Transport Layer Security) is its modern replacement. People still say "SSL certificate".
- 💡 Gives **encryption in transit** and server authentication using certificates issued by a CA.
- **DNSSEC**: protects against DNS spoofing/cache poisoning by signing DNS records (authenticity and integrity, not encryption). 💡 Route 53 supports DNSSEC signing for public hosted zones, with the KSK stored in a KMS asymmetric key in `us-east-1`.

## ACM (AWS Certificate Manager)

- **Public certificates are free**. 💡 They are issued only for use with integrated AWS services (ELB, CloudFront, API Gateway); you can't export the private key.
- You can **buy a cert elsewhere and import it** to ACM. ⚠️ Imported certs are **not auto-renewed**; you must renew and re-import (💡 ACM sends expiry events to EventBridge, and AWS Config has a rule to check expiry).
- ACM-issued public certs **auto-renew**.
- **Private certificates**:
  - Create your own CA (💡 **AWS Private CA**, paid).
  - Applications/clients must **trust this CA**, as it is your private CA.
- ⚠️ ACM is a **regional** service.
  - 💡 Certificates for **CloudFront** must be in **us-east-1**.
  - 💡 For ALB/API Gateway regional endpoints, the cert must be in the same region as the resource.

## CloudHSM

- AWS **provisions dedicated encryption hardware** (single-tenant HSM).
- You manage your own encryption keys **entirely**. AWS has no access to them.
- ⚠️ If you lose your keys, **AWS cannot recover them**.
- **FIPS 140-2 Level 3** compliant. (💡 KMS HSMs are also validated at Level 3 now, but the key difference is single-tenant vs. multi-tenant and who controls keys.)
- Supports **both symmetric and asymmetric** encryption (⚠️ original said "assymetric and assymetric"), including TLS/SSL keys.
- **No free tier.**
- You must use the **CloudHSM client software** (PKCS#11, JCE, CNG/KSP, OpenSSL Dynamic Engine) to use CloudHSM, not AWS API calls for crypto operations.
- Redshift supports CloudHSM for database encryption and key management.
- Good option to pair with **SSE-C** style scenarios, where you must hold the keys yourself.
- 💡 Can also be used as a **KMS custom key store**, so AWS services can use KMS APIs while keys live in your HSM.
- **High availability**: a CloudHSM cluster must be spread across **multiple AZs**.
- Deployed and managed in your **VPC**; can be shared across VPCs using **VPC peering**.
- **Users/permissions** are managed separately inside the HSM (crypto users, crypto officers), unlike KMS which uses IAM. IAM only controls the management APIs.
- Supports **cryptographic acceleration** for SSL/TLS and **Oracle TDE**.
- **SSL termination offload**: webapps (NGINX, Apache, IIS) can offload SSL termination to CloudHSM.
  - You must set up a **Cryptographic User (CU)** on the CloudHSM device and ensure the EC2 instance can use that user.

## S3

### Encryption
- **SSE-S3**: S3 objects encrypted with keys handled and managed by AWS (AES-256). 💡 Since Jan 2023, all new objects are encrypted with SSE-S3 by default.
- **SSE-KMS**: uses KMS to manage the encryption key.
  - Good for **auditing**: API calls using the key appear in **CloudTrail**.
  - Good security if the bucket is made public accidentally: without KMS key permission, public users can't read the file.
  - On upload ensure `kms:GenerateDataKey` is allowed (and `kms:Decrypt` for download).
  - 💡 Watch for KMS API request quotas/throttling on very high request rates. S3 Bucket Keys reduce KMS calls and cost.
- ⚠️ **SSE-C** (customer-provided keys) is its own option, not part of SSE-KMS: you manage your own key and **HTTPS is mandatory**. AWS doesn't store the key.
- 💡 **Client-side encryption**: you encrypt before upload.
- **Glacier**: all data is AES-256 encrypted by default, with keys under AWS control.
- **Enforce HTTPS**: bucket policy that denies requests when `aws:SecureTransport` is `false`.
- 💡 **Enforce encryption on upload**: deny `PutObject` when the `s3:x-amz-server-side-encryption` header is missing or has the wrong value (or just use default bucket encryption).

### Events in S3
- **S3 Access Logs**:
  - Detailed record of the requests made to S3.
  - Might take **hours** to deliver.
  - Best-effort delivery, so might be **incomplete**.
  - 💡 Delivered to another bucket (never log into the same bucket, it creates a loop).
- **Event Notifications**:
  - Events when an object is created, removed, restored, a replication event happens, etc.
  - Destinations: **Lambda, SQS, SNS** (💡 and EventBridge).
  - Events are usually fast (seconds) but can sometimes take a minute or more.
  - If two operations happen on the **same object at the same time**, enable **object versioning** so you get a separate event for each write.
- **Amazon EventBridge**:
  - ⚠️ Two ways: (1) enable the **S3 → EventBridge** option on the bucket for native S3 events; (2) enable **CloudTrail object-level (data event) logging** to react to API calls.
  - Targets: SNS, Lambda, SQS, Step Functions, Kinesis, and many more. 💡 Advanced filtering, archive and replay.

### Trusted Advisor
- ⚠️ (Typo: "Trust Advisor") Checks if a bucket is **public** (S3 bucket permissions check). 💡 Full set of checks needs Business/Enterprise support; Core security checks are free.

### Security
- IAM policies
- Resource-based policies:
  - **Bucket policy**
  - **Object ACL**: finer grain (per object)
  - **Bucket ACL**: less common
- 💡 ACLs are disabled by default for new buckets (**Object Ownership = bucket owner enforced**). AWS recommends policies over ACLs.
- 💡 **S3 Block Public Access** (account and bucket level) overrides policies/ACLs that would make data public.

### Bucket Policy
- Grant **public access** to a bucket
- **Force objects to be encrypted** at upload
- **Cross-account** access
- Optional conditions:
  - Source IP (`aws:SourceIp`), `aws:VpcSourceIp` (through VPC endpoint)
  - Source VPC (`aws:SourceVpc`) and Source VPC endpoint (`aws:SourceVpce`): ⚠️ these only work with VPC endpoints
  - CDN origin identity (**CloudFront OAI**, 💡 now **OAC** is recommended)
  - **MFA** (`aws:MultiFactorAuthPresent`)

### Pre-Signed URL
- Upload / download.
- Valid for **1 hour by default** (⚠️ configurable: console 1 min to 12 hours; CLI/SDK up to 7 days when signed with long-term IAM user credentials; shorter if signed with temporary role credentials, because the URL dies when the credentials expire).
- ⚠️ The user given the pre-signed URL **inherits the permissions of the person who generated it** (typo "inhabit").

### VPC Gateway Endpoint
- In the **bucket policy** use `aws:SourceVpc` or `aws:SourceVpce` to restrict access to your VPC / endpoint.
- 💡 Gateway endpoints are free, use route table entries, and are not reachable from on-prem. Use an **interface endpoint** for on-prem access over DX/VPN.

### Object Lock
- Adopts a **WORM** model (Write Once Read Many).
- Blocks an object version from deletion/overwrite for a specified amount of time.
- 💡 Requires **versioning**. Modes: **Governance** (privileged users with special permission can override) and **Compliance** (nobody, not even root, can delete during retention). Also **Legal Hold** (no expiry, can be set/removed with permission).

### Glacier Vault Lock
- Adopts a **WORM** model.
- **Lock the vault policy** so it can no longer be changed (immutable once locked; 💡 you have a 24-hour window after initiating the lock to validate and abort).
- Helpful for **compliance and data retention**.

### S3 Access Points
- Simplifies S3 access security management at scale (instead of one huge bucket policy).
- Each access point has its **own DNS name** (internet origin or VPC origin).
- Each has an **access point policy** (same format as a bucket policy). 💡 The bucket policy must also allow access via the access point (usually by delegating to access points).
- For **VPC origin** you must create a **VPC endpoint** (gateway or interface) to access it privately.
  - The VPC endpoint policy must also allow access to the S3 bucket and the access point.
- Flow: `EC2 → VPC Endpoint → Access Point (VPC origin) → S3 bucket`
- **S3 Object Lambda**: modifies the S3 object just before it is returned to the caller. The app calls an **Object Lambda access point**, which triggers a Lambda that retrieves and transforms the object from a supporting access point (e.g. redact PII, convert formats, enrich with data).
  - 💡 AWS has been restricting S3 Object Lambda for new customers, so verify availability. It still appears in exam material.

### S3 Multi-Region Access Points
- **Global endpoint** spanning S3 buckets in multiple AWS regions.
- Dynamically routes requests to the **nearest (lowest latency)** bucket.
- **Bi-directional S3 bucket replication** rules are created to keep data in sync across regions.
- **Failover controls**: shift requests across buckets in different regions (**Active-Active** or **Active-Passive**). ⚠️ Shift happens quickly, but expect minutes rather than instant.
- To enable replication you must also enable **bucket versioning**.

## DDoS Attacks

- **AWS Shield Standard**: free, automatic for all customers, protects against common **layer 3/4** attacks (SYN/UDP floods, reflection attacks).
- **AWS Shield Advanced**: paid (about $3,000/month per organization, 1-year commitment).
  - Protects EC2, ELB (ALB/CLB/NLB), CloudFront, Global Accelerator, Route 53, Elastic IPs.
  - 24/7 access to the **Shield Response Team (SRT)**, advanced reporting, **cost protection** against scaling bills during an attack, and automatic layer 7 mitigation via WAF rules.

## WAF (Web Application Firewall)

- Protects from **layer 7 (HTTP)** attacks.
- Deploy on: **ALB, CloudFront, API Gateway, AppSync (GraphQL API)**. 💡 Also Cognito User Pools, App Runner, Verified Access.
- 💡 Web ACL is regional (CloudFront Web ACLs are global and created in `us-east-1`). Rules can match IP sets, geo, headers, body, URI, SQLi, XSS, size, regex; **rate-based rules** limit request rates per IP. Actions: Allow, Block, Count, CAPTCHA/Challenge.
- ⚠️ WAF **does not** protect against layer 3/4 DDoS (that's Shield). It also cannot be attached to NLB.

### Managed Rules (190+)
Ready-to-use rules managed by AWS and by Marketplace sellers.
- **Baseline Rule Groups**
  - `AWSManagedRulesCommonRuleSet`, `AWSManagedRulesAdminProtectionRuleSet`
  - 💡 `AWSManagedRulesKnownBadInputsRuleSet`
- **Use-Case Specific Rule Groups**
  - `AWSManagedRulesSQLiRuleSet`, `AWSManagedRulesWindowsRuleSet`
  - `AWSManagedRulesPHPRuleSet`, `AWSManagedRulesWordPressRuleSet`
  - 💡 `AWSManagedRulesLinuxRuleSet`, `AWSManagedRulesUnixRuleSet`
- **IP Reputation Rule Groups**
  - ⚠️ `AWSManagedRulesAmazonIpReputationList`, `AWSManagedRulesAnonymousIpList`
- **Bot Control Managed Rules**
  - `AWSManagedRulesBotControlRuleSet`

### Logging
- Destinations: **S3, CloudWatch Logs, Kinesis Data Firehose** (use Firehose for large traffic volumes; Firehose destinations can be S3, OpenSearch, Redshift).
- 💡 Destination names must begin with `aws-waf-logs-`.

### Security (Locking the Origin Behind CloudFront)
- CloudFront adds a secret custom header (e.g. `X-Origin-Verify: <secret>`), and WAF on the ALB checks for it, so only traffic that came through CloudFront is allowed.
- The secret can be **rotated automatically using Secrets Manager**, which updates both the WAF rule and the CloudFront custom header.

## AWS Firewall Manager

- Manages firewall rules for **all accounts in an AWS Organization** from one place. 💡 Requires Organizations and AWS Config; set up through a Firewall Manager administrator account.
- **Security policy**: a common set of security rules.
  - WAF rules (ALB, API Gateway, CloudFront)
  - **Shield Advanced** (ALB, CLB, NLB, Elastic IP, CloudFront)
  - **Security groups** policies for EC2, ALB and ENI resources in VPC
  - **AWS Network Firewall** at the VPC level
  - **Amazon Route 53 Resolver DNS Firewall**
- Policies are created at the **region** level.
- Rules are applied **automatically to newly created resources** (e.g. a new ALB) across all accounts, and non-compliant resources are flagged/remediated.

## WAF + Firewall Manager + Shield

- WAF, Shield and Firewall Manager are used **together** for comprehensive protection.
- **Define your Web ACL rules in WAF.**
- For **granular protection of your resources**, WAF alone is the right choice.
- To use WAF **across multiple accounts**, accelerate WAF configuration and **automate protection of new resources**, use **Firewall Manager with WAF**.
- **Shield Advanced** is for DDoS and adds features on top of AWS WAF: dedicated support from the **Shield Response Team (SRT)**, advanced reporting, and it can **automatically create/modify WAF rules** for you.

## Block IP Address

| Layer | What it can do |
|---|---|
| **NACL** | First line of defence (subnet level). Supports **both allow and deny** rules. Stateless. |
| **Security Group** | Instance/ENI level. **Allow rules only**, stateful. |
| **Firewall software on EC2** | e.g. `ufw`, iptables. |
| **ALB** | See below. |
| **WAF** | Layer 7 IP filtering on ALB/CloudFront/API Gateway. |
| **CloudFront** | **Geo Restriction** feature (allow/block list by country). |

- **ALB flow:** `Client → ALB (public subnet) → EC2 (private subnet)`.
  - Use **NACL on the public subnet** (where the ALB lives) for allow/deny rules, plus the SG on the ALB.
  - ⚠️ EC2 behind an ALB sees the **ALB's IP**, not the client's (the client IP is in the `X-Forwarded-For` header), so SG/NACL rules on the EC2 can't block the client.
- **NLB flow:** 💡 NLB **preserves the client IP**, so the EC2 targets' SG/NACL can allow/block client IPs directly (and the NLB now supports its own security group).
- **CloudFront (CDN) in front of ALB:** ⚠️ NACLs won't work for CloudFront users because CloudFront is not in your subnet. Use **WAF** (IP match rules) or **CloudFront geo restriction** at the edge, and at the ALB level use the SG that allows only CloudFront's managed prefix list.

## AWS Inspector

Automated, continuous **vulnerability management** service.

- **EC2**
  - Leverages the **SSM agent** (💡 or agentless scanning of EBS snapshots).
  - Analyzes against **unintended network accessibility** (network reachability).
  - Analyzes the running OS for **known vulnerabilities** (CVEs).
- **ECR**
  - Assesses container images as they get **pushed** to ECR (💡 and continuously re-scans).
- **Lambda**
  - When a function is deployed, it is analyzed for **software vulnerabilities in code and package dependencies**.
- Reports findings in **AWS Security Hub**.
- Sends events of its findings to **Amazon EventBridge**.
- It evaluates:
  - Running EC2 instances, ECR images and Lambda functions
  - **Package vulnerabilities** (EC2, ECR, Lambda): database of **CVEs**
  - **Network reachability** of EC2
  - A **risk score** is associated with each vulnerability for prioritization.
- 💡 Don't confuse with: **GuardDuty** (threat detection from logs), **Macie** (sensitive data in S3), **Config** (config compliance).

## AWS Config

- **Audit and record compliance** of AWS resources.
- Records **configuration and configuration changes over time**.
- ⚠️ Config rules **do not prevent** actions from happening (no deny). They only detect and report (but can trigger remediation).
- Question examples it answers: Is there **unrestricted SSH** access? Are buckets **public**? How has my **ALB config changed** over time?
- **Alerts** to an SNS topic (or EventBridge) on changes.
- It is a **per-region** service, so enable it in every region you need to audit.
- Data from **multiple accounts/regions can be aggregated** in one central account (**aggregator**).
- If **CloudTrail** API calls are enabled you can also see **who made the change**.
- **AWS managed rules**: ⚠️ originally "over 75", now several hundred.
- **Custom config rules** must be defined in a **Lambda function** (💡 or Guard policy-as-code).
  - e.g. evaluate if each EBS volume is `gp3`
  - e.g. evaluate if each EC2 is `t2.micro`
- Rules can be evaluated/triggered:
  - For **each config change**
  - At **regular intervals** (e.g. every day)
  - **EventBridge** can react when a rule becomes non-compliant
- **Remediation** with **SSM Automation documents**: if a resource is non-compliant, trigger an auto-remediation (e.g. change a security group, stop an instance with non-approved tags).
- 💡 **Conformance packs** deploy a bundle of rules and remediations across accounts/regions.

## AWS Managed Logs

| Source | Destination |
|---|---|
| **Load balancer** (ALB/NLB/CLB) access logs | S3 |
| **CloudTrail** logs | S3 (💡 and CloudWatch Logs) |
| **VPC Flow Logs** | S3, CloudWatch Logs, Kinesis Data Firehose |
| **Route 53 query logs** (log of all DNS queries Route 53 receives) | CloudWatch Logs (💡 Resolver query logs can also go to S3 or Firehose) |
| **S3 access logs** (all requests made to a bucket) | Another S3 bucket |
| **CloudFront access logs** (every user request) | S3 (💡 real-time logs go to Kinesis Data Streams) |
| **AWS Config** configuration info | S3 |

## AWS GuardDuty

- **Intelligent threat detection** to protect your AWS account.
- Uses **machine learning, anomaly detection** and **3rd-party threat intelligence** data.
- **One click** to enable, **30-day free trial**, **no software to install**.
- 💡 It analyzes the logs **independently**; you don't need to enable VPC Flow Logs/DNS logs yourself.
- **Input data includes:**
  - **CloudTrail**
    - **Event logs**: unusual API calls, unauthorized deployments
    - **Management events**: create VPC subnets, create trail
    - **S3 data events**: get object, list objects, delete object, ...
  - **VPC Flow Logs**: unusual internal traffic, unusual IP addresses
  - **DNS logs**: compromised instances sending **encrypted data within DNS queries**
  - **Optional features**: EKS audit logs, RDS & Aurora login activity, EBS malware protection, Lambda network activity, S3 data events, Runtime Monitoring.
- Can set up **EventBridge rules** to be notified of findings. Targets can be **Lambda** or **SNS** (💡 great for automated remediation).
- Protects against **cryptocurrency attacks** (has a dedicated finding for it).
- **AWS Organizations**: a member account can be designated as the **GuardDuty delegated administrator**.
  - That account can enable and manage GuardDuty for all accounts in the organization.
  - ⚠️ Only the **Organization management account** can name a delegated administrator.

## IAM Advanced Policies

Useful condition keys:
- `aws:SourceIp`
- `aws:RequestedRegion` (restrict which regions can be used)
- `ec2:ResourceTag`: e.g. `ec2:ResourceTag/Department: "Finance"`
- `aws:PrincipalTag`: e.g. `aws:PrincipalTag/Department: "Data"` (applies to the **user's/role's tag**)
- `aws:MultiFactorAuthPresent`: e.g. deny when `false` (💡 use `BoolIfExists`)
- `aws:PrincipalOrgID`: for **member account access** in an organization
  - 💡 Also `aws:PrincipalOrgPaths` (OU level) and `aws:ResourceOrgID`.

Example (⚠️ as a resource-based policy this needs a `Principal` element, usually `"*"`, restricted by the org condition):

```json
{
  "Effect": "Allow",
  "Principal": "*",
  "Action": ["s3:PutObject", "s3:GetObject"],
  "Resource": "arn:aws:s3:::2022-financial-data/*",
  "Condition": {
    "StringEquals": {
      "aws:PrincipalOrgID": ["o-yyyyyyyyy"]
    }
  }
}
```

## EC2 Instance Connect Service (SendSSHPublicKey API) ⭐ IMPORTANT

- Pushes a **temporary SSH public key** to the instance metadata (valid for about **60 seconds**) using the **`SendSSHPublicKey`** API. 💡 No long-lived key pairs to manage; access is controlled by IAM (`ec2-instance-connect:SendSSHPublicKey`).
- **Interface VPC endpoint** for the EC2 Instance Connect API (service name): `com.amazonaws.ap-south-1.ec2-instance-connect`
- Typical inbound rule on the instance: **SSH, TCP 22**.
- **For a private EC2 instance** (using an **EC2 Instance Connect Endpoint**, EICE):
  - Instance SG: **TCP 22 → source = the security group attached to the EC2 Instance Connect Endpoint**.
  - 💡 EICE lives in a subnet. It lets you connect to private instances **without a public IP, bastion host or internet gateway**. IAM permission `ec2-instance-connect:OpenTunnel` is needed.
- 💡 Alternative with no inbound ports at all: **SSM Session Manager** (needs SSM agent + IAM role, logs sessions to S3/CloudWatch).
- 💡 For browser-based Instance Connect on a public instance, SG must allow port 22 from the EC2 Instance Connect service IP range for the region.

## AWS Security Hub (Multi-Account)

- Manages security across **several accounts** and **automates security checks**.
- Three major security standards can be enabled (💡 plus NIST 800-53 and others):
  - **AWS Foundational Security Best Practices**
  - **CIS AWS Foundations Benchmark**
  - **PCI DSS**
- **Aggregates findings** from various services (in the **ASFF** format):
  - Config, GuardDuty, Inspector, Macie, IAM Access Analyzer, Systems Manager (Patch Manager), Firewall Manager, Health, Partner Network solutions, and more.
- ⚠️ **You must first enable the AWS Config service** (Security Hub's automated checks depend on Config resource recording).
- Generated findings can be used in **EventBridge** (custom actions, auto-remediation).
- **Automated checks** run continuously.
- Use **AWS Detective** to investigate its findings.
- **30-day trial**; first 10,000 finding ingestion events per account per region per month are free.
- 💡 Multi-account: use a **delegated administrator** account with Organizations, and **cross-region aggregation** to a home region.
- 💡 AWS has been evolving this product (the original posture-management feature is now called Security Hub CSPM), so check the current docs for naming/pricing.

## AWS Detective

- **Analyzes, investigates and quickly identifies the root cause** of security issues or suspicious activities using **machine learning and graph analysis**.
- Automatically collects and processes data from **VPC Flow Logs, CloudTrail, GuardDuty findings, EKS audit logs** (⚠️ Macie/Security Hub findings are things you can pivot from, rather than core data sources) and builds a **unified view** (behavior graph).
- Provides **visualizations and detailed context** to get to the root cause.
- 💡 GuardDuty must be enabled in the account for at least **48 hours** before Detective can be enabled. Retains up to a year of history.

## EC2 Placement Groups

| Strategy | Layout | Key points | Use case |
|---|---|---|---|
| **Cluster** | Same rack/hardware, **single AZ** | Low latency, **10 Gbps** per flow, highest failure risk (all-or-nothing) | **HPC**, big data jobs needing fast node-to-node network |
| **Spread** | Each instance on **distinct hardware/rack** | **Max 7 running instances per AZ per group**, can span **multiple AZs**, minimizes correlated failure, high availability | Critical apps needing isolation |
| **Partition** | Instances divided into **logical partitions (racks)**, up to **7 partitions per AZ** | Scales to **hundreds of instances per group**, partitions don't share racks, instance can read its partition info via **EC2 metadata** | **Hadoop, Kafka, Cassandra**, HDFS, HBase |

- ⚠️ Original note said partition instances are all in the same AZ. Partition groups **can span multiple AZs** in a region.
- ⚠️ Original note said spread has "higher latency than cluster": true, since instances are on different racks.
- You can **move an instance in and out** of a placement group:
  - **Stop** the EC2 instance first.
  - Use the CLI command `aws ec2 modify-instance-placement` to change it, then start it.
- 💡 Tip: use a single instance type, launch all at once, and use the same AZ to avoid "insufficient capacity" errors in cluster groups.

## EC2 Launch Types

- **On-Demand**: pay per second/hour, no commitment.
- **Spot Instances**: up to ~90% off, can be reclaimed with a **2-minute warning**. Good for fault-tolerant, flexible workloads.
- **Reserved Instances** (minimum **1 year**, up to 3 years)
  - For **long workloads**.
  - **Convertible RI**: long workload with **flexible instance type** (can change family/OS/tenancy).
  - Discount order, highest to lowest: **All Upfront > Partial Upfront > No Upfront**.
  - 💡 **Savings Plans** (Compute / EC2 Instance) are the newer, more flexible commitment model ($/hour commitment).
- **Dedicated Instances**: hardware dedicated to you, **no other customer shares the same hardware** (but you don't control placement).
- **Dedicated Hosts**: book an **entire physical server** and control instance placement.
  - Great for **software licenses** that operate at a **core or CPU socket** level (BYOL).
  - Can define **host affinity** so instance reboots stay on the **same host**.
- 💡 **Capacity Reservations**: reserve capacity in a specific AZ for any duration (no discount by itself; combine with RI/Savings Plans).
- **Graviton** (AWS ARM-based processors)
  - Delivers the **best price/performance** in EC2.
  - Supports Linux, Red Hat, SUSE, Ubuntu. **Windows is not supported.**
  - **Graviton2**: ~40% better price/performance than comparable 5th gen x86 instances.
  - **Graviton3**: ⚠️ up to ~25% better compute performance than Graviton2 (and up to 2x floating point, up to 3x for ML workloads). The original "3x better than Graviton2" holds only for ML workloads.
  - 💡 Graviton4 is the newer generation. Apps must be ARM-compatible (recompile/multi-arch images).

## EC2 Included Metrics (CloudWatch)

- **CPU**: CPU utilization + **credit usage/balance** (burstable T instances)
- **Network**: in/out
- **Status checks**:
  - **System status**: checks the underlying **hardware/host**
  - **Instance status**: checks the **EC2 VM/OS**
- **Disk I/O**:
  - ⚠️ Only for **instance store** (instance with its own local storage): read/write ops/bytes. (EBS volumes have their own metrics.)
- **RAM**: ⚠️ **Not included by default**. Use the **CloudWatch agent** for memory, disk space, etc.
- 💡 Basic monitoring = 5-minute granularity (free); detailed monitoring = 1 minute (paid).

## Instance Recovery

- **System checks**:
  - **Instance status**: problem with the OS/VM (fix: reboot)
  - **System status**: problem with the underlying hardware/host (fix: recover/migrate)
- A **CloudWatch Alarm** on `StatusCheckFailed_System` → define the action **EC2 instance recovery**.
- Things that are **retained/recovered**: private IP, public IP, Elastic IP, metadata, placement group.
- 💡 Instance **RAM contents and instance store data are lost** (recovery = restart on new hardware). Applies to EBS-backed instances only.
- Push to an **SNS topic** so others know.
- 💡 For `StatusCheckFailed_Instance` use the **reboot** action instead.

## HPC (High Performance Computing)

- Compute **very high numbers of resources**, in **no time**.

### Data Management and Transfer
- **Direct Connect**: move data into the cloud over a private, high-bandwidth link.
- **Snowball**: **petabytes** of data (offline).
- **DataSync**: install the DataSync **agent** on-prem.
  - Moves large data from on-prem to **S3, EFS, FSx for Windows** (💡 and other FSx types).

### Compute and Networking
- **CPU/GPU optimized** instances.
- **Spot instances** for low cost and **Auto Scaling**.
- 💡 Use **cluster placement groups** for the lowest latency between nodes.
- **EC2 Enhanced Networking** (SR-IOV): higher bandwidth, higher PPS (packets per second), lower latency.
  - **Option 1: Elastic Network Adapter (ENA)**, up to **100 Gbps**.
  - **Option 2: Intel 82599 VF**, up to **10 Gbps** (legacy).
- **Elastic Fabric Adapter (EFA)**
  - **Improved ENA for HPC**, only works for **Linux**.
  - Great for **inter-node communication** and **tightly coupled workloads**.
  - Leverages the **Message Passing Interface (MPI)** standard.
  - Bypasses the underlying Linux OS (OS-bypass) to provide **low-latency, reliable transport**.
  - 💡 Also used for ML training with NCCL.
- 💡 Orchestration: **AWS Batch**, **AWS ParallelCluster**.

### Storage
- **Instance-attached storage**
  - **EBS**: scales up to **256,000 IOPS with io2 Block Express**.
  - **Instance Store**: scales to **millions of IOPS**, linked to the EC2 instance, **low latency** (⚠️ ephemeral: data is lost when the instance stops/terminates).
- **Network storage**
  - **S3**: large object storage.
  - **EFS**: ⚠️ scale by using **Max I/O performance mode** and/or **Provisioned Throughput** mode (EFS has no "provisioned IOPS" mode).
  - **Amazon FSx for Lustre**
    - **Linux only**: HPC-optimized **distributed file system**, **millions of IOPS**, sub-millisecond latency.
    - **Backed by S3** (can read/write S3 data as files).
    - 💡 **Scratch** (temporary, fastest, no replication) vs **Persistent** (replicated, long-term) deployment types.
