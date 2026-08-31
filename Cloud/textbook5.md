# Cloud Foundations: CloudFormation

In the EC2 topic you built a web server with the console and your hands. In the storage topic you built a bucket website the same way. Both worked, and both share a weakness: they exist only in your account and in your memory of what you clicked. In this topic you rebuild both as text files, and by the end the pass-or-fail check you wrote in the EC2 exercises will pass against an address that a stack computed and handed to you.

This chapter set covers everything in the lecture in depth, plus the features you will not use in the exercises but will meet later in this course and in the field.

## Chapter 5.1 What CloudFormation is

CloudFormation is a service that takes a description of resources (a *template*) and makes the API calls needed to bring those resources into existence. That is the whole idea. You have already made plenty of API calls yourself: through the console in the EC2 topic, through the CLI from the reconnaissance exercise ("Who Are These People?") onward and throughout the storage topic. Every one of those clicks and commands ended up as a call to the same AWS API. CloudFormation is a third client of that same API. Console, CLI, and template are three doors into one room.

The difference between the doors is who decides what happens next. With the console and CLI, you issue one operation at a time and hold the plan in your head. With CloudFormation, you declare the end state (one bucket configured like this, one instance configured like that) and the service works out which calls to make and in what order. If the instance references the security group, the group is created first. You did that ordering by hand in the EC2 exercises without noticing you were doing it.

The unit CloudFormation works in is the **stack**: one deployment of one template, tracked under a name you choose. Everything a template creates belongs to its stack. The stack is created as a unit, updated as a unit, and (this is the part that pays off) deleted as a unit. No more hunting for the security group that belonged to a terminated instance. Resources cost money from the moment they exist; a stack gives you a single handle for making a whole group of them stop existing.

Because the template is the input, the same template deployed twice produces the same result. Deploy it now, next month, in a colleague's account: the same configuration comes out every time. Note what "same result" means: the same *configuration*, not the same physical resources. Each deployment creates fresh resources with fresh identifiers. The distinction between the name in your file and the identifier AWS assigns is the subject of the next chapter, and it matters throughout this topic.

A template is also the most honest documentation infrastructure can have. A wiki page describes what someone believed was built. The template *is* what was built, because it is what the build was made from.

**Key points**

- A template describes resources; CloudFormation makes the API calls: the same API behind the console and the CLI.
- The stack is the unit: one deployment of one template, created, updated, and deleted as a whole.
- The same template deployed twice produces the same configuration, in any account, every time.
- Declaring the end state replaces holding the plan in your head.

## Chapter 5.2 Template anatomy

### YAML, and why

CloudFormation accepts templates in JSON or YAML. This course uses YAML, for two practical reasons: YAML has comments, and a template you cannot annotate is a template nobody will understand in six months; and YAML has no braces or trailing commas to mismatch, which is where JSON authoring time actually goes. Every example in the AWS resource reference is shown in both syntaxes, so you never translate by hand.

The price of YAML is that indentation is not style; it is the structure. Two spaces of indent means "this is a child of the line above". Get it wrong and the template means something else or fails to parse. YAML also forbids tabs outright. A template that will not parse almost always has a tab in it or inconsistent indentation from a browser copy-paste. Configure your editor for spaces before you write a line.

### The smallest legal template

Only one top-level section is required: `Resources`. This is a complete, deployable template:

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Storage topic bucket site, first cut

Resources:
  SiteBucket:
    Type: AWS::S3::Bucket
```

`AWSTemplateFormatVersion` has had exactly one legal value since 2010. It is optional, but convention includes it. `Description` is also optional and you should always write one: it appears in the console's stack list and in `describe-stacks` output, and it is how you tell twelve stacks apart in a month. Neither line creates anything. The `Resources` section does.

### Parameters

Parameters are the template's inputs: values supplied at deploy time, so one template can serve many deployments. Each parameter has a `Type` (required) and, optionally, a `Default`, validation, and documentation:

```yaml
Parameters:
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small]
    Description: Size of the web server
  DbPassword:
    Type: String
    NoEcho: true
    Description: Masked in console and API output
```

- `Type` is `String`, `Number`, a list type, or one of the AWS-specific types below.
- `Default` makes the parameter optional at deploy time. No default means the deployment fails immediately unless a value is supplied.
- `AllowedValues` (and its cousins `AllowedPattern`, `MinLength`, `MaxValue`, …) rejects bad input *before* any resource is touched. The API would also reject an illegal instance type, but three minutes into the deployment, leaving a half-built stack to clean up. Fail before creating anything.
- `NoEcho: true` masks the value wherever CloudFormation displays it. It masks display; it does not encrypt. Real secrets belong in Secrets Manager or SSM SecureString, not in parameters. For this course, `NoEcho` awareness is enough.

Beyond `String` and `Number`, CloudFormation has AWS-aware parameter types. Declare a parameter as `AWS::EC2::KeyPair::KeyName` and CloudFormation verifies the value names a key pair that exists in your account before deploying anything; in the console the field becomes a dropdown of your real key pairs. The family includes `AWS::EC2::VPC::Id`, `AWS::EC2::Subnet::Id`, `AWS::EC2::AvailabilityZone::Name`, and more. Same idea every time: the typo dies at validation, not mid-rollback.

### The AMI problem, and the SSM answer

This topic's server template must name its machine image, and image IDs are the worst identifiers in AWS: opaque, different in every region, and superseded every few weeks as Amazon patches the OS. A pasted `ami-` ID is stale the day you commit it, and no reviewer can tell what it points at.

The answer is a parameter type that resolves through the SSM Parameter Store, where AWS publishes the current image IDs under stable public paths:

```yaml
Parameters:
  AmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64
