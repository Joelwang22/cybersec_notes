# Cloud Foundations: AWS and EC2

This chapter covers everything from the lecture in depth, plus several topics the lecture skipped: how instance type names are constructed, the EBS volume types, Elastic IP addresses, the instance metadata service, what the free tier actually covers, what keeps charging after you stop an instance, and how to get a shell on a machine without opening a single firewall port.

## Chapter 1.1 What cloud computing is

Strip away the marketing and cloud computing is a rental business. AWS owns warehouses full of servers, storage, and network equipment. You rent slices of it (by the hour, by the gigabyte, by the request) and you do all of it remotely, through an API, with no human on the other end approving anything.

The formal definition adds one word that matters: *on-demand*. You ask for a server and you have one in under a minute. You release it and the charge stops. No purchase order, no delivery window, no minimum term. The speed is the product; nearly everything else about the cloud is a consequence of it.

### Capital expenditure versus operational expenditure

Before the cloud, running a server meant buying one. Buying hardware is *capital expenditure* (CapEx): a large upfront cost for an asset you then own for years, whether your capacity guess was right or not. Guess high and money sits idle in a rack. Guess low and your site falls over on its best day, and the fix is another procurement cycle measured in weeks.

Cloud computing converts that to *operational expenditure* (OpEx): no upfront purchase, a bill at the end of the month for what you actually ran. The trade is worth stating honestly. Renting a server around the clock for three years usually costs more than buying it. What you are paying for is the option to stop, resize, or walk away at any moment. For a steady, predictable, always-on workload, owning can win. For anything uncertain (a new product, a course, an experiment), the flexibility is worth the premium.

### The shared responsibility model

AWS states the security arrangement in one sentence: AWS is responsible for security *of* the cloud; you are responsible for security *in* the cloud.

Security of the cloud is the physical and foundational layer: buildings, power, guards, the physical network, the hardware lifecycle, and the virtualization layer that keeps tenants apart. You will never audit an AWS data center, and you do not need to; that layer is their job and their reputation.

Security in the cloud is everything you configure:

| AWS secures                   | You secure                            |
| ----------------------------- | ------------------------------------- |
| Physical data centers         | Your data                             |
| Host hardware and firmware    | Your operating system and its patches |
| The hypervisor                | Your firewall (security group) rules  |
| The underlying network fabric | Your credentials and who holds them   |
| Managed service internals     | What you make publicly reachable      |

The line between the columns moves per service. On EC2 you manage the operating system, so patching it is yours. On Lambda (covered in the storage topic) AWS manages the operating system and your share shrinks to your code and its permissions. More managed means less yours, never none yours.

Almost every cloud security incident in the news is a failure in the right column: an open storage bucket, a leaked key, a firewall rule allowing the world. The parts were secure; the assembly was not. This course spends most of its security attention on the right column for exactly that reason.

**Key points**

- Cloud computing is on-demand, self-service rental of compute, storage, and networking through an API.
- It converts capital expenditure to operational expenditure; you pay for flexibility, not for cheapness.
- AWS secures the cloud itself; everything you configure (OS, firewall rules, credentials, data exposure) remains your responsibility.
- The managed-ness of a service moves the line, but never moves it all the way.

## Chapter 1.2 Regions and Availability Zones

AWS is not one place. It is more than thirty *regions*, separate geographic areas, each a cluster of data centers: `us-east-1` is Northern Virginia, `ap-southeast-1` is Singapore, `eu-west-1` is Ireland. Regions are independent by design: separate power, separate control systems, so a failure in one cannot cascade into another.

Nearly every resource you create lives in exactly one region. An instance launched in `us-east-1` does not exist in `eu-west-1` in any sense; the two regions might as well be different providers that happen to share your login. Organizations choose regions for latency (near the users), law (some data must stay in-country), price (regions differ), and service availability (new features reach the large regions first).

Inside a region sit *Availability Zones* (AZs): isolated groups of data centers with independent power, cooling, and networking, far enough apart that one flood or grid failure cannot take out two, close enough for single-digit-millisecond links. They are named with a letter suffix: `us-east-1a`, `us-east-1b`, and so on. Production systems that matter are spread across at least two AZs, so that an entire zone can fail (and occasionally one does) without taking the application down. In this course we work in single AZs, because we are learning parts, not designing for availability; but every instance you launch lands in a specific AZ, and it is worth noticing which.

### Why the region you are working in matters

