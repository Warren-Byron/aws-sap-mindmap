---
title: AWS SAP-C02 Mindmap
markmap:
  colorFreezeLevel: 2
  initialExpandLevel: 2
  maxWidth: 380
  spacingVertical: 8
  spacingHorizontal: 90
  paddingX: 10
  color:
    - '#FF9900'
    - '#DD344C'
    - '#E7157B'
    - '#ED7100'
    - '#7AA116'
    - '#F2B705'
    - '#3B48CC'
    - '#C925D1'
    - '#01A88D'
    - '#8C4FFF'
    - '#5A6B86'
    - '#00875A'
    - '#D86613'
    - '#4D7CFE'
    - '#B0084D'
    - '#0E9AA7'
    - '#A0522D'
---

# ☁️ AWS SAP-C02

## 🪪 Identity & Federation

- [ ] **IAM**
  - Building blocks
    - Users = long-term credentials · Roles = short-term creds via STS
    - EC2 instance role: delivered through instance metadata, **one role per instance**
    - Policy types: AWS managed · customer managed · inline · **resource-based** (S3, SQS, KMS…)
  - Policy evaluation
    - JSON: `Effect` · `Action` · `Resource` · `Condition`
    - ⚠️ **Explicit Deny always wins** over any Allow
    - `PowerUserAccess` uses `NotAction` (allow everything *except* IAM/Orgs) rather than Deny
  - Condition operators
    - `StringEquals` / `StringLike` · `Numeric*` · `Date*` · `Bool` · `IpAddress` · `ArnLike` · `Null`
    - Common keys: `aws:SecureTransport` · `aws:MultiFactorAuthPresent` · `aws:SourceIp` · `aws:RequestedRegion` · `aws:PrincipalTag/…` · `aws:TagKeys`
    - Policy variables: `${aws:username}` → per-user S3 prefixes
  - Role vs resource-based policy
    - Assuming a role → you **give up** your original permissions
    - Resource-based policy → principal **keeps** its own permissions
    - 💡 Exam: copy DynamoDB in Acct A → S3 in Acct B ⇒ use a **bucket policy**, not a role
  - Permission boundaries
    - Users & roles only (**not groups**); sets the *maximum* permissions
    - Effective = boundary ∩ identity policy (∩ SCP)
    - 💡 Let devs self-manage IAM without privilege escalation
  - Tag-based control
    - `aws:TagKeys` with `ForAllValues` (only these keys) vs `ForAnyValue` (at least one)

- [ ] **IAM Access Analyzer**
  - Zone of trust = your account or Organization; anything shared outside it ⇒ **finding**
  - Covers S3, IAM roles, KMS keys, Lambda, SQS, Secrets Manager
  - **Policy validation**: grammar + best-practice checks
  - **Policy generation**: builds least-privilege policy from **CloudTrail** (up to 90 days)

- [ ] **STS**
  - `AssumeRole` → temporary creds, **15 min – 12 h**
  - Key APIs
    - `AssumeRole` – same or cross-account
    - `AssumeRoleWithSAML` – SAML-authenticated users
    - `AssumeRoleWithWebIdentity` – OIDC IdP (AWS prefers **Cognito**)
    - `GetSessionToken` – MFA-protected calls
    - `GetFederationToken` – custom identity broker / proxy app
  - Cross-account: trust policy on role + `sts:AssumeRole` permission on caller; can require MFA
  - Third-party access
    - Needs their account ID + an **External ID** (chosen by them)
    - 💡 External ID prevents the **confused deputy** problem
  - Session tags → match with `aws:PrincipalTag` conditions
  - Revoke active sessions: `AWSRevokeOlderSessions` (time-based deny)

- [ ] **Identity Federation & Cognito**
  - Federation = external identities, no IAM users needed
  - **SAML 2.0** (e.g. ADFS)
    - Two-way trust IdP ↔ IAM · uses `AssumeRoleWithSAML`
    - Console access via `signin.aws.amazon.com/saml`
    - "Old way" – IAM Identity Center is the modern choice
  - **Custom identity broker**
    - Only when IdP is **not SAML-compatible**
    - Broker picks the role · `AssumeRole` / `GetFederationToken`
  - **Web identity federation**
    - Without Cognito: `AssumeRoleWithWebIdentity` – not recommended
    - With **Cognito** ✅ – anonymous users, MFA, data sync; replaces Token Vending Machine
    - Identify user in policy: `cognito-identity.amazonaws.com:sub`

- [ ] **AWS Directory Services**
  - **AWS Managed Microsoft AD**
    - Real AD in your VPC, multi-AZ, MFA, auto backups, multi-Region replication
    - Trusts with on-prem AD (one-way either way, or two-way) over **DX / VPN**
    - Integrates: RDS SQL Server, WorkSpaces, QuickSight, IAM Identity Center
    - Seamless domain join of EC2 across accounts & VPCs
    - 💡 Trust ≠ synchronisation
  - **AD Connector**
    - Proxy to on-prem AD, **no caching**, users managed on-prem only
    - No trust, no RDS SQL Server, no seamless join
  - **Simple AD**
    - Cheap Samba-based, standalone · **no MFA, no trust**, no RDS SQL Server
  - Low-latency / resilience pattern: AD replica on EC2 + trust

- [ ] **AWS Organizations**
  - Management account + OUs + member accounts
  - `OrganizationAccountAccessRole`
    - Auto-created in **new** accounts; create manually for **invited** ones
  - Feature modes
    - Consolidated billing only, or **All features** (adds SCPs)
    - Can't switch back from All features
  - Billing: volume discounts aggregated; RI & Savings Plans shared (both accounts must allow sharing)
  - Moving an account: leave old org → invite → accept
  - Multi-account strategy
    - Split by dept / env / cost centre / regulation / service limits
    - Central CloudTrail + CloudWatch Logs in a logging account

- [ ] **AWS Organizations Policies**
  - **SCPs**
    - Allow/deny lists at OU or account level
    - Apply to all users & roles **including root** of member accounts
    - ⚠️ Don't apply to the **management account** or **service-linked roles**
    - Need an explicit Allow at **every level** from root to the account
    - Patterns: deny a Region (`aws:RequestedRegion`), require tags on create
    - Example: deny everything outside approved Regions (global services exempt)
      ```json
      {
        "Effect": "Deny",
        "NotAction": ["iam:*", "organizations:*", "sts:*", "support:*"],
        "Resource": "*",
        "Condition": {
          "StringNotEquals": { "aws:RequestedRegion": ["eu-west-1", "eu-west-2"] }
        }
      }
      ```
  - **Tag policies** – standardise tag keys/values, report non-compliance (EventBridge)
  - **AI services opt-out** – stop AWS using your content (Lex, Comprehend…)
  - **Backup policies** – org-wide AWS Backup plans, read-only in members
  - [Policy evaluation logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)

- [ ] **AWS IAM Identity Center**
  - Successor to AWS SSO: one login for all Org accounts + SAML apps + EC2 Windows
  - Identity source: built-in store, AD, or external IdP (Okta, OneLogin…)
  - **Permission sets** → become IAM roles in each assigned account
  - **ABAC** via user attributes (cost centre, title…)

- [ ] **AWS Control Tower**
  - Sets up a governed multi-account **landing zone** on top of Organizations
  - **Account Factory** – pre-baselined accounts via **Service Catalog**
  - Guardrails (controls)
    - **Preventive** = SCPs · **Detective** = AWS Config
    - Levels: Mandatory · Strongly recommended · Elective
  - Non-compliance → SNS / Lambda remediation