```

The person deploying supplies an SSM *path* (or takes the default); at deploy time CloudFormation reads that path in the current region and substitutes the actual image ID, validated as a real image. The default above always points at the latest Amazon Linux 2023 kernel 6.1 x86_64 image. This is the course house rule: **never paste an `ami-` ID into a template**.

One consequence to file away for Chapter 5.5: resolution happens per deployment. Redeploy in three months and the path resolves to that day's AMI, which changes `ImageId`, which, as you will see, replaces the instance.

### Resources: the logical ID and the physical ID

Each entry in `Resources` starts with a name you invent: the **logical ID** (`SiteBucket` above). It must be alphanumeric and unique within the template, and it is how the rest of the template, and every event and error message, refers to the resource.

When CloudFormation creates the resource, AWS assigns the **physical ID**: the actual bucket name, or for an instance something like `i-0f3a19c2b8a7d4e21`. The mapping between the two is precisely what a stack stores: "in this stack, `SiteBucket` is bucket `cfn-site-sitebucket-1a2b3c4d5e6f`". When Chapter 5.5 talks about *replacement*, it means: the logical ID stays, and the physical resource (and its ID) changes underneath it.

Every resource has a `Type` string of the form `AWS::Service::Resource`: `AWS::S3::Bucket`, `AWS::EC2::Instance`, `AWS::EC2::SecurityGroup`. There are over a thousand types and nobody memorizes them. You look them up in the CloudFormation resource reference, one page per type. That page lists every property with its type, a **Required: Yes/No** marking, and an **Update requires** line whose importance becomes clear in Chapter 5.5. Notice how little is actually required: an S3 bucket requires no properties at all (AWS generates a name), and an instance requires almost nothing. Start minimal.

Growing the bucket toward the storage topic's site: website hosting is the `WebsiteConfiguration` property, and because new buckets block public policies by default, two of the four bucket-level Block Public Access switches you met in the storage topic must be off before a public-read policy can attach and be honored. The other two govern ACLs, which this design does not use; they stay on; loosen only what the design needs:

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
```

There is deliberately no `BucketName` property. A generated name can never collide with another student's, and (Chapter 5.5 again) a name you never chose is a name you will never be tempted to change.

### Outputs

Outputs are what the stack reports when it finishes: the values you would otherwise hunt for in the console.

```yaml
Outputs:
  SiteURL:
    Value: !GetAtt SiteBucket.WebsiteURL
    Description: Open this in a browser
    Export:
      Name: cfn-site-url
```

`Value` is required, and is almost always built from `Ref` or `GetAtt` (next chapter), because the interesting values are the ones AWS assigned during creation. `Description` accompanies the value in `describe-stacks`. `Export` publishes the value under a name that must be unique in the region, so *other* stacks can import it, the cross-stack mechanism covered in Chapter 5.7. This topic's templates need only `Value` and `Description`.

The exercises enforce a rule worth internalizing: the site address must come out of an Output, not out of your clipboard. Prove it, don't assert it.

**Key points**

- Only `Resources` is required; `Description` is optional and always worth writing. YAML indentation is structure, and tabs are fatal.
- Parameters validate input before anything is created; AWS-specific types (`AWS::EC2::KeyPair::KeyName`) validate against your account.
- AMIs are resolved from the SSM public parameter, type `AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>`, never pasted.
- You choose the logical ID; AWS assigns the physical ID; the stack stores the mapping.
- The resource reference page is the authority: required properties, return values, update behavior.

## Chapter 5.3 Intrinsic functions

A list of independent resources is not a system. The bucket policy needs the bucket's name; the instance needs the security group's ID; the output needs the instance's DNS name, and none of those values exist until deployment. Intrinsic functions are how a template refers to values that will only exist later. Each has a long form (`Fn::Sub`) and a YAML short form (`!Sub`); this course uses short form.

### Ref

`!Ref Thing` means "the value of that thing". Pointed at a parameter, it returns the value supplied at deploy time. Pointed at a resource's logical ID, it returns that resource's *main identifying value* and, quietly, declares a dependency: CloudFormation now knows the referenced resource must exist first. Wiring and ordering are the same mechanism.

What "main identifying value" means **depends on the resource type**, and is stated on each type's reference page under **Return values**:

| Resource type | `!Ref` returns |
| --- | --- |
| `AWS::S3::Bucket` | the bucket name |
| `AWS::EC2::Instance` | the instance ID |
| `AWS::EC2::SecurityGroup` | the group ID (see below) |
| `AWS::IAM::Role` | the role name |

The security group row comes with history. Nowadays `!Ref` returns the group ID for every security group, because every group lives in a VPC. For years, though, it returned the group *name* for groups on EC2-Classic (the pre-VPC platform AWS retired in 2023), and templates, blog posts, and answers written in that era still show name-based wiring. `!GetAtt WebSG.GroupId` names the ID explicitly, which is why this topic's template uses it for `SecurityGroupIds`: a property that takes IDs, with a sibling, `SecurityGroups`, that takes names. The habit: before Ref-ing a resource, spend ten seconds on its Return values section.

### GetAtt

`Ref` gives the one main value; `!GetAtt Logical.Attribute` gives any of the others. Every type's Return values section lists its attributes: an instance exposes `PublicDnsName`, `PublicIp`, `PrivateIp`, `AvailabilityZone`; a bucket exposes `WebsiteURL`, `Arn`, `DomainName`, `RegionalDomainName`; a security group exposes `GroupId`. Like `Ref`, a `GetAtt` creates an ordering dependency: you cannot read an attribute of something that does not exist.

