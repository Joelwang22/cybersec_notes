# Cloud Foundations: Networks and Identity

Everything you have built in this course ran inside a network you never looked at, under permissions you never read. In this topic you replace both with versions you can explain. The first half of this book builds a VPC from nothing: every resource written into a CloudFormation template, because after the CloudFormation topic nothing in this course is built by hand. The second half is identity: how to read an IAM policy and predict what it will do before you test it. The last chapter is about this topic's characteristic failure, a stack that deploys perfectly and serves nothing, and the method for finding out why.

One idea runs under all of it: deny by default. A new VPC passes no traffic. A new policy grants nothing. Everything that works, works because something explicitly says so, and here you learn to find that something.

## Chapter 6.1 The VPC

### What the EC2 topic gave you for free

In the EC2 exercises you launched an instance and reached it from a browser within the hour. That worked because every AWS account ships with a **default VPC** in each region, pre-assembled: an address block of `172.31.0.0/16`, one subnet in every Availability Zone, an internet gateway already attached, a main route table already carrying a route to that gateway, and auto-assign public IP switched on. Five decisions, all made before you arrived.

The default VPC is convenient and opaque in equal measure. When it works, you learn nothing; when something breaks, you have no model of what is inside it. Real workloads do not run there. Here you build the same shape yourself, piece by piece, in a fresh VPC that starts sealed.

A VPC (Virtual Private Cloud) is a private, isolated IP network inside a region. Nothing enters or leaves it except through gateways you create, and nothing inside it is filtered except by rules you write. It is the networking you spent two weeks on, delivered as API calls.

### CIDR blocks, and sizing /16 against /24

A VPC is defined by its CIDR block. AWS accepts anything from `/16` (65,536 addresses) down to `/28` (16 addresses). The course VPC uses `10.0.0.0/16`: the whole private 10.0 slice of RFC 1918 space at maximum VPC width.

Why take 65,536 addresses for a course that needs two? Because address space inside a VPC costs nothing, and a VPC sized too small is painful to fix. You can bolt on secondary CIDR blocks later, but you cannot widen the original. The block is a commitment made at creation for the network's whole life. Size the block for the plan, not just for now.

Subnets carve slices out of the VPC block. Each subnet CIDR must sit inside the VPC range and must not overlap any other subnet. Our public subnet is `10.0.1.0/24`: 256 addresses, leaving `10.0.2.0/24` (the private subnet built in the exercises), `10.0.3.0/24`, and 250-odd further /24s free for whatever comes later.

### The five reserved addresses

Your networking course taught you to subtract two addresses from every subnet: network and broadcast. AWS subtracts five. In `10.0.1.0/24`:

| Address | Reserved for |
| --- | --- |
| `10.0.1.0` | Network address |
| `10.0.1.1` | VPC router, the implicit router the `local` route points at |
| `10.0.1.2` | Amazon DNS resolver |
| `10.0.1.3` | Reserved by AWS for future use |
| `10.0.1.255` | Broadcast address, reserved even though a VPC does not support broadcast |

So a /24 yields 251 usable addresses, not 254. This matters at the small end: a /28 has 16 addresses and AWS takes 5, leaving 11. It is also a standard certification question.

### A subnet belongs to exactly one Availability Zone

The VPC is regional: it spans every Availability Zone in its region. A subnet is not. It lives in exactly one AZ, chosen at creation, and can never move. High availability across zones therefore means one subnet per zone, which is why the default VPC ships with a subnet in every zone. The network here uses one subnet in one zone, because this topic is about the shape, not about surviving a zone failure. The template picks the region's first zone with `!Select [0, !GetAZs ""]` so every student's stack is identical.

### What a bare VPC can and cannot do

Create a VPC and nothing else, and it arrives with three invisible companions:

- a **main route table**, containing only the local route;
- a **default network ACL**, which allows all traffic in both directions;
- a **default security group**, which allows members to talk to each other and all outbound.

Add a subnet and two instances, and they can reach each other; the local route covers the entire VPC block. They can resolve DNS through the Amazon resolver. And that is the complete list. No packet can leave the VPC; no packet can enter. There is no gateway, so there is nowhere for outside traffic to go in either direction. A new VPC is a sealed box, and Chapter 6.2 is about cutting exactly one deliberate opening in it.

One flag pair worth knowing now: `EnableDnsSupport` turns on the Amazon resolver inside the VPC, and `EnableDnsHostnames` gives instances DNS names. A custom VPC created through the API has hostnames off by default. The template sets both true; without them, the `PublicDnsName` your reachability check curls against does not exist.

### The template, stage one

Everything in this topic deploys through CloudFormation. The build starts with two resources:

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: A network built from nothing

Parameters:
  VpcCidr:
    Type: String
    Default: 10.0.0.0/16
  PublicSubnetCidr:
    Type: String
    Default: 10.0.1.0/24

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags: [{Key: Name, Value: vpc-vpc}]

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PublicSubnetCidr
      AvailabilityZone: !Select [0, !GetAZs ""]
      MapPublicIpOnLaunch: true
      Tags: [{Key: Name, Value: vpc-public-a}]
```

This deploys, and it produces a sealed network. Every following section adds resources to this same template.

**Key points**

- The default VPC pre-assembled everything the EC2 topic needed; here you build each piece yourself, from a template.
- A VPC CIDR is a one-time commitment: `/16` costs nothing extra and leaves room; subnets carve non-overlapping slices from it.
- AWS reserves five addresses per subnet, not two: a /24 has 251 usable.
- A subnet belongs to exactly one Availability Zone, forever.
- A bare VPC passes internal traffic and nothing else: no gateway, no way in or out.

## Chapter 6.2 Reaching the internet

### The internet gateway: creating and attaching are two resources

An internet gateway (IGW) is the VPC's connection to the internet. In a template it is **two** resources, and the distinction is not bureaucracy:

```yaml
  IGW:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags: [{Key: Name, Value: vpc-igw}]

  VPCGW:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref IGW