The console shows one region at a time, selected in the top bar. Launch an instance, switch region, and the instance list is empty. Nothing was lost. The instance is running, and billing, in the region you left. "My instance disappeared" is, nine times in ten, "I am looking at the wrong region." The CLI has the same property: a default region in your profile, overridable per command with `--region`.

Your sandbox account removes most of this hazard: an organization guardrail denies every action outside your course region, except global services such as IAM. The corollary is worth remembering throughout this course: if you get an unexplained `AccessDenied` or an inexplicably empty list, check the region before anything else.

**Key points**

- A region is an independent geographic cluster of data centers; resources live in exactly one.
- Availability Zones are isolated failure domains inside a region; real systems span two or more.
- The console and CLI operate on one region at a time; the wrong region looks like missing resources.
- The sandbox is locked to a single course region, named in your welcome handout. All course examples assume you are in it.

## Chapter 1.3 Resource pooling, multi-tenancy, and virtualization

AWS can hand you a server in thirty seconds because the hardware already exists: racked, powered, and waiting in a pool shared by every customer. You are not triggering procurement; you are being allocated a slice of capacity that was already there. Pooling millions of customers' demand smooths it (one customer's spike is noise at that scale), keeps utilization high, and is the economic engine that makes renting a slice cost cents per hour.

Sharing hardware between unrelated customers is called *multi-tenancy*, and the technology that makes it safe is *virtualization*. Each physical host runs a hypervisor: a small, privileged layer that slices the machine into virtual machines, each with virtual CPUs, memory, disk, and network, each convinced it is a whole computer. The hypervisor's core duty is isolation: your neighbor's VM cannot read your memory or disk. That isolation boundary has held up under twenty years of hostile, internet-scale use; it is the load-bearing wall of the entire cloud business.

So what are you actually renting? Not a machine, a *specification*. When you launch a `t3.micro` you are asking the pool for 2 vCPUs and 1 GiB of memory, satisfied on whatever host has room. Stop the instance and start it again and it may materialize on different hardware; you cannot tell and are not meant to care. This shift, from servers as physical objects to servers as requested quantities, is what makes the CloudFormation topic possible, where a text file describes infrastructure and a template creates it.

**Key points**

- Instant provisioning works because capacity is pooled and pre-built; you are allocated, not purchased for.
- Multi-tenancy means sharing physical hosts with strangers; hypervisor isolation is what makes that safe.
- You rent a specification, not a specific box; instances can move between hosts across stop/start.

## Chapter 1.4 Elasticity and measured service

*Elasticity* means capacity can follow demand in both directions. Ten servers during the spike, two afterward, and the eight you released stop costing anything the moment they are gone. The shrinking half is the novel half; buying more hardware was always possible, giving it back was not. One honest caveat: elasticity is a property you build, not one you get. Nothing scales itself unless you set up the machinery (auto scaling exists; it is beyond this course). What the cloud provides is the possibility: an API that can add and remove capacity in seconds, which means automation can too.

*Measured service* means everything is metered, at fine grain: instances per second, storage per GB-month, requests per thousand, transfer per byte. The bill is the sum of the meters. This cuts both ways. Small experiments are nearly free: the server you build in the exercises costs pennies per hour. But the meter does not care about your attention:

> Nothing stops charging because you stopped looking at it.

A resource exists until something deletes it, and it bills until it stops existing. The meter does not know the demo ended, or that it is the weekend. Every experienced cloud engineer has found a forgotten resource from months ago; the skill that prevents it is a habit, not knowledge: when you finish, tear down, then *verify* the teardown by looking at the account rather than trusting your memory.

The distinction that makes bills predictable is between two kinds of charging:

| Billed for existing                         | Billed per use      |
| ------------------------------------------- | ------------------- |
| Running instances                           | S3 requests         |
| EBS volumes (running, stopped, or orphaned) | Lambda invocations  |
| Public IPv4 addresses                       | Data transfer out   |
| NAT gateways (the VPC and IAM topic)        | API calls generally |

For every resource you meet in this course, ask which column it belongs to. The dangerous leftovers are always in the left one. Chapter 1.10 returns to the left column for the exercise build.

**Key points**

- Elasticity works in both directions; releasing capacity is the half that did not exist before the cloud.
- Billing is metered per second, per GB, per request; the bill is the sum of the meters.
- Resources bill from the moment they exist until the moment they are deleted, attended or not.
- Distinguish billed-for-existing from billed-per-use; leftovers in the first category are the classic cost mistake.

## Chapter 1.5 Deployment models

