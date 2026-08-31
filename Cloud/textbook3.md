# Cloud Foundations: ECR and Fargate

The S3 Storage topic served the page with no server; this topic brings the server back, in a box. Your prerequisites covered building and running container images locally. What they could not cover is the cloud half: how an image you built ends up running on compute you never see, reachable at a public address, without a machine of yours anywhere. That takes a registry the cloud can pull from (ECR), a form that states everything the platform cannot guess (the task definition), and compute that appears on demand (Fargate).

## Chapter 3.1 Containers: images, registries, and ECR

Your prerequisites covered containers locally; this topic assumes you can build an image and adds one question: how does an image you built end up running in a cloud that has never seen your laptop?

First, the vocabulary made precise. An **image** is a frozen filesystem plus metadata including a start command: built once from a `Dockerfile`, immutable thereafter. A **container** is an image running. The build is where dependencies are installed, so the environment travels with the code because the environment *is* the artifact. That is most of the argument for containers. (The Lambda Functions topic shows the painful alternative: shipping a dependency to a runtime you do not control, with no build step to carry it.)

Here is the complete image the container exercise has you build (your page, served by the same Apache, in a box):

```dockerfile
FROM httpd:2.4-alpine
COPY index.html /usr/local/apache2/htdocs/index.html
EXPOSE 80
```

Three lines, each earning its place. `FROM` starts from a public base image that already contains Apache on a minimal Linux. Note this is `httpd`, the same server you installed with `dnf` in the EC2 exercises, now arriving as a layer of filesystem instead of a package. `COPY` bakes your page into the image at the path Apache serves from. `EXPOSE 80` documents which port the process listens on (documentation only); actually reaching that port is decided at run time, by the task definition and networking in the next chapter. Build and test locally with `docker build -t static-page .` and `docker run -p 8080:80 static-page`, then browse to localhost:8080. Prove it works on your machine before involving the cloud; it splits the debugging space in half.

### Registries and ECR

For AWS to run this image, AWS must be able to pull it, which means it must live in a **registry** the service can reach; a local image on your laptop might as well not exist. Docker Hub is the public registry your base image came from. **ECR** (Elastic Container Registry) is the private registry inside your AWS account: you create a **repository** per image name, and it stores your pushed, tagged versions.

Getting an image there is three commands, and the first is the one with a lesson in it:

```bash
aws ecr get-login-password \
  | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com

docker tag static-page:latest <account-id>.dkr.ecr.<region>.amazonaws.com/static-page:latest
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/static-page:latest
```

ECR is private, so docker must authenticate: `get-login-password` mints a temporary registry password from your AWS credentials and the pipe hands it straight to `docker login` without it landing in your shell history or clipboard. The token lasts about twelve hours. When a push fails with an authentication error hours after an earlier one worked, log in again before suspecting anything deeper; and remember your SSO session itself expires on its own schedule underneath. The `docker tag` line is doing real work, not ceremony: an image's full name encodes its registry, so tagging with your ECR hostname is what aims `docker push` at your account instead of Docker Hub. And notice the shape of the whole first line: an AWS API call feeding a non-AWS tool. Everything is an API call, including obtaining a password.

### Image scanning

A registry that holds your images can also inspect them, and ECR does: **image scanning** compares the packages inside an image against databases of known vulnerabilities (CVEs) and attaches findings to the image (visible in the console per image, with severities). Basic scanning can run automatically on every push; an enhanced mode (via Amazon Inspector, continuous rather than push-time, covering OS and language packages) exists beyond it.

The finding worth expecting: a scan of the three-line image above will likely report vulnerabilities *you did not write*, inherited from the base image's OS packages. That is the flip side of `FROM`: you imported a hundred packages in one line, and their CVEs came along. The working remedy is rebuilding regularly on updated base images: on Fargate, nobody patches your image but you.

**Key points**

- An image freezes filesystem plus start command at build time; dependencies are installed then, so the environment travels with the code.
- `EXPOSE` documents a port; run-time configuration actually publishes it.
- Cloud services pull from registries; ECR is your account's private one, one repository per image name.
- ECR auth is `aws ecr get-login-password | docker login`; tokens last ~12 hours. The registry hostname in the tag aims the push.
- Scanning reports known CVEs per image; most are inherited from the base image, fixed by rebuilding on updated bases.

## Chapter 3.2 Running containers on Fargate

**ECS** (Elastic Container Service) is the orchestrator: it takes descriptions of containers and keeps running copies of them. Its resources, quickly: a **cluster** is a logical grouping (the default one suffices for this course); a **task** is a running copy of a task definition; a **service** is standing instruction to keep N tasks running, replacing any that die. The Fargate exercise runs a bare task; services reappear in the failure-modes discussion.

### The task definition: what the platform must be told

On your laptop, `docker run static-page` works almost bare because the machine supplies every default: its own CPU, its own network, your logged-in registry session. In the cloud there is no machine standing by with defaults: everything must be stated, and the statement is a **task definition**: a registered, versioned document that is to a running task roughly what a Dockerfile is to an image. Instructions, not a running thing.

The essential fields, as a complete registrable document:

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
      "image": "<account-id>.dkr.ecr.<region>.amazonaws.com/static-page:latest",
      "portMappings": [{ "containerPort": 80, "protocol": "tcp" }],
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

