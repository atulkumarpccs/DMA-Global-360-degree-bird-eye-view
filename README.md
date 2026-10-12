# DMA Global 360-degree bird's-eye view

## Interview design answers

This document summarizes hands-on engineering experience and the approach to designing secure, reliable transaction platforms. It distinguishes work personally delivered from work owned by application, integration, architecture, and data-science teams.

## 1. Experience and hands-on engineering

I am a hands-on platform and cloud engineer with more than 14 years of engineering experience. The last seven years were spent at two APRA-regulated banks (ING and IMB Bank) and in an IRAP PROTECTED federal environment (the NDIS Quality and Safeguards Commission). I have not held the title of Principal Engineer, but have performed equivalent technical work: authoring high- and low-level designs and option papers for an architecture design authority, then building the approved solutions.

### Personally delivered in the last 12 months

At the Commission, in production:

- An AWS landing zone using Control Tower, Account Factory for Terraform, networking, and security controls. Control Tower applied baseline SCPs and RCPs to accounts vended through Account Factory; I authored additional SCPs.
- Azure DevOps SaaS with private build agents on EC2: an Image Builder AMI factory, an Auto Scaling fleet, and lifecycle hooks that deregister agents through EventBridge and Lambda. Reusable multi-stage pipelines run on the fleet.
- Self-hosted Terraform Enterprise running active-active on EKS, plus Terraform and CloudFormation automation using a private module registry.
- Azure Artifacts integrated with Azure DevOps as the controlled package feed. I also ran JFrog Artifactory at IMB and ING.
- Developer environments on Amazon WorkSpaces (more than 200 seats), with automated provisioning and decommissioning pipelines and a phased directory migration.
- Integration between AWS, Azure DevOps, and enterprise delivery tools, including Salesforce delivery through Copado, ServiceNow, and Okta.

### Engineering scope

My languages and tools include Python, PowerShell, Bash, Terraform/HCL, CloudFormation, YAML pipelines, and Ansible. I have deployed and troubleshot Java services extensively, but was not the primary developer of Java business logic. Earlier in my career, I wrote production C/C++ for embedded automotive systems.

Direct engineering has made up 60–70% of my recent roles. The rest was architecture work: option papers, high- and low-level designs, and design-authority review. I did this at ING, IMB, and the Commission; at the Commission I also led five vendor DevOps engineers, graduates, and internal staff. I design what I build, and build what I design.

## 2. Ledger and balance design

I have not personally designed or built a production double-entry ledger. My banking work was in the APIs, messaging, and platforms around transaction processing. This is how I would approach a ledger:

- **Journal:** Use an append-only, double-entry journal as the system of record. Each posting is a set of debit and credit lines that must sum to zero. Validate and persist it in one atomic transaction; never update or delete it.
- **Balance states:** Keep posted, pending, and available balances as separate, versioned projections per account. Use optimistic concurrency on the account version to prevent double-spend under concurrent postings.
- **Holds:** Represent a hold as a posting that moves funds from available to held. Capture, release, or expire it with a linked posting, so every hold reaches a terminal state.
- **Idempotency:** Give each request a client-supplied key with a unique constraint. Store the first result and return it on retries. Detect duplicates at every boundary where a message can be redelivered.
- **Reversals:** Make a reversal a new posting that references the original. Correct errors without mutating history.
- **Reconciliation and suspense:** Match internal postings against scheme, bank, and settlement files on a schedule. Send unmatched items to an owned suspense account with ageing thresholds and an exception queue.
- **Audit trail:** Record the actor, source, correlation ID, and timestamp for each posting, and write the records to tamper-evident (WORM) storage so balances can be rebuilt from the journal.

I would validate the accounting model with the ledger and finance specialists before building it.

## 3. Transactional platforms

My experience is with the platforms that move transactions between channels, integration layers, and core banking. I have not owned a core ledger or settlement engine.

### IMB Bank: mobile and internet banking integration modernisation (2024–2025)

Digital channels call APIs managed in MuleSoft Anypoint and running on CloudHub. Those APIs connect to core banking and payment-connected services, including BPAY and RTGS; core banking remained the system of record.

