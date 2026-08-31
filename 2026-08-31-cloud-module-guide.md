# Cloud Module Guide Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce a concise, module-aligned Markdown textbook that teaches all six supplied cloud topics in one full day without designing or implementing the group project.

**Architecture:** One root-level Markdown document follows the course module order. Each chapter synthesizes its primary textbook, supporting slides, and practices into concise prose with embedded PowerShell-compatible AWS CLI commands and focused CloudFormation fragments; a final integration task checks factual accuracy, coverage, length, links, and Markdown structure.

**Tech Stack:** GitHub-flavored Markdown, PowerShell, AWS CLI v2 command syntax, YAML CloudFormation fragments, official AWS documentation for disputed facts

**Spec:** `docs/superpowers/specs/2026-08-31-cloud-module-guide-design.md`

## Global Constraints

- Create `Cloud Module Guide.md` at the workspace root.
- Target approximately 12,000–15,000 words.
- Begin with the title and a contents section containing exactly six links, each targeting a top-level chapter start.
- Use the module order: AWS and EC2; S3 Storage; ECR and Fargate; Lambda Functions; AWS CloudFormation; VPC and IAM.
- Include no preface, overview chapter, quizzes, exercises, interactive checkpoints, cross-chapter links, project architecture, or project implementation advice.
- Use the six `textbook*.md` files as primary sources; use slides and completed practices as supporting sources.
- Use official AWS documentation only to verify current behavior or resolve contradictions.
- Use PowerShell-compatible AWS CLI commands and the configured default Region; do not add routine `--region` flags.
- Keep CloudFormation examples small, valid, and independent of the group project.
- End every chapter with `### Key Points`.
- End the document with `## Sources`.
- This workspace is not a Git repository, so record review checkpoints in command output rather than attempting commits.

---

### Task 1: Source inventory and document framework

**Files:**
- Read: `1. AWS and EC2/textbook1.md`
- Read: `2. S3 Storage/textbook2.md`
- Read: `3. ECR and Fargate/textbook3.md`
- Read: `4. Lambda Functions/textbook4.md`
- Read: `5. AWS CloudFormation/textbook5.md`
- Read: `6. VPC and IAM/textbook6.md`
- Read: all `.pptx`, practice `question.txt`, completed practice `.txt`, and relevant `.png` resources under the six module directories
- Create: `Cloud Module Guide.md`

**Interfaces:**
- Consumes: the approved design specification and all supplied module resources
- Produces: the exact document title, six-link contents, top-level chapter boundaries, and final sources boundary used by Tasks 2–8

- [ ] **Step 1: Read the complete primary sources**

Read each `textbook*.md` from start to finish. Record its heading hierarchy and flag statements that are time-sensitive, internally inconsistent, or contradicted by observed practice behavior.

- [ ] **Step 2: Inspect supporting materials for unique coverage**

Extract slide text in presentation order and inspect practice prompts, answers, and screenshots. Record only concepts, examples, or cautions not already represented in the primary textbooks; do not copy slide wording wholesale.

- [ ] **Step 3: Create the exact document framework**

Use `apply_patch` to create this structure, preserving the six anchors exactly:

```markdown
# Cloud Module Guide

## Contents

1. [AWS and EC2](#1-aws-and-ec2)
2. [S3 Storage](#2-s3-storage)
3. [ECR and Fargate](#3-ecr-and-fargate)
4. [Lambda Functions](#4-lambda-functions)
5. [AWS CloudFormation](#5-aws-cloudformation)
6. [VPC and IAM](#6-vpc-and-iam)

## 1. AWS and EC2

## 2. S3 Storage

## 3. ECR and Fargate

## 4. Lambda Functions

## 5. AWS CloudFormation

## 6. VPC and IAM

## Sources
```

- [ ] **Step 4: Verify the framework**

Run:

```powershell
$guide = Get-Content -Raw -LiteralPath '.\Cloud Module Guide.md'
([regex]::Matches($guide, '(?m)^\d\. \[[^]]+\]\(#[^)]+\)$')).Count
([regex]::Matches($guide, '(?m)^## [1-6]\. ')).Count
```

Expected output: `6` and `6`.

---

### Task 2: AWS and EC2 chapter