Two attributes are literally this topic's deliverables: `!GetAtt SiteBucket.WebsiteURL` (the bucket exercise's required Output) and `!GetAtt WebServer.PublicDnsName` (the address your reachability check gets pointed at).

### Sub

`!Sub` builds strings with deploy-time values embedded. Inside the string, `${LogicalId}` behaves like `Ref` and `${LogicalId.Attribute}` behaves like `GetAtt`:

```yaml
Value: !Sub "http://${WebServer.PublicDnsName}"
Resource: !Sub "arn:aws:s3:::${SiteBucket}/*"
```

The second line is how a bucket policy names its objects: `${SiteBucket}` is the bucket name (that is what `Ref` returns for a bucket), wrapped in the ARN pattern. `!Sub "${SiteBucket.Arn}/*"` is an equivalent spelling.

`Sub` earns its keep in `UserData`, where an entire shell script is one string with stack values dropped in. One trap: shell syntax like `${PATH}` looks like a Sub reference and fails validation (*Unresolved resource dependencies [PATH] in the Resources block*). Escape it as `${!PATH}` and Sub emits a literal `${PATH}`. Your UserData from the EC2 exercises used shell variables; remember this when you port it.

### Pseudo parameters

Some values you never declare because AWS already knows them. Pseudo parameters are referenced like parameters (`!Ref AWS::Region`, or `${AWS::Region}` inside a `Sub`):

- `AWS::Region`: where the stack is running. A template should never hardcode a region it can look up.
- `AWS::AccountId`: the twelve-digit account number, needed for ARNs and for making globally unique names.
- `AWS::StackName`: useful for naming and tagging resources after the stack that owns them.
- `AWS::NoValue` is a special one: assigning `!Ref AWS::NoValue` to a property is equivalent to omitting the property. It only makes sense inside conditionals (Chapter 5.7): `!If [HasBucketName, !Ref BucketName, !Ref "AWS::NoValue"]`.

### DependsOn, and why you rarely need it

`DependsOn` is a resource attribute that forces creation order explicitly. The point to internalize is that you almost never need it: every `Ref`, `GetAtt`, and `Sub` reference already declares a dependency, and CloudFormation derives its ordering from them. If your data flows through references, the order falls out for free.

The legitimate case is a dependency with **no data flowing along it**. The classic, which the VPC and IAM topic makes you build: a default route references the internet gateway (`GatewayId: !Ref IGW`), but nothing in the route references the gateway's *attachment* to the VPC, a separate resource. No reference, no ordering, and the route can race the attachment and fail intermittently. `DependsOn: VPCGW` closes the gap. You will write exactly this line in the VPC and IAM topic.

If you find yourself writing `DependsOn` on every resource, you are wiring by hand what `Ref` would wire for you.

### The first running example: the bucket site

Everything so far, assembled. This is the storage topic's site as a complete, deployable template, the model for the first exercise:

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Storage topic bucket website, rebuilt as a stack

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

  SiteBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref SiteBucket
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Sid: PublicRead
            Effect: Allow
            Principal: "*"
            Action: s3:GetObject
            Resource: !Sub "arn:aws:s3:::${SiteBucket}/*"

Outputs:
  SiteURL:
    Value: !GetAtt SiteBucket.WebsiteURL
    Description: The website endpoint - open this in a browser
  BucketName:
    Value: !Ref SiteBucket
    Description: Upload index.html here
```

Read the wiring: the policy attaches to `!Ref SiteBucket` (the bucket name), its `Resource` is built with `Sub` around that same name, and both references guarantee the bucket exists before the policy attaches. The outputs hand back the two values the next steps need: the address for your browser, the name for the upload. Notice also what the template does *not* contain: the page itself. There is no CloudFormation resource for an S3 object (Chapter 5.9); uploading `index.html` remains an `aws s3 cp`, exactly as in the storage topic.

### The second running example: the web server

The EC2 topic's server as a template, the model for the second exercise. It uses every function in this chapter:

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: EC2 topic web server, rebuilt as a stack

Parameters:
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: Existing key pair, e.g. <username>-key
  AmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64

Resources:
  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP from anywhere
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref AmiId
      InstanceType: t3.micro
      KeyName: !Ref KeyName
      SecurityGroupIds:
        - !GetAtt WebSG.GroupId
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-web"
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          dnf -y install httpd
          echo "<h1>Hello from ${AWS::StackName}</h1>" > /var/www/html/index.html
          systemctl enable --now httpd

Outputs:
  PublicDNS:
    Value: !GetAtt WebServer.PublicDnsName
    Description: Point the reachability check at this
  SiteURL:
    Value: !Sub "http://${WebServer.PublicDnsName}"
    Description: The page in a browser
  InstanceId:
    Value: !Ref WebServer
    Description: The physical ID - compare after updates
```

Things to notice. The AMI arrives through the SSM parameter, never pasted. `SecurityGroupIds` uses `!GetAtt WebSG.GroupId` rather than `!Ref WebSG`: both return the group ID, but the attribute states it explicitly, which the security group's history above makes worth doing. `UserData` is Base64-encoded (EC2 requires it) around a `Sub`, so `${AWS::StackName}` lands inside the shell script; AL2023 means `dnf` and `httpd`, as everywhere in this course. And there is no `DependsOn` anywhere: the `GetAtt` on the security group orders it before the instance, and nothing else needs ordering.

**Key points**

- `Ref` returns a type-dependent main value; the Return values section of the resource page is the authority.
- `GetAtt` reads any listed attribute; `Sub` embeds `${Ref}` and `${Logical.Attr}` in strings; `${!x}` escapes a literal.
- Pseudo parameters (`AWS::Region`, `AWS::AccountId`, `AWS::StackName`) supply context you never declare.
- References create ordering; `DependsOn` is only for dependencies no reference expresses.