**My work:**

- Built the AWS landing zone and secure connectivity, including a new Transit Gateway for MuleSoft connectivity and the network path into core banking.
- Managed environment configuration and deployment automation for the MuleSoft estate.
- Implemented API security policies (OAuth2/OIDC through Okta in API Manager).
- Migrated more than 130 APIs to the new platform and provided production support.

Integration specialists and solution architects owned API contracts, business processes, and integration logic. In production support, I traced failed transactions across channel, API, and backend layers using correlation IDs. Before a retry or replay, I confirmed the processing state in the system of record: API acceptance or a message acknowledgement does not prove that a transaction completed.

### ING Bank: digital banking microservices and real-time offer engine (2021–2023)

More than 15 Java microservices connected digital banking with core banking, legacy services, and a real-time offer engine through synchronous APIs, Kafka, Pub/Sub, and batch.

**My work:** CI/CD pipelines, deployment automation, environment configuration, OpenShift cluster builds and upgrades across two data centres, GCP/GKE infrastructure, and level 3 production support, including Kafka consumer issues.

Application teams owned service logic, and data science owned the models. My automation reduced application deployment time from about eight hours to 30–60 minutes.

## 4. Financial integrations

My direct experience is with the API estate supporting BPAY and RTGS at IMB Bank during its move to a newly hosted MuleSoft API ecosystem on AWS. I have no direct NPP, BECS, SWIFT, FIX, or ISO 8583 experience.

### Example: BPAY-supporting APIs on AWS-hosted MuleSoft

I supported every API, including BPAY-supporting APIs, as each was adapted from Anypoint Studio projects to the new AWS-hosted runtime. I owned the platform side of the move: deployment, environment configuration, connectivity, and production support. Integration specialists owned payload mapping and scheme rules.

- **Message flow:** Built the Transit Gateway routes, private connectivity to core banking, TLS certificates, and API Manager security policies (OAuth2/OIDC through Okta). Automated each environment's configuration so non-production and production behaved consistently.
- **Acknowledgements:** Treated API acceptance or message acknowledgement as confirmation of receipt, not payment completion. In support, confirmed the final state in the downstream system before closing an incident.
- **Idempotency and reconciliation:** Supported stable transaction identifiers and duplicate detection at the integration boundary so timed-out requests could be retried safely after migration. Used correlation IDs to trace records across API logs, messages, and backend status, showing integration and operations staff where a record had stopped.
- **Failure handling:** Worked within explicit timeout and retry settings, escalating partially processed records rather than replaying them blindly. Every API moved through automated, repeatable pipelines with rollback so a release could not leave a payment path half-configured.

## 5. Platform architecture

### Example 1: IMB Bank integration modernisation

The architecture boundaries were digital channels; API and integration management; MuleSoft execution; core and legacy banking; payment-connected services; and cloud networking, identity, and deployment controls.

My decisions included the landing zone and network segmentation, identity federation, API security policy placement, and repeatable environment builds through infrastructure as code. I built a dedicated Transit Gateway for inter-organisation, AWS-to-AWS connectivity to MuleSoft. It attached to the bank's existing Transit Gateway and was built in Terraform. MuleSoft did not join the core network directly; separate route tables controlled which banking networks were reachable through transitive routing, limiting and making auditable the third-party blast radius.

I was also the sole author of the option paper, HLD, and LLD for moving the bank's Terraform to HCP Terraform. I migrated more than 20 workspaces for online, mobile, and internet banking without disrupting delivery. Solution architecture and application teams owned API contracts, data models, and business processes.

### Example 2: Terraform Enterprise as an internal platform

At the Commission, I designed the platform end to end. I chose self-hosting over SaaS because of data-location rules in a PROTECTED environment. It runs active-active on EKS with external PostgreSQL and Redis. Azure DevOps triggers CLI-driven runs over a private load balancer with private-CA TLS. Workspaces and modules are bounded by delivery stream, and OPA and Sentinel policy sets are enforced on every run.

### Patterns for financial systems

