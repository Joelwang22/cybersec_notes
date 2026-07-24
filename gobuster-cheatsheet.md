# Gobuster: Quick Enumeration Guide

Gobuster discovers web paths, DNS subdomains, virtual hosts, cloud-storage
buckets, and TFTP filenames by trying entries from a wordlist.

Only scan systems you own or have explicit permission to test. Gobuster can
generate substantial traffic; begin with the default thread count or lower.

```bash
gobuster MODE [options]
```

- Use `gobuster help` to list modes and `gobuster MODE --help` for exact flags.
- Examples use Gobuster 3.8 syntax. Older packaged versions may differ.
- Replace example targets, wordlists, credentials, and filters before running.

## Contents

- [Pick a mode](#pick-a-mode)
- [Setup and help](#setup-and-help)
- [Directory and file enumeration](#directory-and-file-enumeration)
- [DNS subdomain enumeration](#dns-subdomain-enumeration)
- [Virtual-host enumeration](#virtual-host-enumeration)
- [Fuzz mode](#fuzz-mode)
- [Other modes](#other-modes)
- [Common options](#common-options)
- [Wordlists](#wordlists)
- [Filtering results](#filtering-results)
- [Authenticated requests and proxies](#authenticated-requests-and-proxies)
- [Troubleshooting](#troubleshooting)
- [Common recipes](#common-recipes)
- [Safety and scope](#safety-and-scope)

<a id="pick-a-mode"></a>

## [Pick a mode](#contents "Back to contents")

| Goal | Mode | Starting command |
|---|---|---|
| Find web directories and files | `dir` | `gobuster dir -u URL -w WORDLIST` |
| Find DNS subdomains | `dns` | `gobuster dns -do DOMAIN -w WORDLIST` |
| Find name-based virtual hosts | `vhost` | `gobuster vhost -u URL -w WORDLIST --append-domain` |
| Replace a marker in a URL, header, or body | `fuzz` | `gobuster fuzz -u 'URL/FUZZ' -w WORDLIST` |
| Find public Amazon S3 buckets | `s3` | `gobuster s3 -w WORDLIST` |
| Find public Google Cloud Storage buckets | `gcs` | `gobuster gcs -w WORDLIST` |
| Probe filenames on a TFTP server | `tftp` | `gobuster tftp -s HOST -w WORDLIST` |

`dir`, `dns`, and `vhost` answer different questions. A DNS result means a
name resolves. A virtual-host result means a web server changes its response
for a particular `Host` header. A path result means a URL path produced a
response worth inspecting.

<a id="setup-and-help"></a>

## [Setup and help](#contents "Back to contents")

| Goal | Command |
|---|---|
| Show installed version | `gobuster version` |
| Show general help | `gobuster help` |
| Show mode-specific help | `gobuster dir --help` |
| Install with Go | `go install github.com/OJ/gobuster/v3@latest` |
| Check the installed executable | `command -v gobuster` |

The Go installation places the binary in your configured `GOBIN`, or normally
in `$(go env GOPATH)/bin`. Distribution packages may lag behind the current
release, so check `gobuster version` before copying commands from elsewhere.

<details>
<summary>Show a quick preflight</summary>

```bash
gobuster version
gobuster dir --help
curl -I https://example.com
wc -l /path/to/wordlist.txt
```

</details>

<a id="directory-and-file-enumeration"></a>

## [Directory and file enumeration](#contents "Back to contents")

Use `dir` mode to append each wordlist entry to a base URL.

```bash
gobuster dir -u https://example.com -w /path/to/wordlist.txt
```

| Goal | Option |
|---|---|
| Add file extensions | `-x php,html,txt` |
| Load extensions from a file | `-X extensions.txt` |
| Show full URLs | `-e` |
| Append `/` to each request | `-f` |
| Follow redirects | `-r` |
| Show only selected status codes | `-s 200,204,301,302,307,401,403` |
| Exclude status codes | `-b 404,429` |
| Exclude response lengths | `--exclude-length 1234,2000-2100` |
| Search for backups after a hit | `--discover-backup` |
| Ignore TLS certificate errors | `-k` |

`dir` excludes `404` by default. To use an allowlist with `-s`, clear the
default blacklist by also passing `-b ''`.

<details>
<summary>Show directory examples</summary>

```bash
# Common paths
gobuster dir -u https://example.com -w /path/to/common.txt

# Paths plus likely application extensions
gobuster dir -u https://example.com -w /path/to/common.txt \
  -x php,html,js,json,txt

# Scan an application below a base path
gobuster dir -u https://example.com/app/ -w /path/to/common.txt

# Keep only useful status codes and display full URLs
gobuster dir -u https://example.com -w /path/to/common.txt \
  -s 200,204,301,302,307,401,403 -b '' -e

# Filter a custom not-found page by its stable byte length
gobuster dir -u https://example.com -w /path/to/common.txt \
  --exclude-length 1542
```

</details>

Treat `301` and `302` as leads, not noise: their locations often reveal path
normalization or authentication flows. A `401` or `403` response can still
confirm that a resource exists.

<a id="dns-subdomain-enumeration"></a>

## [DNS subdomain enumeration](#contents "Back to contents")

Use `dns` mode to prepend each word to a domain and resolve the result.

```bash
gobuster dns -do example.com -w /path/to/subdomains.txt
```

| Goal | Option |
|---|---|
| Set the target domain | `-do example.com` or `--domain example.com` |
| Also check CNAME records | `-c` or `--check-cname` |
| Use a custom resolver | `--resolver 1.1.1.1:53` |
| Use TCP with that resolver | `--resolver 1.1.1.1:53 --protocol tcp` |
| Set DNS lookup timeout | `--timeout 2s` |
| Continue after detecting wildcard DNS | `--wildcard` |
| Avoid adding a trailing dot | `--no-fqdn` |

<details>
<summary>Show DNS examples</summary>

```bash
# Basic subdomain discovery
gobuster dns -do example.com -w /path/to/subdomains.txt

# Inspect aliases as well as address records
gobuster dns -do example.com -w /path/to/subdomains.txt --check-cname

# Use an explicitly selected resolver over TCP
gobuster dns -do example.com -w /path/to/subdomains.txt \
  --resolver 1.1.1.1:53 --protocol tcp --timeout 2s
```

</details>

If random names resolve, the domain probably uses wildcard DNS. Confirm with
`dig random-unlikely-name.example.com`, then decide whether the results can be
filtered meaningfully. `--wildcard` merely continues the scan; it does not make
wildcard-generated answers genuine.

Custom DNS resolvers are not supported by Gobuster on Windows. Use the system
resolver there, or run Gobuster in a suitable Linux environment.

<a id="virtual-host-enumeration"></a>

## [Virtual-host enumeration](#contents "Back to contents")

Use `vhost` when multiple websites may share one IP. Each guess is sent as an
HTTP `Host` value; DNS does not need to resolve the guessed names.

```bash
gobuster vhost -u https://192.0.2.10 -do example.com \
  --append-domain -w /path/to/subdomains.txt -k
```

| Goal | Option |
|---|---|
| Append the base domain to each word | `--append-domain` |
| Set the base domain when URL is an IP | `-do example.com` |
| Exclude status codes | `--exclude-status 404,429` |
| Exclude response lengths | `--exclude-length 1234` |
| Adjust length filtering for hostname length | `--exclude-length 1234 --exclude-hostname-length` |
| Continue when result reliability is uncertain | `--force` |

Without `--append-domain`, the wordlist must contain fully qualified hostnames.
When connecting to an IP over HTTPS, `-k` is commonly needed because the TLS
certificate is issued to a hostname rather than the IP address.

<details>
<summary>Show virtual-host examples</summary>

```bash
# Wordlist contains prefixes such as admin, api, and dev
gobuster vhost -u https://192.0.2.10 -do example.com \
  --append-domain -w /path/to/subdomains.txt -k

# Wordlist already contains complete names
gobuster vhost -u https://192.0.2.10 \
  -w /path/to/full-hostnames.txt -k

# Remove the uniform default response
gobuster vhost -u https://192.0.2.10 -do example.com \
  --append-domain -w /path/to/subdomains.txt -k \
  --exclude-status 404 --exclude-length 1678
```

</details>

Plain HTTP vhost mode can behave unexpectedly through an HTTP proxy because
the proxy may route using the guessed hostname. Prefer direct access or HTTPS;
read Gobuster's warning carefully before considering `--force`.

<a id="fuzz-mode"></a>

## [Fuzz mode](#contents "Back to contents")

Use the literal marker `FUZZ` where each word should be substituted. It can
appear in the URL, an HTTP header, Basic Auth credentials, or the request body.

| Goal | Command shape |
|---|---|
| Fuzz a path segment | `gobuster fuzz -u 'https://example.com/FUZZ' -w WORDLIST` |
| Fuzz a query value | `gobuster fuzz -u 'https://example.com/?id=FUZZ' -w WORDLIST` |
| Fuzz a header | `gobuster fuzz -u URL -H 'X-Env: FUZZ' -w WORDLIST` |
| Fuzz a POST body | `gobuster fuzz -u URL -m POST -B 'name=FUZZ' -w WORDLIST` |
| Exclude status codes | `-b 404,429` |
| Exclude response lengths | `--exclude-length 1234` |

Quote URLs containing `?`, `&`, or shell metacharacters.

<details>
<summary>Show fuzz examples</summary>

```bash
# Parameter names
gobuster fuzz -u 'https://example.com/search?FUZZ=test' \
  -w /path/to/parameters.txt

# Parameter values
gobuster fuzz -u 'https://example.com/profile?id=FUZZ' \
  -w /path/to/ids.txt --exclude-length 842

# JSON body; send the matching content type
gobuster fuzz -u https://example.com/api/login -m POST \
  -H 'Content-Type: application/json' \
  -B '{"username":"admin","password":"FUZZ"}' \
  -w /path/to/test-values.txt
```

</details>

Gobuster makes one substitution stream from one wordlist. For workflows that
need multiple independent markers, complex matchers, or response-content
rules, use a purpose-built web fuzzer.

<a id="other-modes"></a>

## [Other modes](#contents "Back to contents")

| Mode | Basic command | Purpose |
|---|---|---|
| `s3` | `gobuster s3 -w bucket-names.txt` | Check candidate Amazon S3 bucket names |
| `gcs` | `gobuster gcs -w bucket-names.txt` | Check candidate Google Cloud Storage bucket names |
| `tftp` | `gobuster tftp -s 192.0.2.20 -w filenames.txt` | Request candidate files from a TFTP server |

Bucket names are globally visible identifiers, so use organization-specific
patterns only within an authorized assessment. TFTP has no directory-listing
operation; results depend heavily on a precise filename wordlist.

Run `gobuster MODE --help` before using these modes because their specialized
flags do not match the HTTP modes.

<a id="common-options"></a>

## [Common options](#contents "Back to contents")

### All wordlist-based modes

| Goal | Option |
|---|---|
| Set wordlist | `-w wordlist.txt` |
| Read words from standard input | `-w -` |
| Set concurrent threads | `-t 10` |
| Delay each thread between attempts | `--delay 500ms` |
| Resume at a wordlist line | `--wordlist-offset 25000` |
| Write results to a file | `-o results.txt` |
| Hide banner and informational noise | `-q` |
| Hide progress | `--no-progress` |
| Hide request errors | `--no-error` |
| Disable colour | `--no-color` |
| Show diagnostic output | `--debug` |

### HTTP-based modes

| Goal | Option |
|---|---|
| Set target URL | `-u https://example.com` |
| Set a header; repeat as needed | `-H 'Name: value'` |
| Set cookies | `-c 'session=value'` |
| Use Basic Auth | `-U username -P password` |
| Set HTTP method | `-m POST` |
| Follow redirects | `-r` |
| Skip TLS certificate validation | `-k` |
| Set request timeout | `--timeout 15s` |
| Use an HTTP or SOCKS5 proxy | `--proxy http://127.0.0.1:8080` |
| Set User-Agent | `-a 'Mozilla/5.0'` |
| Retry timeouts | `--retry --retry-attempts 3` |

Increasing `-t` raises concurrency. `--delay` applies per thread, so several
threads can still produce overlapping requests. Reduce threads as well as
adding delay when a target is fragile or rate-limited.

<a id="wordlists"></a>

## [Wordlists](#contents "Back to contents")

Choose a list for the mode and technology instead of automatically selecting
the largest available file.

| Task | Useful wordlist contents |
|---|---|
| Web paths | Common directories, framework routes, API nouns |
| Extensions | Technology-appropriate suffixes such as `php`, `aspx`, or `json` |
| DNS | Host prefixes such as `www`, `api`, `dev`, and `vpn` |
| Vhosts with `--append-domain` | Host prefixes |
| Vhosts without `--append-domain` | Fully qualified hostnames |
| S3 / GCS | Valid bucket-name candidates |
| TFTP | Exact filenames, often including extensions and paths |

On Kali Linux, packaged lists are commonly under `/usr/share/wordlists/`, and
SecLists is commonly under `/usr/share/seclists/` when installed. Confirm paths
with `find` because packaging varies.

<details>
<summary>Show wordlist preparation</summary>

```bash
# Remove blank lines, sort, and deduplicate
sed '/^[[:space:]]*$/d' raw.txt | sort -u > words.txt

# Feed generated prefixes without creating a temporary wordlist
printf '%s\n' admin api dev staging | \
  gobuster dns -do example.com -w -
```

</details>

Gobuster does not ignore wordlist lines merely because they begin with `#`.
Clean comments yourself if the source list uses them.

<a id="filtering-results"></a>

## [Filtering results](#contents "Back to contents")

Filtering is usually the difference between a useful scan and pages of false
positives.

1. Request one deliberately nonexistent path or hostname.
2. Record its status code, response length, redirect, and visible content.
3. Repeat with another random value to see whether the response is stable.
4. Exclude only the stable property, then manually verify remaining hits.

<details>
<summary>Show baseline checks</summary>

```bash
curl -sk -o /dev/null -w 'status=%{http_code} bytes=%{size_download}\n' \
  https://example.com/definitely-not-a-real-path-92841

curl -sk -o /dev/null -w 'status=%{http_code} bytes=%{size_download}\n' \
  -H 'Host: definitely-not-real-92841.example.com' https://192.0.2.10/
```

</details>

Status and byte length are signals, not proof. Dynamic error pages may vary by
timestamp, reflected hostname, cookies, compression, or localization. Inspect
candidate responses directly with `curl` or a browser.

<a id="authenticated-requests-and-proxies"></a>

## [Authenticated requests and proxies](#contents "Back to contents")

<details>
<summary>Show request examples</summary>

```bash
# Session cookie
gobuster dir -u https://example.com/app/ -w /path/to/common.txt \
  -c 'session=REPLACE_ME'

# Bearer token and a custom header
gobuster dir -u https://example.com/api/ -w /path/to/api.txt \
  -H 'Authorization: Bearer REPLACE_ME' \
  -H 'Accept: application/json'

# Basic Auth; omit -P to be prompted without placing the password in history
gobuster dir -u https://example.com/private/ -w /path/to/common.txt \
  -U analyst

# Inspect HTTPS requests through a local intercepting proxy
gobuster dir -u https://example.com -w /path/to/small.txt \
  --proxy http://127.0.0.1:8080 -k
```

</details>

Command-line secrets can be stored in shell history and exposed in process
listings. Prefer short-lived test credentials, environment-specific secure
workflows, and Gobuster's password prompt for Basic Auth.

<a id="troubleshooting"></a>

## [Troubleshooting](#contents "Back to contents")

| Symptom | Likely cause | What to check |
|---|---|---|
| Every path looks valid | Custom or soft-404 response | Baseline random paths; exclude stable status or length |
| `-s` conflicts with `-b` | Default `-b 404` is still active | Add `-b ''` when using `-s` |
| Certificate error | Name mismatch or private CA | Use the correct hostname/CA; use `-k` only when justified |
| Many timeouts or `429` results | Too much concurrency | Lower `-t`, add `--delay`, and increase `--timeout` carefully |
| DNS wildcard warning | Random subdomains resolve | Verify with `dig`; do not assume forced results are real |
| Vhost results all have one size | Default site reflects guessed host | Use length/status filtering and verify candidates manually |
| No vhost results | Wrong domain construction | Check `-do`, `--append-domain`, scheme, port, and baseline request |
| Flags are rejected | Version mismatch | Run `gobuster version` and `gobuster MODE --help` |
| Empty output file | Filters removed every response | Rerun a tiny sample with fewer filters and without `-q` |

Use `--debug` on a small wordlist when the request itself is unclear. Avoid
debug output with production secrets because it can contain request details.

<a id="common-recipes"></a>

## [Common recipes](#contents "Back to contents")

<details>
<summary>Show recipes</summary>

```bash
# Conservative first pass with saved, script-friendly output
gobuster dir -u https://example.com -w /path/to/small.txt \
  -t 5 --delay 200ms -q --no-progress --no-color -o first-pass.txt

# Technology-specific second pass
gobuster dir -u https://example.com -w /path/to/medium.txt \
  -x php,html,txt,bak -t 10 -o content-results.txt

# Discover DNS names, including aliases
gobuster dns -do example.com -w /path/to/subdomains.txt \
  --check-cname -o dns-results.txt

# Test vhosts at a known address
gobuster vhost -u https://192.0.2.10 -do example.com \
  --append-domain -w /path/to/subdomains.txt -k \
  --exclude-length 1678 -o vhost-results.txt

# Resume a long scan from a known wordlist position
gobuster dir -u https://example.com -w /path/to/large.txt \
  --wordlist-offset 25000 -o resumed-results.txt

# Query parameter discovery
gobuster fuzz -u 'https://example.com/search?FUZZ=test' \
  -w /path/to/parameters.txt --exclude-length 842
```

</details>

Keep the exact command, Gobuster version, wordlist identity, and scan time with
results. They make findings reproducible and explain differences between runs.

<a id="safety-and-scope"></a>

## [Safety and scope](#contents "Back to contents")

- Obtain explicit authorization and confirm the permitted domains, IPs, ports,
  time window, and testing techniques.
- Start with a small, relevant wordlist and conservative concurrency.
- Stop or slow down if the service degrades, rate limits, or returns errors.
- Do not treat discovered access as permission to retrieve sensitive data.
- Protect output files: URLs, hostnames, response metadata, and credentials may
  be sensitive.
- Verify findings manually; wildcard DNS, soft 404s, redirects, and uniform
  vhost responses commonly produce false positives.
- Prefer the target hostname over `-k`; skipping TLS validation removes an
  important identity check.

<details>
<summary>Show the minimum help commands</summary>

```bash
gobuster version
gobuster help
gobuster dir --help
gobuster dns --help
gobuster vhost --help
gobuster fuzz --help
```

</details>

Official project and current CLI reference:
<https://github.com/OJ/gobuster>