```

A gateway exists on its own and attaches to at most one VPC. Creation and attachment are two separate API calls. Everything in the console is an API call, and CloudFormation puts both calls where you can see them. The console wizard hides the second one, which is exactly how people end up with a gateway created, never attached, doing nothing, producing no error. In a template, the absence of a `VPCGatewayAttachment` is visible at review time.

An attached gateway still moves no traffic by itself. Nothing sends packets to a gateway until a route table says to.

### Route tables

A route table is a list of rules (destination CIDR, target) consulted for every packet leaving a subnet. Most specific prefix wins, exactly as in your routing labs.

**The local route.** Every route table begins with one entry you did not create and cannot delete: destination `10.0.0.0/16` (the VPC's own block), target `local`. This is how everything inside the VPC reaches everything else. Because it is more specific than any default route, internal traffic stays internal no matter what else the table says.

**The default route.** To let traffic out, add one entry: destination `0.0.0.0/0`, target the internet gateway. `0.0.0.0/0` matches every address; being the least specific route possible, it loses to `local` for internal traffic and catches everything else.

```yaml
  PublicRT:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags: [{Key: Name, Value: vpc-public-rt}]

  DefaultRoute:
    Type: AWS::EC2::Route
    DependsOn: VPCGW
    Properties:
      RouteTableId: !Ref PublicRT
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref IGW
```

Note `DependsOn: VPCGW`. CloudFormation creates resources in parallel, and a route pointing at a gateway that is not yet attached fails the stack. This is the topic's first hard ordering constraint, and it comes straight from the fact that attachment is its own resource.

**Main table versus explicit association.** Every VPC has one main route table. Any subnet not explicitly associated with another table follows the main one, silently, with no marker on the subnet. A route table you create yourself governs nothing until an association ties it to a subnet:

```yaml
  PublicRTA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRT
      SubnetId: !Ref PublicSubnet