## Chapter 5.4 Driving stacks from the CLI

### validate-template

```bash
aws cloudformation validate-template --template-body file://site.yml
```

Run after every edit. It checks that the file parses and is structurally a template: legal sections, resolvable references. It does *not* predict success: a valid template can still request a bucket name that is taken. Note the argument form: `--template-body` with a `file://` prefix. Omit the prefix and the CLI treats the file *name* as the template text, and the error message will not tell you that.

### deploy

```bash
aws cloudformation deploy \
  --template-file site.yml \
  --stack-name cfn-site
```

`deploy` is the workhorse here. `--template-file` (a plain path this time; a different convention from `validate-template`, and yes, that is annoying) and `--stack-name` are both required. Its behavior is the reason this course teaches it instead of the lower-level `create-stack`/`update-stack` pair it wraps: **if no stack by that name exists, deploy creates it; if one exists, deploy computes the difference and applies it as an update; if the difference is empty, it prints "No changes to deploy" and touches nothing.** One idempotent command, safe to run repeatedly.

Parameters ride along as `--parameter-overrides`:

```bash
aws cloudformation deploy \
  --template-file server.yml \
  --stack-name cfn-web \
  --parameter-overrides KeyName=<username>-key
```

The stack name is the identity `deploy` matches on. Deploy the same template under a different `--stack-name` and you get a second, complete, separately billed copy of everything.

### describe-stacks

```bash
aws cloudformation describe-stacks \
  --stack-name cfn-web \
  --query "Stacks[0].Outputs"
```

```json
[
    {
        "OutputKey": "PublicDNS",
        "OutputValue": "ec2-54-91-118-23.compute-1.amazonaws.com",
        "Description": "Point the reachability check at this"
    }
]
```

Status and outputs. The `--query` trick is the one you learned in the EC2 topic with `describe-instances`. Add `--output text` and the value pipes straight into the reachability check, which is how "done" is defined in the exercises (no human copying values in between):

```bash
./check.sh "$(aws cloudformation describe-stacks \
  --stack-name cfn-web \
  --query "Stacks[0].Outputs[?OutputKey=='PublicDNS'].OutputValue" \
  --output text)"
```

### describe-stack-events

```bash
aws cloudformation describe-stack-events --stack-name cfn-web
```

Every step CloudFormation took, per resource, timestamped: `WebSG CREATE_IN_PROGRESS`, `WebSG CREATE_COMPLETE`, `WebServer CREATE_IN_PROGRESS`, and so on. This is the console's Events tab from the terminal, and it is where Chapter 5.6 lives. One ergonomic trap: events are listed **newest first**; the beginning of the story is at the bottom. Skim the events even on success once: you will see your references become sequencing, the group before the instance every time.

### Change sets: seeing what will happen before it happens

`deploy` always builds a **change set** internally (a computed diff between the running stack and the new template) and then executes it. Add `--no-execute-changeset` and it stops halfway:

```bash
aws cloudformation deploy \
  --template-file server.yml \
  --stack-name cfn-web \
  --no-execute-changeset
```

The diff now exists as an object; nothing has been touched; and deploy prints the exact `describe-change-set` command to inspect it. The inspection shows each resource that would be added, modified, or removed, and for each modification a `Replacement` field: `True`, `False`, or `Conditional`. That field is the difference between "your instance gains a tag" and "your instance is destroyed and rebuilt", and reading it before executing is the difference between a deployment and an incident. Execute with `execute-change-set`, or discard with `delete-change-set`. The lower-level verbs `create-change-set` / `describe-change-set` / `execute-change-set` do the same dance without `deploy`.

For a stack nobody depends on, deploying directly is fine. The moment anything you care about is inside, preview first. The update exercise has you record the prediction, then verify it came true.

### Capabilities

If a template creates IAM resources, `deploy` refuses:

```text
An error occurred (InsufficientCapabilitiesException) ...
Requires capabilities : [CAPABILITY_IAM]
```

until you acknowledge:

```bash
aws cloudformation deploy ... --capabilities CAPABILITY_NAMED_IAM
```

`CAPABILITY_IAM` acknowledges IAM resources with generated names; `CAPABILITY_NAMED_IAM` is required when the template assigns explicit names, and satisfies both cases. The acknowledgment is manual by design and cannot be disabled. IAM resources grant permissions; templates are files; files get copied from the internet; and a copied template could quietly contain a role with administrator access. The flag is your signature that you read the template and know it changes who can do what. Deny by default; the acknowledgment is the explicit exception. You will meet it in the final merge exercise, which creates a role, and again in the VPC and IAM topic.

**Key points**

- `validate-template --template-body file://…` checks structure; `deploy --template-file …` (required flag) creates or updates by stack name and reports "No changes to deploy" when idle.
- `describe-stacks` for status and outputs; `describe-stack-events` for the step-by-step record, newest first.
- `deploy --no-execute-changeset` previews; the `Replacement` field is the line to read.
- IAM-creating templates require `--capabilities CAPABILITY_IAM` or `CAPABILITY_NAMED_IAM`, acknowledged by hand, on purpose.

## Chapter 5.5 What an update actually does

Editing a file feels safe. Whether the resulting *update* is safe depends entirely on which property you touched, and the resource reference states the consequence per property, on the line labeled **Update requires**. There are three values.

**No interruption.** The resource is modified in place and keeps running. Tags on an instance, most bucket configuration. Same physical ID, no downtime.

