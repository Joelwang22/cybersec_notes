# Cloud Foundations: S3 Storage

In the EC2 topic you built a web server: an EC2 instance, an operating system, Apache, and a page. It works, and it will keep working for as long as you keep it working: patched, running, and paid for. This topic and the two after it (ECR and Fargate, then Lambda Functions) are about handing pieces of that job to AWS, and three recurring questions run through all of them: what do you manage, what does it cost while nothing is happening, and how does it fail? Here the answer starts at the simplest end: the same page served from object storage, with no server anywhere.

## Chapter 2.1 Buckets, objects, and keys

S3 (Simple Storage Service) stores objects in buckets. Those two nouns carry the whole model, so it is worth being precise about both.

A **bucket** is a named container for objects. Its name is unique across every AWS account in the world, not only yours, because the name becomes part of a public DNS hostname. `reports` was claimed sometime around 2006; a name you can actually get looks more like `reports-die4-`. Despite the global namespace, a bucket lives in exactly one region (for this course, always your course region) and its data does not leave that region unless you move it.

An **object** is a key, a blob of bytes, and some metadata (content type, timestamps, and so on). An object can be one byte or five terabytes. The operations available on it are deliberately few: put an object, get an object, delete an object, list keys. That is nearly the entire API surface you will use in this course.

Notice what is missing from that list: open, seek, append, edit. S3 is *object* storage, not the *block* storage you met in the EC2 topic. The EBS volume on that instance is a disk, where a program can seek to byte 4,096 and overwrite three bytes. S3 offers none of that. To change one character of a stored page, you upload the whole page again under the same key, replacing the old object. In exchange you get capacity you never provision, access from anywhere over HTTPS, and the durability numbers in the next chapter. The practical rule: databases and filesystems want block storage; pages, images, uploads, backups, and logs want object storage.

### Keys, prefixes, and the directory illusion

Every object has a **key**, and the key is the entire string, slashes included. The key `photos/2026/cat.jpg` is not a file named `cat.jpg` inside two nested directories. It is one flat name, in one flat namespace, that happens to contain `/` characters.

The console shows you folders anyway, because humans expect them. It builds them on the fly: list all keys beginning with the **prefix** `photos/2026/`, treat `/` as a delimiter, and group what comes back. The API call underneath is `ListObjectsV2` with `prefix` and `delimiter` parameters; a "folder" is a query, not a thing. (Everything in the console is an API call; this one is a nice example of a console feature that exists purely to translate between an API's reality and a human's expectations.)

Consequences that surprise people exactly once:

- There is no rename. "Renaming a folder" of 10,000 objects means 10,000 copies to new keys and 10,000 deletes. The console does this quietly when you ask.
- You cannot make an empty directory, because there are no directories. The console fakes even this, by writing a zero-byte object whose key ends in `/`.
- Deleting a "folder" means deleting every object under the prefix. There is no directory entry to remove.

Prefixes are still worth designing deliberately. They are how you organize a bucket, how you filter listings, how lifecycle rules select objects (Chapter 2.3), and, from the IAM topic onward, how IAM policies scope access to "everything under `uploads/`".

**Key points**

- A bucket is a globally-named container living in one region; an object is a key plus bytes plus metadata.
- The API is put, get, delete, list: whole objects only. No append, no editing in place.
- Keys are flat strings; slashes are convention. Folders are a console illusion built from prefix queries.
- Block storage (EBS) is for filesystems and databases; object storage is for content.

## Chapter 2.2 Serving a static site from a bucket

Your EC2 page is static: Apache read the same bytes from disk for every visitor and wrote them to a socket. S3 can do that part itself, and the first exercise is moving that page into a bucket and retiring the middleman.

The mechanics are short. Static website hosting is a bucket property: you enable it, name an **index document** (the object served when a visitor requests `/`, conventionally `index.html`), and optionally an error document. Upload the page. The bucket then answers plain HTTP at a **website endpoint** with a hostname of this shape:

```text
http://<bucket-name>.s3-website-<region>.amazonaws.com
```

No instance, no operating system, no daemon, no security group, nothing to patch, and nothing that can crash at 3 a.m. Also, honestly: no server-side code (every visitor gets identical bytes) and no HTTPS on this endpoint, which we return to below.

### Public access, and why you are refused by default

Try this and your first reward is `403 Access Denied`. That is not a mistake; it is the lesson. S3 refuses public access twice over, and you must deliberately unlock both.

**Lock one: Block Public Access.** Every new bucket carries a set of four switches, all on, that refuse any configuration that would make the bucket public, including refusing to let you attach a public bucket policy at all. They exist because a decade of data-leak headlines traces back to buckets made public by accident. The switches apply at the account level and at the bucket level. The instructors have already cleared the account-level setting in your sandbox accounts; the bucket-level switches are yours to find and deliberately turn off, reading the warning the console shows when you do. Making you consciously accept "objects can be public" is the feature's entire design intent.