- [ ] **AWS Resource Access Manager (RAM)**
  - Share resources across accounts / your Org – avoid duplication
  - **VPC subnet sharing**
    - Same Org only; not default VPC / default SGs
    - Each account owns its own resources; private-IP traffic; can reference SGs across accounts
  - Also: Transit Gateway, Route 53 Resolver rules, prefix lists, License Manager, Aurora, Private CA, Glue, Network Firewall…
  - **Managed prefix lists** – share CIDR sets; update once, all SGs/route tables follow

- [ ] **Summary of Identity & Federation**
  - 🧭 Which federation?
    - SAML IdP → Identity Center (or SAML 2.0)
    - Non-SAML IdP → custom broker + `GetFederationToken`
    - Web/mobile apps → **Cognito**
  - 🧭 Which directory?
    - Full AD in AWS / trusts → Managed Microsoft AD
    - Just proxy to on-prem → AD Connector
    - Cheap & standalone → Simple AD

## 🛡️ Security

- [ ] **CloudTrail**
  - API audit history, **on by default**, 90-day retention → S3 + Athena for longer
  - Event types
    - **Management** (logged by default) · read vs write split
    - **Data** events – S3 object ops, Lambda Invoke (off by default, high volume)
    - **Insights** – baseline of normal writes → detects unusual bursts/limits
  - Hardening the log bucket: versioning, MFA Delete, Object Lock, SSE-KMS, lifecycle, **log file integrity validation** (SHA-256)
  - Org trail → one central bucket (bucket policy allows cross-account delivery)
  - 💡 Fastest reaction to an API call
    - **EventBridge** (near real-time) ⚡
    - CloudWatch Logs (streamed + metric filters)
    - S3 delivery (~every 5 min) · overall up to 15 min
  - Deleted resource? Check CloudTrail first

- [ ] **KMS**
  - Regional keys · IAM-integrated · every call audited in CloudTrail
  - Key types
    - **Symmetric** AES-256 – used by AWS services, envelope encryption
    - **Asymmetric** RSA/ECC – sign/verify or encrypt outside AWS with public key
  - Ownership
    - Customer managed – key policy, rotation 90–2560 days (optional)
    - AWS managed (`aws/s3`…) – yearly rotation, view only
    - AWS owned – invisible to you
  - Key material origin
    - `AWS_KMS` (default) · `EXTERNAL` (BYOK, **no auto rotation**) · `AWS_CLOUDHSM` custom key store
  - **Multi-Region keys** – same key ID/material in many Regions; primary + replicas (not global)
    - 💡 DR, DynamoDB Global Tables, client-side encryption across Regions

- [ ] **Parameter Store & Secrets Manager**
  - **SSM Parameter Store** ($)
    - Hierarchical paths `/dept/app/env/key` · versioned · optional KMS
    - Advanced tier → **parameter policies** (TTL/expiry, EventBridge notifications)
    - No native rotation (DIY with EventBridge + Lambda)
  - **Secrets Manager** ($$$)
    - **Automatic rotation** via Lambda (built-in for RDS, Redshift, DocumentDB)
    - KMS mandatory · resource policy for cross-account sharing
  - 💡 "Rotate DB password automatically" ⇒ Secrets Manager

- [ ] **RDS Security**
  - At rest: KMS (EBS + snapshots); **TDE** for Oracle & SQL Server
  - In transit: SSL for all engines
  - **IAM DB auth**: MySQL, PostgreSQL, MariaDB (authorization still inside DB)
  - Encrypt existing DB: snapshot → copy encrypted → restore
  - ⚠️ CloudTrail doesn't log SQL queries

- [ ] **SSL/TLS, ACM & CloudHSM**
  - **SNI** – many certs on one listener; ALB, NLB, CloudFront (**not CLB**)
  - MITM defence: HTTPS everywhere + **DNSSEC** (Route 53 supports it)
  - **ACM**
    - Free public certs, auto-renewed; private certs via Private CA
    - Integrates with ELB, CloudFront, API Gateway
    - ⚠️ Regional – issue per Region; **CloudFront certs must be in `us-east-1`**
  - **CloudHSM**
    - Dedicated single-tenant HSM, **you** manage keys, FIPS 140-2 L3
    - Multi-AZ cluster for HA · good with SSE-C, Oracle TDE
    - SSL offload on EC2 (NGINX/Apache) – private key never leaves HSM
  - KMS vs CloudHSM: multi-tenant & AWS-managed vs single-tenant & customer-managed

- [ ] **S3 Security**
  - Encryption: **SSE-S3** · **SSE-KMS** (audited; needs `kms:GenerateDataKey`) · **SSE-C** (HTTPS required) · client-side
  - Force HTTPS: bucket policy `aws:SecureTransport = false → Deny`
  - Bucket policy conditions: `aws:SourceIp` (public) vs `aws:SourceVpce` / `aws:SourceVpc` (endpoints)
  - `s3:ListBucket` → bucket ARN · `GetObject/PutObject` → `bucket/*`
  - **Pre-signed URLs** – inherit creator's permissions, default 1 h
  - **Object Lock** / **Glacier Vault Lock** – WORM compliance
  - **Access Points** – per-team DNS name + policy; VPC-origin access points
  - **Multi-Region Access Points** – one global endpoint, nearest bucket, failover controls
  - **S3 Object Lambda** – transform on read (redact PII, resize, XML→JSON)
  - Event options: access logs (best effort) · event notifications · EventBridge (needs CloudTrail data events)

- [ ] **DDoS, Shield & WAF**
  - Attack types: SYN flood, UDP reflection, DNS flood, Slowloris, cache-busting
  - **Shield Standard** – free L3/L4 · **Shield Advanced** – $3k/mo, DRT 24/7, cost protection
  - **WAF** (Layer 7) on ALB, API Gateway, CloudFront, AppSync, Cognito
    - Web ACL rules: IP, headers, body, geo, size, SQLi, XSS, **rate-based**
    - Managed rule groups: common, SQLi, IP reputation, **Bot Control**
    - Logs → CloudWatch Logs, S3, Firehose
    - ⚠️ WAF ≠ DDoS protection on its own
  - Pattern: CloudFront adds secret header (rotated via Secrets Manager) → WAF on ALB checks it
  - **Firewall Manager** – org-wide WAF, Shield Advanced, SGs, Network Firewall, DNS Firewall; auto-applies to new resources
  - 💡 Blocking an IP
    - NACL (only works if client IP is visible – not behind CloudFront)
    - Behind CloudFront ⇒ **WAF on CloudFront**

- [ ] **Inspector, Config & GuardDuty**
  - **Inspector** – CVE scans for **EC2 (via SSM agent), ECR images, Lambda**; network reachability
  - **AWS Config**
    - Records config history & compliance; **doesn't prevent** anything
    - Managed or custom (Lambda) rules · auto-remediation via **SSM Automation**
    - Aggregate across accounts/Regions
  - **GuardDuty**
    - ML threat detection on CloudTrail, VPC Flow Logs, DNS logs (+ EKS, RDS, S3, Lambda…)
    - Findings → EventBridge → Lambda/SNS · crypto-mining detection
    - Delegated administrator for the Org
  - Managed log destinations: ALB/NLB, S3, CloudFront → S3 · Flow Logs → S3/CWL/Firehose · Route 53 → CWL

- [ ] **Security Hub & Detective**
  - **Security Hub** – central findings dashboard across accounts (needs Config)
    - Aggregates GuardDuty, Inspector, Macie, Config, Access Analyzer, Firewall Manager…
  - **Detective** – root-cause investigation using graphs over Flow Logs, CloudTrail, GuardDuty
  - Useful condition: `aws:PrincipalOrgID` – restrict resource policy to your Org
  - EC2 Instance Connect – pushes temp SSH key; allow its IP range in SGs