**Some interruptions.** Modified in place, disruptively. The canonical case is `InstanceType` on an EBS-backed instance: CloudFormation stops the instance, changes the type, and restarts it. Same instance, same physical ID, but it was down for the duration, and a stop releases the auto-assigned public IP, so the instance comes back at a *different address*. Your reachability check now fails not because the server broke but because the address moved. (The stack Output reflects the new value after the update; re-query it.)

**Replacement.** The property cannot be changed on a live resource at all, so CloudFormation creates a **new physical resource**, points the logical ID at it, and deletes the old one in the cleanup phase at the end of the update. The canonical case is `ImageId`: an instance cannot swap its own disk image. After the update, `!Ref WebServer` returns a *different instance ID*; `InstanceId` is in the template's outputs precisely so you can compare before and after. Everything that lived only on the old resource (local disk contents, public IP, DNS name) is gone.

This is why the change set's `Replacement: True` matters, and why the professional habit is: read Update requires, preview the change set, then execute.

### Names are identity

Two consequences follow from names being identity.

Renaming a bucket destroys it. `BucketName` is documented as *Update requires: Replacement*. S3 has no rename API, so CloudFormation creates a bucket under the new name and deletes the old one. If the old bucket held your site, your site is gone. If the old bucket still has objects in it, its deletion during cleanup *fails*, and the bucket is left behind: outside the stack, unmanaged, still billed. Both outcomes are bad, which is why this topic's templates omit `BucketName` and let AWS generate one: a name you never chose is a name you will never be tempted to change.

And one level up: **a stack cannot be renamed at all.** The name is how `deploy` finds the stack; there is no rename operation. A different name means a new stack under the new name and a deletion of the old: replacement, at stack scale, performed by hand.

### DeletionPolicy and UpdateReplacePolicy

Sometimes "deleted as a unit" is exactly what you do not want: the stack is disposable, the data is not. `DeletionPolicy` is a resource attribute (a sibling of `Type` and `Properties`, not inside them):

```yaml
  SiteBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    Properties:
      ...
```

- `Delete` is the default for almost every type: the resource goes with the stack.
- `Retain`: CloudFormation forgets the resource instead of deleting it. Stack gone, bucket survives: now unmanaged, invisible to the stack, and still billed. A retained resource is a resource you have chosen to keep paying for; retain deliberately, never by reflex.
- `Snapshot` applies to the types that support snapshots (EBS volumes, RDS instances and clusters, Redshift, ElastiCache, Neptune): take a final snapshot, then delete. Not applicable to buckets or instances.

Its twin `UpdateReplacePolicy` answers the same question at *replacement* time: when an update replaces this resource, is the old one deleted (default) or retained or snapshotted? Same values, different trigger. On anything holding data, set both.

Related deletion behavior you will prove in the teardown exercise: a bucket that still contains objects cannot be deleted, so `delete-stack` on a stack whose (non-retained) bucket is non-empty fails and leaves the stack in `DELETE_FAILED`. From there you either empty the bucket and retry, or retry with `delete-stack --retain-resources SiteBucket` (an option only valid in the `DELETE_FAILED` state), which deletes the stack and abandons the bucket.

**Key points**

- Every property's **Update requires** line predicts the update: no interruption, some interruptions, or replacement. Read it before deploying.
- Replacement means a new physical resource and a new physical ID under the same logical ID; the old resource is deleted at cleanup.
- `InstanceType` changes in place but stops the instance (and moves its public address); `ImageId` and `BucketName` replace.
- A stack cannot be renamed; its name is its identity.
- `DeletionPolicy` / `UpdateReplacePolicy` (`Delete`, `Retain`, `Snapshot`) decouple a resource's fate from its stack's.

## Chapter 5.6 Reading failures

Stacks fail, and the failure workflow is a career skill. The exercises here include a stack that fails on purpose so you practice it where it is cheap.

### Status values, and which are terminal

A stack always has exactly one status, named systematically: *operation*_*phase*. You do not memorize the list; you parse it.

| Status | Meaning |
| --- | --- |
| `CREATE_IN_PROGRESS` / `UPDATE_IN_PROGRESS` / `DELETE_IN_PROGRESS` | Working. Wait. |
| `CREATE_COMPLETE` / `UPDATE_COMPLETE` | Done, healthy. |
| `UPDATE_COMPLETE_CLEANUP_IN_PROGRESS` | Update done; deleting replaced resources. |
| `ROLLBACK_IN_PROGRESS` / `UPDATE_ROLLBACK_IN_PROGRESS` | A failure is being undone. Wait. |
| `ROLLBACK_COMPLETE` | Failed create, fully undone. Terminal: delete only. |
| `UPDATE_ROLLBACK_COMPLETE` | Failed update, rolled back to the last working version. Usable. |
| `CREATE_FAILED` / `DELETE_FAILED` / `ROLLBACK_FAILED` / `UPDATE_ROLLBACK_FAILED` | Stuck. Action required. |

Anything `IN_PROGRESS` is transitional: wait, do not re-deploy. `COMPLETE` and `FAILED` are resting states, and note that `ROLLBACK_COMPLETE` is a *complete*: the rollback succeeded. The stack "successfully failed": everything it created was successfully undone. The status describes the last operation, not your happiness with it.

### The first event with a reason is the cause

A failed stack produces a wall of red events, because when one resource fails, CloudFormation cancels everything in flight and rolls back everything already created, and each cancellation and rollback is itself an event. One cause, many consequences.

The method: read the events **in time order** (oldest first: bottom-up in the CLI output, since `describe-stack-events` lists newest first). The first event carrying a failure status reason is the cause. Everything after it is CloudFormation reacting. Here is a deliberate failure (a template given a bucket name that already exists), trimmed and reordered oldest-first:

```text
CREATE_IN_PROGRESS  AWS::CloudFormation::Stack  cfn-fail   User Initiated
CREATE_IN_PROGRESS  AWS::S3::Bucket             SiteBucket
CREATE_FAILED       AWS::S3::Bucket             SiteBucket  s3-site already exists
ROLLBACK_IN_PROGRESS  AWS::CloudFormation::Stack  cfn-fail  The following resource(s) failed to create: [SiteBucket].
DELETE_COMPLETE     AWS::S3::Bucket             SiteBucket
ROLLBACK_COMPLETE   AWS::CloudFormation::Stack  cfn-fail
```

The cause is the third line (*s3-site already exists*), and it names the problem in plain language. Read the error message; it usually says what is wrong. The failure mode to train yourself out of is staring at the loudest, most recent red line, which is almost always a consequence.

### ROLLBACK_COMPLETE locks the name

A stack in `ROLLBACK_COMPLETE` is a dead end: there was never a working version to return to, so it cannot be updated or retried. The stack shell (the record, the events, and crucially the *name*) still exists. Fix your template and redeploy under the same name and you get an error, because the name points at the corpse. The only legal operation is:

```bash
aws cloudformation delete-stack --stack-name cfn-fail
```

Delete first, then redeploy. (A failed *update* is different: `UPDATE_ROLLBACK_COMPLETE` means the stack rolled back to its previous working version and is fine.)

### Keeping a failed resource for inspection

Rollback destroys the evidence: the instance whose logs you wanted is terminated by the time you look. When you are debugging, tell CloudFormation not to clean up:

```bash
aws cloudformation deploy ... --disable-rollback
```

The stack stops where it failed and leaves everything standing, failed resource included, for you to inspect. The lower-level `create-stack` spells the same idea `--on-failure DO_NOTHING` (its other values are `ROLLBACK`, the default, and `DELETE`). Two cautions: the half-built stack is still billing while you investigate (this is a debugging posture, not a default), and delete it when you are done.

### Failures the events cannot explain

The events record what CloudFormation asked AWS to do. Some failures happen below that horizon. The classic: your `UserData` script has a typo (`dnf -y install htppd`) and the stack goes `CREATE_COMPLETE` while the site is dead. No contradiction: CloudFormation asked EC2 for an instance and got one in state `running`. Promise kept. `UserData` runs later, inside the instance, as root, once, at first boot, and nothing reports its exit status back to CloudFormation.

So when the stack is green and the reachability check fails, stop reading stack events; the answer is not there. Get onto the instance. Everything `UserData` printed (every command, every error) is captured in:

```text
/var/log/cloud-init-output.log
```

Read it top to bottom; the first error is the cause: the same method as stack events, one level down. (There is a mechanism for making the stack genuinely wait on UserData: `CreationPolicy` and `cfn-signal`, in the next chapter.)

**Key points**

- Parse statuses as *operation_phase*; `IN_PROGRESS` means wait, `FAILED` means act, and `ROLLBACK_COMPLETE` is a successful undo of a failed create.
- Oldest-first, the first event with a status reason is the cause; the rest are consequences.
- `ROLLBACK_COMPLETE` stacks can only be deleted, and they hold the name until you delete them.
- `deploy --disable-rollback` (or `create-stack --on-failure DO_NOTHING`) preserves the failed resource for inspection, at ongoing cost.
- A green stack with a dead site means UserData: read `/var/log/cloud-init-output.log` on the instance.

## Chapter 5.7 Template features not used in this topic

This topic's templates are deliberately plain. These are the features you will meet in this course and in real templates, in rough order of how soon.

### Conditions and Fn::If

A `Conditions` section defines named booleans from parameter values; resources and properties can then exist conditionally. A common use is making a bucket name optional:

```yaml
Parameters:
  BucketName:
    Type: String
    Default: ""

Conditions:
  HasBucketName: !Not [!Equals [!Ref BucketName, ""]]

Resources:
  TestBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !If [HasBucketName, !Ref BucketName, !Ref "AWS::NoValue"]
```

`!If [Cond, then, else]` picks a value; `!Ref AWS::NoValue` as a branch means "omit the property entirely", so an empty parameter falls back to a generated name. A whole resource can also carry `Condition: HasBucketName` and only exist when the condition holds. One template serving dev (no NAT, small instances) and prod (the works) is the standard use.

### Mappings and Fn::FindInMap

A `Mappings` section is a lookup table baked into the template, read with `!FindInMap [MapName, TopKey, SecondKey]`. The historical use was per-region AMI tables (largely obsolete now that SSM resolution exists), but mappings remain the right tool for small fixed lookups like per-environment sizes:

```yaml
Mappings:
  EnvSizes:
    dev:  { InstanceType: t3.micro }
    prod: { InstanceType: t3.small }
```

`!FindInMap [EnvSizes, !Ref EnvType, InstanceType]` then picks the size.

### Join, Select, GetAZs, ImportValue

- `!Join [",", [a, b, c]]` concatenates a list with a delimiter. Most former uses of `Join` read better as `Sub`; you will still see it everywhere in older templates.
- `!Select [0, list]` picks one element by index.
- `!GetAZs ""` returns the Availability Zones of the current region; `!Select [0, !GetAZs ""]` ("the first AZ, wherever this deploys") is the portable way to place a subnet without hard-coding an AZ name.
- `!ImportValue name` reads a value another stack published with `Export` (Chapter 5.2). Exports are the cross-stack contract: an export name must be unique in the region, and a stack cannot be deleted, nor its export removed, while any other stack imports it; the dependency is enforced, which is the point.