**Lock two: nobody has been granted anything.** With Block Public Access out of the way, the bucket is merely *allowed to become* public. Anonymous visitors still have no permissions: deny by default, the standing rule of this course. The grant is a **bucket policy**: a JSON document attached to the bucket. The complete policy a static website needs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForWebsite",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<your-bucket-name>/*"
    }
  ]
}
```

Read it right to left: on every object in this bucket (`Resource`: the `/*` matters; it targets the objects, not the bucket itself), allow the read-an-object action (`Action`), to everyone including anonymous visitors (`Principal": "*"`). Note how narrow a correct grant is: no `s3:ListBucket`, so nobody browses your keys; no `s3:PutObject`, so nobody uploads. This is your first policy in the IAM language, and the shape (Effect, Principal, Action, Resource) is the same one the IAM topic covers in depth.

The two locks fail differently, and the difference is diagnostic. Attaching a public policy while Block Public Access is on is refused loudly, at the moment you try. A cleared Block Public Access with no policy fails quietly: the site exists and every request gets 403. Read the error message; it tells you which lock you are looking at.

One note so old tutorials do not mislead you: ACLs, the legacy per-object permission mechanism, are disabled entirely on new buckets. A tutorial that says to make an object "public read" via its ACL predates the current defaults; use a bucket policy.

### Website endpoint versus REST endpoint

Every bucket actually answers at two different hostnames, and confusing them produces confusing results.

|  | Website endpoint | REST endpoint |
| --- | --- | --- |
| Hostname | `<bucket>.s3-website-<region>.amazonaws.com` | `<bucket>.s3.<region>.amazonaws.com` |
| Protocol | HTTP only | HTTPS (and HTTP) |
| Request for `/` | serves your index document | error, or a key listing if you allowed one |
| Missing key | serves your error document | XML error response |
| Authenticated requests | not supported (anonymous only) | fully supported (SDK, CLI, presigned URLs) |
| Exists when | website hosting is enabled | always |

The REST endpoint is the API: it is what the CLI, boto3, and presigned URLs talk to, it speaks HTTPS, and its errors are XML documents meant for programs. The website endpoint is a thin veneer for browsers: index documents, friendly error pages, redirects, and no TLS. If you need HTTPS in front of a bucket website, the answer is a CDN (CloudFront) in front of the bucket, which is beyond this course but worth knowing the name of.

Symptoms you will meet in the exercises: page downloads instead of rendering: uploaded without a `text/html` content type (the CLI usually guesses right from the extension; some upload paths do not). XML error instead of your page: you are on the REST endpoint. Browser warns "not secure": the website endpoint is HTTP, as designed.

**Key points**

- Static hosting is a bucket property: index document, upload the page, done. The website endpoint is HTTP only.
- Public access must be unlocked twice: turn off bucket-level Block Public Access, then grant `s3:GetObject` to `*` with a bucket policy.
- The grant is narrow by design: objects only, read only.
- Website endpoint and REST endpoint are different hostnames with different behavior; the REST endpoint is the API, the website endpoint is for browsers.

## Chapter 2.3 Durability, storage classes, versioning, and lifecycle

### Eleven nines

S3 is designed for 99.999999999% durability: eleven nines. Concretely: store ten million objects and statistically expect to lose one every ten thousand years. The mechanism is redundancy: every object is stored across at least three Availability Zones (physically separate datacenters) before the write is acknowledged. Disks die and buildings lose power without your objects noticing.

Two distinctions keep this number honest. First, durability is not **availability**. Durability says the data still exists; availability says you can read it right now, and that number is lower (about 99.99% for the Standard class). A rare failed read is a retry, not a loss. Second, durability protects against AWS's hardware, not against you. A delete is executed with full eleven-nines reliability, and so is an overwrite. The defense against your own mistakes is versioning, below.

### Storage classes

Not all bytes deserve the same price. A **storage class** is a per-object setting that trades storage price against retrieval price and speed:

| Class | Storage price | The trade |
| --- | --- | --- |
| Standard | the baseline | instant, no retrieval fee (the default) |
| Standard-IA | cheaper than Standard | per-GB retrieval fee; 30-day and 128 KB minimum charges |
| One Zone-IA | cheaper than Standard-IA | as IA, but one AZ only (reduced resilience) |
| Glacier Instant Retrieval | cheaper still | still instant, higher retrieval fee |
| Glacier Flexible Retrieval | cheaper again | restores take minutes to hours |
| Glacier Deep Archive | cheapest of all | restores take hours |
| Intelligent-Tiering | varies | moves objects between tiers for you, small monitoring fee |

The rule of thumb: the less often you expect to read an object, the cheaper it should sit. A website read hourly belongs in Standard. A backup read once a year belongs in IA. A compliance archive read never, you hope, belongs in Deep Archive. The "minimum" fine print on the cheaper classes exists to stop people from parking hot data there: IA charges at least 30 days of storage and at least 128 KB per object, so a million tiny hot files in IA costs more than Standard, not less. Current per-GB rates live on the S3 pricing page, and the AWS Pricing Calculator (calculator.aws) turns them into an estimate for a real workload.

Requests are billed separately from storage in every class, per PUT and per GET, and are tiny at course scale. This is the "billed per use" half of the course's cost rule: a bucket holding a 1 MB page costs fractions of a cent per month, and rises only when someone actually uses it.

### Versioning

Versioning is a bucket-level switch. Once enabled, every write to a key creates a new **version** rather than replacing the old bytes, and every version is kept. Overwrite `index.html` five times and the bucket holds six objects under one key; you can list them, fetch any of them, and restore an old one by copying it back to the top.

A delete on a versioned bucket does not remove data either. It writes a **delete marker**: a tombstone that becomes the current version, making the key appear gone while every previous version remains underneath. Deleting the delete marker un-deletes the object. Permanently removing data requires deleting specific version IDs, which is a deliberate, unusual act.

Versioning is the actual answer to "eleven nines will not protect me from myself," and it costs what it costs: every version is a full object billed at full storage rates. A frequently rewritten large object accumulates a bill quietly. Which is why versioning and the next feature are usually enabled together. Enabling versioning is also an in-place change to a live bucket (nothing stored is touched), which makes it the gentle kind of update; the CloudFormation topic has a whole exercise on which changes to a managed resource happen in place and which force a replacement.

### Lifecycle rules

A **lifecycle rule** is standing instruction attached to the bucket: objects matching a filter transition between storage classes or expire on a schedule. Written once, applied forever, no cron server anywhere. A complete configuration you could attach right now:

```json
{
  "Rules": [
    {
      "ID": "tidy-uploads",
      "Status": "Enabled",
      "Filter": { "Prefix": "uploads/" },
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" }
      ],
      "Expiration": { "Days": 365 },
      "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
    }
  ]
}
```

In words: everything under `uploads/` moves to Standard-IA after 30 days, is deleted after a year, and old versions (the noncurrent ones that versioning accumulates) are removed 30 days after being superseded. That last rule is the standard companion to versioning: recent mistakes stay recoverable, ancient versions stop billing. Applied with `aws s3api put-bucket-lifecycle-configuration --bucket  --lifecycle-configuration file://lifecycle.json`.

**Key points**

- Eleven nines of durability, via copies in three or more AZs. Durability is not availability, and neither protects you from your own deletes.
- Storage classes trade storage price against retrieval price and speed; minimum-duration and minimum-size charges keep hot data out of cold classes.
- Versioning keeps every version of every key; deletes become markers; every kept version bills as a full object.
- Lifecycle rules automate transitions and expiry per prefix, including expiring old versions.

## Chapter 2.4 Letting others in: presigned URLs and CORS

The bucket policy in Chapter 2.2 opened a bucket to the whole world, permanently, read-only. Most sharing is not like that. "This one person may download this one object for the next ten minutes" (or upload one) is the far more common need, and it has a mechanism that requires no policy changes at all.

### Presigned URLs

Every S3 request is an API call that must be signed by some identity's credentials. Normally the SDK signs each request invisibly as you go. A **presigned URL** is such a signature computed in advance and written into a URL: it names the bucket, the key, the operation, an expiry time, and a cryptographic signature made with *your* credentials. Anyone holding the URL can perform exactly that operation on exactly that object until exactly that time: no AWS account, no credentials of their own. The URL *is* the credential. A presigned URL is an API call, written down.

For downloads the CLI does it in one line:

```bash
aws s3 presign s3://<your-bucket>/report.pdf --expires-in 900
```

which prints something of this shape (one line, wrapped here):

```text
https://<your-bucket>.s3.<region>.amazonaws.com/report.pdf
  ?X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Credential=ASIA...%2F<region>%2Fs3%2Faws4_request
  &X-Amz-Date=20260817T101500Z
  &X-Amz-Expires=900
  &X-Amz-SignedHeaders=host
  &X-Amz-Signature=8f3a1c...
```

Note the hostname: presigned URLs use the REST endpoint, so they are HTTPS. Anyone can open that link in a browser for the next 900 seconds; afterward it returns an expired-signature error.

`aws s3 presign` only produces download (GET) URLs. Upload URLs come from the SDK. Both directions in boto3:

```python
import boto3

s3 = boto3.client("s3")

# Download: holder may GET this object for 15 minutes
download_url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": "<your-bucket>", "Key": "report.pdf"},
    ExpiresIn=900,
)

# Upload: holder may PUT one object at exactly this key
upload_url = s3.generate_presigned_url(
    "put_object",
    Params={"Bucket": "<your-bucket>", "Key": "incoming/photo.jpg"},
    ExpiresIn=900,
)
```

The upload URL is exercised with an HTTP PUT (for instance `curl -X PUT --upload-file photo.jpg ""`), and the object appears at the key the URL was signed for. The uploader chooses nothing: not the bucket, not the key, not the expiry. This is how browsers upload directly to S3 without the application server ever touching the bytes, and it is exactly the pattern the group project uses for capture files.

Two properties matter more than they first appear. A presigned URL can never do more than the identity that signed it: it borrows your permissions, checked at use time, so signing a URL for a bucket you cannot read produces a well-formed URL that yields 403. And the URL is only as durable as the signing credentials: your sandbox credentials come from an SSO session that expires after a few hours, and every URL signed with them dies when the session does, whatever `--expires-in` said.

### Cross-origin requests from a browser

The moment a page served from one origin uses JavaScript to fetch from another (your S3 website calling the REST endpoint, or a future web app PUTting to a presigned URL), you meet the browser's **same-origin policy**. An origin is scheme + hostname + port; your website endpoint and your REST endpoint are different origins even though they front the same bucket. By default the browser blocks scripted cross-origin reads, and the failure appears in the browser console mentioning CORS, while the same URL works perfectly in `curl`. That contrast is the diagnostic: CORS is enforced by the browser, on the server's instruction.

CORS (cross-origin resource sharing) is how a server opts in. For S3 it is a bucket-level configuration:

```json
[
  {
    "AllowedOrigins": ["http://<your-site-bucket>.s3-website-<region>.amazonaws.com"],
    "AllowedMethods": ["GET", "PUT"],
    "AllowedHeaders": ["*"],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3000
  }
]
```

That bare array is what the console's CORS editor expects; the CLI (`aws s3api put-bucket-cors`) wants the same rules wrapped in a top-level object, as `{"CORSRules": [ ... ]}`; feed it the bare array and it rejects the file.

This tells browsers: pages from that one origin may make GET and PUT requests here. For anything beyond a plain GET (a PUT, or custom headers), the browser first sends an `OPTIONS` "preflight" request asking permission, and S3 answers from this configuration. No configuration, no permission, no request.

CORS is not authentication. It controls which *web pages* a browser will let talk to the bucket; it grants nobody any S3 permission. A locked-down bucket with permissive CORS is still locked down. You will not need CORS for this topic's exercises; you will need it in the group project, when a browser uploads captures via presigned URLs, and the error message will send you back to this section.

**Key points**

- A presigned URL embeds a signed, time-limited grant for one operation on one object; the holder needs no AWS credentials.
- `aws s3 presign` covers downloads; uploads need the SDK (`generate_presigned_url("put_object", ...)`) and an HTTP PUT.
- A URL borrows the signer's permissions at use time, and dies with the signer's temporary credentials regardless of its own expiry.
- CORS is a bucket configuration that tells browsers which origins may script requests against it; it is not access control.

## Command reference

| Command | What it does |
| --- | --- |
| `aws s3 mb s3://<bucket>` | create a bucket |
| `aws s3 cp <file> s3://<bucket>/<key>` | upload one object |
| `aws s3 sync <dir> s3://<bucket>/` | upload a directory's contents |
| `aws s3 website s3://<bucket> --index-document index.html` | enable static website hosting |
| `aws s3api put-public-access-block --bucket <bucket> ...` | set/clear the bucket's Block Public Access switches |
| `aws s3api put-bucket-policy --bucket <bucket> --policy file://policy.json` | attach a bucket policy |
| `aws s3api put-bucket-versioning --bucket <bucket> --versioning-configuration Status=Enabled` | turn on versioning |
| `aws s3api put-bucket-lifecycle-configuration --bucket <bucket> --lifecycle-configuration file://rules.json` | attach lifecycle rules |
| `aws s3api put-bucket-cors --bucket <bucket> --cors-configuration file://cors.json` | attach a CORS configuration |
| `aws s3 presign s3://<bucket>/<key> --expires-in 900` | presigned download URL (GET only) |

## Further reading

- **Amazon S3 User Guide**: "Hosting a static website using Amazon S3"; "Blocking public access to your Amazon S3 storage"; "Bucket policy examples"; "Using versioning in S3 buckets"; "Managing your storage lifecycle"; "Sharing objects with presigned URLs"; "Using cross-origin resource sharing (CORS)".
