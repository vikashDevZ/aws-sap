# aws-sap

#### Explicit Deny > Allow > Default Deny
- since Effect: Deny is explicit

#### Operators:
- String
    StringEquals, StringLike
- Numeric
    NumbercEquals, NumericNotQuals, NumericLessThan
- Date
    DateEquals, DateNotEquals, DateLessThan
- Boolean
    Condition: {"Bool": {"aws:secureTransport" : "true"}}
    Condition: {"Bool": {"aws:MultiFactorAuthPresent" : "true"}}
- ip
    Condition: {"IpAddress": {"aws.SourceIp": "203.0.113.0/24"}}
- ArnQuals, ArnLike
- Null
    Condition: {"Null" : {"aws: TokenIssueTime": "true"}}

#### IAM Policies Vairables
- ${aws:username} - arn:aws:s3:::myBucket/${aws:username}/*

### AWS Specific
- aws:CurrentTime, aws:TokenIssueTime, aws:pricipaltype, aws:SecureTrasnport,
    aws:SourceIp, aws: userid, ec2:SourceInstanceARN

#### Service Specific
- s3:prefix, s3:max-keys, s3:x-amz-acl, sns:Endpoint, sns:protocol

##### Tag Based:
- iam:ResouceTag/Key-name. aws:PrincipalTag/key-name

####
- in assume role original role give up its own permisson for the assume permisson
- resource based policy the prinical doesnt change its permission

####
- Identity Federation can have many flavors
  - SAML 2.0
    - Securtiy Accertion Markup Language 2.0
    - Open standard used by many identity providers (eg: Microsoft Active Directory Federation)
      - or any SAML 2.0 compitable IdP with AWS
    - Access to AWS Console CLI, API using temperory credentials
      - No need to create iam users for each of the employees
      - need to setup trust between both aws iam and saml 2.0 idp (both ways)
    - Under the hood: uses STS API AssumeRoleWithSAML
    - user logins in the idp portal -> AD authenticate the request -> gets SAML assertion -> STS verifies assertion and provide temperory access keys 
    - for console sign in same mechanism only change is user will post to https://signin.aws.amazon.com/saml -> that will return security credentials using the STS service and special sign in url for the console to user
    - custom identity broker will be used if users idp does not support SAML 2.0
  - web identity federation
    - restrict policy using iam policy variable
      - eg
        - cognito-identity.amazonaws.com:sub
        - amazon.com:user_id
        - graph.facebook.com:id
        - accounts.google.com:sub
          - {
                "Sid": "AccessOnlyOwnObjects",
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject",
                    "s3:PutObject",
                    "s3:DeleteObject"
                ],
                "Resource": "arn:aws:s3:::my-app-bucket/${amazon.com:user_id}/*"
            }

    - without cognito (not recommended)
      - uses third party mechanism like google, facebook, amazon login or any OpenId connect compatible IdP
      - client uses AssumeRoleWithWebIdentity API
    - with cognito
      - Preferred over web
      - cream iam roles using cognito wiht the least previlege need
      - build trust between OIDC IdP and AWS 
      - uses third party mechanism like google, facebook, amazon login or any OpenId connect compatible IdP
      - the token that client get after authenticating with third party is given to cognito insted of STS, cognito token will then be used to request sts for temperory access token
      - coginto will support anonymous user
      - support MFA
      - data syncronization

  - Custom Identity Broker
  - Web Identity With(out) Amazon Cognito  
  - IAM Identity center

#### MS Azure Directory 
  - Found on any windows server with AD domain serivces
  - Database of objects: User Accounts, Computer, Printers, File Shares, Security Groups
  - Centrzalized Security managment, create account, assign permission
  - Objects are organizad in trees
  - A group of tree is call forest

#### ADFS (AD Federation Services)
  - ADFS provides Single Sign-On across applications
  - SAML across 3rd part: AWS Console, Dropbox, office 365

### KMS is regional key
  - AWS has both symmetric (AES 256) all aws services integrated with KMS use this only
  - asymmetric key 
  - CMK with rotation policy(new key generated every year, and old is preserved)
  - can add a key policy (using resource policy) & audit its usage in cloudtrail	

  - AWS managed keys
  - used by AWS s3, ebs, redshift (automatically roated every 1 year)
  - can view key policy and auth them in cloudtrial

  - aws owned keys:
  - created and managed by aws, used by some aws resource to protect your resources.
  - used in multiple aws account but they are not in you aws account
  - autorotation is varid

  - kms key material origin:
  - kms, external kms (BYOK) (support both symmetric and asymytric) cant be used with cloudhsm, custom key store(cloudHSM)

  - KMS MULTI REGION KEY: replication key across multiple region (not one global key), can promote one replica into their own primary key

### Parameter store
  - serverless, scalable durable, version tracking, security thorugh IAM, integration with Cloudformation to get value for this.
  - KMS encryption is optional
  - Notification from event bridge
  - /aws/reference/secretmanager/secret_ID_in_Secrets_Manager
  - /aws/service/ami-amazon-linux-latest/amaz2-ami-hvm-x86_64 (public parameters)
  - Standard and advanced parameter tiers
  -                         Standard                                Advanced
  - Numbers of parameters        10000                               100,000
      allowed
  - Max size of value             4KB                                 8KB
  - Cost                    No additional charges                   Charges apply
  - Storage pricing              Free                          $ 0.05 per advanced parameter per month

  - Parameter Policies (Advanced Parameters)
    - Allow to assign TTL to a parameter (expiration date) to force updating or deleting sensitive data
    - can allow multiple policies at a time
      - type: Expiration (parameter), ExpirationNotification (event bridge), NoChangeNotification(eventbridge)

### Secret Manager
  - native interation with RDS, Redshift, DocumentDB
  - controls access to resouce using resource based policy
  - KMS encryption is mandatory

### RDS Security
  - KMS encryption at rest
  - (Transparent Data Encyrption)TDE for Oracle and SQL Server
  - SSL encryption to RDS is possible for all DB (in-flight) 
  - IAM authentication for Mysql, Postgres and MariaDB
  - Authorization still happens whitin RDS (not in IAM)
  - can copy unencrypted RDS snapshot to encrypted one
  - Cloudtrail cannot be used to track queries whitin RDS

### SSL / TLS
  - Secure socket layer
  - Transport layer security
  - DNSSEC

### ACM
  - free of cost
  - buy and import it to ACM
  - for private certificate
    - create your own CA
    - application must trst this CA, as its your private CA
  - regional service

### CloudHSM
  - AWS only provision dedicated encryption hardware
  - You mange your own encryotion ket entirely
  - FIPS 140-2 level 3 compliance 
  - supports both assymetric and assymetric encryption (TLS/SSL Key)
  - No Free tier available
  - Must use CloudHSM client software to use cloudHMS not API calls
  - Redshift support CloudHSM for database encryption and key management
  - Good option to use with SSE-C encryption
  - if we lose encryption key, AWS cannot recover
  - CloudHSM cluster must spread multiple AZ
  - Deployed and Managed in VPC, can be shared across VPC using VPC peering
  - can create users and manger their permisson seperatly unline KMS AMI
  - support cryptographic accelleration for SSL/TLS and oracle TDE
  - you can offload SSL termination of your webapps on CloudHSM, it is supported by nginx, apache, iis
    - for ssl termination function you mush setup a cryptographic user (CU) on the CloudHSM device and ensure ec2 instance can use that user

### S3
  - SSE-S3: encrypted s3 object using key handled & managed by AWS
  - SSE-KMS: leverage KMS to manage encryption key
    - Good for auditing, as api call of SSE-KMS will apper in cloudtrail
    - good security in case if bucket is made public, due to kms public user cannot read the file
    - on s3 upload ensure kms:GenereateDataKey is allowed
    - SSE-C: when you want to manage you own encryption (HTTPS is managatory)
    - Glacier: all data is AES-256 encrypted, key under AWS
    - To enforce HTTPS: use bucket policy with aws:SecureTransport 
  - Events in S3:
    - S3 Access logs:
      - Datails record of the request that are made to s3
      - Might take hours to deliver
      - Might be incomplete
    - Event Notification:
      - event when object is restored, removed, created, replication event
      - source: lambda, sqs, sns
      - event are fast (within second), but can take minutes also
      - in case you are doing operation on same object at same time to get two event enable object versioning
  - Trust Advisor
    - Check if bucket is public
  - Amazon EventBridge
    - Need to enable cloudtrail object level logging on s3 first
    - target sns, lambda, sqs, etc
  - Security
    - iam
    - Resrource based policy
    - ACL
    - Object access control list - finer grain
    - Bucket access control list - less common
  - Bucket Policy
    - grant public access to bucket
    - force objects to be encypted at uploda
    - cross account access
    - Optional condition
      - source ip, VpcSourceIp (thorugh VPC endpoint)
      - source VPC and Source VPC endpoint - only works with vpc endpoints
      - CDN origin identity
      - MFA
  - Pre Signed URl
    - Upload/Download
    - valid for default 1h
    - user given the pre signed url, inhabit the permisson of the person who genreate the url
  - VPC Gateway endpoint
    - VPC bucket policy add AWS:SourceVpc and AWS:SourceVpce
  - Object Lock
    - Adopt a WORM model(write once read many)
    - block an object for specified amount of time
  - Glacier Vault Lock
    - Adopt Worm model
    - lock the policy for future edits (means policy no loger be changed)
    - helpful for compliane and data retention
  - S3 Access points
    - simplfies s3 access security management
    - Each access points has it own DNS name (internet origin or VPC origin)
    - an access point policy (same as bucket policy) - manage security at scale
    - must create VPC endpoint (gateway or interface endpoint) for privately accessible to VPC orgin
    - VPC access poolicy should also allow s3 and accsspoint access
    - ec2 -> VPC endpoint -> AcessPoint VPC origin -> s3 Bucket
    - S3 Object lambda to modify the s3 object just before it is retrieved by the caller application by calling
      s3 access point that will trigger the lamnda that will retrieve and transform the s3 object
  - S3 Multi region Access point
    - Global Endpoints that spans s3 buckets in multi AWS region
    - Dynamically route request to nearest bucket
    - Bi-directional S3 bucket replication rules are created to keep the data in sync across region
    - FailoverControl: allows you to shift request acress s3 buckets in diffrent region with seconds (Active-Active or Active-Passive)
    - To enable replicaiton must also enable bucket versioning

### DDOS Attack
    - AWS Shield Standard
    - AWS Shield Advanced

### WAF
  - portect from layer 7 attack
  - ALB, Cloudfront, Api Gateway, App Sync (for GraphQL Api)
  - 190 Managed rules
    - Ready to use rules managed by AWS and Marketplace seller rules
    - Baseline Rule Groups:
      - AWSManagedRuleCommonRuleSet, AWSManagedRuleAdminProtectionRuleSet
    - UseCase Specific RuleGroup:
      - AWSManagedRulesSQLiRuleSet,AWSManagedRulesWindowRuleSet
      - AWSManagedRulesPHPRuleSet, AWSManagedRulesWordPressRuleSet
    - IP Reputation RuleGroup:
      - AWSManagedRulesAmazonReputationList, AWSManagedRulesAnonymousIpList
    - Bot Contorol Manage Rules:
      - AWSManagedRulesBotControlRuleSet
  - Logging
    - s3, cloudwatch
    - kinesis data firehosuse (for large traffic) - destinations can be: s3, opensearch redshift
  - Security
    - X-Origin-Verify: Mysecret to check in WAF
    - Custom HTTP Header will up updated using secret manager automatically in WAF and Cloudfront Request body

### AWS Firewall Manager
  - To manage all firewall rules of all account in AWS Organization
  - Security policy: common set security rules (WAF, Shield)
  - AWS Shield Advance Rules: ALB, CLB, NLB, Elastic Ip, Cloudfront
  - security policies for ec2, alb, eni resources in vpc
  - even network firewall at the VPC level
  - amazon route 53 resolver DNS firewall
  - Policies are create at REGION level
  - apply the same rule in newly created alb automatically

### WAF + Firewall Manager + Shield
  - WAF, Shield and Firewall manager used together for comprehensive protection
  - Define your Awb ACL rules in WAF
  - For granular protection of your resources, WAF alone is correct choice
  - if you want to use WAF across mutliple account, accelerate WAF configuration,
    automate the protection of new resources, use Firewall Manager with WAF
  - Sheild advance for DDOS which adds additonals feature on top of AWS WAF,
    such as dedicated support from SHIELD REPONSE TEAM (SRT) and advanced reporting, and it can also automatically waf rules for you.

### Block IP Address
  - NACL: first line of defence (both deny/allow rule)
  - SG: only allow rule here
  - firewall software in ec2 (eg: ufw)
  - ALB: Client -> (ALB -> ec2 in private subnet) with NACL for allow/deny rule at public subnet rule
  - SAME FLOW FOR NLB
  - WAF: CDN on top of ALB fr ip addres filtrin, NACL Will not work on CND as its not in subnet, so it will be at ALB level with SG
  - Cloudfront: Geo Restiction feature

### AWS Inspector
  - EC2
    - leveraging sms agent
    - analyze agaist unintent network accessiblity
    - analyze the running OS for known vulnerablity
  - ECR
    - Assess the contianer images as they get pushed in ECR
  - Lambda
    - when lambda deployed, it will be analyzed for software vulnerablity in code and package dependencies
  - Reports its finding in AWS Security HUB
  - Send events of its finding in AWS Event Bridge
  - It evaluates
    - running ec2, ecr and lambda funtions
    - package vulnerablity (ec2, ece and lambda) - database of CVE
    - network reachablity of EC2
    - A risk score is assocaited will all the vulnerablity for protection

### AWS CONFIG
  - Audit and recording of compliance of AWS resources
  - have record config and config changes that are done over time
  - config rules do not prevent actions from appending (deny)
  - is there unresticed ssh access? are buckets public? how my alb config are change over time
  - Alert on SNS topic for any kind of changes
  - its a per region service, so need to enable on all region whcih are required to be audited
  - all the accounts data can be aggregated in one central account
  - If Cloudtrail API Calls are enabled you can also see who made the changes here also
  - can use AWS Managed service rules (over 75)
  - we can make own custom config rule (must be defined in lambda fucntion)
    - evaluate if each ebs is of type gp3
    - evaluate if each ec2 is of type t2.micro
  - rule can be evalued/triggered
    - for each config change
    - at certain interval like every day
    - event bridge in case a rule is non compliant
  - Config rules also has deeper interation with SSM Automation
    - if resource are non complaint you can trigger an auto remidation script thorugh SSM
    - eg: change sg, stop instance with non approve tags and so on

### AWS Managed Logs
  - Load Balance: ALB/NLB/CLB access logs -> export to s3
  - Cloudtrail Logs -> export to s3
  - VPC flow logs -> export to s3, cloudwatch, kensis data firehose
  - Route53 access logs -> cloudwathc logs
    - log info of all the queries the route 53 receive
  - S3 access logs -> export to other s3 buckets for all the request made to s3 bucket
  - Cloudfront access logs -> every user requrest
  - AWS Config -> Config info can be export to s3

### AWS GuardDuty
  - Intelligent Threat Discovery to protect your AWS Account
  - Use Machine Leargnign algo, anamoly detection, 3rd party data
  - One click to enable 30 days trial, no need to install any software
  - Input Data includes:
    - Cloudtrail
      - Cloudtrail event Logs -> unusual api calls, unautorized deployments
      - Cloudtrail Management Events: create VPC Subnets, create trail
      - Cloudtrail s3 Data Events: get object, list, delte object...
    - VPC flow logs: unusual internal traffic, unusual Ip addressed
    - DNS Logs: Compromised instances sending encrypted data within DNS queries
    - Optional Features: EKS Audit logs, RDS & Aurora, EBS, Lambda, S3 Data event
  - Can setup an event bridge rules to be notified in case of finding
  - Eventbridge rule can target Lambda or SNS
  - Can protect against CryptoCurrency attack (as it has a dedicated finding for it)
  - AWS Organizatio members accont can be designated to be a GuardDuty Delegated Adminstrator
    - Basically this account will have access to enable and manage GuardDuty for all accounts in the Organization
    - To Name a Deligated Administrator this can only be done by Organization management account

### IAM Advance Policies
  - aws:SourceIp
  - aws:RequestedRegion
  - ec2:ResourceTag - eg: ec2:ResourceTag/Department: "Finance"
  - aws:PrincipalTag - eg: aws:PrincipalTag : "Data"  [This Applies to user tag]
  - aws:MultiFactorAuthPresent: false
  - aws:PrincipalOrgID (for member account access)
    - "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::2022-financial-data/*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": [
            "o-yyyyyyyyy"
          ]
        }
      }

### EC2 instace connect service (Send SSHPublicKey API) !!!IMPORTANT!!!
  - SSH	TCP	22	com.amazonaws.ap-south-1.ec2-instance-connect
  - FOR PRIVATE EC2: TCP 22 → source = security group attached to the EC2 Instance Connect Endpoint

### AWS SECURITY HUB (MULTI ACCOUNT)
  - To Manage security across SEVERAL ACCOUNTS and AUTOMATE SECURITY CHECKS
  - Thee Security Standars can be enabled
    - AWS Foundation security best practices
    - CIS AWS Foundation Benchmark
    - PCI DSS
  - Aggregate findings from various servcies
    - config
    - guardduty
    - inspector
    - macie
    - iam access analyzer
    - system manager
    - firewall manager
    - health
    - partner network solutions
    - even more :D
  - ###### MUST FIRST ENABLE AWS CONFIG SERVICE
  - generated data can be used in event bridge
  - automated checks in it
  - ##### AWS Detective to ivestigate its finding
  - 30 days trial
  - first 10k free

### AWS DETECTIVE
  - analyze, investigate, and quickly identifies the root cause of securiy issues or sus activities using ML and graph
  - automatically collects and process data from VPC FLOW LOGS, CloudTrail, Macie, Security hub, GuardDuty, and create a unfied veiw
  - Provides visualtization and detail context to get ot the root cause


### EC2 Placement Group
  - Cluster: low latency, same rack, same az
    - 10 GB/sec bendwidth
    - high failure chance
    - usecase: HPC
  - spread: spread across diff rack and diff az
    - diff harwarze across diff az
    - max 7 instance per group per az
    - for high availalblity
    - high latency then cluster
  - partiton: spread instance across many different partitions, diff sets of rack within an AZ. Scale 100 of Ec2 instance per group
    - usercase: Hadoop, kafka, casssandra
    - all insuance are within same AZ
    - insatance in partition do not share rack with instance in other partition
    - instance get access to its partition info using ec2 metadata api
  - You can move instance in and out of placement group
    - first stop the ec2
    - cli commadn to modify the instance placement

### Ec2 launch type
  - on Demand
  - spot instance
  - reserved (minimum one year)
    - long workload
    - convertable instance, long workload with flexible instance type
    - higest to lowest discount: all upfront, partial upfront, no upfront
  - dedicated instances
    - no other customer will share the same hardware
  - dedicated hosts
    - book an entire physical server, control instance placement
    - great for software licenses that operate at a core, or CPU socket level
    - can define **host affinity** so that instacne reboot are kept on the same host
  - Graviton
    - AWS Graviton Process delivers the best price performance
    - support: linux, redhat, suse, ubuntu
    - window not suppoted
    - Graviton2
      - 40% better performanace than comparable 5th gen x86 based instance
    - Graviton3
      - upto 3x better performance than **Graviton2**

### EC2 included matrix
  - CPU: CPU utilization + credit usage/ balance
  - Network: in/out
  - System Check
    - system status: underlyign hardware
    - Instance status: check the ec2 VM
  - Iops
    - Only for instacne store (instance with its own storage)
    - read/write ops/bytes
  - RAM: By default it is not included

### Instace Recovery
  - System Check
    - instance status:
    - system status:
      - StatusCheckFailed_System: Cloudwatch Alarm with cehck this status -> define an action called ec2 instance recovery**
      - things that will be recovered: private id, public ip, elastic ip, metadata, placement group
      - push to SNS topic for other to know

### HPC
  - compute very high number on resources
  - very high number of resources in no time
  - Data management and Transfer
    - Direct connect to move data into the cloud
    - Snowball: PB of data
    - DataSync: install datasync agent
      - move large data from on prem to s3, EFS, FSx for windows
  - Compute and networking
    - CPU/ GPU optimized
    - Sport instance for low cost and autoscaling
    - Ec2 Enhanve networking
      - **ENA**
        - higher bandwidth, higher PPS (packet per second), lower latency
        - Option1: **Elastic network adapter (ENA)** upto 100 Gbps
        - Option2: Intel 82599 VF upto 10 Gbps - Legacy (old ENA)
      - **Elastic Fabric Adapter (EFA)**
        - **Improved ENA for HPC**, only works for linux
        - great for interconnect nodes, tightly coupled workloads
        - Leverage *Message passing interface (MPI)* standard
        - this standard bypass underlying Linux OS to provide low latency, reliable transport

### Storage
  - Instance attached storage
    - EBS: scale up to **256,000 IOPS with io2 Block Express**
    - Instance Store: **scale to millions of IOPS, Linked to EC2 instace, low latency**
  - Network storage
    - S3
    - EFS: provisoined iops mode on EFS
    - Amazon FSx for Luster
      - Linux only: HPC optimized distributed fil system, million of IOPS
      - Backed by S3


#### Automamtion and Orchestration
  - 