### Metadata, cfn-init, and CreationPolicy

Chapter 5.6 left a loose end: the stack goes green even if UserData fails. The full-dress fix has three parts. The resource's `Metadata` section carries `AWS::CloudFormation::Init`: a declarative description of packages, files, and services, instead of a shell script. The `cfn-init` helper, run from a now-tiny UserData, reads and applies it, and `cfn-signal` reports the result back. A `CreationPolicy` makes CloudFormation *wait* for that signal:

```yaml
  WebServer:
    Type: AWS::EC2::Instance
    CreationPolicy:
      ResourceSignal:
        Count: 1
        Timeout: PT10M
    Properties:
      # ImageId, InstanceType, and the rest as before
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          dnf -y install aws-cfn-bootstrap
          /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} \
            --resource WebServer --region ${AWS::Region}
          /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} \
            --resource WebServer --region ${AWS::Region}
```

No signal within the timeout, or a failure signal, and the resource (and stack) fail instead of lying green. (On AL2023 the helpers are installed with `dnf install aws-cfn-bootstrap`; Amazon Linux 2 shipped them preinstalled.) For this course, plain UserData plus reading the cloud-init log is enough; know that this exists and what it buys.

### Transform, and SAM in one paragraph

A `Transform` line names a macro that rewrites the template before CloudFormation processes it. The one you will actually meet is `Transform: AWS::Serverless-2016-10-31`, the AWS Serverless Application Model (SAM). SAM defines compact resource types like `AWS::Serverless::Function` which the transform expands into plain CloudFormation: the Lambda function, its role, its permissions, its event wiring. The Lambda you built in the storage topic with a function, a role, and a trigger would be a dozen lines of SAM. Under the hood it is all still CloudFormation; `get-template --template-stage Processed` shows you the expanded truth.

### Nested stacks and cross-stack exports

Templates grow, and there are two ways to split them. **Nested stacks**: an `AWS::CloudFormation::Stack` resource whose template lives at an S3 URL; a stack containing stacks, one deployment, parent passes parameters down and reads outputs up. **Cross-stack exports**: independent stacks sharing values through `Export`/`!ImportValue`; separate deployments, separate lifecycles, with the deletion protection described above. Rule of thumb: nested stacks for one system too big for one file; exports for genuinely separate systems that share a boundary, like the VPC and IAM topic's network stack feeding everything deployed into it.

### Custom resources

When CloudFormation has no resource type for what you need, a **custom resource** hands the work to code you write. The template declares `Type: Custom::Whatever` with a `ServiceToken` pointing at a Lambda function; CloudFormation invokes it on create, update, and delete, and the function reports success or failure back. The classic uses are exactly the gaps you will notice in the exercises: there is no resource that puts an *object* into a bucket, and no resource that empties a bucket so it can be deleted. A custom resource can do both, at the cost of owning that code, including its delete path. Know the pattern exists; reach for it late, not early.

**Key points**

- `Conditions` + `!If` + `AWS::NoValue` make properties and resources optional.
- `Mappings`/`!FindInMap` are baked-in lookup tables; SSM resolution replaced their AMI use case.
- `Export`/`!ImportValue` share values across stacks and enforce the dependency; nested stacks compose one system from parts.
- `CreationPolicy` + `cfn-signal` make the stack wait for in-instance work instead of going green on a promise.
- `Transform`/SAM is shorthand that expands to plain CloudFormation; custom resources cover what CloudFormation cannot.

## Chapter 5.8 Operating on stacks

Features for stacks that live longer than a lab session.

**Drift detection.** A stack believes the world matches its template, until someone edits a resource in the console. Drift detection compares belief to reality: `aws cloudformation detect-stack-drift --stack-name cfn-web` starts a detection, `describe-stack-drift-detection-status` reports when it is done, and `describe-stack-resource-drifts` lists each resource as `IN_SYNC`, `MODIFIED`, or `DELETED`, with a property-level diff. Not every resource type supports it, and detection is on demand, not continuous. The discipline it enforces is the real feature: once a resource is in a stack, change it through the stack. Console edits to stacked resources are how production incidents are made: the next update either reverts the console change silently or collides with it.

**Stack policies.** A stack policy is a JSON document that limits what *updates* may do to specific resources, typically denying `Update:Replace` and `Update:Delete` on a database while allowing everything else. Once a policy is set (`set-stack-policy`), all resources are protected by default, so a real policy contains a broad Allow plus targeted Denies. Overriding for a planned migration takes a deliberate, temporary policy override at update time. It protects against accidental updates, not against `delete-stack`.

**Termination protection.** One flag, per stack: `aws cloudformation update-termination-protection --enable-termination-protection --stack-name cfn-web`. While enabled, `delete-stack` fails, full stop; disabling it is a separate explicit call. Cheap insurance on anything that matters, and a deliberate two-step on the way to any deletion.

**Importing an existing resource.** The bridge for infrastructure built by hand, like the server you built in the EC2 topic: a resource can be brought under an existing or new stack's management without recreating it. You write the resource into a template exactly matching its live configuration, give it a `DeletionPolicy` (required for imports, the safety net if the import goes wrong), and run an import-type change set (`create-change-set --change-set-type IMPORT --resources-to-import ...`). In this topic you rebuild rather than import (rebuilding is the skill being taught), but real migrations to CloudFormation lean on imports heavily.

**StackSets.** A StackSet deploys one template to many accounts and regions as one operation (the fleet version of a stack), used for org-wide baselines: audit roles, logging buckets, guardrail alarms in every account. It needs cross-account roles (or Organizations integration) and is how the environment you are sitting in was itself stamped out: one template, one stack per student account. Out of scope to drive in this course; recognize it.

