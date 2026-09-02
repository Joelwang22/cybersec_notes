# Cloud Module Command Reference

Use this file beside the audio tracks. Commands assume PowerShell, AWS CLI version 2, and a configured default Region.

## Conventions and safety

- Commands labelled **read-only** inspect state.
- Commands labelled **changes state** create or modify resources and may incur charges.
- Commands labelled **destructive** delete, terminate, or release resources. Verify variables immediately before running them.
- Replace every `<placeholder>` before running a command. Do not type the angle brackets.
- Do not add `--region` routinely. Check the configured default and override only for an intentional cross-Region test.
- Never paste access keys, private-key contents, session tokens, or presigned URLs into a submission or group chat.

## 1. AWS and EC2

### Identity, configuration, and inventory

**Read-only: confirm the CLI and current identity.**

```powershell
aws --version
aws sts get-caller-identity
aws configure list
aws configure get region
```

**Read-only: inspect Regions, zones, VPCs, subnets, and key pairs.**

```powershell
aws ec2 describe-regions `
  --query "Regions[].RegionName" `
  --output table

aws ec2 describe-availability-zones `
  --filters "Name=state,Values=available" `
  --query "AvailabilityZones[].{Zone:ZoneName,ZoneId:ZoneId}" `
  --output table

aws ec2 describe-vpcs `
  --query "Vpcs[].{VpcId:VpcId,CIDR:CidrBlock,Default:IsDefault}" `
  --output table

aws ec2 describe-subnets `
  --query "Subnets[].{SubnetId:SubnetId,VpcId:VpcId,CIDR:CidrBlock,AZ:AvailabilityZone,PublicIpDefault:MapPublicIpOnLaunch}" `
  --output table

aws ec2 describe-key-pairs `
  --query "KeyPairs[].{Name:KeyName,Type:KeyType,Fingerprint:KeyFingerprint}" `
  --output table
```

**Read-only: resolve the current Amazon Linux 2023 x86-64 AMI.**

```powershell
$amiId = aws ssm get-parameter `
  --name "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64" `
  --query "Parameter.Value" `
  --output text
$amiId
```

**Read-only: inspect instances and one selected instance.**

```powershell
aws ec2 describe-instances `
  --query "Reservations[].Instances[].{Id:InstanceId,State:State.Name,Type:InstanceType,AZ:Placement.AvailabilityZone,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress,Subnet:SubnetId,Key:KeyName}" `
  --output table

$instanceId = "<instance-id>"
aws ec2 describe-instances `
  --instance-ids $instanceId `
  --query "Reservations[0].Instances[0]" `
  --output json
```

### Reachability and lifecycle

**Read-only: test an HTTP endpoint and prove the exit code.**

```powershell
$url = "http://<public-dns-name>"
curl.exe -sf --max-time 10 -o NUL "$url/"
"Exit code: $LASTEXITCODE"
```

Exit code `0` means curl received a successful HTTP response. HTTP status `000` means no HTTP response was received.

**Changes state: stop and wait.**

```powershell
aws ec2 stop-instances --instance-ids $instanceId
aws ec2 wait instance-stopped --instance-ids $instanceId
```

**Changes state: start and wait.**

```powershell
aws ec2 start-instances --instance-ids $instanceId
aws ec2 wait instance-running --instance-ids $instanceId
```

**Destructive: terminate only the verified instance.**

```powershell
$instanceId
aws ec2 terminate-instances --instance-ids $instanceId
aws ec2 wait instance-terminated --instance-ids $instanceId
```

**Read-only: prove no instances are running and check common leftovers.**

```powershell
aws ec2 describe-instances `
  --filters "Name=instance-state-name,Values=running" `
  --query "Reservations[].Instances[].InstanceId"

aws ec2 describe-volumes `
  --query "Volumes[].{Id:VolumeId,State:State,SizeGiB:Size,Type:VolumeType,AttachedTo:Attachments[0].InstanceId}" `
  --output table

aws ec2 describe-addresses `
  --query "Addresses[].{AllocationId:AllocationId,PublicIp:PublicIp,InstanceId:InstanceId,AssociationId:AssociationId}" `
  --output table
```

### Secure Shell, Session Manager, and metadata

```powershell
ssh -i "C:\path\to\<key-name>.pem" ec2-user@<public-dns-or-ip>