Three vocabulary terms you will meet in every cloud conversation:

- **Public cloud**: the model this whole chapter has described. The provider owns the hardware, many tenants share it, anyone can be a customer. AWS, Azure, Google Cloud.
- **Private cloud**: the same self-service, pooled, API-driven model built on hardware a single organization owns, on-premises or hosted. You keep control and data locality; you give back elasticity at the edges (the pool is only as large as what you bought) and you take on all the operations.
- **Hybrid**: both, connected, usually by a private network link between your data center and a provider. Common in practice because migrations take years and some systems never move: the mainframe stays, new services go to the cloud, and packets flow between them.

This course is public cloud on AWS throughout. The concepts (pooling, elasticity, measured service, shared responsibility) transfer to all three models.

**Key points**

- Public: provider-owned, shared, rented. Private: the cloud model on your own hardware. Hybrid: both, connected.
- Hybrid is common because real migrations are slow and partial, not because it is anyone's ideal end state.

## Chapter 1.6 Compute: instances, images, and volumes

EC2 (Elastic Compute Cloud) is AWS's virtual machine service. One rented VM is an *instance*. Of everything in AWS it is the least new thing: a computer with CPU, memory, disk, and a network interface, running an operating system you chose, reached over the network. The vocabulary of its lifecycle matters, because AWS chose the words to prevent a specific mistake:

- **Launched**: created and booted.
- **Stopped**: shut down. The instance and its disk still exist; compute billing pauses; it can be started again.
- **Terminated**: destroyed. There is no trash can and no undo.

### How instance type names are constructed

You do not configure CPU and memory with sliders. You pick a *type* from a catalog, and the type's name is a compact spec sheet. Take `t3.micro` and `m7g.large`:

```text
 t        3         .  micro
 m        7    g    .  large
 │        │    │       │
 family   gen  attrs   size
```

- **Family** (first letter): what the type is optimized for. `t` is burstable general purpose, `m` general purpose, `c` compute optimized (more CPU per GB), `r` memory optimized (more GB per CPU). Others exist (storage-dense, GPU); recognize the pattern and look up details when you need them.
- **Generation** (the digit): the hardware iteration. Higher is newer, and newer is almost always both faster and cheaper per unit; there is rarely a reason to pick `t2` over `t3` nowadays.
- **Attributes** (letters after the digit): variants. `g` means a Graviton processor: AWS's own ARM-based CPU, cheaper per unit of work, but ARM binaries only. `a` means AMD. No letter means Intel.
- **Size** (after the dot): shirt sizing (`micro`, `small`, `medium`, `large`, `xlarge`, `2xlarge`, and up). Each step roughly doubles vCPUs and memory, and roughly doubles the price.

So `m7g.large` reads: general purpose, 7th generation, Graviton, 2 vCPU / 8 GiB. And `t3.micro` (the course workhorse) reads: burstable, 3rd generation, Intel, 2 vCPU / 1 GiB, at a rate that amounts to pennies per hour.

"Burstable" is a real mechanism, not marketing. `t`-family instances have a modest baseline CPU allowance and accumulate credits while idle, which they spend to run at full speed in bursts. For a web server that is mostly idle (the exercise workload exactly), that is ideal and cheap. Run one flat out for hours and the credits run dry and it slows to baseline. That is the trade, and it is why `t` types undercut `m` types of similar size.

Your sandbox permits only `t2`, `t3`, `t3a`, and `t4g` types, nothing `xlarge` or larger. Requests outside that list fail with an explicit denial from the guardrail policy, not an error in your command.

### Machine images

An instance boots from an *AMI*, an Amazon Machine Image: an operating system plus anything baked in, from which the instance's root disk is created at launch. AWS publishes official images (Amazon Linux, Ubuntu, Windows Server), vendors publish theirs, and you can snapshot a configured instance into an AMI of your own and launch clones, which is how fleets are built, though in this course we install everything by hand so you see each step.

This course uses **Amazon Linux 2023** (AL2023), AWS's own RHEL-family distribution. Two things to know before touching it:

- The package manager is `dnf`. Tutorials saying `yum` are describing older Amazon Linux; on AL2023 the command is `dnf`.
- The web server we install is Apache; both the package and the service are named `httpd`.

A concrete AMI has an ID like `ami-0abcdef1234567890`. AMI IDs are region-specific and constantly superseded as AWS publishes patched images, so an ID pasted from a blog post is stale or foreign almost by definition. The clean approach is to resolve the current ID at launch time from a public pointer AWS maintains. The one this course uses everywhere:

```bash
aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64 \
  --query 'Parameter.Value' --output text
```

which returns something of the form:

```text
ami-0abcdef1234567890
```

That is the current AL2023 image for your region. The console's quick start list does the equivalent for you; the parameter matters most for the CLI and for the templates in the CloudFormation topic. House rule: never paste a bare `ami-` ID into anything.

### Attached storage: EBS volumes

The instance's disk is an *EBS volume* (Elastic Block Store). It mounts and behaves like a local drive but is actually a replicated, network-attached device, which is what lets an instance stop on one physical host and start on another with its data intact.

The lifecycle point matters more than the technology: a volume is a *separate resource* from the instance. It exists, therefore it bills: running, stopped, attached, or orphaned. When you terminate an instance, the root volume is deleted with it by default, but any additionally attached volumes are *not*: they stay behind, billing, invisible unless you look at the volumes list (in the EC2 console, under Elastic Block Store, or `aws ec2 describe-volumes`).

Volume types, and when each fits:

| Type    | What it is                                                               | Fits                                                          | Price posture                   |
| ------- | ------------------------------------------------------------------------ | ------------------------------------------------------------- | ------------------------------- |
| `gp3` | General purpose SSD; 3,000 IOPS and 125 MB/s baseline regardless of size | Almost everything; the default                                | Modest; billed per GB-month     |
| `io2` | Provisioned-IOPS SSD; you pay for guaranteed IOPS                        | Databases with hard latency/IOPS requirements                 | Expensive; per-GB plus per-IOPS |
| `st1` | Throughput-optimized HDD                                                 | Large, sequential workloads: log processing, big file streams | Cheaper per GB than SSD         |
| `sc1` | Cold HDD                                                                 | Rarely accessed bulk data that must stay on a block device    | Cheapest per GB                 |

You will use `gp3` for everything in this course; the others exist so that when a real workload has real I/O requirements you know the knobs exist. (You may also meet `gp2`, the previous-generation default, on older systems: its performance scaled with size, which forced people to buy space they did not need to get speed. `gp3` decoupled the two.) The sandbox caps volumes at 100 GB, far more than the course needs; the AL2023 default root volume is 8 GB, which at gp3 rates costs pennies per month.

**Key points**

- An instance is a rented VM; *stopped* is reversible, *terminated* is gone.
- Type names decode as family, generation, attributes, size: `t3.micro`, `m7g.large`. Newer generations are faster and cheaper.
- `t`-family instances are burstable: cheap for idle-mostly workloads, throttled under sustained load.
- Resolve AMIs from the SSM public parameter at launch; never hardcode an `ami-` ID.
- EBS volumes are separate resources that bill for existing; non-root volumes survive instance termination.
- `gp3` is the default volume answer; `io2`, `st1`, `sc1` exist for specific I/O and cost profiles.

## Chapter 1.7 Addresses: private, public, and Elastic

Every instance gets a *private IP address* from its subnet's range; the sandbox's default network hands out addresses in `172.31.0.0/16`, squarely in the RFC 1918 space you know. The private address is stable for the instance's entire life, through stops and starts, and is reachable only from inside the same private network. For most machines in a real system (databases, workers, caches), that is the correct and only address. Internet exposure is the exception you grant deliberately, not the default.

To be reachable from a browser, an instance also needs a *public IPv4 address*. The usual route is auto-assignment at launch: AWS lends the instance an address from its pool. Two properties of that loan surprise people:

1. **It does not survive a stop.** Stop the instance and the address goes back to the pool; start it again and you get a different one. Anything that pointed at the old address (bookmarks, DNS, scripts, your reachability check) now points at a stranger.
2. **It costs money.** Since February 2024, every public IPv4 address bills by the hour whether or not it is used. IPv4 addresses are genuinely scarce now, and AWS priced them accordingly.

When an address must survive stops, you allocate an *Elastic IP* (EIP): a public IPv4 address reserved to your account, which you associate with an instance and re-associate at will. It is yours until you explicitly release it, which is precisely its hazard. An EIP left allocated after its instance is terminated keeps billing at the same hourly rate: a fee for a number nobody is using. It is the single most common leftover in teardown exercises and in real accounts, and "release the Elastic IP" is a standard line in every teardown checklist in this course. Left allocated and forgotten, that small hourly fee quietly compounds into real money.

Assigned versus reserved, in one table:

|                      | Auto-assigned public IP               | Elastic IP                           |
| -------------------- | ------------------------------------- | ------------------------------------ |
| Where it comes from  | Borrowed from AWS pool at launch      | Allocated to your account on request |
| Survives stop/start  | No, replaced with a new one           | Yes, stays associated                |
| Survives termination | Gone with the instance                | Stays allocated to you, now idle     |
| Cost                 | Billed hourly while the instance runs | Billed hourly, associated or not     |
| Cleanup step         | None                                  | Must be explicitly released          |

**Key points**

- Every instance has a stable private IP; a public IP is optional and deliberate.
- Auto-assigned public IPs change across stop/start; Elastic IPs persist until released.
- All public IPv4 addresses bill hourly since February 2024, used or idle.
- An unreleased Elastic IP is the classic billing leftover; releasing it is part of every teardown.

## Chapter 1.8 Security groups

Between the internet and your instance stands a *security group*: a firewall attached to the instance itself. Not a subnet ACL, not `iptables` inside the OS: enforcement happens at the virtualization layer, outside anything running on the machine. Rules specify protocol, port, and source (a CIDR block or another security group).

Three properties define how they behave:

- **Deny by default, inbound.** A fresh instance answers nothing (not ping, not SSH, not HTTP) until a rule explicitly allows it. AWS's answer to "nobody said anything about this traffic" is always no. You will meet the same posture again in the IAM topic.
- **Stateful.** Allow a connection in and its reply traffic is permitted out automatically; no matching outbound rule is needed. If you learned stateless packet filters, where both directions are written explicitly, consciously drop that habit here. The default outbound rule allows everything, and for this course that is fine; inbound rules are the whole game.
- **Allow rules only.** A security group cannot express "block this address"; the only deny is the absence of an allow. (Explicit deny rules live in network ACLs, which are out of scope for this course.)

One group can be attached to many instances, so "web server rules" is defined once and reused.

Here is the complete rule set for the web server you build in the exercises, and the reasoning:

| Direction | Type        | Protocol | Port | Source           | Why                               |
| --------- | ----------- | -------- | ---- | ---------------- | --------------------------------- |
| Inbound   | HTTP        | TCP      | 80   | `0.0.0.0/0`    | A public page is meant for anyone |
| Inbound   | SSH         | TCP      | 22   | `<your-ip>/32` | Administration, from you alone    |
| Outbound  | All traffic | All      | All  | `0.0.0.0/0`    | Default; updates and replies      |

Note the SSH source. Port 22 open to `0.0.0.0/0` works, and every internet-facing SSH port is under continuous automated attack within minutes of existing. "Works" and "defensible" are different standards; `/32` meets both. Chapter 1.9 shows a way to skip the SSH rule entirely.

The diagnostic signature to memorize: a page that *hangs and times out* is a firewall problem: the security group is silently dropping packets. A page that fails *instantly* ("connection refused") reached the instance and found nothing listening; that is a web server problem. The two symptoms point at different layers, and telling them apart saves you from debugging Apache when the packet never arrived.

**Key points**

- A security group is an instance-attached firewall enforced outside the OS.
- Inbound is deny-by-default; only explicit allows pass traffic.
- Stateful: reply traffic is automatic; you write inbound rules and usually leave outbound alone.
- Open 80 to the world for a public page; open 22 to your own address only, or not at all.
- Timeout means firewall; connection refused means nothing listening.

## Chapter 1.9 Getting in: key pairs, Session Manager, and the metadata service

### Key pairs and what the private key is for

SSH to an EC2 instance does not use passwords. At launch you name a *key pair*; AWS takes the pair's public half and places it into `~/.ssh/authorized_keys` for the default user (`ec2-user` on Amazon Linux) during first boot. The private half (the `.pem` file) exists only where you keep it. AWS does not have a copy, cannot recover it, and cannot log in to your instance. That is the design: proof of identity is possession of the private key, and only you possess it.

Practicalities that follow from the design:

- Lose the `.pem` and there is no reset button. (There are recovery paths involving detaching volumes; the honest summary is "do not lose it.")
- SSH refuses to use a key file that other users on your machine can read. On Linux and macOS: `chmod 400 .pem` once, before first use.
- The connection is then `ssh -i .pem ec2-user@`.

Your sandbox account came with a key pair already created, named `-key`; the `.pem` was issued in your welcome handout. Select that key pair at launch rather than creating new ones.

### Connecting without opening a port: Session Manager