## 🖥️ Compute & Load Balancing

- [ ] **EC2 Fundamentals**
  - Families: **R** RAM · **C** CPU · **M** balanced · **I** local I/O · **G** GPU · **T** burstable
  - **Graviton** (ARM) – best price/perf, Linux only (no Windows)
  - Placement groups
    - **Cluster** – one AZ, lowest latency, 10 Gbps+ · one rack failure = all down
    - **Spread** – separate hardware, **max 7 per AZ** · critical HA
    - **Partition** – up to 7 partitions/AZ, 100s of instances · Hadoop, Cassandra, Kafka
    - Move instance: stop → `modify-instance-placement` → start
  - Purchasing: On-Demand · Spot (≤90% off) · Reserved / Convertible (1–3 yr) · Dedicated Instance · **Dedicated Host** (per-socket/core licences, host affinity)
  - Metrics: CPU, network, disk (instance store only), status checks – ⚠️ **RAM needs CloudWatch agent**
  - **Instance recovery** (system check alarm) keeps private/public/Elastic IP, metadata, placement group

- [ ] **High Performance Computing (HPC)**
  - Data in: Direct Connect · Snowball · DataSync
  - Networking
    - **ENA** enhanced networking up to 100 Gbps (SR-IOV)
    - **EFA** – Linux only, OS-bypass, **MPI** for tightly coupled jobs
  - Storage: io2 Block Express (256k IOPS) · instance store (millions IOPS) · **FSx for Lustre** (S3-backed)
  - Orchestration: **AWS Batch** (multi-node parallel jobs) · **ParallelCluster** (open-source, text config)
  - 💡 Cluster placement group + EFA = HPC answer

- [ ] **Auto Scaling**
  - Dynamic policies: **target tracking** · simple/step · scheduled · **predictive**
  - Good metrics: CPU, `RequestCountPerTarget`, network in/out, custom
  - **Lifecycle hooks** – act before in-service / before terminate (logs, cleanup)
  - New AMI → update launch template → **Instance Refresh** (min healthy %, warm-up)
  - Suspendable processes: Launch, Terminate, HealthCheck, ReplaceUnhealthy, **AZRebalance**, AddToLoadBalancer…
  - Health checks: EC2 status · ELB · custom (`set-instance-health`) – keep them simple
  - Blue/green: second ASG / target group behind ALB, or Route 53 weighted

- [ ] **Spot Instances & Spot Fleet**
  - Max price; 2-min interruption notice → stop/terminate
  - Batch, analytics, fault-tolerant work – ⚠️ not for DBs/critical jobs
  - Spot Fleet allocation strategies
    - `lowestPrice` · `diversified` (availability) · `capacityOptimized`
    - **`priceCapacityOptimized`** ✅ recommended

- [ ] **Amazon ECS & Fargate**
  - Concepts: cluster · service · task definition · task
  - IAM: **instance profile** (agent) vs **task role** (per-task app permissions)
  - ALB **dynamic port mapping** – many tasks per host
  - Secrets: inject from Parameter Store / Secrets Manager as env vars
  - Network modes: none · bridge · host · **awsvpc** (ENI per task, SGs; Fargate default)
  - Service Auto Scaling (Application Auto Scaling) ≠ EC2 ASG scaling
  - Spot: EC2 Spot capacity (draining) or **FARGATE_SPOT** on top of on-demand baseline
  - **ECS Anywhere** – `EXTERNAL` launch type on-prem, needs SSM agent + stable connection

- [ ] **Amazon ECR**
  - Private & public registries, IAM-controlled
  - Cross-Region & cross-account **replication**
  - Image scanning: basic (on push / manual) or **enhanced via Inspector**
  - Lifecycle policies, tags, versioning

- [ ] **Amazon EKS**
  - Managed Kubernetes – choose when already using K8s / cloud-agnostic
  - Nodes: managed node groups · self-managed · Fargate
  - Storage via CSI: EBS, **EFS (works with Fargate)**, FSx Lustre, FSx ONTAP
  - One cluster per Region · Container Insights for metrics/logs
  - **EKS Anywhere** – on-prem using EKS Distro (connected or fully disconnected)

- [ ] **AWS Lambda**
  - Limits
    - RAM 128 MB–10 GB (CPU scales with RAM) · **15 min** timeout · `/tmp` 10 GB
    - Package 50 MB zip / 250 MB unzipped · container image 10 GB
    - Payload **6 MB sync / 256 KB async**
    - **1,000** concurrent (soft) · reserved concurrency caps a function
  - Invocation
    - **Sync** (API GW, SDK) – client retries
    - **Async** (S3, SNS, EventBridge) – 3 attempts, make idempotent, DLQ / destinations
  - CodeDeploy traffic shifting on aliases: Linear · **Canary** · AllAtOnce + pre/post hooks
  - In a VPC: needs NAT for internet; fixed public IP ⇒ private subnet + NAT GW + EIP
  - Observability: CloudWatch Logs/Metrics · X-Ray (enable + IAM)

- [ ] **Elastic Load Balancing**
  - Types: **CLB** (legacy) · **ALB** L7 · **NLB** L4 TCP/UDP/TLS · **GWLB** L3
  - **ALB** – path/host/header/query routing, redirects, HTTP/2, WebSocket
    - Targets: EC2, ECS, **Lambda**, private IPs
  - **NLB** – millions RPS, ultra-low latency, **static IP / EIP per AZ**, zonal DNS names
    - Targets: EC2, private IPs, **an ALB**
  - **GWLB** – inline 3rd-party appliances (firewalls, IDS) via **GENEVE 6081**
  - **CLB** – only one with 2-way TLS passthrough via TCP (legacy)
  - Cross-zone LB: ALB always on (free) · NLB/GWLB off by default (inter-AZ $)
  - Sticky sessions (cookie) – can unbalance load
  - Routing algorithms: least outstanding requests (ALB) · round robin · **flow hash** (NLB)

- [ ] **API Gateway**
  - Limits: **29 s** timeout · **10 MB** payload ⇒ large uploads via **S3 pre-signed URL**
  - Integrations: Lambda · HTTP (on-prem, ALB) · **AWS service proxy** (SQS, Step Functions…)
  - Endpoints: **Edge-optimised** (default) · **Regional** · **Private** (interface endpoint + resource policy)
  - Caching per stage (default TTL 300 s, 0.5–237 GB)
  - Errors: 403 WAF/denied · **429 throttled** · 502 bad Lambda output · **504 timeout**
  - Auth: IAM (SigV4) · **Lambda authorizer** (OAuth/SAML/JWT) · **Cognito User Pools**
  - Usage plans + API keys – throttle & quota per customer
  - **WebSocket APIs** – chat/games; reply via `@connections` URL
  - Logs → CloudWatch or Firehose · X-Ray tracing

- [ ] **AWS AppSync**
  - Managed **GraphQL** – combine DynamoDB, Aurora, OpenSearch, HTTP, Lambda
  - Real-time subscriptions over WebSocket · offline sync for mobile
  - Authorisation by **Cognito groups** in schema

- [ ] **Route 53**
  - Records: A · AAAA · CNAME (**not at zone apex**) · NS · **Alias** (free, apex OK, AWS targets)
    - Alias targets: ELB, CloudFront, API GW, Beanstalk, S3 website, VPC endpoint, Global Accelerator – ⚠️ **not EC2 DNS**
  - Routing policies
    - Simple · **Weighted** · **Latency** · **Failover** · **Geolocation** (needs default) · **Geoproximity** (bias, Traffic Flow) · **Multi-value** (≤8 healthy) · **IP-based** (CIDR → endpoint)
  - Health checks
    - Endpoint (≈15 global checkers, 2xx/3xx, text in first 5,120 bytes)
    - **Calculated** (AND/OR/NOT of ≤256 children)
    - **CloudWatch alarm** ⇒ only way to check **private** resources
  - Private hosted zones need `enableDnsHostnames` + `enableDnsSupport`
  - DNSSEC – public zones only · 3rd-party registrar: update NS records