**Files:**
- Read: `1. AWS and EC2/textbook1.md`
- Read: `1. AWS and EC2/aws_and_ec2_slides.pptx`
- Read: `1. AWS and EC2/1. Introduction to Cloud Infrastructure (3).pptx`
- Read: `1. AWS and EC2/ec_2_cli.txt`
- Read: `1. AWS and EC2/1. the_price_is_right/question.txt`
- Read: `1. AWS and EC2/1. the_price_is_right/the_price_is_right.txt`
- Modify: `Cloud Module Guide.md`, between `## 1. AWS and EC2` and `## 2. S3 Storage`

**Interfaces:**
- Consumes: Task 1 chapter boundary and AWS/EC2 sources
- Produces: a 2,500–3,000 word Chapter 1 ending in `### Key Points`

- [ ] **Step 1: Draft foundations and infrastructure concepts**

Explain cloud computing, CapEx versus OpEx, shared responsibility, Regions, Availability Zones, resource pooling, multi-tenancy, virtualization, elasticity, measured service, and deployment models. Keep these ideas concrete and relate each one to what AWS does without adding a general overview before the chapter.

- [ ] **Step 2: Draft the EC2 resource model**

Explain instances, instance-family naming, AMIs, EBS volume persistence and types, private addresses, auto-assigned public IPv4 addresses, Elastic IPs, and stop/start/terminate behavior. Include a compact lifecycle table and make continuing costs after stop explicit.

- [ ] **Step 3: Draft access, security, and identity mechanics**

Explain security groups as stateful instance-attached firewalls, key-pair authentication, Amazon Linux versus Ubuntu usernames, Session Manager, IMDSv2, the console/API relationship, CLI credentials, and caller identity.

- [ ] **Step 4: Add essential commands and failure signatures**

Include focused PowerShell-compatible commands for caller identity, resolving an AMI through SSM, describing instances, security-group ingress, SSH, IMDSv2 from inside an instance, and cost/teardown inspection. Explain timeout versus connection-refused versus public-key-denied symptoms.

- [ ] **Step 5: Add costs and Key Points**

Summarize compute hours, EBS, public IPv4, Elastic IP, and data-transfer considerations. End with 8–12 compact key points.

- [ ] **Step 6: Verify Chapter 1 coverage and size**

Run a PowerShell extraction between the Chapter 1 and Chapter 2 headings and confirm it includes the terms `shared responsibility`, `Availability Zone`, `AMI`, `EBS`, `security group`, `IMDSv2`, and `public IPv4`; confirm the chapter is within 2,500–3,000 words.

---

### Task 3: S3 Storage chapter

**Files:**
- Read: `2. S3 Storage/textbook2.md`
- Read: `2. S3 Storage/s3_storage_slides.pptx`
- Read: `2. S3 Storage/S3_and_Object_Storage.pptx`
- Read: `2. S3 Storage/a_site_about_nothing/question.txt`
- Read: `2. S3 Storage/a_site_about_nothing/a_site_about_nothing.txt`
- Modify: `Cloud Module Guide.md`, between `## 2. S3 Storage` and `## 3. ECR and Fargate`

**Interfaces:**
- Consumes: Task 1 chapter boundary and S3 sources
- Produces: a 1,600–1,900 word Chapter 2 ending in `### Key Points`

- [ ] **Step 1: Draft the object-storage model**

Explain buckets, global bucket naming, objects, keys, prefixes, metadata, object replacement, consistency, and the difference between object storage and a filesystem.

- [ ] **Step 2: Draft access and website behavior**

Explain Block Public Access layers, bucket policies, public `s3:GetObject`, website configuration, HTTP-only website endpoints, and website versus REST endpoint behavior. Correctly state that a website endpoint returns 404 for a missing object and 403 for an existing unreadable object; separately explain the REST `GetObject` behavior when `s3:ListBucket` is absent.

- [ ] **Step 3: Draft durability, cost, and data-management features**

Explain durability versus availability, storage classes, versioning, delete markers, lifecycle transitions and expiry, per-request pricing, and storage accumulation hazards.

- [ ] **Step 4: Draft controlled sharing and browser access**

Explain presigned GET/PUT URLs, expiry and bearer-token risk, CORS purpose, and why CORS is not authorization.

- [ ] **Step 5: Add commands, policy fragment, and Key Points**