```

Sit with the failure mode hiding here. A route table with a perfect default route, associated with nothing, has zero effect and raises zero errors. The subnet follows the main table, which has no internet route, and every outbound packet dies quietly. CloudFormation deploys the whole arrangement green; every resource in it is individually valid. When you diagram your stack in the exercises from `describe-route-tables` output, the association is one of the things the diagram must show precisely because the template can lie about it by omission.

### What makes a subnet public

**A subnet is public because of a route, not because of a property.**

There is no `Public: true` on a subnet resource; look through `AWS::EC2::Subnet`'s properties and it is not there. A subnet is public when, and only when, its route table carries a route to an internet gateway. When the console labels a subnet "public", it computed that label by reading the route table. When this book says "public subnet", that is shorthand for "a subnet whose route table routes `0.0.0.0/0` to an IGW".

The consequences run in both directions. To make a subnet private, you do not set anything on the subnet; you change what its route table says (the exercises use `10.0.2.0/24` as the private subnet, with a different route table). To find out whether a subnet is public, you do not look at the subnet; you run `describe-route-tables` and read the routes. Most VPC confusion in practice traces back to looking for a property where there is only a route.

### MapPublicIpOnLaunch, and the address itself

The route makes the subnet reachable in principle. For a specific instance to be reachable in fact, it needs a public IPv4 address of its own. These are two independent requirements and either can be missing:

- **Route without address**: the subnet is public, the instance has only `10.0.1.x`, nobody on the internet can name it. This was the EC2 topic's debugging lab: a correctly routed subnet and an instance launched with public IP association disabled.
- **Address without route**: the instance has a public IP, and packets to it die because the subnet's route table never sends replies to the gateway.

`MapPublicIpOnLaunch: true` on the subnet makes every instance launched into it receive a public IPv4 automatically. It is a launch-time default, not a guarantee: launch settings can override it per instance, which is exactly how the EC2 topic's planted bug worked.

One detail that surprises everyone once: the instance's operating system never sees its public address. Run `ip addr` on the instance and you find only the private `10.0.1.x`. The public address lives at the internet gateway, which performs one-to-one NAT between the two. The instance learns its own public address only by asking the metadata service.

### Public IPv4 addresses cost money

Since February 2024, every public IPv4 address bills **by the hour**, whether or not a single packet ever uses it. An Elastic IP allocated but attached to nothing bills the same. The rate looks tiny on the Amazon EC2 pricing page, but it is a meaningful share of what the t3.micro underneath it costs.

The recurring theme applies: resources are billed for existing, not for being used. One forgotten address is noise in your account; a company account with three hundred forgotten Elastic IPs has a line item with someone's name on it. When you tear down the NAT exercise, releasing the Elastic IP is an explicit teardown step for exactly this reason.

**Key points**

- An IGW is two template resources (the gateway and its attachment) because they are two API calls; an unattached gateway does nothing.
- Every route table starts with the immovable `local` route; adding `0.0.0.0/0 → IGW` is what opens the way out, and the `Route` must `DependsOn` the attachment.
- Subnets with no explicit association silently follow the main route table; an unassociated route table governs nothing and raises no error.
- What makes a subnet public is a route, not a property.
- The instance additionally needs its own public IPv4 (`MapPublicIpOnLaunch` supplies one at launch), and that address bills by the hour from the moment it exists.

## Chapter 6.3 Controlling traffic

### Security groups: stateful, attached to the instance

A security group is a firewall attached to an instance, or more precisely to its elastic network interface (ENI). Three properties define it:

**Stateful.** AWS tracks connections. When an inbound rule admits a request on port 80, the response leaves without needing any outbound rule; return traffic rides the connection state. If your firewall unit covered stateful inspection or iptables conntrack, this is that, managed for you.

**Allow-only.** Rules can only allow. There is no deny rule; whatever no rule allows is dropped. This keeps a group readable (it is a plain list of what may enter), but it means "everyone except this range" cannot be written in a security group. Denies belong to the NACL.

**Silent.** A dropped packet triggers no rejection, no log entry, no error anywhere by default. The sender sees a connection that hangs and eventually times out. Chapter 6.7 turns this silence into a diagnostic.

A new group allows all outbound and no inbound. An instance can wear several groups; the effective rules are the union of all of them. This topic's web server needs exactly one inbound rule, TCP 80 from `0.0.0.0/0`, and nothing else.

### Network ACLs: stateless, attached to the subnet

The network ACL (NACL) is the other filter. It attaches to the **subnet** and evaluates every packet crossing the subnet boundary, in both directions, with no connection tracking.

**Stateless** means return traffic gets no credit for belonging to an established connection; it is judged fresh, against the outbound rules. To serve a web page through a restrictive NACL you need both:

- an inbound rule allowing TCP 80, and
- an outbound rule allowing TCP **1024-65535**, because the response goes back to whatever ephemeral source port the client happened to pick.

Forget the ephemeral range and requests arrive but responses die on the way out; the symptom is indistinguishable from a security group block. This is the moment the ephemeral-ports lecture from your networking course becomes an outage.

**Numbered rules, evaluated in order.** NACL rules carry numbers and are evaluated ascending; the first match decides and evaluation stops. At the bottom of every NACL sits an unnumbered `*` rule that denies everything nothing else matched: deny by default, written down where you can see it. Order is a real hazard: rule 100 allow-all followed by rule 200 deny-SSH allows SSH, because 100 matched first. Number with gaps (100, 200, 300) so rules can be inserted later.

Every VPC's **default NACL allows everything** in both directions, which is why you have not noticed NACLs in this course. A NACL you create yourself starts the other way: nothing but the `*` deny, blocking everything until rules are added.

### Referring to a security group instead of an address range

A security group rule's source can be another security group's ID instead of a CIDR:

```yaml
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          SourceSecurityGroupId: !Ref BastionSG
```

This reads "accept TCP 22 from any interface that wears `BastionSG`", whatever its IP address is today. Tiers reference tiers and no one maintains address lists: the database group admits 5432 from the app group, instances come and go, the rule never changes. The CIDR version breaks the day an instance is replaced and its address changes; the group version does not. The private-subnet exercise requires it: the private instance admits SSH from the public instance's group, not from an address.

### Which to reach for, and the order traffic passes

Reach for the **security group** by default: stateful, attached to the thing being protected, easy to read. Reach for the **NACL** when you need what only it can do: an explicit deny, applied subnet-wide, covering every instance present and future ("this address range never talks to this subnet, no exceptions").

The order of traversal matters for diagnosis:

```
inbound:   internet → IGW → route table → NACL (subnet edge) → SG (ENI) → instance
outbound:  instance → SG → NACL → route table → IGW → internet
```

Inbound traffic meets the NACL before the security group; outbound is the mirror. Either filter alone can kill the connection, and this layering is the skeleton of Chapter 6.7's method.

### The template, complete

Adding the security group and the instance finishes the build. The complete, runnable template:

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: A VPC built from nothing, one public subnet, one web server

Parameters:
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: Your <username>-key pair
  VpcCidr:
    Type: String
    Default: 10.0.0.0/16
  PublicSubnetCidr:
    Type: String
    Default: 10.0.1.0/24
  AmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags: [{Key: Name, Value: vpc-vpc}]

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PublicSubnetCidr
      AvailabilityZone: !Select [0, !GetAZs ""]
      MapPublicIpOnLaunch: true
      Tags: [{Key: Name, Value: vpc-public-a}]

  IGW:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags: [{Key: Name, Value: vpc-igw}]

  VPCGW:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref IGW

  PublicRT:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags: [{Key: Name, Value: vpc-public-rt}]

  DefaultRoute:
    Type: AWS::EC2::Route
    DependsOn: VPCGW
    Properties:
      RouteTableId: !Ref PublicRT
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref IGW

  PublicRTA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRT
      SubnetId: !Ref PublicSubnet

  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !Ref VPC
      GroupDescription: HTTP from anywhere
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags: [{Key: Name, Value: vpc-web-sg}]

  WebInstance:
    Type: AWS::EC2::Instance
    DependsOn: DefaultRoute
    Properties:
      ImageId: !Ref AmiId
      InstanceType: t3.micro
      KeyName: !Ref KeyName
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds: [!Ref WebSG]
      Tags: [{Key: Name, Value: vpc-web}]
      UserData:
        Fn::Base64: |
          #!/bin/bash
          dnf -y install httpd
          echo "built from nothing" > /var/www/html/index.html
          systemctl enable --now httpd

Outputs:
  WebURL:
    Value: !Sub "http://${WebInstance.PublicDnsName}"
    Description: The reachability check should pass against this
  VpcId:
    Value: !Ref VPC
  SubnetId:
    Value: !Ref PublicSubnet
```

Two deliberate choices to notice. The AMI resolves through the SSM public parameter, never a pasted `ami-` ID, the same rule as the CloudFormation topic. And the instance declares `DependsOn: DefaultRoute`: its user data runs `dnf install` at first boot, which needs the internet, which needs the route to exist first. CloudFormation cannot infer that dependency from any `!Ref`, so it is stated.

Deploy it, then run your pass-or-fail reachability check against the `WebURL` output. That check passing is what "done" means here, same as it meant in the CloudFormation topic.

**Key points**

- Security groups: stateful, attached to the ENI, allow-only; return traffic needs no rule, and drops are silent.
- NACLs: stateless, attached to the subnet, numbered rules evaluated in order with a `*` deny at the end; both directions must be written, including ephemeral ports 1024-65535.
- A rule can name another security group as its source: membership instead of addresses.
- Default to security groups; use NACLs for subnet-wide guardrails and explicit denies.
- Inbound traffic passes NACL then SG; outbound passes SG then NACL.

## Chapter 6.4 Identity

### Users, groups and roles, and why this course uses roles

IAM has three kinds of principal:

- A **user** is a permanent identity with long-lived credentials: a console password, access keys that work until revoked.
- A **group** is a set of users managed together; policies attached to the group apply to the members.
- A **role** is an identity with no credentials of its own. Something *assumes* it (a person, a service, an application) and receives temporary credentials that expire.

This course uses roles for everything, and the reasons are practical. Leaked long-lived access keys are the canonical AWS security incident, and every leaked key belonged to a user. Roles have nothing durable to leak. You have been living on roles throughout this course without the vocabulary: your Identity Center sign-in hands you a role session (that is why CLI access expires after a few hours and `aws sso login` revives it), your Lambda in the storage topic executed as a role, and this topic's instances read S3 with a role. There is no IAM user anywhere in this course.