- [ ] **Route 53 Resolver (Hybrid DNS)**
  - **Inbound endpoint** – on-prem → resolves AWS private zones
  - **Outbound endpoint** – VPC → on-prem DNS via **forwarding rules**
  - Rules: forwarding · system (override) · most specific wins
  - Endpoints in 2 AZs · 10k QPS per IP · share rules with **RAM**

- [ ] **AWS Global Accelerator**
  - **2 static anycast IPs** → edge → AWS backbone → EIP/EC2/ALB/NLB
  - Health checks, regional failover < 1 min, Shield built in
  - vs CloudFront
    - CloudFront = HTTP caching at edge
    - GA = TCP/UDP proxy – gaming, IoT, VoIP, **static IPs**, fast failover

- [ ] **Architecture Comparisons**
  - EC2 + EIP – cheap, quick failover, doesn't scale
  - Route 53 multi-record – DNS TTL causes stale clients
  - **ALB + ASG** – classic; slower to scale (boot time), pre-warm for spikes
  - ALB + ECS on EC2 – dynamic ports; two scaling layers to coordinate
  - **ALB + Fargate** – fast, easy service scaling
  - ALB + Lambda – hybrid microservices, WAF support
  - **API GW + Lambda** – serverless, pay per request, cold starts
  - **API GW → AWS service** – no Lambda, cheaper (watch 10 MB limit)
  - API GW → HTTP backend – add auth/throttling to on-prem or ALB

- [ ] **Outposts, Wavelength & Local Zones**
  - **Outposts** – AWS racks on-prem (EC2, EBS, S3 on Outposts, ECS, EKS, RDS, EMR); you own physical security
  - **Wavelength** – AWS inside telco 5G networks; ultra-low latency
  - **Local Zones** – extend a Region to metro areas (EC2, RDS, ECS, EBS, ElastiCache, DX)

## 💾 Storage

- [ ] **EBS & Instance Store**
  - EBS = network block drive, **locked to one AZ** (move via snapshot)
  - Types: **gp2/gp3** general · **io1/io2 Block Express** (up to 256k IOPS) · **st1** throughput HDD · **sc1** cold HDD
    - Boot volumes: SSD types only
  - Snapshots: incremental, stored in S3, copy cross-Region for DR
    - Restored volumes are lazy-loaded ⇒ **Fast Snapshot Restore** or pre-warm with `fio`/`dd`
  - **Data Lifecycle Manager** – tag-based snapshot/AMI schedules (only what DLM created)
    - vs **AWS Backup** – central, multi-service backups
  - Encryption by default = **account setting per Region**
  - **Multi-Attach** (io1/io2) – same AZ, cluster-aware FS required
  - **Instance store** – millions IOPS, ephemeral (lost on stop/terminate), can't resize · buffers/cache/scratch

- [ ] **Amazon EFS**
  - Managed **NFSv4.1**, POSIX, Linux only, multi-AZ, on-prem via DX/VPN, ~3× gp2 price
  - Performance mode: **General Purpose** (latency) · **Max I/O** (parallel big data)
  - Throughput mode: Bursting · Provisioned · **Elastic** (unpredictable)
  - Storage tiers: Standard · **IA** · **Archive** + lifecycle · One Zone options (≈90% cheaper)
  - **Access points** – enforce POSIX user/group + root directory per app
  - **File system policies** – resource policy like S3 bucket policy
  - Cross-Region replication – RPO/RTO in minutes