**Key points**

- Drift detection finds console edits to stacked resources; the rule it enforces is "change stacked resources through the stack".
- Stack policies constrain updates; termination protection blocks deletion; both are explicit, deliberate switches.
- Resource import brings hand-built infrastructure under management without recreating it; imports require a `DeletionPolicy`.
- StackSets deploy one template across many accounts and regions.

## Chapter 5.9 Limits and practicalities

**Size limits, and when the template must live in S3.** A template passed directly in the API request body is capped at **51,200 bytes**. Beyond that, the template must be uploaded to S3 and passed by URL, which raises the cap to **1 MB**. The CLI smooths this over: `aws cloudformation deploy --s3-bucket <bucket>` uploads the template and deploys from the URL in one step. Other quotas you will eventually meet: 500 resources, 200 parameters, and 200 outputs per template. A template approaching those numbers is asking to be split (Chapter 5.7). Nothing this course writes gets near any of these limits, but the error, when you first hit it in the field, is confusing unless you know the two-tier rule.

**What CloudFormation does not cover.** Coverage is broad, not total. New services and features often ship without CloudFormation support and grow it later, and some things are modeled deliberately shallowly. You met the sharpest example earlier in this topic: `AWS::S3::Bucket` exists, but there is no resource type for an S3 *object*. The bucket is infrastructure, its contents are data, and uploading `index.html` stays an `aws s3 cp` after the stack completes. Data-plane operations generally (objects, queue messages, table rows) are outside the model. When you need a resource type that does not exist, the escape hatches are, in order: check the CloudFormation registry (AWS and third parties publish additional resource types there), or write a custom resource (Chapter 5.7).

**Retrieving what was actually deployed.** The template on your disk and the template a stack was built from can quietly diverge: you edited the file and never redeployed, or a colleague deployed from another machine. The stack keeps the truth:

```bash
aws cloudformation get-template --stack-name cfn-web
```

returns the exact template body the running stack was deployed from. For transformed templates (Chapter 5.7), `--template-stage Original` returns what you wrote and `--template-stage Processed` what the transform expanded it into. When stack behavior contradicts the file in front of you, this command settles the argument.

**Key points**

- Direct template bodies cap at 51,200 bytes; via S3 the cap is 1 MB; `deploy --s3-bucket` handles the upload.
- 500 resources / 200 parameters / 200 outputs per template; approaching them means splitting the template.
- CloudFormation models infrastructure, not data: buckets yes, objects no. Registry types and custom resources fill gaps.
- `get-template` retrieves the template a stack was *actually* deployed from; it outranks the file on your disk.

## Command reference

Cost note: this topic's resources are one `t3.micro` and its public IPv4 address, each billing per hour, and buckets, whose storage bills per GB-month and rounds to pennies. The habit being built is the same as in every topic: stacks make creation easy, so deletion discipline matters more, not less.

| Command | What it does |
| --- | --- |
| `aws cloudformation validate-template --template-body file://t.yml` | Checks structure and references; does not predict deployment success |
| `aws cloudformation deploy --template-file t.yml --stack-name NAME` | Creates the stack if absent, updates if present, "No changes to deploy" if identical |
| `... deploy --parameter-overrides Key=Value` | Supplies parameter values |
| `... deploy --capabilities CAPABILITY_NAMED_IAM` | Acknowledges IAM resource creation |
| `... deploy --no-execute-changeset` | Builds the change set and stops for review |
| `... deploy --disable-rollback` | On failure, keeps resources standing for inspection |
| `aws cloudformation describe-stacks --stack-name NAME` | Status, description, outputs |
| `aws cloudformation describe-stack-events --stack-name NAME` | Per-resource event history, newest first |
| `aws cloudformation describe-change-set --change-set-name ARN` | Shows Add/Modify/Remove and the Replacement field |
| `aws cloudformation execute-change-set --change-set-name ARN` | Applies a reviewed change set |
| `aws cloudformation delete-stack --stack-name NAME` | Deletes the stack and its non-retained resources |
| `... delete-stack --retain-resources LogicalId` | From `DELETE_FAILED` only: delete the stack, abandon the resource |
| `aws cloudformation get-template --stack-name NAME` | Returns the template the stack was deployed from |
| `aws cloudformation detect-stack-drift --stack-name NAME` | Starts a drift detection |
| `aws cloudformation update-termination-protection --enable-termination-protection --stack-name NAME` | Blocks deletion until disabled |

## Further reading

- **AWS CloudFormation User Guide**: "Template anatomy"; "Parameters" (including the AWS-specific and SSM parameter types); "Intrinsic function reference"; "Pseudo parameters reference".
- **AWS resource and property types reference**: the `AWS::S3::Bucket`, `AWS::S3::BucketPolicy`, `AWS::EC2::Instance`, and `AWS::EC2::SecurityGroup` pages, especially each property's *Update requires* line and each type's *Return values* section.
- **AWS CloudFormation User Guide**: "Update behaviors of stack resources"; "DeletionPolicy attribute"; "Troubleshooting CloudFormation" (the stack status codes table); "Detect unmanaged configuration changes" (drift); "CloudFormation helper scripts reference" (`cfn-init`, `cfn-signal`); "CloudFormation quotas".
- **AWS CLI Command Reference**: the `aws cloudformation` command pages, particularly `deploy`, `validate-template`, `describe-stack-events`, and `delete-stack`.
- **AWS Compute Blog**: "Query for the latest Amazon Linux AMI IDs using AWS Systems Manager Parameter Store".