### Anatomy of a policy

A policy is a JSON document. Six fields carry all the meaning:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadCourseBucket",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<your-cfn-bucket>/*",
      "Condition": {
        "IpAddress": {"aws:SourceIp": "203.0.113.0/24"}
      }
    }
  ]
}
```

- **Version**: the literal string `2012-10-17`. It is the version of the policy *language*, not a date you choose; older or missing versions disable features like policy variables.
- **Statement**: a list of independent rules. Each stands alone.
- **Effect**: `Allow` or `Deny`. Nothing else.
- **Action**: which API calls, as `service:Operation`. Wildcards work: `s3:Get*`, `s3:*`, `*`.
- **Resource**: which ARNs the statement covers.
- **Condition**: optional extra tests on the request: caller's IP, region, tags, time. The statement applies only when every condition passes.

Read policies right to left: *objects in the site bucket, may be fetched, when the caller is inside 203.0.113.0/24, this is permitted*. Every policy in this course, including the per-user isolation policies you will write for the group project, is this shape with different values.

### Identity-based versus resource-based

The same JSON grammar attaches in two places.

An **identity-based policy** attaches to a user, group or role and answers "what may this identity do?" It needs no `Principal` field; it is attached to its principal.

A **resource-based policy** attaches to the resource itself and answers "who may touch this thing?" It carries an extra field, `Principal`, naming who it speaks about. You wrote one in the storage topic without the vocabulary, the bucket policy that made your static site publicly readable:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicRead",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<your-s3-site-bucket>/*"
    }
  ]
}
```

The `Lambda::Permission` that let S3 invoke your function from the storage topic was also resource-based: a policy on the function naming `s3.amazonaws.com` as the principal. Within one account, an allow from either side (identity policy or resource policy) is enough. Across accounts, both sides must allow.

### How a request is evaluated

Every API call (console click, CLI command, SDK call, they are all API calls) is evaluated by the same algorithm:

1. **Start from deny.** The request is denied unless something changes that.
2. **Gather every applicable policy**: identity-based, resource-based, and the organization-level policies covering your account.
3. **Any explicit `Deny` that matches?** The request is denied. Nothing else is consulted.
4. **Otherwise, any `Allow` that matches?** The request is allowed.
5. **Otherwise: denied.** This is the *implicit deny*, the default answer when no policy speaks.

Two rules of this topic fall out of step 3 and step 5:

**Deny by default.** Silence means no. A new role can do nothing. An `AccessDenied` does not necessarily mean something forbade you; usually it means nothing permitted you. The error message distinguishes the cases; read it.

**An explicit deny cannot be overridden.** No allow, from any policy, attached anywhere, at any level, outranks a matching explicit deny. Ten allows and one deny is deny. Administrator access and one deny is deny. This is what makes deny statements strong enough to build guardrails from: the author of a deny does not need to know what allows exist now or might exist later.

A walkthrough. Suppose a role carries two attached policies (one allowing `s3:GetObject` on `arn:aws:s3:::<your-cfn-bucket>/*`, one denying `s3:GetObject` on `arn:aws:s3:::<your-cfn-bucket>/secret.txt`) inside the sandbox with its region-lock guardrail:

| Request | Matching Allow | Matching Deny | Result | Deciding rule |
| --- | --- | --- | --- | --- |
| `GetObject` on `.../index.html` | yes | no | Allowed | explicit allow, no deny |
| `GetObject` on `.../secret.txt` | yes | yes | Denied | explicit deny wins |
| `PutObject` on `.../new.txt` | no | no | Denied | implicit deny, nothing spoke |
| `ListAllMyBuckets` | no | no | Denied | implicit deny, action never allowed |
| `RunInstances` t3.micro, course region | yes (StudentAdmin) | no | Allowed | allow survives the guardrails |
| `RunInstances` anything, eu-west-1 | yes (StudentAdmin) | yes (SCP) | Denied | explicit deny wins, even over admin |

Predicting the table's fourth column before testing is the skill the exercises measure.

### Roles, trust policies, and credentials that expire

A role carries **two separate policy documents**, and confusing them costs hours:

- The **trust policy** answers *who may assume this role*. Its principal is whoever is allowed to pick the role up.
- The **permissions policy** answers *what the holder may do* once inside.

For an EC2 instance role, the trust policy names the EC2 service:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Get the trust policy wrong and the permissions never matter, because nobody can assume the role. The EC2 topic's lab planted exactly this: a role trusting `lambda.amazonaws.com` attached to an EC2 instance: flawless permissions, an unassumable role, and a metadata service returning no credentials.

Assuming a role yields temporary credentials (access key, secret, session token) valid for hours, not years. On an instance, the metadata service serves them and rotates them automatically; nothing lands on disk. Your Identity Center session works the same way, which is why it expires mid-lab and `aws sso login` fixes it. Expiry is not an inconvenience; it is the security model.

### Which actions scope to an ARN, and which only accept `*`

Every API action declares which resource types it accepts, and they are not all the same:

| Action | Valid `Resource` | Why |
| --- | --- | --- |
| `s3:GetObject` | `arn:aws:s3:::bucket/*`, an object ARN | operates on one object |
| `s3:ListBucket` | `arn:aws:s3:::bucket`, the bucket ARN, no `/*` | operates on one bucket |
| `s3:ListAllMyBuckets` | `*` only | account-wide; there is no single resource to name |
| `ec2:DescribeInstances` | `*` only | same, a listing across the account |

Two traps live in this table. First, `s3:ListBucket` on `bucket/*` matches nothing: the bucket ARN and the object ARN are different resources, and statements about one say nothing about the other. Second, writing `s3:ListAllMyBuckets` with your bucket's ARN does not fail loudly; the statement is valid JSON that matches no possible request, so the action remains implicitly denied. In the least-privilege exercise, every `AccessDenied` you produce names the action and the ARN it was tried against; the fastest way to learn this table is to read those messages. The authority is the **Service Authorization Reference**, one page per service, listing every action and what it accepts.