There is a second way to get a shell, and it requires no key file, no SSH client configuration, and (the interesting part) *no open inbound port at all*. AWS Systems Manager **Session Manager** works inside-out: an agent on the instance (preinstalled on AL2023) opens an outbound HTTPS connection to the Systems Manager service, and your terminal session travels down that connection. Since the instance dials out and nothing dials in, the security group needs no inbound rule for it; port 22 can stay closed to the entire internet.

The requirements, all three of which matter when it does not work:

1. The SSM agent is running (it is, on AL2023 AMIs).
2. The instance has an IAM role attached that permits Systems Manager to manage it (the standard managed policy is `AmazonSSMManagedInstanceCore`; roles get their full treatment in the IAM topic).
3. The instance can reach the Systems Manager service outbound over HTTPS.

Then either use the Connect option on the instance in the EC2 console and choose Session Manager, or from a terminal:

```bash
aws ssm start-session --target i-0abc12345def67890
```

(The CLI route additionally needs the Session Manager plugin installed locally; the console route needs nothing.)

In real environments this is increasingly the default: no distributed key files, no internet-facing port 22, and every session is logged with who and when. For this course, SSH with your key remains the primary route; you should know both.

### The instance metadata service

From inside any EC2 instance, the address `169.254.169.254` answers. It is not on any network you built; it is the *instance metadata service* (IMDS), a link-local endpoint the platform provides so software on the instance can ask questions about the instance itself: What is my instance ID? My type? My public IP? What credentials do I hold?

That last question is why the service matters. When an instance has an IAM role (as in the Session Manager setup above), the role's temporary credentials are fetched from the metadata service. Any process on the instance that can make an HTTP request can retrieve them. That made IMDS version 1 (a plain, unauthenticated GET) a famous attack path: trick a server-side application into fetching a URL of your choice (a server-side request forgery), point it at `169.254.169.254`, and it hands you the instance's credentials. Real, headline-grade breaches worked exactly this way.

IMDSv2 closes that path by requiring a session token obtained with a PUT request first, something a typical request-forgery primitive cannot do. On AL2023 AMIs, **IMDSv2 is enforced by default**: plain GETs are refused with `401 Unauthorized`. The two-step, from inside an instance:

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

```text
i-0abc12345def67890
```

Browse `http://169.254.169.254/latest/meta-data/` (with the token header) and you get a directory of everything knowable from inside: `instance-type`, `local-ipv4`, `public-ipv4`, `placement/availability-zone`, and more. Scripts on the instance use this constantly; the exercises include a one-liner that writes the instance's own ID into the web page it serves.

**Key points**

- The private key proves identity; AWS keeps only the public half and cannot log in or recover a lost key.
- `chmod 400` the `.pem`, connect as `ec2-user` on Amazon Linux; your sandbox key is named `-key`.
- Session Manager gives a shell over an outbound connection: no inbound port, no key file; needs the agent, a role, and outbound HTTPS.
- The metadata service at `169.254.169.254` tells an instance about itself, including its role credentials.
- IMDSv2 requires a token first and is enforced by default on AL2023; plain GETs get `401`.

## Chapter 1.10 The console, the API, and who you are

### Everything is an API call

The AWS Management Console (the website) is not AWS. It is one client of the AWS API, the same HTTPS API anyone can call. Click "Launch instance" and your browser sends a `RunInstances` API call; every screen and button resolves to API calls underneath. Three consequences:

- **Anything clickable is scriptable.** The AWS CLI calls the same API from a terminal; Python calls it from code (the storage topic); CloudFormation drives it from templates (the CloudFormation topic). Skills transfer because the API is the single real interface.
- **Clicks do not repeat; commands do.** A 40-step console procedure done twice produces two subtly different results. The industry moved to commands and code for repeatability, not fashion.
- **Everything is auditable.** Every call (clicked or typed) can be logged with who made it, when, and from where (the service that does this is called CloudTrail). "Prove it, don't assert it" works because the API is the single point everything passes through.

### Identity Center, and credentials that expire

You reach AWS through IAM Identity Center, AWS's single sign-on. You sign in at the portal URL from your welcome handout, and the portal lists the account you may enter and the role you hold there: `StudentAdmin`. Clicking through opens the console with that role's permissions. `StudentAdmin` is administrative *within* your account, but organization guardrails cap it: your course region only, small instance types only, a list of expensive services denied outright. A guardrail refusal arrives as an explicit `AccessDenied`. Read it; it names what was denied.