Why each field is non-negotiable. The **image**, by full registry path: there is no local cache to fall back on; this string is why Chapter 3.1 happened. **cpu and memory**: someone must reserve real hardware, and Fargate will not guess; the values are from a fixed menu, below. The **port mapping**: so the platform knows where the process listens. The **execution role**: because pulling a private image from ECR and writing logs to CloudWatch are AWS API calls made on your task's behalf, and by now you know no API call succeeds without a granted permission. The failure mode to know: a task dying immediately with a pull error has a role or networking problem, not an image problem. ECS also has a second, separate role (the *task role*) for what your application code itself may call: platform plumbing and code permissions are two separate questions, a split the Lambda Functions topic meets again as the function's execution role. Our Apache calls no AWS APIs, so it needs no task role. The log configuration sends the container's stdout to CloudWatch Logs, which is the only place output goes when there is no machine to SSH into.

### Fargate, and the launch-type choice

An orchestrator must answer: running on *what*? ECS has two answers, called launch types. The **EC2 launch type**: a fleet of container-hosting EC2 instances that you own; you size the fleet, patch its OS, pay for it whether full or empty, and ECS packs tasks onto it. The **Fargate launch type**: you submit a task, AWS finds hardware, and no instance ever appears in your account.

The trade: Fargate removes the fleet (no capacity planning, no OS patching, no idle instances, per-task isolation) and charges a premium per unit of compute for it. The EC2 launch type earns its keep when a fleet would run hot and steady anyway, or when tasks need what Fargate cannot give (GPUs, specific instance hardware, daemons on every host). For a course, for spiky workloads, and for most teams' first production services, Fargate is the honest default. (Kubernetes-shaped orchestration exists as EKS; it is denied by the sandbox SCPs and out of scope.)

Fargate pricing: per vCPU-hour plus per GB-hour, metered per second, from image pull to task stop: *while the task runs, traffic or not*. In the course's cost language: billed for existing, not per use. A Fargate task is a server in every economic sense; the Lambda Functions topic introduces the one compute in this course whose idle is genuinely free.

### Sizing, and what a quarter vCPU means

The cpu and memory values come from a fixed menu of valid pairs. The smallest tiers:

| cpu (units) | vCPU | valid memory |
| --- | --- | --- |
| 256 | 0.25 | 512 MB, 1 GB, 2 GB |
| 512 | 0.5 | 1-4 GB |
| 1024 | 1 | 2-8 GB |

CPU is counted in units of 1/1024 of a vCPU, so `"cpu": "256"` is a quarter of one vCPU, meaning the task is entitled to a quarter of a vCPU's time, continuously, as a hard allocation. Contrast this with the t3.micro from the EC2 exercises, which advertises two vCPUs but is *burstable*: a baseline share plus credits for sprints. The Fargate quarter-vCPU is smaller on paper and steadier in fact: no credits to exhaust, no throttling surprise after a busy hour. For Apache serving one static page, a quarter vCPU and 512 MB is generous. Price the smallest task in the AWS Pricing Calculator (calculator.aws) and it lands in the same neighborhood as the t3.micro. Fargate's value is never raw price per CPU; it is the fleet you do not run.

### It still runs somewhere, and still needs an address

"Serverless" containers obey networking. In `awsvpc` mode (required on Fargate), every task receives its own network interface in your VPC, and at `run-task` time you must supply the coordinates:

```bash
aws ecs run-task \
  --cluster default \
  --launch-type FARGATE \
  --task-definition static-page \
  --network-configuration "awsvpcConfiguration={subnets=[<subnet-id>],securityGroups=[<sg-id>],assignPublicIp=ENABLED}"
```

A subnet, a security group, and an explicit yes on a public IP. Omit the public IP in a public subnet and two things go wrong at once: nobody can reach the task, and the task may not be able to reach ECR to pull its image in the first place. A task that dies at startup with a pull error is as often a networking problem as a permissions one.

When the page does not load, the diagnosis is the EC2 topic's, verbatim, because nothing about containers changed the physics of reaching a socket: does the task have a public IP; does the security group allow port 80 from you; is the process listening? Outside-in, stop at the first failure. And one familiar cost note: a public IPv4 address on a Fargate task bills by the hour exactly as it did on the EC2 instance. Addresses bill for existing, whatever compute sits behind them.

**Key points**

- A task definition states what no host is there to default: image path, cpu/memory from the valid-pairs menu, port, execution role.
- Execution role (pull image, write logs) versus task role (what your code may call): two separate permission questions, and the Lambda Functions topic meets the same split.
- Fargate removes the fleet and bills per task-second while running: billed for existing, not per use.
- `"cpu": "256"` is a hard quarter-vCPU allocation; contrast the t3.micro's burstable credits.
- Every task gets an ENI in a subnet with a security group; no public IP means unreachable and possibly unable to pull. The EC2 topic's outside-in diagnosis applies unchanged.

## Command reference

| Command | What it does |
| --- | --- |
| `aws ecr create-repository --repository-name <name>` | create an image repository |
| `aws ecr get-login-password \\| docker login --username AWS --password-stdin <registry>` | authenticate docker to ECR |
| `docker build -t <name> .` / `docker tag` / `docker push` | build, aim, and push an image |
| `aws ecs register-task-definition --cli-input-json file://taskdef.json` | register a task definition |
| `aws ecs run-task --launch-type FARGATE ...` | run a task once |
| `aws ecs describe-tasks --cluster <cluster> --tasks <id>` | task status, including stop reasons |

## Further reading

- **Amazon ECR User Guide**: "Private registry authentication"; "Image scanning".
- **Amazon ECS Developer Guide**: "Amazon ECS task definitions"; "AWS Fargate for Amazon ECS"; "Task CPU and memory"; "Amazon ECS task IAM roles".