### Where Identity Center fits, and what StudentAdmin grants

Your own sign-in runs through **IAM Identity Center**, which lives at the organization level, above your account. Your username and password are not in your account's IAM; there are no IAM users there at all. The **permission set** named `StudentAdmin` is a template that Identity Center stamps into each student account as a role; signing in through the portal assumes that role in your account.

Prove it, don't assert it:

```console
$ aws sts get-caller-identity
{
    "UserId": "AROA...:rony",
    "Account": "111122223333",
    "Arn": "arn:aws:sts::111122223333:assumed-role/AWSReservedSSO_StudentAdmin_a1b2c3d4/rony"
}
```

`assumed-role`, `AWSReservedSSO_StudentAdmin`: your administrative access is a role session with expiring credentials, the same machinery as everything else in this chapter.

What StudentAdmin actually grants: administrative access *within the account*, minus whatever the organization denies. The region lock, the instance-type caps, the blocked services: those are denies imposed from above your account (Chapter 6.6 shows their shape), and by the rule of this chapter, your administrator allow cannot override them. Administrative and unlimited are different words.

**Key points**

- Users hold permanent credentials; roles are assumed and hand out expiring ones. This course is roles-only, deliberately.
- A policy is Version, Statement, Effect, Action, Resource, Condition; learn to read it right to left.
- Identity-based policies attach to principals; resource-based policies attach to things and name a `Principal`.
- Evaluation: deny by default; explicit deny beats every allow, always.
- A role's trust policy (who may assume) is separate from its permissions (what they may do); wrong trust means the permissions never matter.
- Some actions scope to ARNs; others, like `s3:ListAllMyBuckets`, only accept `*`. The Service Authorization Reference says which.

## Chapter 6.5 Networking not built here

This topic's network is one public subnet. Real VPCs contain more, and you will meet each of these pieces (one of them in the exercises).

### Private subnets, and the NAT gateway they need

A **private subnet** is one whose route table has no route to an internet gateway; that is the entire definition, by the rule of Chapter 6.2. Instances there cannot be reached from the internet, which is the point: databases, workers, anything that should only be reached through something in front of it.

But private instances still need *outbound* access: package updates, API calls. The answer is a **NAT gateway**: it lives in a *public* subnet, holds an Elastic IP, and the private subnet's route table sends `0.0.0.0/0` to it. Outbound connections work; inbound connections from outside are impossible, because NAT only maps traffic for connections that began on the inside.

The placement is the classic mistake: the NAT gateway goes in the **public** subnet (it needs the IGW route to function), while the **private** subnet's route table points at it. Put the NAT gateway in the private subnet and nothing works, with no error anywhere; it has no path to the internet either.

Cost, and it matters here: a NAT gateway bills **per hour plus per gigabyte processed**, and the hourly part runs whether any traffic flows or not. It is the most expensive resource in this course, billed from the moment it exists, and its Elastic IP bills by the hour alongside it. The hard exercise has you pull the current rates from the AWS Pricing Calculator (calculator.aws), compute the daily figure yourself, and then delete the stack. Deleting the gateway does not release its Elastic IP; that is a separate step, and skipping it leaves an address billing for nothing.

### VPC endpoints

Without help, an instance in a private subnet reaching S3 sends traffic out through the NAT gateway to S3's public endpoint, paying the NAT gateway's per-gigabyte rate for the privilege. A **VPC endpoint** connects the VPC to an AWS service privately, no internet path involved.

- **Gateway endpoints** exist for exactly two services: S3 and DynamoDB. They work as route-table entries (the pattern of Chapter 6.2 again: a route, not a property) and they are **free**. There is rarely a reason not to have the S3 one.
- **Interface endpoints** cover nearly every other service. Each is an ENI in your subnet with a private IP, billed per hour per AZ plus per gigabyte (cheaper than NAT for service traffic, but not free).

### VPC peering and Transit Gateway, in brief

**VPC peering** connects two VPCs so they route to each other's private addresses. Requirements: non-overlapping CIDRs (a reason to plan blocks: two VPCs both using `10.0.0.0/16` can never peer), routes added on both sides, security groups updated. Peering is **not transitive**: A-B and B-C peerings do not let A reach C. Ten VPCs fully meshed need 45 peerings, which is why **Transit Gateway** exists: a regional hub that VPCs attach to once, with routing between attachments, at an hourly-plus-per-GB price. Peer for two or three VPCs; hub past that.

### IPv6, and the egress-only internet gateway

A VPC can carry an IPv6 block alongside IPv4: AWS assigns a `/56`, subnets take `/64`s, and the addresses are free, which matters now that IPv4 is not. The catch: IPv6 addresses are globally routable. There is no NAT and no private-address regime; any IPv6 instance with a route to an IGW is reachable from the internet. The IPv6 equivalent of "outbound only" is the **egress-only internet gateway**: architecturally a NAT-gateway analog that allows connections out but never in, and unlike the NAT gateway it is free.

### VPC Flow Logs

Flow logs record metadata about traffic crossing network interfaces: not packet contents, one summary record per flow per aggregation interval. They are **off by default** and can be enabled per VPC, subnet, or ENI, delivered to CloudWatch Logs or S3:

```bash
aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids vpc-0abc1234 \
  --traffic-type ALL \
  --log-group-name vpc-flow-logs \
  --deliver-logs-permission-arn arn:aws:iam::111122223333:role/flow-logs-role
```

A default-format record, field by field:

```
2 111122223333 eni-0a1b2c3d4e5f6a7b8 203.0.113.7 10.0.1.25 55432 80 6 12 5203 1755400000 1755400060 ACCEPT OK
```

| Field | Value | Meaning |
| --- | --- | --- |
| version | `2` | record format version |
| account-id | `111122223333` | account owning the ENI |
| interface-id | `eni-0a1b...` | which network interface |
| srcaddr / dstaddr | `203.0.113.7` / `10.0.1.25` | source and destination |
| srcport / dstport | `55432` / `80` | note the ephemeral source port |
| protocol | `6` | IANA number; 6 is TCP |
| packets / bytes | `12` / `5203` | volume in this window |
| start / end | Unix timestamps | aggregation window |
| action | `ACCEPT` | the rules allowed this flow |
| log-status | `OK` | the record itself is sound |

