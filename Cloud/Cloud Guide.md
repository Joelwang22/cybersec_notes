# Cloud Module Guide

## Contents

1. [AWS and EC2](#1-aws-and-ec2)
2. [S3 Storage](#2-s3-storage)
3. [ECR and Fargate](#3-ecr-and-fargate)
4. [Lambda Functions](#4-lambda-functions)
5. [AWS CloudFormation](#5-aws-cloudformation)
6. [VPC and IAM](#6-vpc-and-iam)

## 1. AWS and EC2

### 1.1 Cloud computing as an operating model

Cloud computing is on-demand access to compute, storage, networking, and managed services through APIs. AWS owns the physical equipment and maintains a pool of capacity; customers request resources in minutes and release them when they are no longer needed. The decisive change is not that the servers belong to someone else. It is that infrastructure becomes self-service, programmable, elastic, and finely metered.

Traditional infrastructure is mainly **capital expenditure (CapEx)**: buy hardware in advance, own it for years, and accept the risk that the capacity forecast was wrong. Cloud infrastructure is mainly **operational expenditure (OpEx)**: consume resources and receive a bill for the measured usage. Renting continuously can cost more than buying equivalent hardware. The value is flexibility—the ability to experiment, resize, scale, or stop without another procurement cycle.

The **shared responsibility model** divides security and maintenance between AWS and the customer:

| AWS secures                                    | The customer secures                                 |
| ---------------------------------------------- | ---------------------------------------------------- |
| Buildings, power, cooling, and physical access | Data and its classification                          |
| Physical servers and network fabric            | Credentials and permissions                          |
| Hypervisor and managed-service infrastructure  | Network exposure and firewall rules                  |
| Availability of the underlying service         | Guest operating system, software, and patches on EC2 |

The boundary moves with the service. EC2 leaves the operating system and everything above it to you. Fargate removes the host operating system from your responsibilities. Lambda also supplies the language runtime. S3 supplies the whole serving platform, leaving you responsible for content and access policy. More managed never means responsibility disappears; it means your responsibility moves higher in the stack.

### 1.2 Regions, Availability Zones, and pooled capacity

An AWS **Region** is a separate geographic area such as `us-east-1` or `ap-southeast-1`. Most resources live in one Region, and the console and CLI normally display or operate on one Region at a time. Choosing a Region affects latency, data-residency obligations, service availability, and price. An empty resource list often means the operator selected the wrong Region, not that the resources vanished.

Each Region contains multiple **Availability Zones (AZs)**. An AZ is an isolated failure domain made from one or more data centres with independent power and networking. AZs in a Region have low-latency links, allowing systems to replicate across them. A single EC2 instance occupies one AZ; a production design that must survive an AZ failure normally places capacity in at least two.

Fast provisioning depends on **resource pooling** and **multi-tenancy**. AWS already owns the capacity and allocates slices from the pool. A hypervisor isolates virtual machines sharing a physical host, giving each instance virtual CPU, memory, storage, and networking. You rent a specification, not a particular metal box. A stopped instance can later run on different hardware without changing its identity or EBS data.

**Elasticity** means adding and removing capacity as demand changes. Vertical scaling gives one machine more CPU or memory; horizontal scaling changes the number of machines. AWS makes both programmable, but applications do not scale automatically unless scaling rules and suitable architecture are configured. **Measured service** means every resource has meters—seconds of compute, GB-months of storage, public-address hours, requests, or transferred bytes. Always ask whether a resource charges for existing or per use.

Cloud deployment models use the same ideas in different ownership arrangements:

- **Public cloud:** provider-owned, shared infrastructure offered to customers.
- **Private cloud:** self-service cloud methods on infrastructure dedicated to one organisation.
- **Hybrid cloud:** connected private and public environments.
- **Multi-cloud:** deliberate use of more than one public provider, increasing choice and management complexity.

### 1.3 EC2 instances, types, and images

Amazon Elastic Compute Cloud (**EC2**) provides virtual machines called **instances**. An instance has an operating system, CPU and memory allocation, block storage, network interfaces, and a lifecycle:

| State/action | Meaning                        | Typical charging consequence                                       |
| ------------ | ------------------------------ | ------------------------------------------------------------------ |
| Running      | Booted and able to execute     | Compute, attached storage, and public IPv4 can charge              |
| Stopped      | Shut down but retained         | Compute pauses; EBS continues; auto-assigned public IP is released |
| Started      | A stopped instance boots again | Compute resumes; a new auto-assigned public IP may appear          |
| Terminated   | Instance is destroyed          | Irreversible; attached resources follow their own deletion rules   |

An **instance type** describes the hardware allocation. In `t3.micro`, `t` is the family, `3` is the generation, and `micro` is the size. Common families include `t` for burstable general-purpose workloads, `m` for balanced general purpose, `c` for compute-intensive work, and `r` for memory-intensive work. Letters after the generation can identify a processor variant: for example, `g` commonly indicates AWS Graviton (Arm) and `a` an AMD variant. Newer generations often improve price/performance, but software architecture must match the processor architecture.

Burstable `t` instances earn CPU credits during light use and spend them during bursts. They suit small services that are idle much of the time; sustained CPU-heavy work can exhaust the burst allowance. The type must be selected from workload measurements rather than from the word “micro.”

An **Amazon Machine Image (AMI)** is the boot template: operating system plus any software and configuration baked into the image. AMI IDs are Region-specific and are replaced as publishers release patched versions, so durable automation should resolve a current image rather than paste an opaque `ami-...` value. AWS publishes stable SSM Parameter Store paths for current Amazon Linux images:

```powershell
$amiId = aws ssm get-parameter `
  --name "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64" `
  --query "Parameter.Value" `
  --output text
$amiId
```

AMI choice controls operational details. Amazon Linux 2023 uses `dnf`, calls Apache `httpd`, and normally uses the SSH user `ec2-user`. Ubuntu uses `apt`, calls Apache `apache2`, and normally uses the SSH user `ubuntu`. A server banner identifying Ubuntu is evidence to use `ubuntu`; repeatedly presenting the right key to the wrong account still produces `Permission denied (publickey)`.

### 1.4 EBS: storage with an independent lifecycle

Elastic Block Store (**EBS**) provides network-attached block devices that behave like disks. The root filesystem normally resides on an EBS volume, allowing the instance to stop and later boot with its data intact. A volume is a separate resource: it has its own identifier, size, type, attachment, deletion settings, and bill.

The usual default is `gp3`, a general-purpose SSD whose baseline performance is decoupled from its capacity. `io2` is for workloads needing provisioned, durable IOPS; `st1` favours large sequential throughput on HDD; `sc1` is low-cost cold HDD storage. A stopped instance still has its EBS volumes. The root volume is commonly configured for deletion when the instance terminates, while extra volumes often survive. Teardown therefore includes listing volumes and checking for unattached leftovers.

Snapshots are point-in-time backups stored independently of the volume. They also have storage charges and survive instance deletion. “The instance is gone” is not proof that its disks and backups are gone.

### 1.5 Private addresses, public addresses, and Elastic IPs

Every instance network interface receives a private IP address from its subnet. The address remains associated with the interface through stop/start and is routable within connected private networks. Internet access is separate.

An auto-assigned public IPv4 address is mapped to the private address at the internet gateway. The guest operating system sees only its private address. Stop the instance and the public address is released; start it and a different public address is normally assigned. Bookmarks and scripts pointing at the previous address then fail.

An **Elastic IP** is a public IPv4 address explicitly allocated to the account. It can remain stable and be reassociated, but it continues to exist after the instance is terminated until someone releases it. Public IPv4 addresses are metered while allocated or in use, so stable addressing is both an architectural and cost decision.

Useful inventory commands keep identity, state, placement, and addresses visible:

```powershell
aws ec2 describe-instances `
  --query "Reservations[].Instances[].{Id:InstanceId,State:State.Name,Type:InstanceType,AZ:Placement.AvailabilityZone,PrivateIp:PrivateIpAddress,PublicIp:PublicIpAddress,Image:ImageId,Key:KeyName}"

aws ec2 describe-volumes `
  --query "Volumes[].{Id:VolumeId,State:State,SizeGiB:Size,Type:VolumeType,Instance:Attachments[0].InstanceId}"

aws ec2 describe-addresses
```

### 1.6 Security groups and reachability

A **security group** is a stateful, allow-only firewall attached to an elastic network interface. New groups allow no inbound traffic. Rules permit traffic by protocol, port, and source CIDR or source security group. If no rule allows a packet, AWS silently drops it.

Stateful means a permitted inbound connection automatically permits its return traffic. Multiple groups on one interface combine as a union: any matching allow is sufficient. A public web server might allow TCP 80 from `0.0.0.0/0`, but administration should be restricted to a trusted address such as `<your-public-ip>/32`, or replaced with Session Manager.

```powershell
aws ec2 authorize-security-group-ingress `
  --group-id <security-group-id> `
  --protocol tcp `
  --port 80 `
  --cidr 0.0.0.0/0
```

Failure shape is diagnostic:

- **Timeout:** traffic was probably dropped by routing or a network filter.
- **Connection refused:** the packet arrived, but nothing listened on the port.
- **Permission denied (publickey):** SSH reached the daemon, but authentication failed—check username, key pair, and installed public key.
- **Host-key warning:** the address now identifies a host whose SSH host key differs from the remembered one; verify the target before removing old trust data.

### 1.7 Keys, Session Manager, and instance metadata

At launch, EC2 installs the selected key pair's public key for the AMI's default operating-system user. The `.pem` file is the private half and must remain private. AWS cannot download it again. The basic Windows OpenSSH form is:

```powershell
ssh -i ".\<key-name>.pem" ec2-user@<public-ip>  # Amazon Linux
ssh -i ".\<key-name>.pem" ubuntu@<public-ip>    # Ubuntu
```

AWS Systems Manager **Session Manager** can provide a shell without an inbound SSH rule or distributed key file. The instance needs the SSM Agent, an IAM role containing the required managed-instance permissions, and outbound HTTPS access to Systems Manager endpoints:

```powershell
aws ssm start-session --target <instance-id>
```

From inside an instance, the link-local **Instance Metadata Service (IMDS)** at `169.254.169.254` exposes instance identity, placement, networking, user data, and temporary role credentials. IMDSv2 requires a token before metadata can be read. From a Linux shell on the instance:

```bash
TOKEN=$(curl -sS -X PUT http://169.254.169.254/latest/api/token \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -sS -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

Requiring the token reduces exposure to simple server-side request-forgery attacks. Application code should use an SDK's credential provider rather than manually copying role credentials from metadata.

### 1.8 Bootstrapping and proving a service

Launching an instance creates a computer, not a working application. Software still has to be installed, configured, started, and kept running. Manual SSH is useful while learning, but repeatable instances normally use **user data**: a script supplied at launch and executed as root by cloud-init during the first boot. On Amazon Linux, a minimal web-server script might install `httpd`, create a page, and enable the service:

```bash
#!/bin/bash
set -euxo pipefail
dnf -y install httpd
printf '%s\n' '<h1>Hello from EC2</h1>' > /var/www/html/index.html
systemctl enable --now httpd
```

User data is not interactive, and EC2 reaching the `running` state does not mean the script succeeded. Its output is recorded in `/var/log/cloud-init-output.log`. If an instance exists and networking is correct but the application is absent, read that log from the first error downward. The same principle applies to system services: `systemctl status httpd`, `journalctl -u httpd`, and `ss -ltnp` reveal whether the process is healthy and which address and port it actually listens on.

A deployment is not proven by a green console icon. Test the externally observable result with a command whose exit status expresses success:

```powershell
curl.exe -sf --max-time 5 "http://<public-dns-name>/" -o NUL
if ($LASTEXITCODE -eq 0) { "PASS" } else { "FAIL" }
```

This separates infrastructure state from service behavior. EC2 can correctly report a running instance while Apache is stopped, and Apache can be healthy while a security group or route blocks every visitor. Keep the check small and rerunnable so it can be used after every change.

### 1.9 The API, credentials, cost, and cleanup

The Management Console, AWS CLI, SDKs, and CloudFormation all call the same service APIs. This gives three durable lessons: console actions can be automated; automation is repeatable; and management calls can be audited in CloudTrail. Begin credential troubleshooting by asking AWS who you are:

```powershell
aws --version
aws configure get region
aws sts get-caller-identity
```

Temporary credentials expire. When many unrelated commands suddenly report expired or missing credentials, renew the login before debugging individual services. An `AccessDenied` can also come from an organisation guardrail, such as a Region or instance-family restriction; read the full message because it often names the denying policy layer.

EC2 costs are the sum of separate meters, not one “server price.” Common components include running instance time, EBS capacity and provisioned performance, public IPv4 time, snapshots, and outbound data transfer. Stopping normally pauses only compute. Cost tools also lag behind resource creation, so teardown is verified from live resource inventories rather than from a same-day bill.

**Tags** are key-value metadata attached to resources. A `Name` tag makes console lists readable; ownership, environment, application, and cost-centre tags support inventory and cost allocation. Tags do not create security boundaries by themselves, but IAM conditions and organisational policies can require or evaluate them. Consistent tagging is especially valuable when an account contains resources from several people or exercises.

Treat free-tier or promotional credits as discounts, not architecture. Eligibility changes by account and date, and a free allowance for compute does not necessarily cover its EBS volume or public address. Estimate the complete set of resources in AWS Pricing Calculator, then use Billing and Cost Explorer for actual spend. Pricing data can arrive hours later, whereas resource inventory is current.

Cleanup follows dependency and lifecycle rules. Terminate disposable instances, confirm root-volume deletion behavior, remove unneeded snapshots, release Elastic IPs, and check for security groups or interfaces still referenced by another resource. A resource that fails to delete is evidence of a remaining dependency, not permission to forget it.

```powershell
# Prove that no instances remain running.
aws ec2 describe-instances `
  --filters "Name=instance-state-name,Values=running" `
  --query "Reservations[].Instances[].InstanceId"

# Look for storage and stable addresses that can outlive instances.
aws ec2 describe-volumes --filters "Name=status,Values=available"
aws ec2 describe-addresses
```

## 2. S3 Storage

### 2.1 Buckets, objects, and keys

Amazon Simple Storage Service (**S3**) is object storage. A **bucket** is a named container created in one AWS Region; its name must be unique across the global S3 namespace. An **object** consists of bytes, metadata, and a **key** that uniquely names it within the bucket. S3 scales without provisioning a disk size or filesystem server.

Object storage has a different interface from block or file storage. The main operations create or replace a whole object (`PutObject`), retrieve it (`GetObject`), delete it, and list keys. There is no append or in-place editing. Replacing an object means uploading a complete new value under the same key.

Keys are flat strings. `reports/2026/august.csv` contains slashes, but S3 does not store three nested directories. Consoles group keys by **prefix** to draw a familiar folder tree. This distinction matters operationally: renaming a prefix means copying every matching object to new keys and deleting the old keys, and an “empty folder” is usually a zero-byte placeholder object.

S3 provides strong read-after-write consistency: after a successful write or delete, subsequent reads and listings reflect the completed operation. The service is reached through HTTPS APIs, allowing the same objects to be used by the CLI, SDKs, applications, and other AWS services without mounting a disk.

```powershell
aws s3 mb "s3://<globally-unique-bucket-name>"
aws s3 cp ".\report.pdf" "s3://<bucket-name>/reports/report.pdf"
aws s3 sync ".\website" "s3://<bucket-name>/"
aws s3api head-object --bucket <bucket-name> --key reports/report.pdf
aws s3 ls "s3://<bucket-name>/reports/" --recursive
```

Block storage such as EBS fits filesystems, databases, and software expecting sectors and random writes. S3 fits uploads, media, logs, backups, static content, datasets, and durable build artifacts.

### 2.2 Access control and encryption

New buckets are private. S3 **Block Public Access** adds preventive controls above ordinary permissions. The four settings can reject public ACLs, ignore existing public ACLs, reject public bucket policies, or restrict public buckets. They exist at account and bucket levels, and S3 applies the most restrictive combination. Turning off a block merely permits a public configuration; it does not itself grant access.

A **bucket policy** is a resource-based IAM policy attached to the bucket. This minimal static-site policy grants anonymous read access to objects but not listing or writing:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<bucket-name>/*"
    }
  ]
}
```

The object ARN ends in `/*`; the bucket ARN without that suffix is a different resource. Keep public access narrow and deliberate. Modern buckets normally disable object ACLs through bucket-owner-enforced object ownership, so old tutorials recommending a `public-read` ACL do not match current defaults.

S3 encrypts new objects at rest by default. SSE-S3 uses S3-managed keys. SSE-KMS uses AWS KMS keys when control over key policy, audit trails, or rotation is required, but adds KMS permissions and request costs. Encryption at rest does not replace IAM authorization, and HTTPS protects data in transit.

### 2.3 Static website hosting and two endpoint types

Static website hosting is a bucket configuration naming an index document and, optionally, an error document:

```powershell
aws s3 website "s3://<bucket-name>/" --index-document index.html
```

The resulting **website endpoint** serves index documents and browser-oriented errors over HTTP. It supports anonymous requests only and does not support HTTPS. The **REST endpoint** is the normal S3 API endpoint used by the CLI, SDKs, signed requests, and presigned URLs; it supports HTTPS and authentication.

| Behavior        | Website endpoint                       | REST endpoint                      |
| --------------- | -------------------------------------- | ---------------------------------- |
| Primary purpose | Browser-oriented static hosting        | S3 API operations                  |
| Root request    | Resolves the configured index document | Addresses an object/API operation  |
| Authentication  | Anonymous access                       | IAM/SigV4 authentication supported |
| HTTPS           | Not supported directly                 | Supported                          |
| Missing object  | `404 Not Found`                      | Depends on API permissions         |

The error distinction is important. A website endpoint returns 404 when the requested object does not exist and 403 when the object exists but is not publicly readable. For REST `GetObject`, a missing object returns 404 when the caller also has `s3:ListBucket`; without that listing permission, the API can return 403 rather than reveal existence. These are different endpoint behaviors, not contradictory results.

Typical diagnostics follow directly: raw XML usually indicates the REST endpoint; a browser security warning reflects the website endpoint's HTTP-only design; a downloaded page can indicate incorrect `Content-Type`; and 403 on an existing object means the public read grant or Block Public Access configuration is incomplete.

### 2.4 Durability, availability, and storage classes

S3 Standard is designed for eleven nines of object durability by redundantly storing data across multiple Availability Zones. **Durability** asks whether the data survives; **availability** asks whether a request succeeds now. Neither prevents an authorised user from deleting or overwriting data.

Storage classes trade storage price against retrieval price, minimum duration, resilience, and retrieval delay:

| Class                      | Appropriate shape                                                |
| -------------------------- | ---------------------------------------------------------------- |
| S3 Standard                | Frequently accessed data requiring immediate retrieval           |
| Standard-IA                | Infrequent access with immediate retrieval and retrieval charges |
| One Zone-IA                | Re-creatable infrequent data that can tolerate one-AZ storage    |
| Glacier Instant Retrieval  | Rare access but immediate reads                                  |
| Glacier Flexible Retrieval | Archives that can wait minutes or hours for restoration          |
| Glacier Deep Archive       | Long-term archives with the longest restoration times            |
| Intelligent-Tiering        | Uncertain or changing access patterns, with monitoring overhead  |

Cold classes can have minimum-storage-duration and minimum-object-size charging rules. A cheaper per-GB storage rate is not automatically cheaper for short-lived, tiny, or frequently retrieved objects. Requests and data transfer are separate meters.

### 2.5 Versioning and lifecycle management

With **versioning** enabled, a new write under an existing key creates a new version rather than destroying the old one. A simple delete creates a **delete marker** that hides prior versions; removing the marker can restore visibility. Permanently erasing a versioned object requires deleting each version and marker explicitly.

```powershell
aws s3api put-bucket-versioning `
  --bucket <bucket-name> `
  --versioning-configuration Status=Enabled

aws s3api list-object-versions --bucket <bucket-name>
```

Versioning protects against accidental overwrite and deletion, but every retained version consumes billed storage. A **lifecycle configuration** applies rules to matching objects or prefixes: transition to colder storage, expire current objects, remove incomplete multipart uploads, or expire noncurrent versions. Versioning and lifecycle are therefore commonly paired—retain recent recovery points while removing old accumulation automatically.

Bucket deletion also follows the object model. A bucket must be empty before it can be deleted, and a versioned bucket is not empty until every object version and delete marker has been removed.

### 2.6 Presigned URLs and CORS

A **presigned URL** contains a Signature Version 4 authorization calculated in advance. Its holder may perform one specified operation on one specified object until expiry, without possessing AWS credentials. The URL is a bearer credential and must be protected accordingly. It can never grant more than the signing principal is allowed to do, and a URL made with temporary credentials cannot outlive those credentials.

```powershell
aws s3 presign "s3://<bucket-name>/reports/report.pdf" --expires-in 900
```

That CLI command creates a presigned GET URL. SDKs can generate presigned PUT requests for direct uploads. Presigned operations use the HTTPS REST endpoint and do not require making the bucket public.

**Cross-Origin Resource Sharing (CORS)** controls whether browser JavaScript loaded from one origin may call another origin. An origin is the scheme, hostname, and port combination. The browser can send an `OPTIONS` preflight before methods such as PUT. An S3 CORS configuration names allowed origins, methods, and headers.

CORS is not authentication or authorization. A permissive CORS rule does not grant `s3:GetObject` or `s3:PutObject`; IAM and bucket policies still decide whether S3 accepts the request. A request that works in `curl` but is blocked by browser JavaScript is characteristic CORS evidence.

### 2.7 Operational details and stronger protection

Object metadata affects how clients interpret stored bytes. `Content-Type: text/html` tells a browser to render HTML; `Content-Disposition: attachment` can instruct it to download. Metadata supplied at upload belongs to the object and normally requires a copy operation to replace. Tags are separate key-value labels used by lifecycle, access policies, inventory, and cost reporting.

Large objects are uploaded in parts. A **multipart upload** creates an upload ID, accepts independently retryable parts, and completes by assembling them. This improves throughput and recovery from network interruption, but abandoned multipart uploads consume storage until aborted. Lifecycle rules can remove incomplete uploads after a chosen age.

**Replication** can copy eligible objects to another bucket, including a bucket in another Region. It requires versioning and an IAM role that S3 can assume. Replication supports resilience, compliance, and locality, but it creates additional stored copies and therefore additional cost; it is not a substitute for deciding retention and deletion behavior.

**S3 Object Lock** adds write-once-read-many retention to versioned objects. Governance or compliance modes can prevent versions from being deleted or overwritten until a retention date, providing protection even against powerful credentials. This is intentionally stronger than ordinary versioning and must be planned before relying on it for regulated or ransomware-resistant data.

Useful cleanup commands distinguish current keys from retained history:

```powershell
aws s3 rm "s3://<bucket-name>" --recursive
aws s3api list-object-versions --bucket <bucket-name>
aws s3 rb "s3://<bucket-name>"
```

The recursive remove clears current objects in an unversioned bucket, but it does not erase every historical version in a versioned bucket. Always list versions and delete markers before treating an empty-looking console view as proof of deletion.

## 3. ECR and Fargate

### 3.1 Images, containers, and layers

A **container image** is an immutable filesystem plus metadata such as its startup command. A **container** is a running instance of that image. Containers share the host kernel rather than booting a complete guest operating system, so they generally start faster and occupy less space than virtual machines. Isolation is still real, but the boundary and operational model differ from a VM.

A Dockerfile builds an image as ordered, content-addressed **layers**. Unchanged layers can be cached and reused; frequently changing source code is therefore usually copied after stable dependency-installation steps. The image packages the application, runtime, libraries, and OS userland that must travel together, reducing “works on my machine” differences.

```dockerfile
FROM httpd:2.4-alpine
COPY index.html /usr/local/apache2/htdocs/index.html
EXPOSE 80
```

`EXPOSE` documents the listening port; it does not make the process reachable. Networking at run time still decides which interfaces, ports, security groups, and routes admit traffic. Build and test locally before adding cloud components:

```powershell
docker build -t static-page:1.0 .
docker run --rm -p 8080:80 static-page:1.0
```

### 3.2 Amazon ECR

A cloud runtime cannot pull an image that exists only on a laptop. A **registry** stores and distributes images. Amazon Elastic Container Registry (**ECR**) is a private registry in an AWS account. A repository normally represents one application or image name and contains tagged versions, each backed by a content digest such as `sha256:...`.

A tag is a human-friendly pointer and can be moved unless tag immutability is enabled. A digest identifies exact image content. Deploying a meaningful immutable version tag or digest makes the running artifact auditable; relying on `latest` makes it difficult to know which bytes were actually launched.

```powershell
$account = aws sts get-caller-identity --query Account --output text
$region = aws configure get region
$registry = "$account.dkr.ecr.$region.amazonaws.com"

aws ecr create-repository `
  --repository-name static-page `
  --image-tag-mutability IMMUTABLE `
  --image-scanning-configuration scanOnPush=true

aws ecr get-login-password |
  docker login --username AWS --password-stdin $registry

docker tag static-page:1.0 "$registry/static-page:1.0"
docker push "$registry/static-page:1.0"
```

The login password is temporary and derived from the caller's AWS credentials; an authentication failure after earlier success often means either the registry token or the underlying AWS session expired. Piping the password avoids putting it on the command line or clipboard.

ECR scanning compares packages in an image with vulnerability databases. Findings frequently come from a base image rather than application code. The remedy is still yours: update the base, rebuild, retest, and redeploy. A scan proves only what the scanner knew about that image at that time.

Repository lifecycle policies can expire untagged images or retain only a chosen number of releases. This limits quiet storage growth from CI builds. ECR can also replicate images across Regions or accounts when distribution and recovery requirements justify the extra copies.

```powershell
aws ecr describe-images --repository-name static-page
aws ecr describe-image-scan-findings `
  --repository-name static-page `
  --image-id imageTag=1.0
```

### 3.3 ECS resources and task definitions

Amazon Elastic Container Service (**ECS**) is an orchestrator. Its main concepts are:

- A **cluster** is a logical place in which tasks and services run.
- A **task definition** is an inert, versioned specification for running one or more related containers.
- A **task** is one running copy of a task-definition revision.
- A **service** maintains a desired number of tasks, replaces failures, and performs controlled deployments.

The task definition supplies everything a missing host cannot guess: image URI, CPU and memory, ports, environment, secrets, logging, roles, and compatible launch type. A focused Fargate example is:

```json
{
  "family": "static-page",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::<account-id>:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "<account-id>.dkr.ecr.<region>.amazonaws.com/static-page:1.0",
      "portMappings": [{"containerPort": 80, "protocol": "tcp"}],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/static-page",
          "awslogs-region": "<region>",
          "awslogs-stream-prefix": "web"
        }
      }
    }
  ]
}
```

Fargate accepts only supported CPU/memory combinations. The values `256` and `512` mean 0.25 vCPU and 512 MiB, suitable for a very small process. Sizing too low causes throttling or out-of-memory termination; sizing too high wastes money. Measure CPU, memory, and latency, then choose the smallest allocation with safe headroom.

### 3.4 Fargate and responsibility boundaries

ECS can place tasks on an EC2 fleet you manage or on **AWS Fargate**. With the EC2 launch type, you patch, scale, secure, and pay for container hosts. With Fargate, AWS supplies the host for each task and no EC2 instance appears in your account. You still own everything inside the image, the task configuration, IAM permissions, logs, and network exposure.

Fargate bills for requested vCPU, memory, and certain additional resources while the task runs. An idle long-running task still incurs compute charges. ECR separately charges for stored image data and scanning features where applicable. Fargate is attractive when avoiding host management matters more than obtaining the lowest unit price from a heavily utilised fleet.

Two IAM roles must not be confused:

- The **task execution role** is used by ECS/Fargate agents for platform work such as pulling a private ECR image and sending logs.
- The **task role** is delivered to the containers and controls AWS API calls made by application code.

A static web server may need an execution role but no task role. An application reading S3 needs a task role with narrow S3 permissions; adding those permissions to the execution role does not correctly grant them to the application.

### 3.5 Networking, logs, and failure diagnosis

Fargate requires `awsvpc` networking. Each task receives an elastic network interface (**ENI**) and private IP in a selected subnet, and consumes an address from that subnet. Security groups attach to the task ENI. Public reachability additionally needs suitable routing, a public IP or load balancer, and an inbound rule for the application port.

```powershell
aws ecs register-task-definition --cli-input-json file://taskdef.json

aws ecs run-task `
  --cluster default `
  --launch-type FARGATE `
  --task-definition static-page `
  --network-configuration "awsvpcConfiguration={subnets=[<subnet-id>],securityGroups=[<security-group-id>],assignPublicIp=ENABLED}"
```

There is no host to SSH into. Container stdout and stderr should go to CloudWatch Logs, and ECS records lifecycle details. Diagnose a failed task from actual state rather than repeatedly launching replacements:

```powershell
aws ecs describe-tasks `
  --cluster default `
  --tasks <task-arn> `
  --query "tasks[].{LastStatus:lastStatus,StopCode:stopCode,StoppedReason:stoppedReason,Containers:containers[].reason}"

aws logs tail "/ecs/static-page" --follow
```

An image-pull failure points toward the image URI, execution role, registry authentication, or network path to ECR. An application exit code points inside the container. `OutOfMemoryError` points at task sizing. A running but unreachable task points at its ENI, address, route, security group, port mapping, or listening process. A bare task stays stopped; an ECS service replaces failed tasks, which improves recovery but can create a loop if every replacement has the same bad image or configuration.

Long-running public applications normally place an Application Load Balancer in front of an ECS service rather than assigning a public address to every task. The load balancer supplies a stable endpoint and health checks while tasks remain replaceable private targets. During a deployment, the service can start tasks from a new task-definition revision, wait for health checks, and drain the old revision. Health checks must test meaningful application behavior: a check that only proves a TCP port opened can mark a broken application healthy.

Operationally, use specific image versions, enable logs before the first failure, set service deployment limits that preserve capacity, and configure alarms around stopped tasks or an unhealthy desired count. Orchestration restarts processes; it does not decide whether repeated restarts need a person.

## 4. Lambda Functions

### 4.1 Functions and the handler contract

AWS Lambda runs code in response to invocations without exposing a server to manage. AWS supplies and patches the host, operating system, and language runtime; you supply function code, dependencies, configuration, and permissions. Nothing belonging to the function listens on a port or runs continuously between invocations.

Each runtime defines a **handler** contract. For Python:

```python
def lambda_handler(event, context):
    return {"status": "ok"}
```

The handler setting `lambda_function.lambda_handler` means “load `lambda_function.py` and call `lambda_handler`.” The `event` contains data from the invoker. Its shape is determined by the event source, not Lambda. The `context` describes the current invocation, including request identity and remaining time.

Module-level code runs when an execution environment is initialised; the handler can then run many times in that environment. Create reusable SDK clients and other safe, expensive setup outside the handler to amortise it. Do not assume process memory is durable or exclusive storage. A later invocation may reuse it, while another simultaneous invocation runs in a different environment. Durable state belongs in an external service.

Lambda billing is mainly per request and execution duration weighted by configured memory. With no invocations, there is no function execution charge. This suits short, event-shaped work; it does not suit an indefinite daemon, a listening network server, or a job longer than the per-invocation timeout ceiling.

### 4.2 Events, S3 triggers, and idempotency

A **trigger** is persistent wiring from an event source to a function. An S3 event notification can invoke Lambda when an object is created, optionally filtered by key prefix or suffix. The event contains object coordinates and metadata, not the object's bytes:

```python
import urllib.parse
import boto3

s3 = boto3.client("s3")

def lambda_handler(event, context):
    for record in event["Records"]:
        bucket = record["s3"]["bucket"]["name"]
        key = urllib.parse.unquote_plus(record["s3"]["object"]["key"])
        response = s3.get_object(Bucket=bucket, Key=key)
        print({"bucket": bucket, "key": key,
               "bytes": response["ContentLength"]})
```

S3 URL-encodes object keys in notifications; `unquote_plus` prevents filenames containing spaces or special characters from becoming misleading `NoSuchKey` errors. Iterate over `Records` rather than assuming exactly one. Event-driven delivery can be retried and may deliver the same logical event more than once, so handlers should be **idempotent**: repeating an event produces the same final result rather than duplicating side effects.

S3 invokes Lambda asynchronously. Lambda retries certain failed asynchronous invocations, so a permanent code defect can execute repeatedly. An on-failure destination or dead-letter queue can retain failed event information, while CloudWatch alarms can notify a person when error or throttle metrics rise.

A function that writes new objects into the same bucket and prefix that triggers it can create an invocation loop. Prevent recursion by using a separate output bucket or a trigger filter that excludes output keys.

### 4.3 Two permission directions

Lambda involves two independent authorisation questions:

1. **Who may invoke the function?** A resource-based policy on the function can allow a service such as S3 to call `lambda:InvokeFunction`, often restricted by source bucket ARN and account.
2. **What may the function code do?** The function's **execution role** supplies temporary credentials and identity-based permissions to the running code.

The basic Lambda execution managed policy primarily permits CloudWatch logging. Reading an S3 object additionally needs an explicit `s3:GetObject` allow on the relevant object ARN. Do not place access keys in source code or environment variables; SDKs automatically obtain the role credentials.

The failure shapes differ. No invocation and no log stream suggest trigger configuration or invoke permission. A log entry showing `AccessDenied` from `GetObject` means the function ran but its execution role lacked permission. A function unable to create logs can appear silent, so logging permission is foundational.

### 4.4 CloudWatch Logs and observable execution

Lambda sends standard output, exceptions, and platform records to a CloudWatch Logs group named `/aws/lambda/<function-name>`. Each invocation has `START`, `END`, and `REPORT` records. The report includes duration, billed duration, configured memory, maximum memory used, and request ID. This is both diagnostic evidence and a performance/cost measurement.

```powershell
aws lambda invoke `
  --function-name <function-name> `
  --cli-binary-format raw-in-base64-out `
  --payload '{}' `
  .\response.json

aws logs tail "/aws/lambda/<function-name>" --follow
aws lambda get-function-configuration --function-name <function-name>
aws lambda get-policy --function-name <function-name>
```

Log retention must be set deliberately; indefinite retention can accumulate cost and sensitive application data. Logs are not notification by themselves. Metrics and alarms convert error counts, duration, throttles, or dead-letter activity into messages that reach operators.

### 4.5 Deployment packages and dependencies

A ZIP-based Python function receives the runtime, standard library, and AWS SDK components provided by that runtime. Other libraries must ship in the **deployment package** or an attached **layer**. Installing a package on a laptop does not install it in Lambda.

```powershell
New-Item -ItemType Directory -Path .\package
pip install --target .\package <library-name>
Copy-Item .\lambda_function.py .\package\
Compress-Archive -Path .\package\* -DestinationPath .\function.zip

aws lambda update-function-code `
  --function-name <function-name> `
  --zip-file fileb://function.zip
```

Libraries with compiled native components must match Lambda's Linux operating system and processor architecture. Packaging a Windows binary into a ZIP produces an import failure even though the module exists. Build in a compatible environment or request compatible wheels from the package index.

A **layer** is a separately versioned dependency archive shared by functions. Python packages must appear under the layer's `python/` directory. Layers reduce duplicate packaging but remain part of the function's combined deployment and compatibility surface. A Lambda **container image** in ECR is the alternative when dependencies are large or need a controlled filesystem and build process.

`Runtime.ImportModuleError` is packaging evidence: the handler module or one of its imports could not load. It is not evidence that the handler's business logic ran and failed.

Function configuration should remain outside the deployment package when it varies by environment. Environment variables are suitable for non-secret values such as bucket names, feature flags, and log levels. Secrets belong in a managed secret store and are fetched under the execution role; marking a value as an environment variable does not make it secret from principals allowed to inspect function configuration.

Lambda functions run with service-managed networking by default and can reach public AWS endpoints. Attaching a function to customer VPC subnets creates network interfaces so it can reach private resources. That attachment does not automatically preserve internet access: private-subnet routing, NAT, or VPC endpoints must provide the required outbound paths. A VPC-attached function that times out only when calling an external service often has a routing problem rather than a handler problem.

### 4.6 Memory, timeout, concurrency, and cold starts

A **cold start** occurs when Lambda creates a new execution environment, loads code, starts the runtime, and runs module-level initialisation. Warm reuse avoids much of that work. Package size, runtime, network placement, and initialisation influence latency. Provisioned concurrency can keep environments initialised for latency-sensitive functions but adds an always-ready cost.

The memory setting also controls available CPU. More memory can make CPU-bound code finish sufficiently faster to reduce both latency and total GB-seconds. Tune from report data rather than choosing the lowest value automatically. Timeout is a hard ceiling: when reached, Lambda terminates the invocation, so side effects may be partially complete and a retry may follow.

**Concurrency** is the number of simultaneous invocations. Lambda creates environments as demand rises until quotas or configured limits intervene. Throttled synchronous requests fail back to callers; asynchronous sources follow their retry behavior. Reserved concurrency can protect one function or cap a runaway one. Concurrency makes thread-safe local code insufficient for global coordination; shared limits and deduplication need external state.

### 4.7 Choosing the compute shape

| Property               | EC2                        | Fargate                           | Lambda                       | S3 static hosting              |
| ---------------------- | -------------------------- | --------------------------------- | ---------------------------- | ------------------------------ |
| Unit deployed          | Virtual machine            | Container task                    | Function                     | Objects/configuration          |
| Idle compute charge    | Yes while running          | Yes while task runs               | No invocation charge         | No compute process             |
| Runtime duration       | Continuous                 | Continuous or batch               | Bounded invocation           | Not applicable                 |
| Operating-system work  | Customer                   | AWS host; customer image          | AWS                          | AWS                            |
| Characteristic failure | Machine/service stays down | Task stops or service replaces it | One invocation fails/retries | Request refused or key missing |

Static bytes fit object storage. Short event-driven work fits Lambda. A long-running process with a packaged runtime fits a container. EC2 remains suitable when the workload needs machine-level control, unusual protocols, specialised hardware, or software that does not fit managed constraints.

## 5. AWS CloudFormation

### 5.1 Infrastructure as code and stacks

AWS CloudFormation turns a declarative **template** into AWS API calls. Instead of recording a sequence of console clicks, the template describes the desired resources and their configuration. CloudFormation determines creation order from references, records the resources it created, and manages them as a **stack**.

A stack is one deployment of a template under a stable stack name. It is created, updated, inspected, and deleted as a unit. Deploying the same template twice gives equivalent configuration but different physical resources. This makes the template repeatable documentation: it is not a description written after the fact; it is an input used to create the infrastructure.

CloudFormation does not make resource charges disappear. It makes creating many connected resources easy, which makes disciplined change review and stack deletion more important.

### 5.2 Template anatomy

CloudFormation accepts JSON or YAML. YAML is usually easier to read and permits comments, but indentation is syntax: use spaces, never tabs. Only `Resources` is required, while a useful template commonly contains:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Small demonstrative stack

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, test, prod]

Resources:
  DataBucket:
    Type: AWS::S3::Bucket

Outputs:
  BucketName:
    Description: Generated physical bucket name
    Value: !Ref DataBucket
```

**Parameters** are deployment-time inputs. They can supply defaults and validation such as `AllowedValues`, patterns, and minimum or maximum sizes. AWS-specific types validate real account resources and provide suitable console controls:

```yaml
Parameters:
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
  AmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64
```

`NoEcho: true` masks a parameter in CloudFormation displays but does not encrypt it or make every downstream use secret. Sensitive values belong in a purpose-built secret service and should not appear in outputs, metadata, or plaintext properties.

**Resources** are entries with a logical ID, type such as `AWS::S3::Bucket`, and type-specific properties. You choose the logical ID; AWS assigns the physical ID, such as a bucket name or instance ID. The stack maintains that mapping. The CloudFormation resource reference is authoritative for required properties, return values, and each property's update behavior.

**Outputs** return values discovered during deployment, such as URLs, names, and IDs. Scripts should query outputs rather than copy values from the console.

### 5.3 Intrinsic functions and dependencies

Intrinsic functions connect resources using values that do not exist until deployment.

**`Ref`** returns a parameter value or a resource's type-specific primary value. For an S3 bucket it returns the bucket name; for an EC2 instance it returns the instance ID. Always check the resource's “Return values” documentation.

**`GetAtt`** retrieves a named resource attribute:

```yaml
Outputs:
  BucketArn:
    Value: !GetAtt DataBucket.Arn
  BucketDomain:
    Value: !GetAtt DataBucket.RegionalDomainName
```

**`Sub`** inserts references and attributes into a string:

```yaml
Resource: !Sub "${DataBucket.Arn}/*"
Value: !Sub "deployed-by-${AWS::StackName}-in-${AWS::Region}"
```

`${LogicalId}` behaves like `Ref`, while `${LogicalId.Attribute}` behaves like `GetAtt`. In a substituted shell script, `${!SHELL_VARIABLE}` emits the literal `${SHELL_VARIABLE}` rather than treating it as a template reference.

**Pseudo parameters** such as `AWS::Region`, `AWS::AccountId`, and `AWS::StackName` provide deployment context without declarations. `AWS::NoValue`, normally used through `Fn::If`, removes a property entirely.

References also declare dependencies. If an instance uses `!GetAtt WebSecurityGroup.GroupId`, CloudFormation knows the group must exist first. Explicit **`DependsOn`** is needed only when ordering exists without data flowing through a reference. A classic case is a route that references an internet gateway but not the separate gateway-attachment resource:

```yaml
DefaultRoute:
  Type: AWS::EC2::Route
  DependsOn: GatewayAttachment
  Properties:
    RouteTableId: !Ref PublicRouteTable
    DestinationCidrBlock: 0.0.0.0/0
    GatewayId: !Ref InternetGateway
```

The following focused fragment shows the functions working together without hardcoding a generated name:

```yaml
Resources:
  SiteBucket:
    Type: AWS::S3::Bucket
    Properties:
      WebsiteConfiguration:
        IndexDocument: index.html
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        IgnorePublicAcls: true
        BlockPublicPolicy: false
        RestrictPublicBuckets: false

  SitePolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref SiteBucket
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal: "*"
            Action: s3:GetObject
            Resource: !Sub "${SiteBucket.Arn}/*"

Outputs:
  SiteUrl:
    Value: !GetAtt SiteBucket.WebsiteURL
  BucketName:
    Value: !Ref SiteBucket
```

`!Ref SiteBucket` supplies the generated bucket name to the policy attachment and output. `${SiteBucket.Arn}` reads the bucket ARN, and `/*` converts it into the object-resource pattern. `!GetAtt SiteBucket.WebsiteURL` asks the resource for a value known only after creation. These references also order the bucket before its policy. The template still does not upload `index.html`: the bucket is infrastructure, while object content is a data-plane operation performed separately.

CloudFormation supports resource-level attributes outside `Properties`. `DependsOn`, `DeletionPolicy`, `UpdateReplacePolicy`, `Condition`, and `Metadata` belong beside `Type`. Placing one inside `Properties` sends it to the resource type, where it is normally rejected as an unknown property. Tags, when the resource supports them, usually do belong inside `Properties` and should identify ownership, environment, and stack purpose.

### 5.4 Driving stacks from PowerShell

Validate after edits. Validation checks parsing, section structure, and references, but it cannot predict external failures such as a globally duplicated bucket name:

```powershell
aws cloudformation validate-template `
  --template-body file://template.yml
```

Validation also does not prove that user data works, that quotas permit creation, or that the caller has every required permission. Treat it as a fast structural gate followed by change-set review and an observable post-deployment check.

`deploy` creates a missing stack, updates an existing stack with the same name, and reports when there are no changes:

```powershell
aws cloudformation deploy `
  --template-file .\template.yml `
  --stack-name example-stack `
  --parameter-overrides Environment=dev

aws cloudformation describe-stacks `
  --stack-name example-stack `
  --query "Stacks[0].{Status:StackStatus,Outputs:Outputs}"
```

A template creating IAM resources requires explicit acknowledgment. `CAPABILITY_IAM` covers generated IAM names; `CAPABILITY_NAMED_IAM` is required when the template assigns names and satisfies both cases:

```powershell
aws cloudformation deploy `
  --template-file .\template.yml `
  --stack-name example-stack `
  --capabilities CAPABILITY_NAMED_IAM
```

The requirement is a security boundary: a copied template can create identities and permissions, so deployment requires the operator to acknowledge that consequence.

### 5.5 Change sets and update behavior

A **change set** compares a proposed template with the current stack and identifies additions, modifications, removals, and possible replacements. Preview before changing important infrastructure:

```powershell
aws cloudformation deploy `
  --template-file .\template.yml `
  --stack-name example-stack `
  --no-execute-changeset

aws cloudformation describe-change-set --change-set-name <change-set-arn>
aws cloudformation execute-change-set --change-set-name <change-set-arn>
```

Every resource property documents one of three update outcomes:

- **No interruption:** modified in place while remaining available.
- **Some interruptions:** modified in place but temporarily disrupted.
- **Replacement:** a new physical resource is created for the same logical ID; the old one is then cleaned up.

Changing an EBS-backed instance type commonly stops and restarts the same instance, potentially changing an auto-assigned public address. Changing an instance `ImageId` replaces the instance because a running machine cannot swap its boot image. Changing `BucketName` replaces the bucket because S3 has no rename operation. Read the property's **Update requires** line and the change set's replacement field before execution.

Explicit physical names tighten identity. A stack cannot be renamed, and many named resources cannot be transparently replaced because the new resource cannot temporarily take the same name. Generated names make repeatable deployment and replacement safer.

### 5.6 Retention, deletion, and drift

`DeletionPolicy` controls what happens to a resource when it leaves the stack or the stack is deleted. `UpdateReplacePolicy` controls the old resource during replacement:

```yaml
DatabaseVolume:
  Type: AWS::EC2::Volume
  DeletionPolicy: Snapshot
  UpdateReplacePolicy: Snapshot
  Properties:
    AvailabilityZone: !Select [0, !GetAZs ""]
    Size: 20
    VolumeType: gp3
```

The main choices are `Delete`, `Retain`, and—for supported data resources—`Snapshot`. `Retain` removes the resource from stack management but leaves it existing and billing. Retention is a deliberate data-lifecycle decision, not free insurance. Non-empty S3 buckets commonly cause `DELETE_FAILED`; empty the bucket and retry, or explicitly retain the resource if that is the intended outcome.

**Drift** occurs when someone changes a stacked resource outside CloudFormation. Drift detection compares supported live properties with the template's expected values:

```powershell
$detectionId = aws cloudformation detect-stack-drift `
  --stack-name example-stack `
  --query StackDriftDetectionId --output text

aws cloudformation describe-stack-drift-detection-status `
  --stack-drift-detection-id $detectionId
```

Termination protection blocks accidental stack deletion until explicitly disabled. Stack policies can constrain sensitive updates. Both add guardrails; neither substitutes for reviewing change sets and backups.

Long-lived stacks need a disciplined operating model. Change stacked resources through a template so the next deployment does not silently reverse a console edit. Use stack outputs as contracts for automation, tags for ownership and allocation, drift detection for exceptions, and termination protection for important stacks. `get-template` retrieves the deployed truth when the local file and stack behavior disagree.

CloudFormation can also **import** certain existing resources without recreating them. The template must accurately describe the live resource, and an import-type change set maps its physical identifier to a logical ID. Imports are useful during migration from manually built infrastructure, but importing configuration you do not understand merely moves uncertainty into a stack. A required retention policy protects the resource during the import process.

An **export** gives an output a Region-unique name; another stack reads it with `ImportValue`. CloudFormation prevents the exporting stack from deleting or changing the exported value while consumers still import it. This enforcement is valuable but couples lifecycle across stacks. Use exports for stable boundaries, not as a shortcut for every value.

### 5.7 Reading failures

Stack statuses combine an operation with a phase: `CREATE_IN_PROGRESS`, `UPDATE_COMPLETE`, `ROLLBACK_COMPLETE`, or `DELETE_FAILED`. An `IN_PROGRESS` state means wait. `ROLLBACK_COMPLETE` after a failed create means CloudFormation successfully removed what it could create; the stack record remains but cannot be updated into existence. Delete it before reusing the stack name.

```powershell
aws cloudformation describe-stack-events --stack-name example-stack
```

Events are returned newest first. Reconstruct them oldest first and find the first failed resource carrying a specific status reason. Later cancellations and rollback events are consequences. Reading only the newest red event often hides the cause.

Rollback can delete the resource needed for diagnosis. During deliberate debugging, `deploy --disable-rollback` or a lower-level create with `DO_NOTHING` preserves the partial stack, which continues to incur charges until cleaned up.

A failed update differs from a failed initial create. `UPDATE_ROLLBACK_COMPLETE` normally means CloudFormation restored the previous working configuration, so the stack can be corrected and updated again. `UPDATE_ROLLBACK_FAILED` means rollback itself needs intervention. `DELETE_FAILED` means at least one resource could not be removed—commonly a non-empty bucket, termination protection, or an external dependency. Read that resource's failure event, correct the cause, and retry deletion; do not assume the remaining resources are harmless or free.

CloudFormation reports success when its API calls succeed, not when software inside a resource works. An EC2 instance can reach `CREATE_COMPLETE` while user data fails. In that case, the evidence is inside the instance at `/var/log/cloud-init-output.log`. A `CreationPolicy` with `cfn-signal` can make CloudFormation wait for in-instance configuration and fail if the signal never arrives.

### 5.8 Conditions, composition, and practical limits

`Conditions` and `Fn::If` include resources or properties based on evaluated inputs. A condition can guard a whole resource through `Condition: Name`, while `!If` chooses a property value. `AWS::NoValue` means omit the property rather than supply an empty value:

```yaml
Parameters:
  RequestedBucketName:
    Type: String
    Default: ""

Conditions:
  HasBucketName: !Not [!Equals [!Ref RequestedBucketName, ""]]

Resources:
  ConditionalBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !If
        - HasBucketName
        - !Ref RequestedBucketName
        - !Ref AWS::NoValue
```

`Mappings` and `FindInMap` provide static two-level lookup tables embedded in a template, such as approved instance sizes by environment. They are suitable for values that change only when the template changes. SSM parameters or other external configuration are better for publisher-maintained values such as current AMI IDs.

A **transform** rewrites a template before normal CloudFormation processing. `Transform: AWS::Serverless-2016-10-31` enables AWS Serverless Application Model (**SAM**) types such as `AWS::Serverless::Function`; the transform expands them into ordinary functions, permissions, roles, and event wiring. The processed template can be inspected, which is useful when shorthand behavior is unclear.

**Nested stacks** are `AWS::CloudFormation::Stack` resources pointing to component templates. They split one system into smaller templates while preserving one parent-controlled deployment. Cross-stack exports instead connect independently deployed stacks. Choose nested stacks when components share one lifecycle and exports when genuinely separate stacks have a stable interface.

A **custom resource** delegates create, update, and delete handling to code, often Lambda, when no native type models the requirement. CloudFormation sends lifecycle events and waits for a response. The delete path is as important as create: custom code that provisions something but cannot remove it breaks stack teardown. Prefer native resource types and registry extensions before accepting that maintenance burden.

`Metadata` with `AWS::CloudFormation::Init`, the `cfn-init` helper, `CreationPolicy`, and `cfn-signal` provide a more structured alternative to unobserved user-data scripts. CloudFormation can wait for a success signal and roll back if configuration never completes, making the stack's status better reflect readiness.

**StackSets** distribute one template across multiple accounts or Regions, commonly for organisation-wide baselines. They add delegated administration, target selection, rollout, and failure-tolerance concerns and are distinct from ordinary single-account stacks.

CloudFormation models infrastructure resources more readily than data-plane contents. It can create a bucket but has no ordinary `AWS::S3::Object` resource for uploading application files. Template and resource-count quotas also encourage decomposition. Larger templates can be stored in S3 instead of sent directly in the request body.

```powershell
aws cloudformation get-template --stack-name example-stack
aws cloudformation delete-stack --stack-name example-stack
aws cloudformation wait stack-delete-complete --stack-name example-stack
```

`get-template` settles which template the live stack actually used, which can differ from an edited local file that was never deployed.

## 6. VPC and IAM

### 6.1 VPCs, CIDR blocks, and subnets

An Amazon Virtual Private Cloud (**VPC**) is an isolated IP network inside one Region. The VPC spans the Region, while each **subnet** belongs permanently to one Availability Zone. A default VPC arrives with subnets, routing, an internet gateway, DNS settings, and public-address defaults already configured. A custom VPC begins with fewer assumptions, making every path explicit.

The VPC and each subnet receive a Classless Inter-Domain Routing (**CIDR**) block. `10.0.0.0/16` contains 65,536 IPv4 addresses; `10.0.1.0/24` carves out 256 of them. Subnet blocks must fit inside the VPC block and cannot overlap. Address space inside the VPC does not have an hourly charge, while redesigning an undersized or overlapping network can be costly. Plan non-overlapping blocks with room for multiple AZs and future tiers.

AWS reserves five IPv4 addresses in every subnet. For `10.0.1.0/24`:

| Address        | Purpose                                                              |
| -------------- | -------------------------------------------------------------------- |
| `10.0.1.0`   | Network address                                                      |
| `10.0.1.1`   | VPC router                                                           |
| `10.0.1.2`   | Amazon-provided DNS                                                  |
| `10.0.1.3`   | Reserved for future use                                              |
| `10.0.1.255` | Broadcast address, reserved even though VPC broadcast is unsupported |

A `/24` therefore has 251 usable addresses, while a `/28` has only 11. This matters for Fargate tasks, load balancers, interface endpoints, and other resources that consume subnet IPs.

A new VPC includes a main route table with a `local` route, a default network ACL, and a default security group. The local route permits communication between addresses in the VPC CIDR. It does not create an internet path. `EnableDnsSupport` enables the Amazon resolver, and `EnableDnsHostnames` allows appropriate public DNS hostnames. DNS names are operational dependencies, not decoration.

```powershell
aws ec2 describe-vpcs `
  --query "Vpcs[].{Id:VpcId,Cidr:CidrBlock,Default:IsDefault}"

aws ec2 describe-subnets `
  --query "Subnets[].{Id:SubnetId,Vpc:VpcId,Cidr:CidrBlock,AZ:AvailabilityZone,PublicIpDefault:MapPublicIpOnLaunch}"
```

### 6.2 Internet gateways, routes, and public addressing

An **internet gateway (IGW)** connects a VPC to the public internet. Creating the gateway and attaching it to a VPC are distinct operations. An unattached gateway exists but moves no traffic. An attached gateway also does nothing until a route table sends traffic to it.

A route table selects a target by longest-prefix match. Every VPC route table contains the VPC CIDR targeting `local`. A public route table adds `0.0.0.0/0` targeting the IGW. Because the local route is more specific, internal traffic remains internal while all other IPv4 destinations use the gateway.

A subnet uses the route table explicitly associated with it; without an explicit association, it silently uses the VPC's main route table. A route table with a perfect default route but no subnet association governs nothing and generates no error. A subnet is **public** because its effective route table has a route to an IGW—not because the subnet has a `Public` property.

CloudFormation expresses the gateway and attachment separately, then forces the route to wait for attachment:

```yaml
InternetGateway:
  Type: AWS::EC2::InternetGateway

GatewayAttachment:
  Type: AWS::EC2::VPCGatewayAttachment
  Properties:
    VpcId: !Ref VPC
    InternetGatewayId: !Ref InternetGateway

DefaultRoute:
  Type: AWS::EC2::Route
  DependsOn: GatewayAttachment
  Properties:
    RouteTableId: !Ref PublicRouteTable
    DestinationCidrBlock: 0.0.0.0/0
    GatewayId: !Ref InternetGateway
```

An internet-routable instance also needs a public address. `MapPublicIpOnLaunch` is a subnet default applied when an instance is launched, and launch settings can override it. The guest sees its private address; the IGW maps an auto-assigned public address to it. Route without address and address without route both fail. Public IPv4 addresses are metered while they exist.

The simplified inbound path is:

```text
client
  -> internet gateway and public/private address mapping
  -> effective route for the subnet
  -> subnet network ACL
  -> ENI security group
  -> operating-system listener
```

Outbound traffic traverses the same controls in reverse order. Each layer must permit the complete round trip.

```powershell
aws ec2 describe-route-tables `
  --filters "Name=association.subnet-id,Values=<subnet-id>" `
  --query "RouteTables[].{Id:RouteTableId,Routes:Routes,Associations:Associations}"

aws ec2 describe-internet-gateways `
  --filters "Name=attachment.vpc-id,Values=<vpc-id>"
```

### 6.3 Security groups and network ACLs

Security groups and network ACLs (**NACLs**) filter different boundaries:

| Property                | Security group           | Network ACL                      |
| ----------------------- | ------------------------ | -------------------------------- |
| Attached to             | ENI/resource             | Subnet                           |
| State                   | Stateful                 | Stateless                        |
| Rule effects            | Allow only               | Allow and deny                   |
| Evaluation              | All allows combine       | Numbered order; first match wins |
| Return traffic          | Automatically recognised | Must be explicitly allowed       |
| Default custom behavior | No inbound, all outbound | Deny until rules are added       |

Use security groups as the primary workload firewall. A rule can name another security group as its source, meaning “permit traffic from interfaces wearing that group.” This survives address changes and scales better than maintaining private-IP lists. For example, a database group can allow its port only from the application group.

Use a NACL for subnet-wide guardrails or explicit CIDR denies. Because it is stateless, a restrictive web-subnet NACL must allow inbound TCP 80 and outbound client ephemeral ports. Rule order matters: rule 100 allowing a flow makes rule 200 denying it irrelevant because evaluation already stopped. Number rules with gaps so new entries can be inserted.

Neither filter proves a process is listening. A timeout suggests a silent filter or route failure; immediate connection refusal means the packet reached the host and no service accepted it.

Statefulness is easiest to understand as connection tracking. If a security group permits a client to begin a TCP connection to port 443, response packets for that connection are automatically permitted even when no outbound rule explicitly names the client's ephemeral port. This does not turn the group into a general return route: unrelated connections still need their own matching rules. NACLs keep no such connection state. A restrictive NACL must allow the service port in the request direction and the operating system's ephemeral-port range in the response direction. The client chooses that temporary port, so allowing only port 443 both ways is a common reason for one-way failure.

Routing and filtering answer different questions. A route table chooses the next hop for a destination; it does not grant application access. A security group or NACL can allow a packet but cannot invent a missing route. Conversely, a correct route can deliver traffic directly into a deny rule. Troubleshooting is faster when each claim is tested separately: effective route, public or private address mapping, subnet filter, ENI filter, and service listener.

```powershell
aws ec2 describe-security-groups --group-ids <security-group-id>
aws ec2 describe-network-acls `
  --filters "Name=association.subnet-id,Values=<subnet-id>"
```

### 6.4 Private subnets and additional network services

A **private subnet** has no route to an internet gateway. Its instances cannot receive unsolicited internet connections, but they may still need outbound package downloads or public API access. A **NAT gateway** is placed in a public subnet with an Elastic IP; the private route table sends `0.0.0.0/0` to it. Connections begun inside can return, while the NAT gateway does not create a general inbound path.

Putting the NAT gateway in the private subnet is a circular failure: it also lacks a route to the IGW. NAT gateways charge for time and processed data, and their Elastic IP is a separate resource. They can dominate the cost of a small architecture.

A **VPC endpoint** reaches supported AWS services without an internet or NAT path. Gateway endpoints for S3 and DynamoDB add route-table targets and have no hourly endpoint charge. Interface endpoints create private ENIs, use security groups, and have hourly and data-processing charges. Endpoint policies provide another permission filter.

VPC peering routes privately between two non-overlapping VPCs but is not transitive. Transit Gateway provides a hub for many networks at additional cost. IPv6 addresses are globally routable; an egress-only internet gateway permits outbound-initiated IPv6 without inbound initiation. Plan address ranges early because overlapping CIDRs obstruct future connectivity.

**VPC Flow Logs** record flow metadata—not packet contents—from a VPC, subnet, or ENI to CloudWatch Logs or S3. `ACCEPT` means security groups and NACLs permitted the recorded flow; `REJECT` means one of them did not. Flow logs are opt-in, so a silent security-group drop leaves no historical evidence when they were not enabled.

### 6.5 IAM identities, roles, and policy anatomy

AWS Identity and Access Management (**IAM**) decides whether an authenticated API request is authorised. A **user** is a long-lived identity, a **group** collects users, and a **role** is assumed by a person, workload, or AWS service to obtain temporary credentials. Prefer roles because their sessions expire and there is no permanent secret inherent in the role.

An IAM policy is a JSON document containing statements:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadReports",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::example-reports/*",
      "Condition": {
        "StringEquals": {"aws:RequestedRegion": "us-east-1"}
      }
    }
  ]
}
```

Read the statement as: when the condition matches, allow this API action against these resources. `Version` is the policy-language version and remains the literal `2012-10-17`; it is not today's date. `Action` identifies API operations, `Resource` identifies ARNs, and `Condition` tests request context.

An **identity-based policy** attaches to a user, group, or role and describes what that principal may do. A **resource-based policy** attaches to a supported resource and names a `Principal`, describing who may use it. Bucket policies and Lambda invoke permissions are resource-based examples.

Actions accept specific resource shapes. `s3:GetObject` uses an object ARN such as `arn:aws:s3:::bucket/*`; `s3:ListBucket` uses the bucket ARN without `/*`; some account-wide list or describe actions support only `"Resource": "*"`. A syntactically valid statement with an unsupported ARN can simply match no request and leave an implicit deny. The Service Authorization Reference documents each action's resource types and condition keys.

### 6.6 Roles, trust, and temporary credentials

A role has two separate permission questions:

- Its **trust policy** states who may call `sts:AssumeRole` for the role.
- Its attached permissions policies state what a successful role session may do.

An EC2 role trust policy names the EC2 service principal:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
```

Perfect S3 permissions on a role trusting only Lambda do not help an EC2 instance; EC2 cannot assume it. Services commonly expose assumed-role credentials automatically—EC2 through instance metadata and Lambda through its execution environment. SDKs refresh these temporary credentials without embedding keys.

IAM Identity Center uses the same role-session model for people. A permission set creates roles in target accounts, and sign-in returns temporary credentials. `aws sts get-caller-identity` reveals the account and assumed-role ARN and is the first diagnostic for identity confusion.

### 6.7 Policy evaluation: deny by default

AWS evaluates a request using every applicable policy layer:

```text
authenticate the principal
  -> gather identity and resource policies
  -> apply permission boundary and organisation SCP filters
  -> matching explicit Deny?  DENY
  -> otherwise matching Allow? ALLOW
  -> otherwise                 DENY (implicit)
```

The fundamental rules are:

1. Start with implicit deny.
2. Any applicable explicit `Deny` wins over every allow.
3. Without an explicit deny, at least one applicable `Allow` is required.

An **implicit deny** usually means no policy granted the requested action/resource combination. An **explicit deny** is a matching `Deny` statement and cannot be overridden, even by administrator permissions. Error messages often distinguish them and may identify the responsible policy layer.

A **permission boundary** caps the maximum permissions of a user or role. Effective identity permission is the intersection of its identity policies and boundary. A **service control policy (SCP)** caps principals in member accounts of AWS Organizations. Boundaries and SCPs do not grant permissions; they filter what ordinary policies could grant. Region locks and instance-type restrictions are commonly implemented with conditional SCP denies.

The word *maximum* prevents a frequent mistake. Giving a role a boundary that allows `s3:*` does not give it S3 access; an identity policy must still allow the requested S3 operation. Giving the role `AdministratorAccess` also cannot exceed a narrower boundary. In an organisation member account, the applicable SCP forms another ceiling. The effective result is the overlap that survives every relevant ceiling, subject to any explicit deny. The management account is not restricted by SCPs attached to the organisation, but identities there still follow ordinary IAM policy evaluation.

Cross-account access normally requires cooperation on both sides. The principal's account must allow the action through an identity policy, and the resource-owning account must trust or grant that principal through a resource policy or role trust relationship. Assuming a role changes the working identity: subsequent calls use the role session's permissions, its boundary if present, session-policy limits, and applicable organisation policies. This is why inspecting only the original user's policies can miss the real decision context.

CloudFormation adds another identity distinction. Without a service role, it operates using the caller's credentials for that stack operation. When a stack has a CloudFormation service role, CloudFormation assumes that role to create, update, and delete resources, even if the person starting the operation could not perform those service actions directly. The caller therefore needs permission to operate the stack and to pass the approved role, while the service role needs least-privilege permissions for the resources in the template. A failure may belong to either side, so the failed stack event and CloudTrail caller identity are more useful than guessing from the console login.

Service-linked roles are predefined roles owned by AWS services for actions those services need in an account. Their names commonly begin `AWSServiceRoleFor`. They should not be treated as unexplained administrator roles or edited like application roles.

Useful global condition keys include `aws:RequestedRegion`, `aws:SourceIp`, and principal or resource tags. Conditions must be tested against the actual request context; calls through service proxies or VPC endpoints can provide context different from a direct public request.

### 6.8 Simulation, analysis, and audit evidence

The IAM policy simulator predicts a principal's result without executing the operation:

```powershell
aws iam simulate-principal-policy `
  --policy-source-arn "arn:aws:iam::<account-id>:role/<role-name>" `
  --action-names s3:GetObject s3:PutObject `
  --resource-arns "arn:aws:s3:::<bucket-name>/report.txt"
```

Results use the same vocabulary: `allowed`, `implicitDeny`, or `explicitDeny`. IAM Access Analyzer validates policy syntax and findings, identifies resources accessible from outside the account, and can help refine permissions from observed access.

**CloudTrail** records management API activity: who called which service operation, when, from which address, with what outcome. Console actions, CLI commands, SDK calls, and CloudFormation operations all pass through APIs. Event history makes recent management events queryable without first creating a trail:

```powershell
aws cloudtrail lookup-events `
  --lookup-attributes AttributeKey=EventName,AttributeValue=PutBucketPolicy `
  --max-results 10
```

An IAM API denial appears in CloudTrail with an error code and message. A dropped network packet does not: CloudTrail is not a packet log. S3 object-level operations are data events and require explicit data-event logging. Know which evidence source could contain the refusal before searching it.

### 6.9 Systematic network and permission diagnosis

`CREATE_COMPLETE` proves that resource API calls succeeded. It does not prove the network path works or that policies express the intended access. Diagnose from observed state, one layer at a time.

For a web endpoint that times out:

1. Define the failure with `curl.exe -sf --max-time 5` and note timeout versus refusal.
2. Describe the instance or task: running, correct subnet, correct private/public address?
3. Describe every attached security group: is the destination port allowed from the caller?
4. Describe the subnet NACL: are both directions and ephemeral return ports allowed?
5. Describe the route table associated with the subnet: is the expected default route present?
6. Describe the gateway: does it exist and is it attached to the correct VPC?
7. Only after the packet path is established, inspect the service listener and application logs.

```powershell
aws ec2 describe-instances --instance-ids <instance-id> `
  --query "Reservations[].Instances[].{State:State.Name,PublicIp:PublicIpAddress,Subnet:SubnetId,Groups:SecurityGroups}"

aws ec2 describe-security-groups --group-ids <security-group-id>
aws ec2 describe-network-acls `
  --filters "Name=association.subnet-id,Values=<subnet-id>"
aws ec2 describe-route-tables `
  --filters "Name=association.subnet-id,Values=<subnet-id>"
aws ec2 describe-internet-gateways `
  --filters "Name=attachment.vpc-id,Values=<vpc-id>"
```

The template states intent; `describe-*` returns actual state. The gap is where silent networking mistakes live. Reachability Analyzer can inspect configuration between a source and destination and identify a blocking component without sending packets. Use it to confirm and accelerate a diagnosis, while retaining the manual layer model.

Test name resolution separately from transport. If `Resolve-DnsName <hostname>` fails, inspect the name, VPC DNS attributes, resolver path, and record before changing routes or firewalls. If the name resolves but `Test-NetConnection <hostname> -Port 443` fails, retain the resolved address as evidence and continue through the packet path. This separation prevents a DNS symptom from being misdiagnosed as a security-group problem.

For an API `AccessDenied`:

1. Run `aws sts get-caller-identity` and confirm the account and role session.
2. Confirm Region and exact action/resource ARN from the full error.
3. Inspect identity policies, resource policy, role trust where assumption is involved, and any permission boundary.
4. Consider invisible organisation SCPs when the message names an explicit organisational deny.
5. Simulate the request when supported.
6. Query CloudTrail for the actual failed event and compare it with the predicted policy outcome.

A diagnosis becomes durable when encoded as a check with a meaningful exit status. End-to-end checks prove behavior; narrower checks assert individual conditions such as a route to an IGW or a policy simulation result.