Include essential commands for bucket creation, object copy/sync, website configuration, object inspection, presigned URLs, version listing, and deletion constraints. Include one minimal public-read bucket-policy fragment and end with 8–12 key points.

- [ ] **Step 6: Verify Chapter 2 coverage and size**

Confirm the chapter contains `Block Public Access`, `website endpoint`, `REST endpoint`, `versioning`, `lifecycle`, `presigned`, and `CORS`; confirm it is within 1,600–1,900 words.

---

### Task 4: ECR and Fargate chapter

**Files:**
- Read: `3. ECR and Fargate/textbook3.md`
- Read: `3. ECR and Fargate/ecr_and_fargate_slides.pptx`
- Read: `3. ECR and Fargate/ECR_and_Fargate.pptx`
- Modify: `Cloud Module Guide.md`, between `## 3. ECR and Fargate` and `## 4. Lambda Functions`

**Interfaces:**
- Consumes: Task 1 chapter boundary and ECR/Fargate sources
- Produces: a 1,300–1,600 word Chapter 3 ending in `### Key Points`

- [ ] **Step 1: Draft container and registry concepts**

Explain image, layer, container, Dockerfile, tag, digest, registry, ECR repository, authentication, immutable deployment value, and image scanning limitations.

- [ ] **Step 2: Draft ECS and Fargate mechanics**

Explain clusters, task definitions, task revisions, tasks, CPU and memory selection, container ports, environment and secrets, log configuration, launch type, ENI allocation, subnets, security groups, and public-IP implications.

- [ ] **Step 3: Draft IAM, lifecycle, costs, and failures**

Distinguish the task execution role from the task role. Explain Fargate billing while a task runs, ECR storage, stopped-task diagnostics, image-pull failures, application exit codes, and the difference between one-off tasks and services.

- [ ] **Step 4: Add essential commands and Key Points**

Include ECR login, repository creation, tag/push, image scan inspection, task-definition registration, task execution, task description, and log lookup commands. End with 8–10 key points.

- [ ] **Step 5: Verify Chapter 3 coverage and size**

Confirm the chapter contains `image`, `digest`, `ECR`, `task definition`, `Fargate`, `execution role`, `task role`, `ENI`, and `stopped reason`; confirm it is within 1,300–1,600 words.

---

### Task 5: Lambda Functions chapter

**Files:**
- Read: `4. Lambda Functions/textbook4.md`
- Read: `4. Lambda Functions/lambda_functions_slides.pptx`
- Modify: `Cloud Module Guide.md`, between `## 4. Lambda Functions` and `## 5. AWS CloudFormation`

**Interfaces:**
- Consumes: Task 1 chapter boundary and Lambda sources
- Produces: a 1,400–1,700 word Chapter 4 ending in `### Key Points`

- [ ] **Step 1: Draft the execution and handler model**

Explain functions, runtimes, handler signatures, invocation events, context, stateless execution, warm environment reuse, and idempotency.

- [ ] **Step 2: Draft event wiring and permissions**

Explain S3 event notifications, event payload navigation, asynchronous invocation, Lambda resource permissions, execution roles, least privilege, CloudWatch Logs, and how trigger permission differs from function permission.

- [ ] **Step 3: Draft packaging and operational controls**

Explain ZIP deployment packages, dependencies, compiled-library compatibility, layers only where useful, memory/CPU coupling, timeout, concurrency, cold starts, retries, throttling, and failure destinations or dead-letter handling at a conceptual level.

- [ ] **Step 4: Draft compute-choice comparison**

Use one focused table to compare EC2, Fargate, and Lambda by unit of deployment, idle cost, runtime duration, scaling, operational responsibility, and characteristic failure.

- [ ] **Step 5: Add essential commands and Key Points**

Include commands to create/update a function package, invoke a function, inspect configuration, list event-source or notification wiring where applicable, and tail CloudWatch logs. End with 8–10 key points.

- [ ] **Step 6: Verify Chapter 4 coverage and size**

Confirm the chapter contains `handler`, `event`, `execution role`, `CloudWatch Logs`, `deployment package`, `cold start`, `concurrency`, and `idempotent`; confirm it is within 1,400–1,700 words.

---

### Task 6: AWS CloudFormation chapter