- Use backward-compatible contracts and schema evolution.
- Keep deployment artefacts versioned and immutable.
- Make consumers idempotent, with explicit retry, offset, and replay policies.
- Use checkpoints for batch processing.
- Keep authoritative state in the system of record and distribute changes downstream as events. Replaying an event must never create a second financial effect.

At ING, I used these patterns in production on Kafka and Pub/Sub, including addressing consumer lag and replay issues.

## 6. AWS, infrastructure, and delivery

AWS is my principal cloud. I also have production Azure (IMB) and GCP (ING) experience.

At the Commission, I built the IRAP PROTECTED AWS environment in `ap-southeast-2`, including:

- **Services:** Control Tower, Organizations, EKS, Lambda, Step Functions, EventBridge, RDS, S3, WorkSpaces, Security Hub, GuardDuty, and Private CA.
- **Networking:** Hub-and-spoke VPCs with private subnets only and Transit Gateway; Direct Connect and VPN termination; PrivateLink and VPC endpoints; AWS Network Firewall with Suricata rules on the egress path; and Route 53 Resolver rules.
- **IAM:** Okta federation to IAM Identity Center with SCIM; permission sets and permission boundaries (the pattern was governed by security and implemented by me); baseline SCPs and RCPs applied through Control Tower and extended with my SCPs.
- **Secrets:** Secrets Manager and KMS customer-managed keys, with no secrets in source or build artefacts.
- **Infrastructure as code:** Terraform with a private module registry on self-hosted Terraform Enterprise, plus CloudFormation where required.
- **Environments:** Account-per-environment through Account Factory for Terraform.
- **Containers and serverless:** EKS, including eight active-active clusters for the data analytics platform, plus Lambda and Step Functions.
- **CI/CD:** Azure DevOps with private agents, separate approval environments per operation, and controlled artefact flow through Azure Artifacts.
- **Monitoring:** CloudWatch with SLOs and error budgets; Security Hub feeds the agency SOC.
- **Resilience and recovery:** Active-active EKS, Multi-AZ RDS, AWS Backup, and DR failover tests and runbooks at IMB and ING.

## 7. Security engineering

At the Commission, I implemented security architecture for the landing zone and developer desktop fleet:

- **Authentication and MFA:** SAML 2.0 federation from Okta, later Entra ID, to AWS and WorkSpaces, with MFA enforced at the identity provider.
- **RBAC:** Permission sets and directory groups mapped to each delivery stream, with least privilege by default.
- **Segregation of duties:** Separate deployment identities and approval environments for provisioning and decommissioning; separate accounts for workloads, security tooling, and log archive.
- **Encryption:** KMS for EBS, RDS, and WorkSpaces volumes; TLS from a private CA with certificate lifecycle management.
- **Secrets:** Secrets Manager. Pipeline identities use scoped service connections, not stored keys.
- **Tenant isolation:** Account-level isolation enforced by SCPs.
- **Logging:** An organisation-wide CloudTrail trail and Security Hub feeding the SOC and SIEM; Cortex XDR deployed across more than 200 WorkSpaces.
- **Secure SDLC:** SAST, SCA, and container scanning in pipelines, and OPA/Sentinel policy checks before infrastructure changes apply.

### Vulnerabilities and control gaps remediated

- **Log4j (CVE-2021-44228) and ongoing vulnerability reports:** At ING and IMB, I identified affected artefacts across the estate, upgraded dependencies, rebuilt images, redeployed through controlled change, and validated the fix. I also handled day-to-day vulnerability reports from the SOC and third-party teams according to severity and priority.
- **IRAP gaps at the Commission:** Findings included unencrypted WorkSpaces volumes (encryption can only be set at launch) and Simple AD's lack of KMS encryption and LDAPS. I designed and delivered migration to AWS Managed Microsoft AD and re-provisioned WorkSpaces with encryption. I then enforced encryption in the provisioning pipeline so the gap could not recur.

### Compliance evidence

I implemented controls and supplied technical evidence for IRAP PROTECTED, aligned with the Information Security Manual (ISM) and Essential Eight. I also implemented platform controls and supplied technical evidence for ISO 27001 and PCI-DSS baseline controls at ING, IMB, and the Commission. I have not done SOC 2 work and was not a formal assessor or certification owner.