Identity Center issues *temporary* credentials. A session lasts a few hours and then everything stops at once: the console bounces you to the sign-in page, and the CLI starts failing with errors mentioning expired tokens or missing credentials. This is deliberate. A leaked password is a permanent problem; a leaked session token is a problem with a countdown. The reflex to build now: when commands that worked earlier start failing with anything credential-shaped, sign in again before debugging anything else.

For the CLI, `aws configure sso` sets up a profile from your Identity Center session, and `aws sso login` renews an expired one. You will need both during the reconnaissance exercise, "Who Are These People?".

### Worked example: who am I?

The first command to run when anything fails with a credentials or access error, because it answers "who does AWS think I am right now":

```bash
aws sts get-caller-identity
```

```json
{
    "UserId": "AROA4EXAMPLE7EXAMPLE:rony",
    "Account": "111122223333",
    "Arn": "arn:aws:sts::111122223333:assumed-role/AWSReservedSSO_StudentAdmin_0123456789abcdef/rony"
}
```

Read the ARN. It says `assumed-role`, not `user`: you are operating under temporary credentials for a role, which is what Identity Center sign-in looks like from the inside. The `AWSReservedSSO_StudentAdmin_...` fragment is the role Identity Center created in your account for your permission set, and `Account` is your twelve-digit account ID. When this command fails, nothing else will work either; when it succeeds, credential problems are off the suspect list.

### Worked example: what does an instance look like?