`action` is the diagnostic field: **ACCEPT** means security groups and NACLs allowed the flow; **REJECT** means one of them refused it. A REJECT with `dstport 22` from an unknown address is the internet's background noise scanning you; a REJECT where you expected your own traffic to flow is Chapter 6.7's evidence. Flow logs are the *only* place a security group drop is ever recorded, and only if you enabled them first.

### DHCP option sets and DNS inside the VPC

Instances learn their network configuration by DHCP; the **DHCP option set** attached to the VPC decides what they are told: domain name, DNS servers, NTP servers. The default hands out `AmazonProvidedDNS`, the resolver living at the subnet's `.2` address (reserved in Chapter 6.1) and reachable VPC-wide at the second address of the VPC block, `10.0.0.2`. Option sets are immutable: to change one, create a new set and associate it.

The resolver answers public queries, and, with `EnableDnsSupport` and `EnableDnsHostnames` on, gives instances names like `ec2-54-x-x-x.compute-1.amazonaws.com`. A useful property: queried from inside the VPC, an instance's public DNS name resolves to its *private* address, keeping internal traffic on the local route instead of hairpinning through the IGW.

**Key points**

- Private subnet = no IGW route; outbound access comes from a NAT gateway in the *public* subnet, metered per hour and per gigabyte, the most expensive resource in this course, billed for existing.
- Gateway endpoints for S3 and DynamoDB are free route-table entries; interface endpoints are paid ENIs for everything else.
- Peering is pairwise and non-transitive; Transit Gateway is the hub once the mesh gets silly. Overlapping CIDRs can never peer.
- IPv6 addresses are free and globally routable; outbound-only IPv6 uses an egress-only internet gateway.
- Flow logs are opt-in and are the only record of a security-group drop; ACCEPT vs REJECT is the field that matters.

## Chapter 6.6 Identity beyond one policy

### When four kinds of policy apply at once

Chapter 6.4 evaluated identity and resource policies. The full picture adds two more layers, and real requests can cross all four:

1. **Service control policies (SCPs)**: organization-level guardrails.
2. **Permission boundaries**: per-principal caps.
3. **Identity-based policies**: what the principal may do.
4. **Resource-based policies**: who may touch the resource.

The algorithm does not change; it lengthens. Any explicit deny anywhere still ends the request. Then the request must survive every *filter* layer: the SCP must allow it, the boundary (if one is set) must allow it; and finally some identity or resource policy must actually allow it. A useful mental model: SCPs and boundaries never grant anything; they only remove things from what the other policies could grant. An allow in an SCP gives you no power; only allows in identity or resource policies do that.

### Permission boundaries

A permission boundary is a policy attached to a user or role that sets the *maximum* it can ever do; effective permissions are the intersection of the boundary and the identity policies. Their real use is delegation: let a team create roles freely, but force every role they create to carry a boundary, so nothing they build can exceed the boundary no matter what policies they attach. The group project's per-user isolation could be built with boundaries; you will use plain identity policies instead, but the evaluation rule is the same intersection logic.

### Service control policies: the ones on your own account

SCPs attach to accounts (or groups of accounts) from the organization above. They are invisible from inside the account (you cannot list them with any IAM call), but you have been living inside two of them all through this course. The region lock is shaped like this:

```json
{
  "Sid": "DenyOutsideUsEast1",
  "Effect": "Deny",
  "NotAction": ["iam:*", "sts:*", "organizations:*", "budgets:*",
                "cloudfront:*", "route53:*", "support:*"],
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {"aws:RequestedRegion": "<course-region>"}
  }
}
```

Deny everything (except the listed global services, which have no region) whenever the requested region is not the course region. The instance-type cap is the same pattern:

```json
{
  "Sid": "DenyBigInstances",
  "Effect": "Deny",
  "Action": "ec2:RunInstances",
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringNotLike": {"ec2:InstanceType": ["t2.*", "t3.*", "t3a.*", "t4g.*"]}
  }
}
```

Both are explicit denies, which is the entire trick: they hold against StudentAdmin's administrative allow because an explicit deny cannot be overridden. When you attempt `aws ec2 describe-instances --region eu-west-1` and read the `AccessDenied` (with an explicit deny in a service control policy in the message text), you are watching Chapter 6.4's evaluation algorithm run with the full four-layer stack. Read the error message; it names the layer that refused you.

### Service-linked roles

Some services need permissions in your account to do their work, and create a **service-linked role** for it the first time you use them: predefined trust and permissions, owned by the service, not editable by you. Run `aws iam list-roles` and look for `AWSServiceRoleFor...`: the `AWSServiceRoleForECS` in your account appeared the moment the storage topic's Fargate cluster was created. Nothing to configure, but when you meet a role you never made, this is what it is.

### The policy simulator, and IAM Access Analyzer

Chapter 6.4 said to predict a request's outcome before making it. The **IAM policy simulator** does the prediction mechanically:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::111122223333:role/vpc-web-role \
  --action-names s3:GetObject s3:PutObject \
  --resource-arns "arn:aws:s3:::<your-cfn-bucket>/index.html"
```

The output's `EvalDecision` is `allowed`, `implicitDeny`, or `explicitDeny`: the three outcomes of the evaluation algorithm, named. Simulation is free and touches nothing, which makes it the safe way to test a policy against actions that would be destructive to try.

**IAM Access Analyzer** works the other direction: instead of answering one question, it reviews what exists: flagging resources reachable from outside the account (a bucket policy with `"Principal": "*"` lights up immediately), validating policies for errors and over-breadth as you write them, and reporting granted-but-unused access. The least-privilege exercise is Access Analyzer's job description performed by hand.

### Condition keys worth knowing

Three conditions carry most of the real-world weight:

- **`aws:SourceIp`**: the caller's public IP. A deny conditioned on `NotIpAddress` outside a CIDR range pins credentials to a network; stolen keys stop working from anywhere else. This is the condition in the Deny wins exercise, using the classroom's address range. One caveat: calls that reach a service through a VPC endpoint do not carry a public source IP, and an `aws:SourceIp` condition will not match them.
- **`aws:PrincipalTag/<key>`**: tags on the calling principal, enabling policies like "allow if the caller's `team` tag equals the resource's", one policy serving many principals.
- **`aws:RequestedRegion`**: the region the call targets; the backbone of the sandbox's region lock above.

### CloudTrail: the record of who called what

Every API call in the account (console click, CLI, SDK, CloudFormation acting on your behalf) lands in **CloudTrail**. Ninety days of management events are queryable free, no setup:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --max-results 5
```