aws ssm start-session --target $instanceId
```

Run these metadata commands inside an EC2 Linux instance:

```bash
TOKEN=$(curl -sS -X PUT \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" \
  http://169.254.169.254/latest/api/token)

curl -sS \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

## 2. S3 Storage

### Create, upload, inspect, and test

**Changes state: create a globally unique bucket.**

```powershell
$bucketName = "<unique-lowercase-bucket-name>"
$bucketName
aws s3 mb "s3://$bucketName"
```

**Changes state: upload and synchronize content.**

```powershell
aws s3 cp .\index.html "s3://$bucketName/index.html"
aws s3 sync .\site "s3://$bucketName/"
```

**Read-only: list and inspect one object.**

```powershell
aws s3 ls "s3://$bucketName/" --recursive

aws s3api head-object `
  --bucket $bucketName `
  --key index.html
```

**Changes state: enable website hosting.**

```powershell
aws s3 website "s3://$bucketName/" --index-document index.html
```

**Read-only: inspect website configuration and Block Public Access.**

```powershell
aws s3api get-bucket-website --bucket $bucketName
aws s3api get-public-access-block --bucket $bucketName
```

### Policy-based public website access

Save the following as `public-read-policy.json`, replacing the bucket placeholder:

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

**Changes state: keep ACL blocks on while permitting a public bucket policy.**

```powershell
aws s3api put-public-access-block `
  --bucket $bucketName `
  --public-access-block-configuration `
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=false,RestrictPublicBuckets=false"

aws s3api put-bucket-policy `
  --bucket $bucketName `
  --policy file://public-read-policy.json
```

**Read-only: inspect policy and test the website.**

```powershell
aws s3api get-bucket-policy `
  --bucket $bucketName `
  --query Policy `
  --output text

$websiteUrl = "http://<bucket-website-endpoint>"
curl.exe -i "$websiteUrl/"
curl.exe -i "$websiteUrl/nothing-here.html"
```

### Versioning, lifecycle, CORS, and sharing

**Changes state: enable versioning.**

```powershell
aws s3api put-bucket-versioning `
  --bucket $bucketName `
  --versioning-configuration Status=Enabled
```

Save a lifecycle configuration as `lifecycle.json`:

```json
{
  "Rules": [
    {
      "ID": "expire-noncurrent-versions",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 30},
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }
  ]
}
```

```powershell
aws s3api put-bucket-lifecycle-configuration `
  --bucket $bucketName `
  --lifecycle-configuration file://lifecycle.json
```

Save a CLI CORS configuration as `cors.json`:

```json
{
  "CORSRules": [
    {
      "AllowedOrigins": ["https://<your-web-origin>"],
      "AllowedMethods": ["GET", "PUT"],
      "AllowedHeaders": ["*"],
      "ExposeHeaders": ["ETag"],
      "MaxAgeSeconds": 3000
    }
  ]
}
```

```powershell
aws s3api put-bucket-cors `
  --bucket $bucketName `
  --cors-configuration file://cors.json

aws s3 presign "s3://$bucketName/<key>" --expires-in 900
```

**Destructive: empty and delete only a verified, disposable bucket.**

```powershell
$bucketName
aws s3 rm "s3://$bucketName/" --recursive
aws s3 rb "s3://$bucketName"
```

Versioned buckets require deletion of all object versions and delete markers before removal.

## 3. ECR and Fargate

### Build and test an x86-64 image

```powershell
docker version

docker build `
  --platform linux/amd64 `
  -t static-page:latest `
  .

docker image inspect static-page:latest `
  --format '{{.Os}}/{{.Architecture}}'

docker run --rm -d `
  --name static-page-test `
  -p 8080:80 `
  static-page:latest

curl.exe -i http://localhost:8080/
docker logs static-page-test
docker stop static-page-test
```

### Login, tag, push, and inspect

**Changes state: create and populate a private repository.**

```powershell
$accountId = aws sts get-caller-identity --query Account --output text
$region = aws configure get region
$registry = "${accountId}.dkr.ecr.${region}.amazonaws.com"
$imageUri = "${registry}/static-page:latest"

aws ecr create-repository --repository-name static-page

aws ecr get-login-password |
  docker login --username AWS --password-stdin $registry

docker tag static-page:latest $imageUri
docker push $imageUri
```

**Read-only: inspect tags, digests, compressed sizes, and push times.**

```powershell
aws ecr describe-images `
  --repository-name static-page `
  --query "imageDetails[].{Tags:imageTags,Digest:imageDigest,SizeBytes:imageSizeInBytes,PushedAt:imagePushedAt}" `
  --output table
```

### Inspect ECS and Fargate

```powershell
aws ecs list-clusters
aws ecs list-task-definitions --family-prefix static-page
aws ecs describe-task-definition --task-definition static-page

aws ecs list-tasks --cluster <cluster-name>
aws ecs describe-tasks `
  --cluster <cluster-name> `
  --tasks <task-arn-or-id> `
  --query "tasks[].{LastStatus:lastStatus,DesiredStatus:desiredStatus,StoppedReason:stoppedReason,Containers:containers[].{Name:name,ExitCode:exitCode,Reason:reason}}"

aws logs tail "/ecs/static-page" --follow
```

**Changes state: stop a verified one-off task.**

```powershell
aws ecs stop-task `
  --cluster <cluster-name> `
  --task <task-arn-or-id> `
  --reason "Course teardown"
```

## 4. Lambda Functions

### Inspect, deploy, invoke, and read logs

```powershell
aws lambda get-function --function-name <function-name>
aws lambda get-function-configuration --function-name <function-name>
aws lambda get-policy --function-name <function-name>
aws lambda list-event-source-mappings --function-name <function-name>
```

**Changes state: deploy a ZIP whose handler is at the archive root.**

```powershell
Compress-Archive `
  -Path .\lambda_function.py `
  -DestinationPath .\function.zip `
  -Force

aws lambda update-function-code `
  --function-name <function-name> `
  --zip-file fileb://function.zip
```

**Read-only plus invocation charge: invoke a function manually.**

```powershell
aws lambda invoke `
  --function-name <function-name> `
  --cli-binary-format raw-in-base64-out `
  --payload '{}' `
  .\response.json

Get-Content .\response.json
```

```powershell
aws logs tail "/aws/lambda/<function-name>" --since 10m
aws logs tail "/aws/lambda/<function-name>" --follow
```

### Package a third-party Python dependency

Use a Python version supported by the selected Lambda runtime. For an x86-64 Linux function, install compatible binary wheels into the package directory:

```powershell
New-Item -ItemType Directory -Path .\package

py -m pip install `
  --platform manylinux2014_x86_64 `
  --implementation cp `
  --python-version <runtime-major-minor-without-dot> `
  --only-binary=:all: `
  --target .\package `
  Pillow

Copy-Item .\lambda_function.py .\package\lambda_function.py

Compress-Archive `
  -Path .\package\* `
  -DestinationPath .\function.zip `
  -Force

tar -tf .\function.zip |
  Select-String "^(lambda_function.py|PIL/__init__.py)$"
```

For Python 3.12, replace the runtime placeholder with `312`. Match ARM functions with ARM-compatible wheels instead.

### Inspect S3 trigger wiring

```powershell
aws s3api get-bucket-notification-configuration --bucket <bucket-name>
aws lambda get-policy --function-name <function-name>
```

**Changes state: allow one bucket to invoke the function.**

```powershell
aws lambda add-permission `
  --function-name <function-name> `
  --statement-id AllowS3Invoke `
  --action lambda:InvokeFunction `
  --principal s3.amazonaws.com `
  --source-arn "arn:aws:s3:::<bucket-name>" `
  --source-account $accountId
```

## 5. CloudFormation

### Validate, deploy, and retrieve outputs

```powershell
aws cloudformation validate-template `
  --template-body file://template.yml

aws cloudformation deploy `
  --template-file .\template.yml `
  --stack-name <stack-name> `
  --parameter-overrides `
    KeyName=<key-pair-name> `
    AllowedSSH=<your-public-ip>/32
```

When the template creates named IAM resources, append `--capabilities CAPABILITY_NAMED_IAM` to the deploy command.

```powershell
aws cloudformation describe-stacks `
  --stack-name <stack-name> `
  --query "Stacks[0].{Status:StackStatus,Outputs:Outputs}" `
  --output json

$url = aws cloudformation describe-stacks `
  --stack-name <stack-name> `
  --query "Stacks[0].Outputs[?OutputKey=='URL'].OutputValue" `
  --output text

curl.exe -sf --max-time 10 -o NUL "$url/"
"Exit code: $LASTEXITCODE"
```

### Preview and diagnose updates

```powershell
aws cloudformation deploy `
  --template-file .\template.yml `
  --stack-name <stack-name> `
  --no-execute-changeset

aws cloudformation list-change-sets --stack-name <stack-name>

aws cloudformation describe-stack-events `
  --stack-name <stack-name> `
  --query "StackEvents[].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]" `
  --output table

aws cloudformation get-template --stack-name <stack-name>
```

```powershell
$driftId = aws cloudformation detect-stack-drift `
  --stack-name <stack-name> `
  --query StackDriftDetectionId `
  --output text

aws cloudformation describe-stack-drift-detection-status `
  --stack-drift-detection-id $driftId

aws cloudformation describe-stack-resource-drifts `
  --stack-name <stack-name>
```

### Delete and wait

**Destructive: delete only the verified stack.**

```powershell
$stackName = "<stack-name>"
$stackName
aws cloudformation delete-stack --stack-name $stackName
aws cloudformation wait stack-delete-complete --stack-name $stackName
```

If a first creation ended in `ROLLBACK_COMPLETE`, delete that stack record before reusing the name.

## 6. VPC and IAM

### Trace the actual network path

```powershell
$instanceId = "<instance-id>"

$instance = aws ec2 describe-instances `
  --instance-ids $instanceId `
  --query "Reservations[0].Instances[0]" `
  --output json |
  ConvertFrom-Json

$subnetId = $instance.SubnetId
$vpcId = $instance.VpcId
$securityGroupIds = @($instance.SecurityGroups | ForEach-Object GroupId)

$instance | Select-Object `
  InstanceId, `
  @{Name='State';Expression={$_.State.Name}}, `
  PrivateIpAddress, `
  PublicIpAddress, `
  SubnetId, `
  VpcId
```

```powershell
aws ec2 describe-security-groups `
  --group-ids $securityGroupIds `
  --query "SecurityGroups[].{Name:GroupName,Id:GroupId,Inbound:IpPermissions,Outbound:IpPermissionsEgress}"

aws ec2 describe-network-acls `
  --filters "Name=association.subnet-id,Values=$subnetId" `
  --query "NetworkAcls[].Entries"

aws ec2 describe-route-tables `
  --filters "Name=association.subnet-id,Values=$subnetId" `
  --query "RouteTables[].{Id:RouteTableId,Routes:Routes,Associations:Associations}"

aws ec2 describe-internet-gateways `
  --filters "Name=attachment.vpc-id,Values=$vpcId" `
  --query "InternetGateways[].{Id:InternetGatewayId,Attachments:Attachments}"
```

If the route-table query returns nothing, inspect the VPC's main route table:

```powershell
aws ec2 describe-route-tables `
  --filters `
    "Name=vpc-id,Values=$vpcId" `
    "Name=association.main,Values=true" `
  --query "RouteTables[].{Id:RouteTableId,Routes:Routes,Associations:Associations}"
```

### Inspect identity and policy attachment

```powershell
aws sts get-caller-identity

aws iam get-role --role-name <role-name>
aws iam list-attached-role-policies --role-name <role-name>
aws iam list-role-policies --role-name <role-name>

aws iam get-role-policy `
  --role-name <role-name> `
  --policy-name <inline-policy-name>
```

**Read-only simulation: predict actions without performing them.**

```powershell
aws iam simulate-principal-policy `
  --policy-source-arn "arn:aws:iam::<account-id>:role/<role-name>" `
  --action-names s3:GetObject s3:PutObject `
  --resource-arns "arn:aws:s3:::<bucket-name>/<key>" `
  --query "EvaluationResults[].{Action:EvalActionName,Decision:EvalDecision}" `
  --output table
```

### Audit API activity

```powershell
aws cloudtrail lookup-events `
  --lookup-attributes `
    AttributeKey=EventName,AttributeValue=RunInstances `
  --max-results 10 `
  --query "Events[].{Time:EventTime,Event:EventName,Username:Username}" `
  --output table
```

### A reusable end-to-end check

```powershell
$url = aws cloudformation describe-stacks `
  --stack-name <network-stack-name> `
  --query "Stacks[0].Outputs[?OutputKey=='URL'].OutputValue" `
  --output text

curl.exe -sf --max-time 10 -o NUL "$url/"
if ($LASTEXITCODE -eq 0) { "PASS" } else { "FAIL" }
"Exit code: $LASTEXITCODE"
```
