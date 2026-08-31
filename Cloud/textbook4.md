# Cloud Foundations: Lambda Functions

The page is served three ways so far: an instance, a bucket, a container on Fargate. This topic adds the strangest shape of compute: a function that does not exist as a running process at all, executes only when something invokes it, and bills only for the milliseconds it runs. It is also where this course's central packaging lesson lands: a dependency does not arrive by magic. The final chapter then puts all four ways side by side; those pages are the ones to remember at the end of the course.

## Chapter 4.1 Lambda: code without a server

Lambda's offer: you write a function (not an application, not a service, a function) and AWS supplies everything beneath it: the machine, the operating system, the language runtime, the scaling, the patching. There is no instance in your account, nothing to SSH into, and no process of yours running between invocations.

Billing follows: per request, plus per GB-second of execution, metered in milliseconds and scaled by the memory setting. A free tier covers both meters, generously enough that this course's Lambda usage will likely cost nothing at all. Contrast this with EC2 in the course's standard terms: an instance is *billed for existing* (every hour, visitors or not) while a function is *billed per use*: zero invocations, zero dollars. No other compute in this course has a true zero for idle.

What Lambda takes in exchange: your code cannot run continuously, cannot listen on a port, and gets at most 15 minutes per invocation. It is not a small EC2; it is a different shape of compute, fitted to work that is *event-shaped*: it happens when something happens, and then it is done.

### The handler contract

The entire interface between your code and the platform is one function:

```python
def lambda_handler(event, context):
    ...
```

You tell Lambda which function this is (the handler setting is a string like `lambda_function.lambda_handler`, file name dot function name), and Lambda calls it once per invocation. `event` is a dict describing why you were called; its shape depends entirely on the event source. `context` is metadata about this invocation: the request ID, the function's configured memory, and `context.get_remaining_time_in_millis()`, which tells you how close the timeout is. The return value is delivered to synchronous callers and recorded-then-discarded for asynchronous ones (S3 events are asynchronous).

Note the inversion from the EC2 topic. Apache ran in a loop, waiting. A Lambda function is *called*: you write the middle of the program and the platform owns the beginning and the end. One structural consequence matters immediately: code at module level (imports, `boto3.client("s3")` and similar setup) runs once when an execution environment starts; the handler then runs once per event, possibly thousands of times in that same environment. Expensive setup belongs at module level. This split is also the mechanics behind cold starts, in Chapter 4.4.

Here is a complete, working handler for this topic's event source, an S3 trigger, showing the two lines of event-parsing that every S3-triggered function begins with:

```python
import urllib.parse

def lambda_handler(event, context):
    record = event["Records"][0]
    bucket = record["s3"]["bucket"]["name"]
    key = urllib.parse.unquote_plus(record["s3"]["object"]["key"])
    size = record["s3"]["object"]["size"]
    print(f"received s3://{bucket}/{key} ({size} bytes)")
    return {"bucket": bucket, "key": key, "size": size}
```

The `unquote_plus` line is explained in the next chapter; leave it out and your function has a bug that only appears for files with spaces in their names.

**Key points**

- You supply a function; AWS supplies machine, OS, runtime, scaling, and patching. Nothing of yours runs between invocations.
- Billed per use (requests plus GB-seconds), with a true zero when idle.
- The handler contract: `handler(event, context)`, configured as `file.function`; the platform calls you.
- Module-level code runs once per environment; the handler runs once per event. Put setup at module level.
- Limits define the shape: no listening on ports, 15 minutes maximum per invocation.

## Chapter 4.2 Events and triggers: an upload as an event