Each event records who (`userIdentity`, down to the assumed-role session name, which for you is your username), what (`eventName`, `eventSource`), from where (`sourceIPAddress`), and the outcome: a denied call carries `"errorCode": "AccessDenied"` with the message you saw. When you produce a string of AccessDenied errors in the least-privilege exercise, every one of them is in CloudTrail with your name attached. Two boundaries to remember: S3 object-level reads and writes are *data events*, not logged by default; and CloudTrail is control plane only: a packet dropped by a security group appears nowhere in it. That trail of evidence belongs to flow logs, and the split between the two is where Chapter 6.7 begins.

**Key points**

- Full evaluation stacks four layers (SCP, boundary, identity, resource); SCPs and boundaries only filter, never grant.
- Your sandbox's region lock and instance-type cap are SCP explicit denies, which is why StudentAdmin cannot override them.
- `simulate-principal-policy` predicts allowed / implicitDeny / explicitDeny without executing anything; Access Analyzer reviews what exists.
- `aws:SourceIp`, `aws:PrincipalTag`, `aws:RequestedRegion` are the workhorse condition keys.
- CloudTrail records every API call and every denial, but never a dropped packet.

## Chapter 6.7 When it deploys but does not work

### CREATE_COMPLETE means the resources exist

CloudFormation reports `CREATE_COMPLETE` when every resource in the stack was created successfully: every API call returned success. That is the whole claim. It does not test that traffic flows, that any route leads anywhere, or that the network you described is the one you meant. A route table associated with nothing, a security group missing its port, a gateway created and never attached: each is a perfectly valid resource, and a stack of them deploys green.

The CloudFormation topic's failures were loud: stack events, rollback, an error message to read. This topic's are silent: a green stack and a page that times out. Silence is why you need a method.

### The method: work outward from the instance