Once your instance exists, its details (in the console's instance page or from the CLI) are a checklist of every concept in this chapter. Trimmed to the fields that matter, via the CLI:

```bash
aws ec2 describe-instances \
  --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,State:State.Name,AZ:Placement.AvailabilityZone,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress,AMI:ImageId,KeyName:KeyName}'
```

```json
[
    {
        "ID": "i-0abc12345def67890",
        "Type": "t3.micro",
        "State": "running",
        "AZ": "<region>a",
        "PrivateIP": "172.31.24.17",
        "PublicIP": "54.234.18.201",
        "AMI": "ami-0abcdef1234567890",
        "KeyName": "rony-key"
    }
]
```

Every field should now mean something: the type you chose from the catalog, the AZ the pool placed you in, the stable private address, the borrowed public one, the image it booted from, the key pair whose public half landed in `authorized_keys`. Note that `describe-instances` with no filters also returns stopped and recently terminated instances, which is why raw output is longer than people expect; `--query` and `--filters` are how you trim it, and learning that CLI output is JSON you can filter is worth more than any single command.

**Key points**

- The console is one client of the API; everything you click is an API call, therefore scriptable and auditable.
- Sign-in is via the Identity Center portal URL; you hold the `StudentAdmin` role, capped by organization guardrails.
- Credentials expire after a few hours; re-authenticate before debugging "sudden" failures.
- `aws sts get-caller-identity` is the universal first move on any credential or access error.

## Chapter 1.11 The bill: free tier, stopped instances, and where to look

### Free tier reality

Nearly every AWS tutorial online says some version of "this is free tier eligible." Treat that phrase with suspicion, for three reasons.

First, the free tier has changed shape. Accounts created before mid-2025 got the classic offer: 12 months of allowances (750 hours per month of a micro instance, some storage, and so on) from account creation. Accounts created after that get a credit-based plan instead (a limited pool of credits with an expiry) plus a set of always-free allowances on specific services (Lambda's monthly request allowance, which matters in the storage topic, is one). Tutorials written for the old model describe an offer your account may not have.

Second, and decisively for this course: your sandbox account is a member of an organization, and free tier benefits do not apply per-account the way tutorials assume: for the classic offer they were shared across the entire organization, once, not once per account. The working assumption for this course is that **the free tier covers nothing** and every resource bills at list price. That assumption keeps the reasoning honest and builds the right habit; anything the free tier does absorb is a rounding error in your favor.

Third, "free tier eligible" never meant "free." It meant "this specific resource draws from an allowance while the allowance lasts"; the public IPv4 next to it and the volume under it may be billing regardless.

### What the exercise build costs

The exercise build runs three meters at once, one per resource:

| Resource              | What the meter counts            |
| --------------------- | -------------------------------- |
| `t3.micro`, running | Instance hours                   |
| Public IPv4 address   | Hours the address exists         |
| 8 GB gp3 root volume  | GB-months of provisioned storage |

While you work, the three together amount to pennies. That is exactly the trap: each meter keeps counting until its resource is deleted, and a forgotten resource turns a rounding error into real money through nothing but neglect. The habit you are practicing on a build that costs almost nothing is the one that prevents four- and five-figure surprises at production scale, where the forgotten resource is a NAT gateway or a large database rather than a micro instance. When you want actual current rates, put the build into the AWS Pricing Calculator at calculator.aws; prices change, the lesson does not.

### Charges that continue after an instance is stopped

Stopping an instance stops the *compute* meter, and only the compute meter. Still billing while your instance sits stopped:

- **The EBS volume.** It exists, so it bills: per GB-month, indefinitely.
- **An Elastic IP associated with the stopped instance**: its hourly meter keeps running. (An *auto-assigned* public IP is released on stop and does stop billing; that is the flip side of it not surviving the stop.)
- **Snapshots** of volumes, if you made any, at their own per-GB rate.

"Stopped" is therefore a low-cost state, not a free one. The only state with no ongoing charge is *terminated, with volumes deleted and addresses released*, and the difference between believing that has happened and verifying it is precisely the final exercise in this topic.

### Where to see current spend

Two places, both reached from the account menu in the console's top bar:

- **The Billing and Cost Management console**, the authoritative view: current month-to-date charges, broken down by service, plus past invoices. When you want the real number, it is here.
- **Cost Explorer**, the analytical view: spend graphed over time, filterable by service, region, or tag. This is where "what is actually costing me money" gets answered in practice, and it is the subject of your first exercise.

One warning that catches everyone: **cost data is not real time.** Charges typically take up to a day to appear. The instance you launch in the exercises will likely not show up in the numbers until the next day. Absence of a charge in Cost Explorer an hour after launch proves nothing, which is why teardown verification means looking at the *resources* (instance list, volume list, address list), never the bill.

**Key points**

- Assume the free tier covers nothing in this course; treat every resource as billing at list price.
- Stopping an instance stops only the compute charge; volumes and Elastic IPs keep billing.
- Only "terminated, volumes gone, addresses released" costs nothing.
- Real spend lives in the Billing console; analysis lives in Cost Explorer; both lag by up to a day.
- Verify teardown by listing resources, not by watching the bill.

## Command reference

| Command                                                                                                    | What it does                                                 |
| ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `aws --version`                                                                                          | Confirm the CLI is installed                                 |
| `aws configure sso`                                                                                      | Create a CLI profile from an Identity Center sign-in         |
| `aws sso login`                                                                                          | Renew an expired Identity Center session                     |
| `aws sts get-caller-identity`                                                                            | Show which identity and account you are operating as         |
| `aws ec2 describe-instances`                                                                             | List instances (all states) with full details                |
| `aws ec2 describe-volumes`                                                                               | List EBS volumes, the teardown leftover check                |
| `aws ec2 describe-addresses`                                                                             | List Elastic IPs, the other leftover check                   |
| `aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64`         | Resolve the current AL2023 AMI ID                            |
| `aws ssm start-session --target <instance-id>`                                                           | Open a shell with no inbound port (needs the session plugin) |
| `ssh -i <your-key>.pem ec2-user@<public-ip>`                                                             | SSH to an Amazon Linux instance                              |
| `chmod 400 <your-key>.pem`                                                                               | Make the private key usable by SSH (Linux/macOS)             |
| `sudo dnf install -y httpd`                                                                              | Install Apache on AL2023                                     |
| `sudo systemctl enable --now httpd`                                                                      | Start Apache and keep it starting on boot                    |
| `curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"` | Get an IMDSv2 token (from inside the instance)               |

## Further reading

All of these are official AWS documentation; find them by name.

- *Amazon EC2 User Guide*: "Instance types" (the full catalog and naming scheme) and "Instance lifecycle" (stop versus terminate, precisely).
- *Amazon EBS User Guide*: "Amazon EBS volume types" (gp3, io2, st1, sc1 compared with numbers).
- *Amazon EC2 User Guide*: "Instance metadata and user data" (IMDS and IMDSv2 in full) and "Amazon EC2 key pairs and Amazon EC2 instances".
- *Amazon VPC User Guide*: "Control traffic to your AWS resources using security groups".
- *AWS Systems Manager User Guide*: "Session Manager" (setup, prerequisites, and logging).
- *AWS Billing User Guide*: "Viewing your monthly charges"; *AWS Cost Management User Guide*: "Analyzing your costs with Cost Explorer".
- *AWS CLI User Guide*: "Configuring the AWS CLI to use IAM Identity Center".
- *Amazon EC2 shared responsibility model*: in the security section of the EC2 User Guide; the general model is on the AWS "Shared Responsibility Model" page.