## 8. Reliability, documentation, and technical authority

At the Commission, I established:

- **Observability:** SLOs with error budgets on the eight-cluster EKS analytics platform. When the budget runs low, infrastructure and application teams review together. I used the same model for the offer engine at ING.
- **Incidents:** Blameless postmortems with tracked actions.
- **Recovery:** Automated rebuild through infrastructure as code, active-active platforms, and DR runbooks.
- **Documentation:** Each change I lead ships with an HLD, LLD, diagrams, deployment procedure, and runbook for the design authority.
- **Transaction-facing services:** Monitoring of technical signals (availability, latency, errors, saturation, and dependency health) and business signals (consumer lag, failed messages, processing exceptions, and reconciliation status).

### Design changed: Simple AD

The developer desktop platform depended on Simple AD. Evidence showed that Simple AD has no KMS encryption or LDAPS, its volumes could not be encrypted after launch, and AWS was deprecating it for new customers. I stopped further investment and wrote a five-phase migration to AWS Managed Microsoft AD, which the design authority approved. The migration was delivered stream by stream without a big-bang cutover, and the control gaps were closed.

### Design changed: build-agent capacity

A production MuleSoft release stalled because only one build agent was available. My analysis identified the shared pool as the bottleneck. I redesigned it into purpose-based pools with pre-baked AMIs, so production releases no longer queue behind routine builds.

### Root cause challenged: database deletion protection

After a Multi-AZ deployment, the SOC escalated a production database alert on the assumption that the deployment had broken the database. I traced the change from repository through pipeline to runtime and found deletion protection had never been declared in Terraform since the database was created. The deployment had surfaced an existing non-compliant state. I fixed it in code rather than in the console and wrote the root-cause analysis for the SOC. The control is now enforced on every run.

## 9. Role alignment and first 90 days

I would own technical direction, architecture, security, and design standards; delivery leadership would retain sequencing, allocation, and people management. To avoid slowing delivery, I would:

- Publish decision boundaries: what needs an architecture decision record and what a team can decide itself.
- Turn standards into reusable patterns, templates, policy-as-code, and pipeline gates so compliance is the default path, not a review queue.
- Review designs within an agreed turnaround. When raising a blocking issue, bring a workable alternative.

### First 30 days

Personally deliver a current-state architecture and dependency map; a transaction-flow and failure-path assessment covering idempotency, retries, duplicate processing, and reconciliation boundaries; a prioritised security and resilience risk register; initial architecture decisions; and agreement with delivery on the highest-value improvements.

### By day 90

Deliver reference patterns for APIs, asynchronous integration, infrastructure, and security, each with a working implementation; improved CI/CD controls; SLOs on critical transaction paths; one end-to-end tested recovery scenario; and remediation of at least one material architectural or operational weakness.

## 10. Motivation and practical matters

### Why this role

I want hands-on engineering with end-to-end technical responsibility. This role combines transaction correctness, secure integration, cloud platform design, and operational resilience. I bring banking integration experience from ING and IMB, together with current enterprise AWS and security engineering experience from a PROTECTED government environment.

### Why I am moving

My role at the NDIS Commission has finished. The final transformation project is the NDIS Commission NQSC Final Public Portal. I am choosing my next role deliberately: I want to retain architecture authority while writing and shipping engineering myself, in financial services, where I spent four years at ING and IMB.

- **Sydney CBD, five days per week:** Confirmed.
- **Work rights:** Australian citizen.
- **Notice period:** None; available immediately.
- **Earliest start date:** As soon as required.
- **Expected base salary, excluding superannuation:** A$185,000.
- **Commencement restrictions:** None.

I have also prepared a short interactive reference design illustrating the approach behind answers 2–9: a double-entry ledger, payment lifecycle with timeout and duplicate handling, reconciliation, and a gated delivery pipeline. It is generic and contains no employer material: **Interactive-demo-ledger**.

I would welcome the chance to discuss the IMB and ING integration work in more depth, including my personal contributions, the constraints, and decisions made with the wider teams.