Start at the instance and walk outward through every layer a packet must cross, in order. One command per layer; each step either finds the fault or rules that layer out. Do not skip ahead: no SSHing into the box to inspect Apache before you have established the packet can arrive at all. (The CloudFormation topic's quick checklist ran outside-in from your laptop; this is the same layer list, walked from the other end, entirely with describe commands.)

**Step 0: define the failure.** One command, pass or fail:

```bash
curl -sf --max-time 5 "http://<public-dns-name>/" && echo PASS || echo FAIL
```

Note *how* it fails. A timeout means something dropped the packet: filter or routing. `Connection refused` means the packet arrived and nothing was listening; the network is fine and the problem is on the instance. This one distinction can skip you four steps.

**Step 1: the instance.** Running, and holding a public address?

```bash
aws ec2 describe-instances --instance-ids i-0abc... \
  --query 'Reservations[].Instances[].[State.Name,PublicIpAddress,SubnetId]'
```

No public IP: the packet never had a destination. Stop here; the fault is address assignment, not filtering.

**Step 2: the security group.** Does a rule admit your traffic?

```bash
aws ec2 describe-security-groups --group-ids sg-0abc... \
  --query 'SecurityGroups[].IpPermissions'
```

You are looking for TCP 80 from `0.0.0.0/0` (or your address). Remember the union rule: check every group the instance wears.

**Step 3: the network ACL.** What guards the subnet?

```bash
aws ec2 describe-network-acls \
  --filters Name=association.subnet-id,Values=subnet-0abc... \
  --query 'NetworkAcls[].Entries'
```

If it is the default allow-all NACL, rule this layer out and move on. If someone tightened it, check both directions, including the ephemeral-port outbound rule from Chapter 6.3.

**Step 4: the route table.** The step that catches the silent fault. Which table does the subnet *actually* follow?

```bash
aws ec2 describe-route-tables \
  --filters Name=association.subnet-id,Values=subnet-0abc... \
  --query 'RouteTables[].Routes'
```

Two distinct failures surface here. The routes shown may lack a `0.0.0.0/0` entry targeting an `igw-`. Or, subtler, the filter returns *nothing*, which means no route table is explicitly associated with the subnet, so it silently follows the main table, whatever your template intended. A route table can exist, hold a perfect default route, and govern nothing. The command shows actual state; your template shows intent; diagnosis is the comparison of the two.

**Step 5: the gateway.** Created *and* attached?

```bash
aws ec2 describe-internet-gateways \
  --filters Name=attachment.vpc-id,Values=vpc-0abc... \
  --query 'InternetGateways[].[InternetGatewayId,Attachments]'
```

An empty result with a gateway existing elsewhere in the account means created-but-never-attached: Chapter 6.2's two-resource split, missing its second half.

If all five layers check out and the symptom was a timeout, re-verify from another network. If it was `Connection refused`, the network was never the problem: go look at whether `httpd` is running.

### Comparing intended state against actual state

Every step above is one comparison: the template says what you meant; the `describe-` call says what exists. These are the same API calls the console makes to draw its screens (everything in the console is an API call), and they are the tooling behind the diagram exercise, where every value must come from describe output rather than from your template or your memory. The habit generalizes: when infrastructure misbehaves, the first question is never "what did I write?" but "what is actually there?"

### Reachability Analyzer, and Network Access Analyzer

**Reachability Analyzer** performs the outward walk mechanically. Give it a source, destination, and port; it analyzes configuration (no packets are sent) and either traces the full path hop by hop or names the exact component that blocks it:

```bash
aws ec2 create-network-insights-path \
  --source igw-0abc... --destination i-0abc... \
  --protocol tcp --destination-port 80

aws ec2 start-network-insights-analysis \
  --network-insights-path-id nip-0abc...

aws ec2 describe-network-insights-analyses \
  --network-insights-analysis-ids nia-0abc...
```

The result is `NetworkPathFound: true` with the hop-by-hop path, or `false` with an explanation naming the blocking component: the route table with no matching route, the NACL entry, the security group. Each analysis run is billed individually: cheap for a named answer, not something to poll in a loop. Use it to *confirm* a diagnosis the manual method produced; the manual method is the skill.

**Network Access Analyzer** asks broader questions: not "can A reach B?" but "does any path exist matching this scope?" ("anything from the internet to any database subnet"). It is an audit tool; know it exists.

### Where a refusal is recorded, and where it leaves no trace

The topic's two halves meet here. When AWS refuses you, where is that written?

| Refusal | Recorded in | If not enabled |
| --- | --- | --- |
| API call denied (IAM, SCP) | CloudTrail, automatically | always on for management events |
| Packet dropped (SG, NACL) | VPC Flow Logs, as `REJECT` | **no trace anywhere** |

A denied API call is loud: an error message to your terminal, an event in CloudTrail with `errorCode: AccessDenied`. A dropped packet is silent by design: nothing returns to the sender, nothing lands on the instance, nothing in CloudTrail; the only witnesses are flow logs, and only if someone enabled them before the drop. With no flow logs, the sole evidence is the shape of the failure at your terminal: timeout means dropped, `Connection refused` means arrived-but-unanswered. Diagnosis leans on exactly this asymmetry, which is why IAM problems are usually found by reading and network problems by walking.

### Turning a diagnosis into a check

A diagnosis you performed once is knowledge; a diagnosis encoded as a command is a check that keeps working. The rule from the EC2 topic still holds: a check is a single command whose exit code means pass or fail. The end-to-end check, against the stack's own output:

```bash
url=$(aws cloudformation describe-stacks --stack-name vpc-net \
  --query "Stacks[0].Outputs[?OutputKey=='WebURL'].OutputValue" --output text)
curl -sf --max-time 5 "$url" > /dev/null
```

And a narrower one, distilled from Step 4, asserting the exact condition this chapter turned on, that the subnet's effective route table has a default route to an internet gateway:

```bash
aws ec2 describe-route-tables \
  --filters Name=association.subnet-id,Values=<subnet-id> \
  --query 'RouteTables[].Routes[?DestinationCidrBlock==`0.0.0.0/0`].GatewayId' \
  --output text | grep -q '^igw-'
```

The first check tells you *that* it broke; the second tells you *where*, for one specific layer, instantly. Prove it, don't assert it: a fix is not finished until the check that caught the fault passes again, and the checks you write here are the ones the group project will lean on when its network misbehaves with four other people watching.

**Key points**

- `CREATE_COMPLETE` certifies that resources exist, not that traffic flows or that they match your intent.
- Work outward from the instance (instance, SG, NACL, route table, gateway), one describe command per layer, no skipping.
- Timeout means dropped; `Connection refused` means arrived and nothing listening. The failure's shape is free evidence.
- The route-table step must ask which table the subnet *actually* follows; an unassociated table governs nothing, silently.
- API denials land in CloudTrail; packet drops appear only in flow logs, and only if enabled beforehand.
- A diagnosis becomes durable when it is a command with a meaningful exit code.

## Command reference

| Command | What it answers |
| --- | --- |
| `aws ec2 describe-vpcs` | which VPCs exist, their CIDR blocks |
| `aws ec2 describe-subnets` | subnet CIDRs, AZs, `MapPublicIpOnLaunch` |
| `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<id>` | which table the subnet actually follows, and its routes |
| `aws ec2 describe-internet-gateways --filters Name=attachment.vpc-id,Values=<id>` | whether a gateway is attached to the VPC |
| `aws ec2 describe-security-groups --group-ids <id>` | actual inbound and outbound rules |
| `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<id>` | the subnet's NACL entries, both directions |
| `aws ec2 describe-instances --instance-ids <id>` | state, public IP, subnet, security groups |
| `aws ec2 describe-nat-gateways` | NAT gateways, the ones billing for every hour they exist |
| `aws ec2 create-flow-logs` | start recording ACCEPT/REJECT per flow |
| `aws ec2 create-network-insights-path` / `start-network-insights-analysis` | Reachability Analyzer: name the blocking component |
| `aws sts get-caller-identity` | who you are: role, session, account |
| `aws iam simulate-principal-policy` | predict allow / implicitDeny / explicitDeny |
| `aws iam list-attached-role-policies --role-name <name>` | which policies a role carries |
| `aws cloudtrail lookup-events` | who called what, and what was denied |
| `aws cloudformation describe-stacks --stack-name <name>` | stack status and outputs |

## Further reading

- **Amazon VPC User Guide**: "How Amazon VPC works"; "Subnets for your VPC" (the five reserved addresses); "Configure route tables" (main table, associations); "Connect to the internet using an internet gateway"; "NAT gateways"; "DHCP option sets"; "DNS attributes for your VPC"; "Logging IP traffic using VPC Flow Logs".
- **Amazon EC2 User Guide**: "Amazon EC2 instance IP addressing" (why the OS never sees the public address); "Security groups".
- **AWS PrivateLink Guide**: "Gateway endpoints" and "Access an AWS service using an interface VPC endpoint".
- **Amazon VPC Peering Guide**: "VPC peering basics" (non-transitivity).
- **IAM User Guide**: "Policy evaluation logic" (the authoritative version of Chapter 6.4's algorithm); "Policies and permissions in IAM"; "IAM roles" (trust policies); "Permissions boundaries for IAM entities"; "Using IAM Access Analyzer".
- **Service Authorization Reference**: "Actions, resources, and condition keys for Amazon S3" (which actions accept which ARNs).
- **AWS Organizations User Guide**: "Service control policies (SCPs)".
- **IAM Identity Center User Guide**: "Permission sets".
- **AWS CloudTrail User Guide**: "Working with CloudTrail Event history".
- **Reachability Analyzer Guide**: "What is Reachability Analyzer".
- **Amazon EC2 pricing page**: the "IPv4 addresses" section for the current per-hour charge.