**Files:**
- Read: `5. AWS CloudFormation/textbook5.md`
- Read: `5. AWS CloudFormation/cloudformation_slides.pptx`
- Modify: `Cloud Module Guide.md`, between `## 5. AWS CloudFormation` and `## 6. VPC and IAM`

**Interfaces:**
- Consumes: Task 1 chapter boundary and CloudFormation sources
- Produces: a 2,300–2,700 word Chapter 5 ending in `### Key Points`

- [ ] **Step 1: Draft declarative infrastructure and template anatomy**

Explain desired state, stacks, YAML, `AWSTemplateFormatVersion`, `Description`, `Parameters`, `Resources`, `Outputs`, logical versus physical IDs, parameter types, and dynamic AMI resolution through an SSM parameter.

- [ ] **Step 2: Draft intrinsic functions and dependency behavior**

Explain `Ref`, `GetAtt`, `Sub`, pseudo parameters, implicit dependencies, and the limited cases for `DependsOn`. Include small valid YAML fragments for each core intrinsic.

- [ ] **Step 3: Draft CLI lifecycle and safe change workflow**

Explain validation, deployment, IAM capabilities, stack outputs, events, change sets, updates in place versus replacement, resource naming consequences, drift, and stack deletion.

- [ ] **Step 4: Draft retention and failure diagnosis**

Explain `DeletionPolicy`, `UpdateReplacePolicy`, terminal states, rollback behavior, the first failed event as root cause, `ROLLBACK_COMPLETE`, preserving failed resources, and `cloud-init-output.log` for failures beneath CloudFormation's visibility.

- [ ] **Step 5: Draft selected advanced features and limitations**

Concise coverage: conditions, mappings, `Fn::If`, transforms/SAM, metadata and `cfn-init`, creation policies and signals, nested stacks, custom resources, stack limits, and the fact that S3 objects are data-plane content rather than native `AWS::S3::Object` resources.

- [ ] **Step 6: Add essential commands and Key Points**

Include `validate-template`, `deploy`, `describe-stacks`, `describe-stack-events`, change-set inspection, drift detection, and delete commands. End with 10–14 key points.

- [ ] **Step 7: Verify Chapter 5 coverage and size**

Confirm the chapter contains `Parameters`, `Resources`, `Outputs`, `!Ref`, `!GetAtt`, `!Sub`, `change set`, `replacement`, `DeletionPolicy`, and `ROLLBACK_COMPLETE`; confirm it is within 2,300–2,700 words.

---

### Task 7: VPC and IAM chapter

**Files:**
- Read: `6. VPC and IAM/textbook6.md`
- Read: `6. VPC and IAM/vpc_and_iam_slides.pptx`
- Modify: `Cloud Module Guide.md`, between `## 6. VPC and IAM` and `## Sources`

**Interfaces:**
- Consumes: Task 1 chapter boundary and VPC/IAM sources
- Produces: a 3,000–3,500 word Chapter 6 ending in `### Key Points`

- [ ] **Step 1: Draft address planning and VPC structure**

Explain CIDR notation, `/16` versus `/24`, five reserved subnet addresses, VPC and subnet boundaries, one-AZ subnet placement, and what a bare VPC provides. Include one compact address-capacity table.

- [ ] **Step 2: Draft internet routing and public-subnet mechanics**

Explain internet-gateway creation and attachment, route tables, route association, local and default routes, `MapPublicIpOnLaunch`, public IPv4 translation, and why an internet gateway alone does not make an instance reachable. Include one compact text diagram of the packet path.

- [ ] **Step 3: Draft network controls**

Compare security groups and network ACLs by attachment, statefulness, rules, and common use. Explain security-group references, traffic order, private subnets, NAT gateways, VPC endpoints, flow logs, and the cost implications of NAT and public IPv4.

- [ ] **Step 4: Draft IAM identities and policy evaluation**

Explain users, groups, roles, temporary credentials, trust policies, identity-based and resource-based policies, policy statement fields, ARN scope, implicit deny, explicit allow, explicit deny, and least privilege. Include one compact text flow showing evaluation order.

- [ ] **Step 5: Draft policy layers, audit, and diagnosis**

Explain permission boundaries, SCPs, service-linked roles, condition keys, policy simulation, IAM Access Analyzer, CloudTrail, and where network refusals versus API refusals leave evidence.

