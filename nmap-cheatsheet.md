# Nmap: Quick Network Scanning Guide

Nmap discovers hosts, identifies exposed network services, and gathers
information about systems you are authorized to assess.

Only scan systems you own or have explicit permission to test. Confirm the
allowed addresses, ports, techniques, timing, and source system before running
a scan.

```bash
nmap [scan type] [options] TARGET
```

- Examples use Nmap 7.99 syntax and documentation.
- Replace documentation-only addresses, ports, and filenames before running.
- Many raw-packet scans require `sudo` on Unix-like systems or Administrator
  privileges with Npcap on Windows.

## Contents

- [Pick the right scan](#pick-the-right-scan)
- [Setup and help](#setup-and-help)
- [Targets and scope](#targets-and-scope)
- [Host discovery](#host-discovery)
- [TCP port scanning](#tcp-port-scanning)
- [UDP port scanning](#udp-port-scanning)
- [Port selection](#port-selection)
- [Service and OS detection](#service-and-os-detection)
- [Nmap Scripting Engine](#nmap-scripting-engine)
- [Timing and performance](#timing-and-performance)
- [Output and reporting](#output-and-reporting)
- [Understanding results](#understanding-results)
- [Troubleshooting](#troubleshooting)
- [Common recipes](#common-recipes)
- [Safety and scope](#safety-and-scope)

<a id="pick-the-right-scan"></a>

## [Pick the right scan](#contents "Back to contents")

| Goal | Starting command |
|---|---|
| List intended targets without scanning | `nmap -sL TARGETS` |
| Find live hosts without a port scan | `nmap -sn NETWORK` |
| Scan the default 1,000 TCP ports | `nmap TARGET` |
| Scan selected TCP ports | `nmap -p 22,80,443 TARGET` |
| Scan every TCP port | `nmap -p- TARGET` |
| Identify services on open ports | `nmap -sV TARGET` |
| Scan common UDP ports | `sudo nmap -sU --top-ports 100 TARGET` |
| Run default NSE scripts | `nmap -sC TARGET` |
| Attempt OS detection | `sudo nmap -O TARGET` |
| Save reusable output formats | `nmap -oA BASENAME TARGET` |

Host discovery asks whether a system appears online. Port scanning asks which
network endpoints respond. Service detection probes those endpoints to infer
the actual application, while OS detection analyzes network-stack behavior.
These are separate stages and may produce different kinds of uncertainty.

<a id="setup-and-help"></a>

## [Setup and help](#contents "Back to contents")

| Goal | Command |
|---|---|
| Show installed version and features | `nmap --version` |
| Show command summary | `nmap --help` |
| Open the local manual on Linux/macOS | `man nmap` |
| Install on Debian / Ubuntu | `sudo apt install nmap` |
| Install on Fedora | `sudo dnf install nmap` |
| Install on Arch Linux | `sudo pacman -S nmap` |
| Install with Homebrew | `brew install nmap` |

Windows users should use the official installer, which includes Npcap. Open an
Administrator terminal for scans that require raw packets. Package-manager
versions may differ, so check `nmap --version` before relying on newer options.

<details>
<summary>Show a quick preflight</summary>

```bash
nmap --version
nmap --help
nmap -sL 192.0.2.0/29
```

</details>

`-sL` performs a list scan. It is useful for catching an incorrect CIDR range
or input file before sending discovery or port-scan probes. It may still make
DNS queries unless `-n` is added.

<a id="targets-and-scope"></a>

## [Targets and scope](#contents "Back to contents")

| Target form | Example |
|---|---|
| One IPv4 address | `192.0.2.10` |
| One hostname | `host.example.com` |
| Several targets | `192.0.2.10 192.0.2.20 host.example.com` |
| CIDR network | `192.0.2.0/24` |
| Address range | `192.0.2.10-50` |
| Input file | `-iL targets.txt` |
| Exclude targets | `--exclude 192.0.2.1,192.0.2.254` |
| Exclusion file | `--excludefile exclusions.txt` |
| IPv6 target | `-6 2001:db8::10` |

Use IANA documentation ranges such as `192.0.2.0/24`, `198.51.100.0/24`,
`203.0.113.0/24`, and `2001:db8::/32` only as placeholders in notes and
examples; replace them with an authorized target.

<details>
<summary>Show target validation examples</summary>

```bash
# Preview a file without probing ports or hosts
nmap -sL -n -iL targets.txt

# Preview a network while excluding infrastructure addresses
nmap -sL -n 192.0.2.0/24 \
  --exclude 192.0.2.1,192.0.2.254

# Scan all addresses returned for a hostname, not only the first
nmap --resolve-all host.example.com
```

</details>

An input file contains one target specification per line. Keep exclusions in
the command or a reviewed file so out-of-scope systems cannot be included by
accident.

<a id="host-discovery"></a>

## [Host discovery](#contents "Back to contents")

By default, Nmap discovers hosts first and performs heavier scans only against
systems it considers online.

| Goal | Option |
|---|---|
| Discover hosts without scanning ports | `-sn` |
| Skip discovery and treat every target as online | `-Pn` |
| Use TCP SYN discovery probes | `-PS22,80,443` |
| Use TCP ACK discovery probes | `-PA80,443` |
| Use UDP discovery probes | `-PU53,161` |
| Use ICMP echo discovery | `-PE` |
| Use ARP discovery on a local Ethernet network | `-PR` |
| Disable reverse DNS lookups | `-n` |
| Always perform reverse DNS lookups | `-R` |
| Trace the route to discovered hosts | `--traceroute` |

<details>
<summary>Show discovery examples</summary>

```bash
# Local network inventory without a port scan
sudo nmap -sn 192.0.2.0/24

# Use probes likely to pass through common firewall rules
sudo nmap -sn -PS22,80,443 -PA80,443 -PE 192.0.2.0/24

# Scan a known host even if discovery probes are blocked
nmap -Pn -p 22,80,443 192.0.2.10
```

</details>

`-Pn` is not a stronger ping. It disables host discovery and scans every
specified address as though it were online. On a large range, this can make a
scan dramatically slower and noisier. Local Ethernet scans normally use ARP
discovery because it is reliable within the broadcast domain.

<a id="tcp-port-scanning"></a>

## [TCP port scanning](#contents "Back to contents")

| Scan | Option | Use |
|---|---|---|
| SYN scan | `-sS` | Efficient raw-packet TCP scan; usually requires privileges |
| Connect scan | `-sT` | Uses the operating system's full TCP connection |
| ACK scan | `-sA` | Maps filtering; does not determine whether a port is open |

With raw-packet privileges, SYN scan is normally the default TCP technique.
Without them, Nmap normally substitutes a connect scan. A connect scan
completes the TCP handshake and is more likely to appear in application logs.

```bash
sudo nmap -sS -p 22,80,443 192.0.2.10
```

<details>
<summary>Show TCP examples</summary>

```bash
# Explicit unprivileged connect scan
nmap -sT -p 22,80,443 192.0.2.10

# SYN scan with state reasons and only responsive port states shown
sudo nmap -sS -p- --open --reason 192.0.2.10

# Check whether a firewall permits ACK probes; not an open-port scan
sudo nmap -sA -p 22,80,443 192.0.2.10
```

</details>

NULL, FIN, Xmas, idle, custom-flag, decoy, spoofing, and source-port techniques
are specialized and easy to misinterpret or misuse. Consult the official scan
technique documentation and the assessment rules before using them.

<a id="udp-port-scanning"></a>

## [UDP port scanning](#contents "Back to contents")

UDP has no TCP-style handshake. Silence may mean an open service ignored the
probe or a firewall dropped it, so Nmap often reports `open|filtered`.

| Goal | Command |
|---|---|
| Scan a few UDP services | `sudo nmap -sU -p 53,67,123,161 TARGET` |
| Scan the 100 most common UDP ports | `sudo nmap -sU --top-ports 100 TARGET` |
| Probe likely services on UDP results | `sudo nmap -sU -sV --top-ports 100 TARGET` |
| Scan selected TCP and UDP ports | `sudo nmap -sS -sU -p T:22,80,443,U:53,161 TARGET` |

Version detection can send protocol-aware probes that turn some
`open|filtered` UDP results into confirmed `open` results. UDP scans are often
slow because closed-port ICMP responses may be rate-limited and silent ports
must time out.

<details>
<summary>Show a staged UDP scan</summary>

```bash
# First pass: common UDP ports
sudo nmap -sU --top-ports 50 --open -oA udp-first 192.0.2.10

# Follow-up: service detection on selected results
sudo nmap -sU -sV -p 53,123,161 192.0.2.10
```

</details>

<a id="port-selection"></a>

## [Port selection](#contents "Back to contents")

Nmap scans the 1,000 most common ports for each selected protocol by default.

| Goal | Option |
|---|---|
| Scan individual ports | `-p 22,80,443` |
| Scan a range | `-p 1-1024` |
| Scan all ports from 1 through 65535 | `-p-` |
| Scan the 100 most common ports | `--top-ports 100` |
| Scan the smaller fast set | `-F` |
| Specify TCP and UDP separately | `-p T:22,80,U:53,161` |
| Exclude ports | `--exclude-ports 9100` |
| Scan ports in numeric order | `-r` |

`-F` scans 100 common ports when Nmap's normal frequency data is available;
it does not mean all ports at a faster packet rate. `-p-` can take much longer
than the default and should be scoped to known hosts.

<details>
<summary>Show port-selection examples</summary>

```bash
# Quick triage of a known host
nmap --top-ports 100 --open 192.0.2.10

# Full TCP port inventory without reverse DNS lookups
sudo nmap -sS -p- -n 192.0.2.10

# Scan by service names known to Nmap
nmap -p http,https,ssh 192.0.2.10
```

</details>

<a id="service-and-os-detection"></a>

## [Service and OS detection](#contents "Back to contents")

| Goal | Option |
|---|---|
| Detect application and version | `-sV` |
| Use fewer, more likely version probes | `--version-light` |
| Set version intensity from 0 to 9 | `--version-intensity LEVEL` |
| Try every version probe | `--version-all` |
| Attempt OS detection | `-O` |
| Limit OS detection to promising hosts | `--osscan-limit` |
| Guess OS more aggressively | `--osscan-guess` |
| Enable OS, versions, default scripts, and traceroute | `-A` |

```bash
nmap -sV -p 22,80,443 192.0.2.10
```

Nmap's `SERVICE` column without `-sV` is often a lookup based on the port
number. With `-sV`, Nmap actively probes the endpoint to identify what is
actually listening. Even then, banners may be hidden, proxied, or misleading.

<details>
<summary>Show detection examples</summary>

```bash
# Lightweight service identification
nmap -sV --version-light -p 22,80,443 192.0.2.10

# OS detection works best after finding at least one open and one closed port
sudo nmap -O --osscan-limit -p 1-1000 192.0.2.10

# Broad detection bundle; review scope before using
sudo nmap -A -p 22,80,443 192.0.2.10
```

</details>

`-A` is a bundle, not a synonym for "accurate." It enables version detection,
OS detection, default NSE scripts, and traceroute, creating more traffic and
potentially triggering application-level actions.

<a id="nmap-scripting-engine"></a>

## [Nmap Scripting Engine](#contents "Back to contents")

NSE scripts perform targeted checks after discovery and port scanning. Review
a script's documentation and categories before running it.

| Goal | Option |
|---|---|
| Run the default script set | `-sC` or `--script default` |
| Run a named script | `--script http-title` |
| Run several named scripts | `--script http-title,ssl-cert` |
| Show script documentation | `--script-help SCRIPT` |
| Supply script arguments | `--script-args name=value` |
| Trace script traffic | `--script-trace` |
| Refresh the local script database | `--script-updatedb` |

<details>
<summary>Show focused NSE examples</summary>

```bash
# Read documentation before running a script
nmap --script-help http-title

# Gather a web title and TLS certificate metadata
nmap -sV -p 80,443 --script http-title,ssl-cert 192.0.2.10

# Retrieve SSH host-key fingerprints
nmap -p 22 --script ssh-hostkey 192.0.2.10

# Run the default set against explicitly selected ports
nmap -sV -sC -p 22,80,443 192.0.2.10
```

</details>

Categories include `default`, `safe`, `discovery`, `version`, `auth`, `vuln`,
`intrusive`, `brute`, `exploit`, `dos`, `fuzzer`, `malware`, `broadcast`, and
`external`. A category name is not a complete risk assessment. In particular,
do not run broad `brute`, `exploit`, `dos`, `fuzzer`, `intrusive`, or `vuln`
sets without explicit authorization and a script-by-script review. `external`
scripts may disclose target information to third-party services.

The official NSE portal documents each script and its accepted arguments:
<https://nmap.org/nsedoc/>

<a id="timing-and-performance"></a>

## [Timing and performance](#contents "Back to contents")

| Goal | Option |
|---|---|
| Use the default normal timing | `-T3` |
| Use faster timing on a reliable network | `-T4` |
| Cap total packet rate | `--max-rate 100` |
| Add a minimum delay between probes | `--scan-delay 100ms` |
| Limit retransmissions | `--max-retries 2` |
| Give up on one host after a duration | `--host-timeout 10m` |
| Print periodic progress | `--stats-every 30s` |

Higher timing templates trade reliability and network courtesy for speed.
`-T5` can miss results on congested or filtered networks and can generate
bursty traffic. Start at `-T3`; use `-T4` only when the network and scope can
handle it.

<details>
<summary>Show controlled timing examples</summary>

```bash
# Rate-capped scan with periodic status
sudo nmap -sS --top-ports 1000 --max-rate 100 \
  --stats-every 30s 192.0.2.0/28

# Conservative scan of a fragile service range
nmap -sT -p 1-1024 -T3 --scan-delay 100ms \
  --max-retries 2 192.0.2.10
```

</details>

`--min-rate` forces a floor rather than a ceiling and can overwhelm a slow
target. Use `--max-rate` when the goal is to control load.

<a id="output-and-reporting"></a>

## [Output and reporting](#contents "Back to contents")

| Goal | Option |
|---|---|
| Save normal human-readable output | `-oN scan.nmap` |
| Save structured XML | `-oX scan.xml` |
| Save normal, XML, and grepable formats | `-oA scan` |
| Show only open or possibly open ports | `--open` |
| Explain host and port states | `--reason` |
| Increase visible detail | `-v` or `-vv` |
| Show periodic status | `--stats-every 30s` |

```bash
nmap -sV -p 22,80,443 --reason -oA service-scan 192.0.2.10
```

`-oA service-scan` creates `service-scan.nmap`, `service-scan.xml`, and
`service-scan.gnmap`. Prefer XML for automated parsing; grepable output is a
legacy format and omits some information. Nmap writes runtime messages to the
terminal even when file output is enabled.

<details>
<summary>Show reporting examples</summary>

```bash
# Inventory with a timestamped basename
nmap -sV --top-ports 1000 -oA inventory-2026-07-24 192.0.2.0/28

# XML on standard output for a parser
nmap -sV -oX - 192.0.2.10

# Explain why Nmap assigned each state
nmap -p 22,80,443 --reason 192.0.2.10
```

</details>

Record the exact command, Nmap version, source host, target scope, start time,
and authorization context with the output. Scan files can reveal sensitive
network structure and software versions, so store them appropriately.

<a id="understanding-results"></a>

## [Understanding results](#contents "Back to contents")

| State | Meaning |
|---|---|
| `open` | An application accepted or answered the probe |
| `closed` | The host responded, but no application is listening there |
| `filtered` | A firewall or network obstacle prevented a clear answer |
| `unfiltered` | The port is reachable, but that scan cannot tell open from closed |
| `open\|filtered` | No response distinguished an open port from a filtered one |
| `closed\|filtered` | Nmap could not distinguish those two states |

Port state describes Nmap's observation from the scanning system at that time.
It is not a permanent property of the target. Firewalls, routing, source
address, load balancers, packet loss, and rate limiting can change the result.

<details>
<summary>Show a reading checklist</summary>

```text
HOST         Confirm the address and resolved name are in scope.
STATE        Read it together with --reason and the chosen scan type.
SERVICE      Without -sV, this may be only a port-number lookup.
VERSION      Treat banners and fingerprints as evidence, not proof.
SCRIPT       Check which script generated the result and its limitations.
LATENCY      Large changes can affect timeouts and scan completeness.
```

</details>

Manually validate important findings with the appropriate client and from the
same authorized network position.

<a id="troubleshooting"></a>

## [Troubleshooting](#contents "Back to contents")

| Symptom | Likely cause | What to check |
|---|---|---|
| Host appears down | Discovery probes are filtered | Confirm address; try a small `-Pn -p PORT` scan |
| SYN scan is unavailable | Raw-packet privileges are missing | Use `sudo`/Administrator or choose `-sT` |
| Every port is filtered | Firewall, routing, or silent target | Add `--reason`; verify route and approved source address |
| UDP scan is very slow | Silence and ICMP rate limiting | Start with `--top-ports`; use selected ports and `-sV` |
| Service is shown but unverified | Port-number lookup only | Add `-sV` and manually validate |
| OS guess is absent or weak | Insufficient fingerprint conditions | Find at least one open and one closed TCP port |
| Results change between runs | Packet loss, rate limits, or load balancing | Slow down, compare sources, and repeat a focused scan |
| Scan covers unexpected hosts | Incorrect CIDR, DNS, or input file | Stop; review with `nmap -sL -n` and exclusions |
| NSE script fails | Missing arguments or unsuitable service | Read `--script-help`; verify service detection and scope |
| Windows capture error | Npcap or privilege problem | Check Npcap installation and Administrator terminal |

For a focused diagnostic, add `-vv --reason` to a tiny target and port set.
`--packet-trace`, `--version-trace`, and `--script-trace` are much noisier and
may expose payloads or secrets, so use them sparingly.

<a id="common-recipes"></a>

## [Common recipes](#contents "Back to contents")

<details>
<summary>Show recipes</summary>

```bash
# 1. Preview and confirm a small authorized network
nmap -sL -n 192.0.2.0/28

# 2. Discover responding hosts without scanning ports
sudo nmap -sn -n -oA discovery 192.0.2.0/28

# 3. Scan common TCP ports on the reviewed live-host list
sudo nmap -sS --top-ports 1000 --open -iL live-hosts.txt \
  --max-rate 100 -oA tcp-common

# 4. Identify services on selected findings
nmap -sV --version-light -p 22,80,443 \
  -iL selected-hosts.txt -oA services

# Full TCP port scan of one known host
sudo nmap -sS -p- -n --open --reason \
  -oA full-tcp 192.0.2.10

# Focused UDP scan
sudo nmap -sU -sV --top-ports 100 --open \
  -oA udp-common 192.0.2.10

# Combined selected TCP and UDP inventory
sudo nmap -sS -sU -p T:22,80,443,U:53,123,161 \
  -sV -oA mixed-services 192.0.2.10

# Focused web and TLS metadata
nmap -sV -p 80,443 --script http-title,ssl-cert \
  -oA web-metadata 192.0.2.10

# IPv6 service scan
nmap -6 -sV -p 22,80,443 2001:db8::10
```

</details>

A staged workflow is easier to review and less wasteful than applying every
detection option to an entire network. Generate `live-hosts.txt` from reviewed
discovery results rather than assuming all addresses remain in scope.

<a id="safety-and-scope"></a>

## [Safety and scope](#contents "Back to contents")

- Obtain explicit authorization for the source host, target addresses, ports,
  scan types, NSE scripts, rate, and testing window.
- Preview ranges and files with `nmap -sL -n`; use exclusions for shared or
  sensitive infrastructure.
- Start with targeted ports and normal timing. Expand only when the initial
  result and scope justify it.
- Use `--max-rate` or `--scan-delay` when load must be constrained, and stop if
  services degrade or monitoring teams request it.
- Review NSE scripts individually. Never assume a category such as `vuln` is
  passive or safe.
- Avoid evasion, spoofing, decoys, brute force, exploitation, denial-of-service,
  and third-party `external` scripts unless they are explicitly authorized.
- Treat scan results as sensitive and verify important findings manually.
- Remember that `open`, `filtered`, service names, and OS guesses are network
  observations with uncertainty, not proof of a vulnerability.

<details>
<summary>Show the minimum help commands</summary>

```bash
nmap --version
nmap --help
man nmap
nmap --script-help SCRIPT
```

</details>

Official Nmap 7.99 reference and downloads:

- <https://nmap.org/book/man.html>
- <https://nmap.org/download.html>