A function that nothing invokes never runs: not once, ever. The design question around every Lambda is therefore: what invokes it? You can invoke by hand (the console's test button, or `aws lambda invoke --function-name <name> --payload '{}' out.json`), and you will, during development. But the point of the platform is **triggers**: standing wiring from an event source to your function, configured once, firing forever. Sources include S3 object events, HTTP requests via an API gateway, queue messages, and schedules. This topic uses the first and most instructive: an upload.

The pipeline: someone PUTs an object into the bucket (console, CLI, presigned URL, it makes no difference). The bucket has an **event notification** configured: on `ObjectCreated` events (optionally filtered by prefix or suffix, such as only keys under `uploads/` ending `.jpg`), invoke this function. Lambda receives the event and calls your handler. A trimmed but structurally faithful event:

```json
{
  "Records": [
    {
      "eventSource": "aws:s3",
      "eventName": "ObjectCreated:Put",
      "awsRegion": "<region>",
      "s3": {
        "bucket": {
          "name": "<upload-bucket>",
          "arn": "arn:aws:s3:::<upload-bucket>"
        },
        "object": {
          "key": "uploads/team+photo.jpg",
          "size": 482091,
          "eTag": "9c1f..."
        }
      }
    }
  ]
}
```

Three things to internalize about this document.

**The event carries coordinates, not content.** Bucket name and key, not bytes. Events are small notes. A function that needs the file's content extracts the coordinates and calls S3 itself, `s3.get_object(Bucket=bucket, Key=key)`, with permissions from its execution role (next chapter).

**The key is URL-encoded.** The file above was named `team photo.jpg`; the event says `team+photo.jpg`. Feed that verbatim to `get_object` and S3 answers `NoSuchKey`: technically true, completely misleading, since the object exists under the decoded name. Hence `urllib.parse.unquote_plus` in every S3 handler. This bug ships in a remarkable number of real systems and reveals itself on the first filename with a space.

**Records is a list.** In practice S3 sends one record per invocation, but the contract is a list, and honest code indexes or iterates it rather than assuming.

### Wiring it, concretely

The trigger can be attached from either end, and both write the same bucket notification configuration. From the function: the function's page in the Lambda console, **Add trigger**, source S3, choose the bucket, event type *All object create events*, and a prefix filter such as `incoming/`. From the bucket: the bucket's **Properties** tab, Event notifications, Create event notification, destination Lambda function. When you attach from the Lambda side, the console quietly does one more thing: it adds a resource-based policy to the function so S3 may invoke it. Doing the same from the CLI takes two calls (`aws s3api put-bucket-notification-configuration` on the bucket and `aws lambda add-permission` on the function), and forgetting the second is a classic silent failure: the notification fires, the invoke is denied, and nothing appears anywhere you are looking.

Verify the wiring the same way every time: upload a file under the filtered prefix, then read the log group. `aws logs tail /aws/lambda/<function-name> --follow` in one terminal while you upload from another is the fastest loop; a new REPORT line per upload means the trigger fires. No log line at all means the invoke never happened: check the trigger, its filter, and where the upload actually landed.

One design warning that costs real money when missed: a function that *writes back to the bucket that triggers it* can invoke itself forever: upload triggers function, function writes an object, the write triggers the function, and so on. The standard defenses are writing output to a different bucket, or scoping the trigger's prefix filter so outputs do not match it. "Look ma, no hands!" takes the second route: the function sorts objects into other prefixes of the very bucket that triggers it, and the `incoming/` prefix filter on the notification is the only thing standing between you and the loop. The exercise notes treat that filter as the safety mechanism it is; extensions like thumbnail generation meet the same hazard.

**Key points**

- A trigger is standing wiring: configured once on the source, it invokes the function per event, forever, with no poller anywhere.
- S3 event notifications fire on object events, filterable by key prefix and suffix.
- The event carries bucket and key, not the object; the key arrives URL-encoded; decode with `unquote_plus`.
- A function writing into its own trigger bucket can recurse indefinitely; separate buckets or prefix filters prevent it.

## Chapter 4.3 Permissions and logs: the execution role and CloudWatch

Your handler calls `s3.get_object` with no credentials anywhere in the code, and no credentials belong there, ever. Permission comes from the **execution role**: an IAM role that the function assumes for the duration of each invocation. Whatever the role's policies allow, the code can do. Everything else is denied: deny by default, the rule you have now met on three services.

The role Lambda offers to create for a new function grants exactly one capability: writing logs (the managed policy behind it is named `AWSLambdaBasicExecutionRole`, which is a misleading name for what is purely a logging permission). A function that must read uploaded objects needs more, added deliberately by you, with a statement like:

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::<upload-bucket>/*"
}
```

Note it is as narrow as the static-website bucket policy from the S3 Storage topic was: one action, one bucket's objects.

Keep two permission questions apart, because they fail apart. *Who may invoke the function?* That is a resource-based policy on the function itself: when you attach an S3 trigger in the console, the console writes this for you, granting the S3 service permission to invoke. *What may the function's code do?* That is the execution role, and nothing writes it for you. The resulting diagnostic: a function that never runs at all has a trigger problem; a function that runs and then dies with `AccessDenied` inside has a role problem. "Look ma, no hands!" contains the second failure by design.

### Where the logs go

No server means no SSH and no log file on a disk, so where did your `print` output go? **CloudWatch Logs**, the managed log service. Every function writes to a log group named `/aws/lambda/<function-name>`; within it, log streams: one per execution environment, each holding many invocations in sequence.

Finding them, two ways. In the console: the function's monitoring tab links directly to its log group: the path of least resistance, and the one to remember when something breaks. From the CLI:

```bash
aws logs tail /aws/lambda/<function-name> --follow
```

which behaves like `tail -f` on a file that lives nowhere and is the fastest debugging loop available in these exercises.

The platform brackets every invocation with `START`, `END`, and a `REPORT` line:

```text
REPORT RequestId: 8f0e...  Duration: 148.21 ms  Billed Duration: 149 ms
Memory Size: 128 MB  Max Memory Used: 41 MB
```

That line is your bill, itemized per invocation, and a free performance profile: duration against billed duration, memory used against memory configured. It is also the input to the memory-dial decision in the next chapter.

Close the loop on permissions: logging works only because the execution role grants it. Strip those permissions and the function fails *silently everywhere*: no logs, because writing the log needed the permission that is missing. A function that appears to produce nothing at all is a role problem until proven otherwise. Prove it, don't assert it: the test button plus the log group answers most Lambda questions in under a minute.

**Key points**

- Code carries no credentials; the execution role defines what the function may do, and the default role can only write logs.
- Who-may-invoke (resource policy, written by the console with the trigger) and what-may-the-code-do (execution role, written by you) are separate and fail separately.
- Logs land in CloudWatch Logs at `/aws/lambda/<name>`; `aws logs tail --follow` is the fast path.
- The REPORT line gives per-invocation duration, billed duration, and memory used.

## Chapter 4.4 The deployment package, and how a library actually arrives

When you deploy a function, what ships is a **deployment package**: a zip of your `.py` files plus everything they import beyond what the runtime provides. The current Python runtime provides the interpreter, the standard library, and **boto3**, the AWS SDK, pre-installed as a deliberate courtesy on the assumption that a Lambda will call AWS. That is the complete list.

Pillow is not on it. Neither is `requests`, nor numpy, nor anything else you have ever installed with pip. `pip install` puts packages into a directory on *your* machine, and your machine does not ship. The image-validation exercise (a function that uses Pillow to verify that an upload is a real image) is scheduled to fail on first invocation with:

```text
Runtime.ImportModuleError: Unable to import module 'lambda_function':
No module named 'PIL'
```

Read it precisely. It does not say your logic is wrong; it says the module is absent from the environment your code landed in. The failure happens at import, before your first line runs, which is why there is no line number. Different problem, different fix, and the error told you which you have. This is the topic's central lesson: **a dependency does not arrive by magic.** Every "works on my machine" in your career will have this anatomy: your machine holding state the target lacks. There are three ways to make the target match.

**Route one: vendor it into the zip.** Install the package into a local directory and zip it together with your code:

```bash
pip install --target ./package pillow
cp lambda_function.py ./package/
cd package && zip -r ../function.zip .
```

The dependency now rides inside the deployment package. One caveat bites with Pillow specifically: it contains compiled C, and the build pip picks for your Windows or macOS laptop will not run on Lambda's Linux. pip can be told to fetch a build for a specific platform; the flags are in pip's documentation for the `install` command, and locating them is part of the exercise. Getting it wrong produces another `ImportModuleError`, this time about a binary module rather than a missing one (different error text, worth actually reading). Limits: 50 MB zipped for direct upload, 250 MB unpacked.

**Route two: a layer.** A **layer** is a zip of dependencies published as its own versioned resource and attached to functions, up to five per function. Lambda unpacks layers alongside your code; for Python the layer zip must place packages under a top-level `python/` directory, the layout the runtime adds to its import path. The payoffs: your deployment package shrinks back to your code only, and one `pillow-layer` serves every function on the team instead of five vendored copies drifting apart. The unpacked total of function plus layers must still fit in 250 MB. One person builds the layer correctly once (same platform caveat as vendoring) and everyone else attaches it by ARN.

**Route three: a container image.** Give up on zips and ship the entire filesystem (base runtime, dependencies, code) as a container image in ECR, up to 10 GB. Full control, standard container tooling, and the natural landing place when dependency weight outgrows zips (scientific stacks, native libraries, model files). It is also the conceptual bridge to the next chapter, because containers solve this exact problem for every kind of compute, not only Lambda.

Choosing: for one function with one library, vendor it; that is this topic's exercise. Several functions sharing dependencies call for a layer. Heavy or exotic dependencies call for an image.

### The dials you cannot see from the code: cold starts, memory, concurrency

Three runtime behaviors are invisible in the source and worth knowing before they surprise you.

**Cold starts.** The first invocation in a fresh execution environment pays for creating it: fetching your package, starting the runtime, running your module-level code. That is a **cold start**: typically well under a second for a small Python package, growing with package size (another reason to keep zips small) and with slow setup code. The environment is then kept warm and reused, which is exactly why expensive setup belongs at module level. You do not control when environments appear or retire; idle functions eventually go cold again. For a bursty image validator the occasional extra half-second is irrelevant; for a latency-sensitive API the paid fix (pre-provisioned warm environments) is called provisioned concurrency, a name to know and no more.

**Memory and timeout: one dial, not two settings.** Timeout: default 3 seconds, maximum 15 minutes; exceed it and the invocation is killed mid-flight, the REPORT line pinned at the limit. Memory (128 MB to 10 GB) allocates memory *and CPU together*: CPU scales in proportion, reaching roughly one full vCPU near 1.8 GB. For CPU-bound work like decoding an image, more memory means faster runs, and since the bill is memory times duration, twice the memory finishing in well under half the time costs *less*. The REPORT line supplies both inputs to that arithmetic; treat memory as a performance dial and let measurements, not instinct, set it. Prove it, don't assert it.

**Concurrency, and the limit.** Each in-flight event occupies one execution environment; a thousand simultaneous uploads means a thousand environments, scaled without your involvement. The ceiling is the account's regional concurrency quota (1,000 by default, though new accounts may start lower), shared by every function in the region. At the ceiling, further invocations are **throttled**: synchronous callers get a rate-exceeded error back immediately, theirs to retry; asynchronous invocations (your S3 events) are retried by Lambda with backoff. The sharp edge in that kindness: failed async events are also retried, up to twice, even when the failure is your bug. A handler that always throws runs three times per upload, and the log group shows every attempt. Because the pool is shared, one runaway function can starve every other in the account; the fix, reserved concurrency, is a name to know, not this topic's material.

**Key points**

- The Python runtime ships the standard library and boto3; everything else must travel with your code.
- `Runtime.ImportModuleError` means the environment lacks a module, not that your logic is wrong; with compiled packages, the bundled build must match Lambda's Linux.
- Vendored zip for one function, a layer (top-level `python/` directory, five max, shared by ARN) for many, a container image for heavyweights.
- Cold starts tax the first invocation per environment; module-level setup is amortized across reuses.
- Memory is a combined memory-and-CPU dial; tuning it can cut both latency and cost. Timeout caps at 15 minutes.
- At the regional concurrency limit, sync callers get throttling errors; async events (S3) are retried automatically, including failures caused by your own bug.

## Chapter 4.5 Choosing between the three

You can now serve the same page four ways: the EC2 instance, a bucket, a container on Fargate, a function behind some HTTP front. None is best. They differ on three axes; "What could possibly go wrong?", back in the S3 Storage topic, asked you to argue two of them in writing, and this chapter gives the full frame.

### What you manage

Under every workload runs the same stack: hardware, OS, language runtime, dependencies, application. The only question that varies is where the line sits between your side and AWS's side.

|  | Hardware | OS | Runtime | Dependencies | Your code/content |
| --- | --- | --- | --- | --- | --- |
| EC2 | AWS | **you** | **you** | **you** | **you** |
| Fargate | AWS | AWS | **you** (in image) | **you** (in image) | **you** |
| Lambda | AWS | AWS | AWS | **you** | **you** |
| S3 static site | AWS | AWS | AWS | none | **you** |

Read the table as a patching schedule. On EC2, the latest Apache CVE is yours to patch. On Fargate the machine is never yours, but everything inside the image is: when a base-image CVE lands, *you* rebuild and repush. On Lambda, AWS rotates the runtime under you; your estate is your code and, as Chapter 4.4 demonstrated, your dependencies. On S3, your estate is the content.

Two corollaries. You never manage *less*, you manage *higher*: failures move up with you, from kernel panics toward misconfigured policies. And each step up surrenders control: the instance runs anything forever; the function runs one blessed runtime for fifteen minutes. "Serverless" is not a virtue in itself; the working question is *which layers did I stop managing, and did I need them?*

### What each costs when nothing is happening

The key table of the topic. Monthly, zero visitors:

| Approach | Doing nothing | The bill | Model |
| --- | --- | --- | --- |
| EC2 (t3.micro + IPv4 + 8 GB EBS) | full price | the same as under load | billed for existing |
| Fargate (0.25 vCPU / 0.5 GB + IPv4) | full price | a shade above the EC2 instance | billed for existing (while the task runs) |
| Lambda | nothing | zero | billed per use |
| S3 site (1 MB of page) | storage only | rounds to zero | billed per use (plus storage) |

The instance and the Fargate task land in the same ballpark, idle, for nobody: both are running things, and running things bill for existing. Nothing stops charging because you stopped looking at it. Lambda is the only true zero on the table, and the S3 site rounds to one.

Idle is half the story; invert the table for the other half. At sustained heavy traffic, always-on wins: a Lambda handling millions of requests per day can out-cost the instance that would have handled them flat-rate, and per-request S3 pennies become real at scale. The durable pattern: spiky and idle-heavy favors per-use; steady and heavy favors always-on. The comparison exercise has you put numbers on this for the EC2 server against the S3 bucket, with current rates from the AWS Pricing Calculator (calculator.aws).

### How each one fails

The neglected axis, and often the deciding one: what does 3 a.m. look like?

**EC2:** one process on one machine. Apache dies, the disk fills, the kernel panics: the page is down until a human acts, and nothing tells you unless you built the telling. That is precisely why the EC2 exercises had you write the pass-or-fail reachability check: it was your monitoring. Failure is total and silent by default.

**Lambda:** failure is scoped to one invocation. A malformed image kills one event; the next upload gets a clean invocation. S3-triggered invocations are asynchronous, so Lambda retries failures automatically (up to twice), which is resilience against transient errors and triple execution of your permanent bugs. Overload does not crash anything; it throttles at the concurrency limit and, for async sources, queues and retries. Failures become log entries.

**Fargate:** the container exits and the task stops. A bare task (the Fargate exercise) stays stopped, with the reason recorded on the stopped task (image pull failure, application exit code, out-of-memory); reading it is the whole diagnostic method. Under a **service**, the orchestrator notices and starts a replacement: restart-on-failure as configuration instead of a 3 a.m. human. The characteristic risk moves up a level: every task runs the same image, so a bad image fails everywhere at once, identically.

**S3:** nothing of yours runs, so nothing of yours crashes. The failure modes are refusals (403 and 404 born of configuration and policy), and they cluster at setup time, not at 3 a.m. Behind the scenes, the service's own redundancy handles the hardware.

The pattern: managed platforms convert outages into log entries and restarts. What no platform takes over is *noticing*, which is why the reachability check keeps its job throughout the course regardless of what serves the page, and why the CloudFormation topic will run that same check against infrastructure built by CloudFormation.

### Matching the job

The compressed decision rule: static content wants storage; there is no cheaper or lower-maintenance way to serve bytes. Event-shaped work (runs when something happens, done in seconds to minutes) wants a function: pay per firing, patch nothing. A long-running server process with its own stack wants a container on managed compute: the environment travels with the code, and nobody owns a machine. The instance remains right when you need the machine itself: GPUs, odd protocols, kernel control, software older than all of this.

These compose rather than compete. This topic's exercises are already a composition: a bucket holding uploads, a function validating them, a container serving pages. The group project hands you a requirements document and this menu, with no further hints.

**Key points**

- The management line moves up the stack, never away: EC2 from the OS up, Fargate the image, Lambda the function and its dependencies, S3 the content.
- Idle costs split the world: running things (EC2, Fargate) bill for existing even at zero traffic; Lambda idles at zero; S3 at storage only.
- Steady heavy load inverts the ranking: per-use pricing loses to flat-rate at volume.
- Failure shapes differ: total-and-silent (EC2), per-event with retries (Lambda), restart-by-configuration (Fargate under a service), refusals-at-setup (S3).
- Monitoring is never included. The reachability check stays employed.

## Command reference

| Command | What it does |
| --- | --- |
| `aws lambda invoke --function-name <name> --payload '{}' out.json` | invoke a function by hand |
| `aws lambda update-function-code --function-name <name> --zip-file fileb://function.zip` | deploy a new package |
| `aws lambda publish-layer-version --layer-name <name> --zip-file fileb://layer.zip` | publish a layer version |
| `aws logs tail /aws/lambda/<name> --follow` | live-tail a function's logs |

## Further reading

- **AWS Lambda Developer Guide**: "Building Lambda functions with Python" (handler, deployment packages, layers); "Lambda execution role"; "Using AWS Lambda with Amazon S3"; "Configuring function memory and timeout"; "Understanding Lambda function scaling".
- **pip documentation**: the `install` command's platform options (the flags the Pillow packaging exercise sends you to find).