- [ ] **Step 6: Draft systematic troubleshooting**

Teach an evidence-first path across resource state, addressing, routes, gateways, security groups, network ACLs, service listeners, IAM evaluation, CloudFormation events, and CloudTrail. Include characteristic symptoms and the command that tests each layer.

- [ ] **Step 7: Add essential commands and Key Points**

Include describe commands for VPCs, subnets, routes, gateways, security groups, and instances; IAM role/policy inspection and simulation; CloudTrail lookup; and reachability checks. End with 12–16 key points.

- [ ] **Step 8: Verify Chapter 6 coverage and size**

Confirm the chapter contains `CIDR`, `reserved`, `internet gateway`, `route table`, `security group`, `network ACL`, `trust policy`, `implicit deny`, `explicit deny`, `permission boundary`, `SCP`, and `CloudTrail`; confirm it is within 3,000–3,500 words.

---

### Task 8: Sources, factual review, and final validation

**Files:**
- Read: `docs/superpowers/specs/2026-08-31-cloud-module-guide-design.md`
- Modify: `Cloud Module Guide.md`

**Interfaces:**
- Consumes: the complete six-chapter draft from Tasks 2–7
- Produces: the final validated one-day textbook

- [ ] **Step 1: Verify time-sensitive and disputed statements**

Check official AWS documentation for current behavior involving public IPv4 charging, S3 website versus REST error behavior, supported CloudFormation semantics, Lambda limits described numerically, Fargate sizing, and any other statement with a meaningful chance of having changed. Remove unnecessary exact limits or prices when the guide can teach the durable rule instead.

- [ ] **Step 2: Write the Sources section**

List the six course textbooks by path, group the supplied slide decks and practices as supporting materials, and add direct official AWS documentation links actually used during verification. Do not add unused or generic sources.

- [ ] **Step 3: Perform a project-content exclusion scan**

Run:

```powershell
rg -n -i 'capture analysis|packet capture|\.pcap|analyst A|analyst B|R1|R2|R3|R4|R5|R6' '.\Cloud Module Guide.md'
```

Expected: no matches outside a source filename if one is ever unavoidable; remove any project design or implementation material.

- [ ] **Step 4: Validate document topology**

Run:

```powershell
$guide = Get-Content -Raw -LiteralPath '.\Cloud Module Guide.md'
$contentsLinks = ([regex]::Matches($guide, '(?m)^\d\. \[[^]]+\]\(#[^)]+\)$')).Count
$chapters = ([regex]::Matches($guide, '(?m)^## [1-6]\. ')).Count
$keyPoints = ([regex]::Matches($guide, '(?m)^### Key Points$')).Count
[pscustomobject]@{ ContentsLinks=$contentsLinks; Chapters=$chapters; KeyPoints=$keyPoints }
```

Expected: `ContentsLinks=6`, `Chapters=6`, `KeyPoints=6`.

- [ ] **Step 5: Validate links and forbidden structure**

Confirm every contents anchor matches its generated chapter anchor. Confirm the first headings are `# Cloud Module Guide`, `## Contents`, and `## 1. AWS and EC2`; confirm there is no `Overview`, `Quiz`, `Exercise`, `Project`, or standalone command-reference chapter.

- [ ] **Step 6: Validate code fences and CloudFormation fragments**

Count opening and closing fences to ensure they balance. Inspect every YAML fragment for valid indentation and defined references. Run `aws cloudformation validate-template` only on examples that form complete templates; label intentionally partial fragments as fragments and validate their syntax manually against the official resource schema.

- [ ] **Step 7: Measure and tighten the prose**

Run:

```powershell
$text = Get-Content -Raw -LiteralPath '.\Cloud Module Guide.md'
$words = ([regex]::Matches($text, '\b[\p{L}\p{N}][\p{L}\p{N}''-]*\b')).Count
"WORDS=$words"
```

Expected: `WORDS` between 12,000 and 15,000. Remove repetition and filler if high; restore missing explanation rather than padding if low.

- [ ] **Step 8: Complete the final requirements audit**

Read the design specification and check all ten quality checks against the rendered Markdown and the fresh command outputs. Report any known factual uncertainty explicitly; otherwise hand off `Cloud Module Guide.md` with the word count and structural-validation results.
