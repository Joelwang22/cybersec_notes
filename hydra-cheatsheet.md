# Hydra: Quick Credential Auditing Guide

THC Hydra tests login credentials against network services. It is intended for
authorized security assessments and controlled password-policy validation.

Only test accounts and systems covered by explicit permission. Confirm the
allowed users, credential sources, services, attempt limits, lockout policy,
rate, and testing window before starting.

```bash
hydra [options] PROTOCOL://TARGET[:PORT]/[MODULE-OPTIONS]
```

The equivalent traditional form is:

```bash
hydra [options] TARGET PROTOCOL [MODULE-OPTIONS]
```

- Examples follow Hydra 9.7 syntax.
- Replace documentation-only targets, accounts, and files before running.
- Start with one known test account, a tiny approved password list, and one
  task. Confirm result detection before expanding the test.

## Contents

- [Build the command](#build-the-command)
- [Setup and module help](#setup-and-module-help)
- [Targets and services](#targets-and-services)
- [Credential inputs](#credential-inputs)
- [Concurrency and delays](#concurrency-and-delays)
- [Common service modules](#common-service-modules)
- [HTTP form modules](#http-form-modules)
- [TLS and custom ports](#tls-and-custom-ports)
- [Multiple targets](#multiple-targets)
- [Output and restore](#output-and-restore)
- [Understanding and verifying results](#understanding-and-verifying-results)
- [Troubleshooting](#troubleshooting)
- [Common recipes](#common-recipes)
- [Safety and scope](#safety-and-scope)

<a id="build-the-command"></a>

## [Build the command](#contents "Back to contents")

Choose one login source, one password source, a target, and a service module.

| Goal | Starting command |
|---|---|
| Test one known pair | `hydra -l USER -p PASS ssh://TARGET` |
| Test one user with an approved list | `hydra -l USER -P PASSWORDS ssh://TARGET` |
| Test approved users and passwords | `hydra -L USERS -P PASSWORDS ssh://TARGET` |
| Test fixed `user:password` pairs | `hydra -C PAIRS ftp://TARGET` |
| Use a custom service port | `hydra -s PORT -l USER -P PASSWORDS TARGET ssh` |
| Inspect module-specific syntax | `hydra -U MODULE` |
| Save successful pairs as JSON | `hydra -o results.json -b json ...` |

```bash
hydra -l audit-user -P approved-passwords.txt \
  -t 1 -W 2 ssh://192.0.2.10
```

Hydra is an online authentication tester: every candidate normally reaches the
service. It is different from an offline password-hash auditing tool, which can
test hashes without touching live accounts or triggering login controls.

<a id="setup-and-module-help"></a>

## [Setup and module help](#contents "Back to contents")

| Goal | Command |
|---|---|
| Show version, compiled modules, and summary | `hydra -h` |
| Show available modules | `hydra` |
| Show help for one module | `hydra -U MODULE` |
| Read the local manual | `man hydra` |
| Install on Debian / Ubuntu | `sudo apt install hydra` |
| Pull the official container image | `docker pull vanhauser/hydra` |

Hydra enables some modules only when their required libraries were present at
build time. A command shown in this guide may therefore be unavailable in a
particular package. The installed `hydra` output is the source of truth for its
compiled modules.

<details>
<summary>Show a safe preflight</summary>

```bash
hydra -h
hydra -U ssh
hydra -U http-post-form
wc -l approved-users.txt approved-passwords.txt
```

</details>

Read module help before supplying module options. Their grammar is not shared
across protocols, and some modules require an option even when others do not.

<a id="targets-and-services"></a>

## [Targets and services](#contents "Back to contents")

| Goal | Syntax |
|---|---|
| One IPv4 address | `ssh://192.0.2.10` |
| One hostname | `ssh://host.example.com` |
| Custom port in URI form | `ssh://192.0.2.10:2222` |
| Custom port in traditional form | `-s 2222 192.0.2.10 ssh` |
| IPv6 target | `-6 ssh://[2001:db8::10]` |
| Module option in URI form | `imap://192.0.2.10/PLAIN` |
| Module option in traditional form | `192.0.2.10 imap PLAIN` |

Identify the actual service and authentication method first. A port number is
not proof of a protocol, and the wrong module can create connection errors or
misleading results.

<details>
<summary>Show target examples</summary>

```bash
# Standard SSH service
hydra -l audit-user -P approved-passwords.txt \
  -t 1 ssh://192.0.2.10

# SSH on a nonstandard port
hydra -l audit-user -P approved-passwords.txt \
  -t 1 ssh://192.0.2.10:2222

# IMAP with a module-specific authentication option
hydra -l audit-user -P approved-passwords.txt \
  -t 1 imap://192.0.2.10/PLAIN
```

</details>

Hydra uses IPv4 unless `-6` is selected. Bracket IPv6 literals in URI syntax.
Use IANA documentation addresses only as placeholders.

<a id="credential-inputs"></a>

## [Credential inputs](#contents "Back to contents")

### Users and passwords

| Goal | Option |
|---|---|
| Use one login | `-l USER` |
| Load logins from a file | `-L users.txt` |
| Use one password | `-p PASS` |
| Load passwords from a file | `-P passwords.txt` |
| Load fixed `login:password` pairs | `-C pairs.txt` |
| Also try an empty password | `-e n` |
| Also try login as password | `-e s` |
| Also try reversed login as password | `-e r` |

`-L users.txt -P passwords.txt` tests the Cartesian product: every selected
password against every selected login. By contrast, `-C pairs.txt` tests only
the exact pair on each line and cannot be combined with `-l`, `-L`, `-p`, or
`-P`.

<details>
<summary>Show input file formats</summary>

`users.txt` contains one login per line:

```text
audit-user-1
audit-user-2
```

`passwords.txt` contains one password per line:

```text
ApprovedCandidate1!
ApprovedCandidate2!
```

`pairs.txt` contains one exact pair per line:

```text
audit-user-1:ApprovedCandidate1!
audit-user-2:ApprovedCandidate2!
```

</details>

Files contain real secrets during many assessments. Restrict permissions,
avoid committing them, and securely remove them according to the engagement's
data-handling rules.

### Attempt ordering and generation

| Goal | Option |
|---|---|
| Try every password for one login before moving on | Default behavior |
| Try one password across every login before the next | `-u` |
| Generate candidates | `-x MIN:MAX:CHARSET` |

In `-x`, `a` means lowercase letters, `A` uppercase letters, and `1` digits;
other characters are included literally. Generated candidate counts grow
exponentially and can quickly exceed safe online-test limits.

<details>
<summary>Show a tiny lab-only generator</summary>

```bash
# Generates only the ten one-digit candidates; use on a disposable lab account
hydra -l lab-user -x 1:1:1 -t 1 -W 2 ssh://127.0.0.1
```

</details>

`-u` resembles password spraying because one candidate is tried across many
accounts. It is not automatically safer: it can affect many users and must be
explicitly approved. Do not use `-e` or `-x` merely because they are convenient;
each adds login attempts.

<a id="concurrency-and-delays"></a>

## [Concurrency and delays](#contents "Back to contents")

| Goal | Option |
|---|---|
| Set parallel connections per target | `-t TASKS` |
| Set maximum response wait | `-w SECONDS` |
| Wait between connections made by each task | `-W SECONDS` |
| Space attempts across all threads | `-c SECONDS` |
| Stop after the first valid pair per host | `-f` |

Hydra defaults to 16 tasks for many modules, which may be inappropriate for
SSH, RDP, web forms, fragile services, or any account-lockout policy. Begin
with `-t 1`. Both `-W` and `-c` are most meaningful with low concurrency; the
official help recommends `-t 1` with `-c`.

<details>
<summary>Show controlled-rate examples</summary>

```bash
# One connection task with two seconds between its connections
hydra -l audit-user -P approved-passwords.txt \
  -t 1 -W 2 -f ssh://192.0.2.10

# One attempt every five seconds across the run
hydra -l audit-user -P approved-passwords.txt \
  -t 1 -c 5 -f ftp://192.0.2.10
```

</details>

A delay does not guarantee that an account will avoid lockout. Calculate the
total candidates per account and coordinate with system owners before running.

<a id="common-service-modules"></a>

## [Common service modules](#contents "Back to contents")

These commands use small, approved credential lists and conservative task
counts. Confirm the module is compiled with `hydra` and read `hydra -U MODULE`.

| Service | Starting command |
|---|---|
| SSH | `hydra -l USER -P PASSWORDS -t 1 ssh://TARGET` |
| FTP | `hydra -l USER -P PASSWORDS -t 1 ftp://TARGET` |
| IMAP | `hydra -l USER -P PASSWORDS -t 1 imap://TARGET` |
| POP3 | `hydra -l USER -P PASSWORDS -t 1 pop3://TARGET` |
| SMTP | `hydra -l USER -P PASSWORDS -t 1 smtp://TARGET` |
| SMB | `hydra -l USER -P PASSWORDS -t 1 smb://TARGET` |
| HTTP Basic Auth | `hydra -l USER -P PASSWORDS TARGET http-get /private/` |

<details>
<summary>Show focused service examples</summary>

```bash
# SSH
hydra -l audit-user -P approved-passwords.txt \
  -t 1 -W 2 -f ssh://192.0.2.10

# FTP over explicit TLS module, if compiled
hydra -l audit-user -P approved-passwords.txt \
  -t 1 -W 2 -f ftps://192.0.2.10

# HTTP Basic Auth protecting a path
hydra -l audit-user -P approved-passwords.txt \
  -t 1 -W 2 -f 192.0.2.10 http-get /private/
```

</details>

HTTP Basic Auth is different from an HTML login form. Use `http-get` or an
HTTPS equivalent for an HTTP authentication challenge; use a form module for
credentials submitted as form fields.

<a id="http-form-modules"></a>

## [HTTP form modules](#contents "Back to contents")

The form-module option is a colon-separated string:

```text
PATH:FORM-FIELDS:FAILURE-OR-SUCCESS-CONDITION
```

`^USER^` and `^PASS^` are replaced for each attempt. The final condition must
identify either a failed login with `F=` or a successful login with `S=`.

```bash
hydra -l audit-user -P approved-passwords.txt -t 1 -W 2 \
  192.0.2.10 http-post-form \
  '/login:username=^USER^&password=^PASS^:F=Invalid credentials'
```

| Form type | Module |
|---|---|
| HTTP form using GET | `http-get-form` |
| HTTP form using POST | `http-post-form` |
| HTTPS form using GET | `https-get-form` |
| HTTPS form using POST | `https-post-form` |

<details>
<summary>Show form examples</summary>

```bash
# Failure marker: present only when credentials are rejected
hydra -l audit-user -P approved-passwords.txt -t 1 -W 2 \
  192.0.2.10 http-post-form \
  '/login:username=^USER^&password=^PASS^:F=Invalid credentials'

# Success marker: present only on an authenticated response
hydra -l audit-user -P approved-passwords.txt -t 1 -W 2 \
  192.0.2.10 http-post-form \
  '/login:username=^USER^&password=^PASS^:S=Account overview'

# HTTPS form on the default HTTPS port
hydra -l audit-user -P approved-passwords.txt -t 1 -W 2 \
  192.0.2.10 https-post-form \
  '/login:username=^USER^&password=^PASS^:F=Invalid credentials'
```

</details>

Capture one authorized failed request and one authorized successful request in
browser developer tools or an approved intercepting proxy. Match a stable,
unique response marker. Status code alone is often insufficient because many
applications return `200` for both outcomes or use redirects.

Form modules commonly fail or produce false positives when the application
uses changing CSRF tokens, CAPTCHA, JavaScript-generated values, MFA, JSON
instead of form encoding, per-attempt cookies, rate limiting, or generic error
pages. Hydra is not a full browser. Use `hydra -U http-post-form` for current
optional parameters and validate the request flow before testing a list.

<a id="tls-and-custom-ports"></a>

## [TLS and custom ports](#contents "Back to contents")

| Goal | Syntax |
|---|---|
| Use a TLS-specific module | `https-post-form`, `ftps`, `imaps`, or `pop3s` |
| Request SSL/TLS for a module | `-S` |
| Set a nondefault port | `-s PORT` or `PROTOCOL://TARGET:PORT` |
| Select IPv6 | `-6` |

Prefer the protocol's TLS-specific module when one is available. `-S` changes
the connection and default port behavior for supported modules; verify the
exact combination with module help.

<details>
<summary>Show TLS and port examples</summary>

```bash
# SSH on TCP 2222
hydra -l audit-user -P approved-passwords.txt \
  -t 1 -W 2 ssh://192.0.2.10:2222

# IMAP over TLS
hydra -l audit-user -P approved-passwords.txt \
  -t 1 -W 2 imaps://192.0.2.10

# Traditional form with a custom HTTPS port
hydra -l audit-user -P approved-passwords.txt \
  -t 1 -W 2 -S -s 8443 192.0.2.10 http-post-form \
  '/login:user=^USER^&pass=^PASS^:F=Invalid credentials'
```

</details>

Do not use `-O`, which enables obsolete SSLv2/SSLv3, unless legacy-protocol
testing is specifically authorized and isolated.

<a id="multiple-targets"></a>

## [Multiple targets](#contents "Back to contents")

Use `-M targets.txt` with traditional syntax to load one target per line.

```text
192.0.2.10
192.0.2.20:2222
host.example.com
```

```bash
hydra -M targets.txt -l audit-user -P approved-passwords.txt \
  -t 1 -W 2 ssh
```

| Goal | Option |
|---|---|
| Load targets from a file | `-M targets.txt` |
| Stop after the first valid pair on each host | `-f` |
| Stop after the first valid pair on any host | `-F` |

The URI form cannot be combined with `-M`. A target file can include a custom
port as `host:port`; bracket IPv6 addresses and add `-6` for an IPv6 run.

Multi-target testing multiplies load and the number of affected accounts.
Review every target and port, keep per-target tasks low, and use it only when
the whole file is explicitly in scope.

<a id="output-and-restore"></a>

## [Output and restore](#contents "Back to contents")

| Goal | Option |
|---|---|
| Write successful pairs to a file | `-o results.txt` |
| Choose plain text output | `-b text` |
| Choose current JSON output | `-b json` |
| Choose stable v1 JSON schema | `-b jsonv1` |
| Show verbose progress | `-v` |
| Show every attempted pair | `-V` |
| Enable debugging | `-d` |
| Restore an interrupted run | `hydra -R` |
| Ignore an existing restore file | `-I` |

```bash
hydra -l audit-user -P approved-passwords.txt -t 1 -W 2 \
  -o results.json -b json ssh://192.0.2.10
```

`-V` and `-d` can expose every tested credential and protocol detail on screen
or in captured logs. Use them only with disposable test secrets and controlled
logging.

Hydra writes `hydra.restore` periodically after interruption or failure. Resume
with `hydra -R` from the directory containing that file. Restore files are
platform-dependent and may contain sensitive assessment state; protect them as
carefully as wordlists and results.

For JSON output, `success` means Hydra completed successfully, not that a valid
credential was found. Check `quantityfound` and the `results` array. Serious
startup errors can also leave invalid JSON, so parsers must handle failure.

<a id="understanding-and-verifying-results"></a>

## [Understanding and verifying results](#contents "Back to contents")

A reported pair means the selected module classified one response as success.
It is evidence to verify, not proof that the credentials work as intended.

1. Stop or allow `-f` to stop the authorized test after a result.
2. Check the service logs and Hydra's errors for rate limits or ambiguous
   responses.
3. Manually validate the pair once using the normal client and approved source.
4. Confirm the account, authentication realm, target, and port.
5. Record whether MFA, conditional access, or a forced password change affects
   real access.
6. Protect the credential and report it through the agreed channel.

False positives are especially common with web forms, redirects, uniform error
pages, unstable markers, proxy responses, and experimental modules. False
negatives can result from lockouts, rate limits, connection failures, unsupported
authentication methods, TLS problems, or insufficient timeouts.

<a id="troubleshooting"></a>

## [Troubleshooting](#contents "Back to contents")

| Symptom | Likely cause | What to check |
|---|---|---|
| Module is unavailable | Hydra was compiled without its dependency | Run `hydra`; install a suitable package or build |
| Connection errors disable workers | Wrong service/port, filtering, or too many tasks | Verify with a client; reduce to `-t 1`; increase `-w` carefully |
| SSH rejects many connections | Server concurrency controls | Use `-t 1`, add `-W`, and coordinate attempt limits |
| Every form candidate succeeds | Success/failure condition is wrong | Capture known good/bad responses and select a unique marker |
| No form candidate succeeds | Dynamic token, MFA, JavaScript, or wrong fields | Inspect the full request flow; Hydra may not fit the application |
| Valid test pair is missed | Wrong module/auth method or unstable response | Run `hydra -U MODULE`; test the single known pair first |
| Account becomes locked | Attempt budget exceeded | Stop immediately and coordinate recovery with the owner |
| Restore warning delays startup | `hydra.restore` already exists | Resume deliberately with `-R` or use `-I` only for a new run |
| Output contains secrets | Verbose/debug output or result file | Restrict access, sanitize reports, and follow retention rules |
| JSON parser fails | Hydra had a serious startup error | Inspect stderr and validate the JSON before processing |

When diagnosing, use one known authorized pair, `-t 1`, and the smallest
possible request. Do not solve connection failures by increasing concurrency.

<a id="common-recipes"></a>

## [Common recipes](#contents "Back to contents")

<details>
<summary>Show recipes</summary>

```bash
# 1. Confirm the module and installed feature set
hydra -h
hydra -U ssh

# 2. Validate result detection with one approved test pair
hydra -l audit-user -p 'REPLACE_WITH_TEST_SECRET' \
  -t 1 -f ssh://192.0.2.10

# 3. Run a small, rate-controlled SSH audit
hydra -l audit-user -P approved-passwords.txt \
  -t 1 -W 2 -f -o ssh-results.txt ssh://192.0.2.10

# Fixed default-account pairs without a Cartesian product
hydra -C approved-pairs.txt -t 1 -W 2 -f \
  ftp://192.0.2.10

# Controlled HTTP Basic Auth audit
hydra -l audit-user -P approved-passwords.txt \
  -t 1 -W 2 -f 192.0.2.10 http-get /private/

# Controlled HTTPS form audit using a verified failure marker
hydra -l audit-user -P approved-passwords.txt \
  -t 1 -W 2 -f 192.0.2.10 https-post-form \
  '/login:username=^USER^&password=^PASS^:F=Invalid credentials'

# JSON results for an approved automation workflow
hydra -l audit-user -P approved-passwords.txt \
  -t 1 -W 2 -f -o results.json -b json ssh://192.0.2.10
```

</details>

The single known-pair check is important: if the module cannot recognize a
controlled success and failure, a larger run will not produce trustworthy
results.

<a id="safety-and-scope"></a>

## [Safety and scope](#contents "Back to contents")

- Obtain explicit authorization for targets, ports, services, accounts,
  credential sources, attempts per account, timing, and source addresses.
- Review the lockout, throttling, MFA, monitoring, and incident-response rules
  with the system owner before testing.
- Prefer dedicated test accounts and tiny, curated candidate lists. Never use
  leaked or unrelated real-world credentials without explicit authorization.
- Start with `-t 1` and an agreed delay. Stop on lockout, degradation, unexpected
  alerts, or repeated connection errors.
- Avoid broad CIDR ranges, generated candidate spaces, or multi-target files
  unless their exact size and impact have been reviewed.
- Treat wordlists, restore files, terminal logs, and output as sensitive because
  they can contain working credentials and account names.
- Manually verify findings once; do not continue testing a credential after it
  has been confirmed.
- Do not use Hydra against third-party identity providers or public services
  unless the provider and account owner have explicitly authorized it.

<details>
<summary>Show the minimum help commands</summary>

```bash
hydra -h
hydra
hydra -U MODULE
man hydra
```

</details>

Official Hydra 9.7 project, manual source, and releases:

- <https://github.com/vanhauser-thc/thc-hydra>
- <https://github.com/vanhauser-thc/thc-hydra/blob/master/hydra.1>
- <https://github.com/vanhauser-thc/thc-hydra/releases>