- [ ] **Amazon S3**
  - Anti-patterns: many tiny files, POSIX/locks, search/query, dynamic websites
  - Storage classes (all 11 9's durability)
    - Standard · **Intelligent-Tiering** · Standard-IA (30 d) · **One Zone-IA** (1 AZ)
    - Glacier **Instant** (90 d) · Glacier **Flexible** (90 d) · **Deep Archive** (180 d, cheapest)
  - Replication (versioning required): **CRR** / **SRR** · **Replication Time Control** = 99.99% in 15 min
  - Event notifications → SNS/SQS/Lambda, or **EventBridge** (advanced filters, archive/replay, many targets)
  - Performance
    - **3,500 writes / 5,500 reads per second per prefix** – spread prefixes to scale
    - **Multipart upload** – recommended >100 MB, required >5 GB; lifecycle rule to abort incomplete parts
    - **Transfer Acceleration** (edge upload) · **byte-range fetches** (parallel/partial GETs)
  - **Storage Class Analysis** – suggests Standard→IA transitions (24–48 h to start)
  - **Storage Lens** – Org-wide dashboards: cost, data protection, activity; free 14 days / advanced 15 months
  - Pattern: S3 event → Lambda → **DynamoDB index** for searchable object metadata

- [ ] **Amazon FSx**
  - **FSx for Windows** – SMB/NTFS, AD integration, DFS namespaces, Multi-AZ, SSD or HDD
  - **FSx for Lustre** – HPC/ML, 100s GB/s
    - **S3-linked**, lazy loads only the data you process
    - **Scratch** (fast, not replicated) vs **Persistent** (replicated in AZ)
  - **FSx for NetApp ONTAP** – NFS + SMB + **iSCSI**, auto-grow, instant clones, dedupe
  - **FSx for OpenZFS** – NFS, 1M IOPS, < 0.5 ms, instant clones
  - 💡 Storage can only **grow** – to shrink or go Single→Multi-AZ, create new FSx + **DataSync**

- [ ] **AWS DataSync**
  - On-prem / other cloud (NFS, SMB, HDFS, S3 API) → S3, EFS, FSx – **agent required**
  - AWS ↔ AWS storage – no agent
  - Scheduled tasks · **preserves permissions & metadata** · 10 Gbps per agent, bandwidth throttling
  - Private transfer: DX private VIF + **PrivateLink interface endpoint**

- [ ] **AWS Data Exchange**
  - Subscribe to third-party datasets → load into S3
  - Also Data Exchange for **Redshift** (query shared data) and for **APIs**

- [ ] **AWS Transfer Family**
  - Managed **SFTP / FTPS / FTP** into S3 or EFS, multi-AZ
  - Auth: built-in, AD, LDAP, Okta, Cognito, custom
  - Endpoints
    - Public (AWS IPs change)
    - VPC internal (static private IPs, SG/NACL allow lists)
    - **VPC internet-facing** (Elastic IPs + SGs) ⇒ partners need fixed IPs

## ⚡ Caching

- [ ] **Amazon CloudFront**
  - CDN with 600+ PoPs; integrates Shield, WAF, Route 53; supports WebSocket
  - Origins
    - S3 (secure with **Origin Access Control**) – download *and* upload
    - **VPC origin** – private ALB/NLB/EC2, nothing exposed publicly
    - Custom HTTP: public ALB, EC2, S3 website, API Gateway, on-prem
  - Protect a public custom origin
    - CloudFront adds **secret custom header** → ALB rule only accepts it
    - and/or SG allows only **CloudFront IP ranges** (managed prefix list)
  - **Origin groups** – primary + secondary failover (even cross-Region)
  - **Geo restriction** – allow/block lists by country
  - Price classes: All · 200 · 100 (cheapest Regions only)
  - **Signed URLs / cookies** (path, IP, expiry; trusted key groups) vs **S3 pre-signed URLs** (IAM identity of signer)
  - Custom error pages + error caching TTL
  - CloudFront vs S3 CRR: global TTL cache vs near real-time copies in a few Regions

- [ ] **CloudFront Functions & Lambda@Edge**
  - **CloudFront Functions**
    - JavaScript, < 1 ms, millions RPS, **viewer request/response only**, 1/6 the price
    - Cache-key normalisation, header rewrite, URL redirect, JWT validation
  - **Lambda@Edge**
    - Node.js/Python, up to 5 s (viewer) / 30 s (origin), all 4 triggers
    - Network & file system access, request body, 3rd-party libs
    - Authored in **us-east-1**, replicated globally
  - Can combine, but not both on the same viewer event
  - Patterns: device-based redirects, A/B testing, route to nearest origin, global app with DynamoDB

- [ ] **Amazon ElastiCache**
  - Managed **Redis** / **Memcached** – needs **app code changes**
  - Patterns: DB read cache (needs invalidation strategy) · **session store** for stateless apps
  - **Redis**: Multi-AZ auto-failover, read replicas, persistence & backups, sharding
  - **Memcached**: multi-threaded, sharding, non-persistent
  - 📊 Handling extreme rates
    - S3 3.5k PUT / 5.5k GET per prefix · SQS FIFO 3k msg/s with batching
    - Kinesis shard 1 MB/s in, 2 MB/s out · DAX up to 10 nodes · Redis up to 500 nodes

## 🗄️ Databases

- [ ] **DynamoDB**
  - Serverless NoSQL, millions RPS, **400 KB item limit**
  - Capacity: provisioned (RCU/WCU + auto scaling) or **on-demand**
  - Keys: partition key only, or partition + sort key – choose a high-cardinality PK
  - Indexes
    - **LSI** – same PK, different sort key, **only at table creation**
    - **GSI** – new PK/SK, add any time
  - Features: TTL · ACID transactions · PITR backups · Standard / Standard-IA table classes
  - **Streams** (24 h) → Lambda · or **Kinesis Data Streams** for longer retention
  - **Global Tables** – multi-Region **active-active** (requires streams)
  - **DAX** – write-through cache, µs reads, fixes **hot keys**, no code rewrite, ≤10 nodes
    - DAX = item/query cache · ElastiCache = computed/aggregated results
  - Pattern: DynamoDB Streams → Lambda → **OpenSearch** for search

- [ ] **Amazon OpenSearch**
  - Managed (ex-Elasticsearch) – cluster or **serverless**
  - Full-text search, log analytics, clickstream, security analytics
  - OpenSearch Dashboards (ex-Kibana) · Logstash ingestion
  - Ingest: CloudWatch Logs subscription → Lambda (real-time) or **Firehose** (near real-time)

- [ ] **Amazon RDS**
  - Engines: PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, Db2
  - **Multi-AZ** = sync standby, one DNS name, auto failover
  - **Read replicas** = async, scale reads (can be cross-Region)
  - Backups: automated PITR · manual snapshots (copy cross-Region)
  - **IAM DB auth** – 15-min token, SSL required (MySQL, PostgreSQL, MariaDB)
  - Oracle tips
    - RDS backups → restore to RDS · **RMAN** → restore to non-RDS
    - ⚠️ **No RAC on RDS** – use EC2 · TDE supported
  - MySQL to non-RDS: `mysqldump`
  - **RDS Proxy** – pools connections (Lambda "too many connections"), IAM auth, never public
  - Pattern: CloudWatch alarm → Route 53 health check → promote replica in DR Region

- [ ] **Amazon Aurora**
  - MySQL/PostgreSQL compatible · storage auto-grows to 256 TB
  - **6 copies across 3 AZs** (4/6 write, 3/6 read quorum), self-healing
  - Up to 15 replicas · failover < 30 s
  - Endpoints: **writer** · **reader** (load-balanced) · **custom** (subset, e.g. analytics) · instance
  - Logs: error, slow query, general, audit → CloudWatch Logs
  - Troubleshooting: **Performance Insights** · Enhanced Monitoring (per-second OS) · slow query log
  - **Aurora Serverless** – auto capacity, intermittent workloads · **Data API** (HTTPS SQL, creds in Secrets Manager)
  - **Aurora Global Database** ✅
    - 1 primary + up to 10 secondary Regions, < 1 s lag
    - Promote secondary for DR (RTO < 1 min) · **write forwarding** from secondaries
  - Migrate RDS → Aurora: snapshot restore, or Aurora read replica then promote

- [ ] **Redshift**
  - Columnar **OLAP** warehouse, MPP, provisioned or serverless
  - Leader node plans · compute nodes execute
  - Load from S3, Firehose, DynamoDB, DMS · **Enhanced VPC Routing** keeps COPY/UNLOAD in VPC
  - Snapshots incremental to S3 · automated (8 h / 5 GB) or manual · **cross-Region snapshot copy** (KMS needs snapshot copy grant)
  - **Spectrum** – query S3 directly from a running cluster
  - **WLM** – query queues/priorities · **Concurrency Scaling** – burst clusters per second
  - 💡 Sporadic queries → Athena; sustained → Redshift

- [ ] **DocumentDB & Timestream**
  - **DocumentDB** – MongoDB-compatible, Aurora-like (3 AZ, grows in 10 GB steps)
  - **Timestream** – serverless time-series DB; memory + magnetic tiers; IoT & ops metrics

## 🔗 Service Communication

- [ ] **AWS Step Functions**
  - JSON state machine: sequence, parallel, choice, wait, retry/catch, **human approval**
  - Integrations: optimised (Lambda, Batch, ECS run-and-wait, DynamoDB, SNS, SQS, EMR, Glue, SageMaker) + **SDK for 200+ services**
  - Task types: Lambda · **Activity** (worker polls – on-prem, EC2) · Service · Wait
  - Triggers: console, SDK, API Gateway, EventBridge, CodePipeline, another state machine
  - **Standard vs Express**
    - Standard – up to **1 year**, exactly-once, priced per transition
    - Express – up to **5 min**, at-least-once, 100k+/s, priced per execution
    - Express sync (wait for result) vs async (fire-and-forget)
  - Failures → EventBridge → SNS alert

- [ ] **Amazon SQS**
  - Serverless queue for decoupling · max message 1 MB (bigger ⇒ pointer to S3)
  - Write buffer in front of DynamoDB / DB
  - **FIFO** – ordered, exactly-once processing, 300 msg/s (3,000 batched)
  - **DLQ** after `maxReceiveCount` – type must match source · keep 14 days · **redrive** back when fixed
  - ⚠️ At-least-once ⇒ make consumers **idempotent** (upsert, not insert)
  - Lambda event source mapping: long polling, batch size, visibility timeout ≥ 6× function timeout; DLQ on the **queue**
  - Request/response pattern with temporary reply queues

- [ ] **Amazon MQ**
  - Managed **ActiveMQ / RabbitMQ** for open protocols: **MQTT, AMQP, STOMP, OpenWire, WSS**
  - 💡 Lift legacy brokers (IBM MQ, TIBCO) without rewriting to SQS/SNS
  - Runs on brokers – Multi-AZ failover, doesn't scale like SQS/SNS

- [ ] **Amazon SNS**
  - Pub/sub: 12.5M subscriptions per topic · 100k topics
  - Mobile push (APNS, FCM) · access policy for cross-account/S3 publish
  - **Fan-out: SNS → many SQS** – persistence, retries, cross-Region queues
    - 💡 Only one S3 event rule per event-type + prefix ⇒ fan-out via SNS
  - **FIFO topics** – order by message group ID, dedupe; SQS FIFO subscribers
  - **Message filtering** – JSON filter policy per subscription
  - Delivery retries (custom policy for HTTP/S only) → **DLQ per subscription**
  - SNS → Firehose → S3 for archiving

## 📊 Data Engineering

- [ ] **Kinesis Data Streams**
  - Real-time, retention **up to 365 days**, replayable, ordered per partition key
  - Records up to 10 MiB · KPL producers · KCL consumers
  - **Provisioned**: per shard 1 MB/s (1k rec/s) in, 2 MB/s out
  - **On-demand**: starts 4 MB/s, scales from 30-day peak
  - Latency ~200 ms (shared) / ~70 ms (**enhanced fan-out**)

- [ ] **Amazon Data Firehose**
  - Fully managed, **near real-time** (buffer by size or time), no storage/replay
  - Destinations: **S3, Redshift (via S3), OpenSearch, Splunk**, HTTP partners
  - Lambda transforms · JSON → **Parquet/ORC** (S3) · compression
  - 💡 Need replay or custom consumers ⇒ Data Streams; just deliver ⇒ Firehose

- [ ] **Managed Service for Apache Flink**
  - Run Flink (Java/Scala/SQL) on streams – windowing, aggregation, checkpoints
  - Sources: Data Streams or MSK · ⚠️ **not Firehose**

- [ ] **Amazon MSK (Kafka)**
  - Managed Kafka brokers + ZooKeeper in your VPC, multi-AZ; data on EBS indefinitely
  - **MSK Serverless** option
  - vs Kinesis: messages > 1 MB configurable, topics/partitions (add only), PLAINTEXT or TLS
  - Consumers: Flink, Glue streaming, Lambda, EC2/ECS/EKS apps

- [ ] **AWS Batch**
  - Batch jobs as **Docker images**, no time limit
  - Managed compute env: EC2 On-Demand/Spot or Fargate/Fargate Spot (private subnet needs NAT or ECS endpoints)
  - Unmanaged: your own EC2/ECS/EKS
  - **Multi-node parallel** jobs for HPC – ⚠️ no Spot, best with cluster placement group
  - Schedule with EventBridge · orchestrate with Step Functions
  - vs Lambda: no 15-min limit, any runtime, EBS/instance store disk

- [ ] **Amazon EMR**
  - Managed Hadoop/Spark/Hive/HBase/Presto/Flink on EC2
  - Nodes: **master** · **core** (HDFS data) · **task** (compute only – ideal for **Spot**)
  - Long-running vs **transient** clusters · uniform instance groups vs **instance fleets**
  - Storage: HDFS on EBS (scratch) · **EMRFS on S3** (durable)

- [ ] **AWS Glue**
  - Serverless ETL (Spark) – prep data for analytics, e.g. CSV → Parquet
  - **Glue Data Catalog** – metadata used by Athena, Redshift Spectrum, EMR
  - Crawlers populate the catalog

- [ ] **Amazon Athena**
  - Serverless SQL on S3 (Presto/Trino) · $5/TB scanned
  - ⚡ Cost/perf: **Parquet/ORC**, compression, **partitioning** (`year=/month=/day=`), files > 128 MB
  - **Federated query** via Lambda connectors (RDS, DynamoDB, CloudWatch Logs…)
  - 💡 Analyse VPC Flow Logs, ALB logs, CloudTrail, CUR in S3 ⇒ Athena

- [ ] **Amazon QuickSight**
  - Serverless BI dashboards · **SPICE** in-memory engine · per-session pricing
  - Users/groups are QuickSight-only (not IAM) · enterprise: row-level security
  - Dashboards = published read-only snapshot of an analysis

- [ ] **Data Pipelines & Comparisons**
  - Ingest: IoT/Kinesis → Firehose → S3 → Glue/Lambda → Athena → QuickSight
  - Warehouse choice
    - **EMR** – Hive/Spark, full control, Spot savings
    - **Athena** – ad-hoc serverless SQL on S3
    - **Redshift** – complex SQL, sustained BI (+ Spectrum for S3)
  - Cost example: Kinesis shards far cheaper than DynamoDB WCU for raw ingest

## 📈 Monitoring

- [ ] **CloudWatch Metrics, Alarms & Synthetics**
  - EC2: 5-min default, 1-min detailed · ⚠️ **RAM needs the agent**
  - Custom metrics: standard 1 min · **high-resolution 1 s**
  - Alarm actions: EC2 (stop/terminate/reboot/**recover**), Auto Scaling, SNS → EventBridge
  - Cross-Region dashboards
  - **Synthetics canaries** – scripted headless Chrome checks (Node.js/Python)
    - Blueprints: heartbeat · API · broken links · visual diff · recorder · GUI workflow

- [ ] **CloudWatch Logs**
  - Sources: SDK, unified agent, Beanstalk, ECS, Lambda, Flow Logs, API GW, CloudTrail, Route 53
  - Log groups / streams · retention policies · KMS encryption
  - **Metric filters** → alarms · **Logs Insights** queries
  - **S3 export** – up to 12 h delay (`CreateExportTask`) – not real-time
  - **Subscription filters** → Kinesis Data Streams / Firehose / Lambda / OpenSearch (real-time)
  - Multi-account/Region aggregation: subscriptions → central Kinesis → Firehose → S3
  - Deploy & configure agent with **SSM Run Command / State Manager + Parameter Store**

- [ ] **Amazon EventBridge**
  - Schedules (cron) and event-pattern rules → Lambda, SQS, SNS, Step Functions…
  - Default bus, partner buses, custom buses
  - **Archive & replay** events · **Schema Registry** (code bindings)
  - Resource policy ⇒ **aggregate events from the whole Org** into one bus

- [ ] **X-Ray & AWS Health**
  - **X-Ray** – distributed tracing; daemon on EC2/ECS, built into Lambda, Beanstalk, API GW; needs IAM
  - **AWS Health Dashboard** (account) – events affecting *your* resources, Org aggregation, Health API
  - Health events → **EventBridge** → notify/remediate (e.g. EC2 scheduled retirement)

## 🚀 Deployment & Instance Management

- [ ] **Elastic Beanstalk**
  - Developer-centric PaaS on EC2/ASG/ELB/RDS – free, pay for resources
  - Great for **replatforming** (Java, .NET, Node, Python, PHP, Go, Docker…)
  - Architectures: single instance (dev) · LB + ASG (prod web) · ASG only (workers)
  - **Worker tier** – pulls from SQS; periodic tasks via `cron.yaml`
  - Blue/green: clone env → test → **swap URLs** (or Route 53 weighted)

- [ ] **AWS CodeDeploy**
  - Targets: EC2/on-prem, ASG, Lambda, ECS
  - EC2: in-place with lifecycle hooks (`appspec.yml`)
  - ASG: in-place or **blue/green** (new ASG, needs ELB)
  - Lambda: alias traffic shifting (canary/linear), hooks, **auto-rollback on CloudWatch alarms**
  - ECS/Fargate: blue/green via new task set, canary supported

- [ ] **AWS CloudFormation**
  - IaC backbone of Beanstalk, Service Catalog, SAM
  - **DeletionPolicy**: `Retain` · `Snapshot` (EBS, RDS, ElastiCache, Redshift) · `Delete` (default; DBCluster defaults Snapshot)
  - ⚠️ Empty S3 bucket before delete – use a **custom resource** (Lambda)
  - **StackSets** – deploy across accounts & Regions; auto-deploy to new Org accounts
  - **Drift detection** – find manual changes
  - **Resource import** – adopt existing resources (needs DeletionPolicy + identifier)
  - Dynamic references to Secrets Manager / Parameter Store

- [ ] **Service Catalog, SAM & CDK**
  - **Service Catalog** – admin-approved CloudFormation **products** in **portfolios**; self-service with governance (ServiceNow integration)
  - **SAM** – YAML shorthand for serverless; local testing; CodeDeploy traffic shifting
  - **CDK** – TypeScript/Python/Java/.NET → synthesises CloudFormation

- [ ] **AWS Systems Manager**
  - Manage EC2 + on-prem at scale; needs **SSM agent + IAM role** (most "not managed" issues = role)
  - **Run Command** – scripts across fleets, rate/error control, no SSH
  - **Patch Manager** – baseline → patch groups (tags) → maintenance window → `AWS-RunPatchBaseline` → compliance
  - **Session Manager** – shell without SSH/bastion; logs to S3/CloudWatch
  - **Automation** runbooks – e.g. ASG lifecycle hook → run doc before terminate
  - **OpsCenter** – OpsItems aggregating Config, CloudTrail, alarms + runbooks

- [ ] **AWS Cloud Map**
  - Service discovery registry with health checks; query via API or DNS

## 💰 Cost Control

- [ ] **Cost Allocation Tags & Tag Editor**
  - AWS-generated (`aws:` prefix, not retroactive) and user tags (`user:` prefix)
  - Appear as report columns; up to 24 h to show
  - **Tag Editor** – bulk tag across Regions, find untagged

- [ ] **Trusted Advisor, Support Plans & Service Quotas**
  - Checks: cost, performance, security, fault tolerance, **service limits**, ops
  - Full checks + Support API need **Business / Enterprise** support
  - ⚠️ Detects public **buckets**, not public **objects** (use Config / EventBridge)
  - **Service Quotas** – CloudWatch alarms near a limit (e.g. Lambda concurrency)

- [ ] **EC2 Pricing & Savings Plans**
  - **EC2 Instance Savings Plan** – ≤72%, fixed family + Region
  - **Compute Savings Plan** – ≤66%, any family/Region, **EC2 + Fargate + Lambda**
  - SageMaker Savings Plan ≤64%
  - RIs/SPs shared across Org (payer can disable) · **queue** RI renewals

- [ ] **S3 Cost Optimisation**
  - Lifecycle transitions · compression · Intelligent-Tiering (no retrieval fees, archive tiers optional)
  - Glacier retrieval: Instant (ms) · Flexible (1–5 min / 3–5 h / 5–12 h) · Deep Archive (12 h / 48 h)
  - **Requester Pays** – requester pays transfer, owner pays storage

- [ ] **Budgets, Cost Explorer & Compute Optimizer**
  - **Budgets**: cost · usage · RI · Savings Plans; up to 5 SNS notifications
    - **Budget actions**: apply IAM policy, apply **SCP**, stop EC2/RDS (auto or approval)
  - **Cost Explorer** – analyse, SP recommendations, **12-month forecast**
  - **Compute Optimizer** – ML right-sizing for EC2, ASG, EBS, Lambda (agent for memory)

## 🚚 Migrations

- [ ] **The 7 Rs**
  - **Retire** – switch off · **Retain** – not yet (compliance, mainframe)
  - **Relocate** – e.g. VMware Cloud on AWS, move VPC/account/Region
  - **Rehost** – lift & shift via **Application Migration Service** (~30% saving)
  - **Replatform** – lift & reshape: RDS, Beanstalk
  - **Repurchase** – move to SaaS (Salesforce, Workday)
  - **Refactor** – re-architect to cloud native / serverless

- [ ] **AWS Storage Gateway**
  - **S3 File Gateway** – NFS/SMB → S3 with local cache; AD for SMB; lifecycle to Glacier
    - Versioning + `RefreshCache` · Object Lock for WORM
  - **Volume Gateway** – iSCSI block · **cached** (recent data local) vs **stored** (all local, async backup)
  - **Tape Gateway** – virtual tape library → S3/Glacier for existing backup software
  - (FSx File Gateway – deprecated, use FSx directly)

- [ ] **AWS Snow Family**
  - Snowball Edge **Storage Optimized** (~210 TB) / **Compute Optimized** (EC2/Lambda at the edge)
  - 💡 Rule of thumb: network transfer > **1 week** ⇒ Snowball
  - Speed tips: parallel writes, batch small files, **S3 Adapter** (250–400 MB/s vs 25–40 MB/s file interface)

- [ ] **DMS & Schema Conversion Tool**
  - DMS runs on a replication instance; source stays live
  - Full load · full load + **CDC** · CDC only
  - Sources: on-prem/EC2 DBs, Azure SQL, RDS, S3, DocumentDB
  - Targets: RDS/Aurora, Redshift, DynamoDB, S3, OpenSearch, Kinesis, DocumentDB
  - **SCT** – only when **engines differ** (Oracle → Aurora PostgreSQL; Teradata → Redshift)
  - Huge DBs: SCT extracts to **Snowball Edge** → S3 → DMS applies (+ CDC)
  - ⚠️ OpenSearch is target-only

- [ ] **Disaster Recovery**
  - **RPO** = data loss window · **RTO** = downtime
  - Strategies (cheaper → faster)
    - 🧊 **Backup & Restore** – hours
    - 🕯️ **Pilot Light** – core DB running, rest off
    - 🌤️ **Warm Standby** – full stack at min size
    - 🔥 **Multi-site / Active-Active** – seconds, $$$
  - Toolkit: snapshots, CRR, Aurora Global, Route 53 failover, VPN backup for DX, CloudFormation rebuilds
  - **Fault Injection Service** – chaos experiments (EC2, ECS, EKS, RDS)

- [ ] **Discovery & Migration Services**
  - **Application Discovery Service**
    - Agentless (VMware OVA) or agent-based (processes, network dependencies)
    - Data → **Migration Hub**, explore with Athena
  - **Application Migration Service (MGN)** – continuous block replication, rehost with minimal downtime
  - **Elastic Disaster Recovery (DRS)** – continuous replication to AWS for DR/ransomware recovery
  - **Migration Evaluator** – business case from agentless collector
  - Amazon Linux 2 available as VM image for on-prem testing

- [ ] **AWS Backup**
  - Central policy-based backups: EC2/EBS, S3, RDS/Aurora, DynamoDB, DocumentDB, Neptune, EFS, FSx, Storage Gateway
  - Cross-Region & cross-account copies · PITR · tag-based **backup plans**
  - **Vault Lock** – WORM; **even root can't delete**

## 🌐 Amazon VPC

- [ ] **VPC Basics**
  - Private ranges: `10.0.0.0/8` · `172.16.0.0/12` · `192.168.0.0/16`
  - VPC CIDR **/28 – /16**; can add secondary CIDRs
  - Subnets: **5 IPs reserved** (first 4 + last) · most specific route wins
  - **IGW** for public subnets · **NAT Gateway** (per AZ, EIP, managed) vs NAT instance (disable src/dst check)
  - **NACL** – subnet, stateless, allow + **deny** (block an IP cheaply)
  - **Security group** – ENI, stateful, allow only, can reference SGs (incl. peered/cross-account)
  - Bastion host vs **Session Manager** (no SSH needed)
  - IPv6: all public · private egress via **Egress-only IGW**

- [ ] **VPC Peering**
  - Private link between 2 VPCs, cross-account & cross-Region
  - ⚠️ **Not transitive** · no overlapping CIDRs · update both route tables
  - Longest prefix match for overlapping routes
  - ⚠️ No **edge-to-edge** routing – can't use peer's IGW, NAT, VPN, DX or gateway endpoint

- [ ] **Transit Gateway**
  - Hub-and-spoke for thousands of VPCs + VPN + DX Gateway
  - Regional, peer across Regions · share via **RAM**
  - **Route tables** to segment who talks to whom
  - Only AWS service with **IP multicast**
  - Pattern: **centralised egress VPC** with shared NAT Gateways

- [ ] **VPC Endpoints & PrivateLink**
  - **Gateway endpoint** – S3 & DynamoDB only, route table entry, free
    - ⚠️ Can't be used from on-prem / peered / TGW
  - **Interface endpoint** – ENI + SG for most services; reachable over **DX & VPN**
    - Private DNS needs `enableDnsHostnames` + `enableDnsSupport`
  - Endpoint policies restrict *what* can be accessed (don't replace IAM/bucket policies)
  - S3 bucket policy: `aws:SourceVpce` / `aws:SourceVpc` – ⚠️ `aws:SourceIp` doesn't work via endpoints
  - **PrivateLink** – expose your service to 1000s of VPCs: **NLB** (provider) + ENI (consumer); no peering, no CIDR worries

- [ ] **Site-to-Site VPN & Client VPN**
  - **VGW** (AWS side) + **Customer Gateway** (on-prem public IP) · 2 IPsec tunnels
  - Static routes or **BGP** dynamic (ASNs)
  - On-prem reaching internet *through* AWS: NAT instance ✅ · NAT Gateway ❌
  - **VPN CloudHub** – up to 10 CGWs per VGW, hub-and-spoke between sites
  - Accelerated VPN via Global Accelerator (with TGW)
  - **Client VPN** – OpenVPN from laptops to VPC, peered VPCs, on-prem, internet, TGW

- [ ] **Direct Connect**
  - Dedicated private link; **not encrypted** by default; not redundant by default
  - VIFs: **Public** (AWS public endpoints) · **Private** (VPC) · **Transit** (TGW)
  - Connections: **Dedicated** 1–400 Gbps · **Hosted** 50 Mbps–25 Gbps · lead time > 1 month
  - Encryption: **VPN over DX** (IPsec) or MACsec
  - **LAG** – up to 4 identical dedicated links at one location
  - **DX Gateway** – one DX to VPCs in many Regions/accounts (+ TGW)
  - HA: two DX locations · or DX + **backup VPN**
  - **SiteLink** – on-prem ↔ on-prem via DX locations, bypassing Regions

- [ ] **VPC Flow Logs**
  - VPC / subnet / ENI level → S3, CloudWatch Logs, Firehose
  - Includes managed ENIs (ELB, RDS, NAT GW, TGW…)
  - Query with Athena or Logs Insights
  - 🔍 Troubleshoot: inbound ACCEPT + outbound REJECT ⇒ **NACL** (stateless); single REJECT ⇒ SG or NACL
  - NAT GW "accepts" inbound in logs but drops unsolicited traffic

- [ ] **AWS Network Firewall**
  - Managed L3–L7 firewall for the whole VPC (built on GWLB)
  - Inspects VPC↔VPC, egress, ingress, DX/VPN traffic
  - Stateful rules, **domain allow lists**, regex, IPS; logs → S3/CWL/Firehose
  - Managed centrally with **Firewall Manager**

## 🤖 Machine Learning

- [ ] **Amazon Bedrock**
  - Managed generative AI via unified API across foundation models
  - Private model copy for **fine-tuning**; your data never trains the base model
  - Built-in RAG (knowledge bases), agents, guardrails

- [ ] **Vision, Speech & Language Services**
  - **Rekognition** – labels, faces, celebrities, text, **content moderation** (+ A2I human review)
  - **Transcribe** – speech → text, **PII redaction**, auto language ID
  - **Polly** – text → speech; lexicons & **SSML**
  - **Translate** – localisation
  - **Lex** – chatbots (ASR + NLU) · **Connect** – cloud contact centre
  - **Comprehend** – NLP sentiment, entities, topics · **Comprehend Medical** – PHI detection
  - **Textract** – forms, tables, handwriting from documents

- [ ] **SageMaker, Kendra & Personalize**
  - **SageMaker AI** – build, train, deploy custom models
  - **Kendra** – ML document search with natural-language answers
  - **Personalize** – real-time recommendations (Amazon.com tech)
  - 🧭 Cheat sheet: faces→Rekognition · audio→Transcribe · speak→Polly · bot→Lex · sentiment→Comprehend · docs→Textract · search→Kendra · recommend→Personalize

## 🧩 Other Services

- [ ] **CI/CD on AWS**
  - CodeCommit/GitHub → **CodeBuild** → **CodeDeploy**, orchestrated by **CodePipeline**
  - Manual approval stages · CodeBuild → ECR images · pipeline-driven CloudFormation
  - Pattern: repo push → Lambda scans for leaked credentials

- [ ] **Kinesis Video Streams**
  - One stream per device (cameras, phones); data stored internally (no direct S3 output)
  - Consumers: EC2, Parser Library, **Rekognition Video** → Kinesis Data Streams for face alerts

- [ ] **WorkSpaces & AppStream 2.0**
  - **WorkSpaces** – managed VDI, AD-integrated; on-demand or monthly
    - WAM for app packaging · maintenance windows for Windows updates
    - Cross-Region redirection via Route 53 (needs **AD Connector**)
    - **IP access control groups** – allow-list user public IPs
  - **AppStream 2.0** – stream a single app to any browser; instance type per app

- [ ] **Device Farm, Macie, SES & Pinpoint**
  - **Device Farm** – test on real phones/browsers, videos & logs
  - **Macie** – ML discovery of **PII in S3**
  - **SES** – transactional/bulk email, DKIM/SPF, reputation dashboard
    - Configuration sets → Firehose (metrics) / SNS (bounces) · dedicated IP pools
  - **Pinpoint** – 2-way marketing campaigns (email, SMS, push, voice) with segments & templates

- [ ] **EC2 Image Builder & IoT Core**
  - **Image Builder** – automate build → test → distribute AMIs/containers on schedule, multi-Region/account
  - **IoT Core** – connect billions of devices; rules route to Lambda, S3, Firehose, SageMaker…

## 🎯 Exam Preparation

- [ ] **Exam Approach**
  - Scenario-heavy; several answers may work – pick the **most appropriate**
  - Watch for qualifiers: *least operational overhead* · *most cost-effective* · *least change*
  - Eliminate options that are manual, not scalable, or break a stated constraint
  - [Official exam page](https://aws.amazon.com/certification/certified-solutions-architect-professional/)

- [ ] **Sample Question Lessons**
  - 💸 Stop runaway spend across many accounts but keep teams autonomous ⇒ **Org + SCP** with `ec2:InstanceType` (alerts/reports only detect)
  - 👀 Third-party needs read-only in every account ⇒ IAM role trusting their account, deployed with **StackSets**
    - Identity Center allows only **one** identity source
  - 🌐 Browser form on S3 calling API Gateway ⇒ **S3 static website** + **CORS on API Gateway**
  - ⚠️ API Gateway **502** under spikes ⇒ Lambda concurrency limit (429 would be throttling)
  - 🔒 No SSH allowed, small ops team ⇒ no key pairs, manage with **Systems Manager**
  - 👥 Dev/ops across two accounts, one credential set ⇒ users in dev account + **assume role** in prod
  - 💰 Kinesis cost: pack more devices per shard (stay < 1 MB/s) · replace idle ASG with **SQS → Lambda**
  - 🐢 Fragile third-party endpoint ⇒ **SQS buffer** + Lambda with **reserved concurrency**
  - 🌍 Active-active two Regions, least change ⇒ ECR CRR + second ECS/ALB stack + Route 53 latency + **Aurora Global write forwarding**
  - ⚙️ Modernise with less ops ⇒ ALB for web tier + **Aurora Serverless** (Graviton ✗ on Windows)

