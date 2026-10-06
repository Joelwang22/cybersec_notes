# Tcpdump: Quick Command Guide

Tcpdump captures packets and reads saved captures from a terminal.
Commands and options are case-sensitive.

```bash
sudo tcpdump [options] 'capture filter'
tcpdump -r capture.pcap [options] 'filter'
```

- Examples use Linux and Bash. Replace `eth0` with your active interface.
- Example addresses are placeholders. Use your own host, subnet and ports.
- Live capture usually needs `sudo`; reading a saved capture normally does not.
- Press `Ctrl+C` to stop. Do not type a shell prompt such as `$`.
- Filters select packets. Options control the interface, output and capture file.
- With no filter, tcpdump accepts all packets visible on the selected interface.

## Contents

- [Setup and help](#setup-and-help)
- [Interfaces and subnets](#interfaces-and-subnets)
- [Basic capture](#basic-capture)
- [Flags and their purpose](#output-options)
- [Saving and reading captures](#saving-and-reading-captures)
- [Filter syntax](#filter-syntax)
- [Hosts and networks](#hosts-and-networks)
- [Ports and protocols](#ports-and-protocols)
- [MAC addresses and VLANs](#mac-addresses-and-vlans)
- [TCP flags](#tcp-flags)
- [Packet lengths and header fields](#packet-lengths-and-header-fields)
- [Reading packet output](#reading-packet-output)
- [Common recipes](#common-recipes)
- [File rotation and buffering](#file-rotation-and-buffering)
- [Troubleshooting](#troubleshooting)
- [Wireshark and Tshark](#wireshark-and-tshark)
- [Limits and safety](#limits-and-safety)
- [References](#references)

<a id="setup-and-help"></a>

## [Setup and help](#contents "Back to contents")

| Goal                       | Command                      |
| -------------------------- | ---------------------------- |
| Install on Ubuntu / Debian | `sudo apt install tcpdump` |
| Install on Fedora          | `sudo dnf install tcpdump` |
| Install on Arch Linux      | `sudo pacman -S tcpdump`   |
| Find the executable        | `command -v tcpdump`       |
| Show version               | `tcpdump --version`        |
| Show help                  | `tcpdump -h`               |
| Read the command manual    | `man tcpdump`              |
| Read the filter manual     | `man pcap-filter`          |

Windows does not include tcpdump by default. A Linux VM provides the commands
used here. WSL sees its own network environment, which may differ from Windows.
For native Windows capture, Wireshark's Tshark is an alternative, not the same
command. macOS uses different interface names, commonly `en0`.

<a id="interfaces-and-subnets"></a>

## [Interfaces and subnets](#contents "Back to contents")

| Goal                                    | Command                     |
| --------------------------------------- | --------------------------- |
| List capture interfaces                 | `sudo tcpdump -D`         |
| Show Linux interface addresses          | `ip -br addr`             |
| Show routes and default interface       | `ip route`                |
| Select an interface by name             | `sudo tcpdump -i eth0`    |
| Select an interface by listed number    | `sudo tcpdump -i 1`       |
| Capture across interfaces on Linux      | `sudo tcpdump -i any`     |
| Capture local loopback traffic on Linux | `sudo tcpdump -i lo`      |
| List link-layer types for an interface  | `sudo tcpdump -i eth0 -L` |

`any` is platform-dependent. On Linux it uses a cooked link-layer format rather
than ordinary Ethernet headers and does not enable promiscuous mode.
Capturing several interfaces can show the same forwarded traffic more than once.

An address such as `192.168.30.223/24` belongs to `192.168.30.0/24`.
The `/24` mask is `255.255.255.0`. Do not assume every network uses `/24`.

<details>
<summary>Show interface selection example</summary>

```bash
ip -br addr
ip route
sudo tcpdump -D

# Replace eth0 with the interface shown on your machine
sudo tcpdump -i eth0 -nn
```

</details>

<a id="basic-capture"></a>

## [Basic capture](#contents "Back to contents")

| Goal                                        | Command                                            |
| ------------------------------------------- | -------------------------------------------------- |
| Capture without a filter                    | `sudo tcpdump -i eth0 -nn`                       |
| Stop after 20 matching packets              | `sudo tcpdump -i eth0 -nn -c 20`                 |
| Capture traffic involving one host          | `sudo tcpdump -i eth0 -nn 'host 192.168.30.223'` |
| Capture traffic to/from a subnet            | `sudo tcpdump -i eth0 -nn 'net 192.168.30.0/24'` |
| Use default snapshot length explicitly      | `sudo tcpdump -i eth0 -nn -s 0`                  |
| Capture only incoming packets, if supported | `sudo tcpdump -i eth0 -nn -Q in`                 |
| Capture only outgoing packets, if supported | `sudo tcpdump -i eth0 -nn -Q out`                |
| Disable requesting promiscuous mode         | `sudo tcpdump -i eth0 -nn -p`                    |

`-s` controls how many bytes of each packet are retained. `-s 0` uses the default
large snapshot length, currently 262144 bytes. Small values can truncate evidence.
`-c` counts packets that match the filter, not all traffic on the interface.

<a id="output-options"></a>

## [Flags and their purpose](#contents "Back to contents")

### Interfaces and capture controls

| Flag                                | Purpose                                                                                   |
| ----------------------------------- | ----------------------------------------------------------------------------------------- |
| `-i interface`                    | Select the capture interface, such as`-i eth0`                                          |
| `-D`                              | List available capture interfaces                                                         |
| `-c count`                        | Stop after this many matching packets, such as`-c 10`                                   |
| `-s bytes`                        | Set the maximum bytes retained per packet;`-s 0` uses the default large snapshot length |
| `-p`                              | Do not request promiscuous mode                                                           |
| `-Q in`, `-Q out`, `-Q inout` | Select capture direction, where supported                                                 |
| `-L`                              | List supported link-layer types for the selected interface                                |
| `-y type`                         | Select a supported link-layer type                                                        |

### Filters and help

| Flag          | Purpose                                                       |
| ------------- | ------------------------------------------------------------- |
| `-F file`   | Read the filter expression from a text file                   |
| `-d`        | Print compiled filter instructions and exit without capturing |
| `-dd`       | Print compiled filter instructions as a C array               |
| `-ddd`      | Print compiled filter instructions as decimal numbers         |
| `-h`        | Show command help                                             |
| `--version` | Show tcpdump and library versions                             |

### Packet display

| Flag                      | Purpose                                                 |
| ------------------------- | ------------------------------------------------------- |
| `-l` | Line-buffer text output so piped commands such as grep receive each line promptly |
| `-n`                    | Disable address name resolution                         |
| `-nn`                   | Also keep port numbers numeric                          |
| `-v`, `-vv`, `-vvv` | Increase decoded detail                                 |
| `-q`                    | Shorter summaries                                       |
| `-e`                    | Show link-layer information, such as Ethernet MACs      |
| `-S`                    | Show absolute TCP sequence numbers                      |
| `-A`                    | Show packet bytes as ASCII, excluding link-layer header |
| `-X`                    | Show hex and ASCII, excluding link-layer header         |
| `-XX`                   | Show hex and ASCII including link-layer header          |
| `-x`, `-xx`           | Hex only, without / with link-layer header              |

### Timestamps

| Flag       | Purpose                                       |
| ---------- | --------------------------------------------- |
| `-t`     | Omit timestamps                               |
| `-tt`    | Show Unix epoch timestamps                    |
| `-ttt`   | Show time since the previous displayed packet |
| `-tttt`  | Show date and time                            |
| `-ttttt` | Show time since the first displayed packet    |

<details>
<summary>Show output examples</summary>

```bash
sudo tcpdump -i eth0 -nn -tttt -vv 'picmp'
tcpdump -nn -e -r capture.pcap 'arp'
tcpdump -nn -X -r capture.pcap 'tcp port 80'

# Save readable summaries, not a packet capture
sudo tcpdump -i eth0 -nn -l 'port 53' | tee dns-summary.txt
```

</details>

<a id="saving-and-reading-captures"></a>

## [Saving and reading captures](#contents "Back to contents")

| Flag                             | Purpose                                                               |
| -------------------------------- | --------------------------------------------------------------------- |
| `-w file`                      | Save packets as binary capture data, such as`-w capture.pcap`       |
| `-r file`                      | Read packets from a saved capture, such as`-r capture.pcap`         |
| `-r input.pcap -w output.pcap` | Read one capture and write a filtered copy when followed by a filter  |
| `--count`                      | Count matching packets in a saved capture, on supported versions      |
| `--print`                      | Print packet summaries while saving with`-w`, on supported versions |

`-w` writes binary packet data, not readable text. Normally it suppresses packet
summaries. Newer versions offer `--print` to print while saving.
`>` writes terminal text and replaces an existing file. Use different filenames
for source captures and filtered copies. Opening an existing `-w` target can
overwrite it.

<details>
<summary>Show capture, inspect and narrow-down workflow</summary>

```bash
# Capture on your own interface, then stop with Ctrl+C
sudo tcpdump -i eth0 -nn -s 0 -w investigation.pcap

# Inspect the first 30 packets
tcpdump -nn -tttt -c 30 -r investigation.pcap

# Extract one host's traffic without changing the original
tcpdump -r investigation.pcap -w host-only.pcap 'host 192.168.30.223'

# Inspect DNS from the extracted file
tcpdump -nn -vv -r host-only.pcap 'port 53'
```

</details>

<a id="filter-syntax"></a>

## [Filter syntax](#contents "Back to contents")

Tcpdump uses libpcap capture filters, often called BPF filters. They are different
from Wireshark display filters. Put the entire filter in single quotes.

| Syntax          | Meaning                           | Example                           |
| --------------- | --------------------------------- | --------------------------------- |
| `host`        | One source or destination address | `host 10.0.0.2`                 |
| `net`         | Source or destination network     | `net 10.0.0.0/24`               |
| `src`         | Source only                       | `src host 10.0.0.2`             |
| `dst`         | Destination only                  | `dst port 443`                  |
| `src or dst`  | Either direction, the default     | `src or dst net 10.0.0.0/24`    |
| `src and dst` | Both directions must match        | `src and dst net 10.0.0.0/24`   |
| `and`         | Both conditions                   | `tcp and port 443`              |
| `or`          | Either condition                  | `port 80 or port 443`           |
| `not`         | Exclude a condition               | `not arp`                       |
| `( ... )`     | Group conditions                  | `tcp and (port 80 or port 443)` |

`and` and `or` have equal precedence in this syntax and associate left to right.
Use parentheses whenever you mix them. `not` takes precedence.
An unqualified `port 53` is not limited to UDP; qualify it when needed.

<details>
<summary>Show combined filters</summary>

```bash
sudo tcpdump -i eth0 -nn 'host 10.0.0.2 and (tcp port 80 or tcp port 443)'
sudo tcpdump -i eth0 -nn 'net 10.0.0.0/24 and not (udp port 5353 or udp port 5355)'
sudo tcpdump -i eth0 -nn '(host 10.0.0.2 or host 10.0.0.3) and tcp'

# Check filter compilation without capturing packets
tcpdump -d 'tcp and (port 80 or port 443)'

# Validate against a saved capture's link-layer type
tcpdump -r capture.pcap -d 'host 10.0.0.2'
```

</details>

<a id="hosts-and-networks"></a>

## [Hosts and networks](#contents "Back to contents")

Use these expressions after the options in a command.

| Goal                                | Filter                                              |
| ----------------------------------- | --------------------------------------------------- |
| To/from one IPv4 host               | `host 10.0.0.2`                                   |
| Sent by a host                      | `src host 10.0.0.2`                               |
| Sent to a host                      | `dst host 10.0.0.2`                               |
| Between two hosts, either direction | `host 10.0.0.2 and host 10.0.0.3`                 |
| One direction between hosts         | `src host 10.0.0.2 and dst host 10.0.0.3`         |
| To/from a subnet                    | `net 10.0.0.0/24`                                 |
| From a subnet                       | `src net 10.0.0.0/24`                             |
| To a subnet                         | `dst net 10.0.0.0/24`                             |
| Both endpoints in a subnet          | `src net 10.0.0.0/24 and dst net 10.0.0.0/24`     |
| Local subnet to outside it          | `src net 10.0.0.0/24 and not dst net 10.0.0.0/24` |
| Outside to local subnet             | `dst net 10.0.0.0/24 and not src net 10.0.0.0/24` |
| Exclude one host                    | `not host 10.0.0.2`                               |
| IPv4 traffic only                   | `ip`                                              |
| IPv6 traffic only                   | `ip6`                                             |
| To/from an IPv6 host                | `ip6 host 2001:db8::2`                            |
| To/from an IPv6 subnet              | `ip6 net 2001:db8::/64`                           |
| IPv4 multicast                      | `ip multicast`                                    |
| IPv6 multicast                      | `ip6 multicast`                                   |

Use literal IPs for repeatable filters. `host example.com` resolves the name when
the filter is compiled; it does not match HTTP hostnames or follow later IP changes.

<a id="ports-and-protocols"></a>

## [Ports and protocols](#contents "Back to contents")

| Goal                                     | Filter                             |
| ---------------------------------------- | ---------------------------------- |
| TCP                                      | `tcp`                            |
| UDP                                      | `udp`                            |
| IPv4 ICMP, such as ping                  | `icmp`                           |
| IPv6 ICMP, including neighbour discovery | `icmp6`                          |
| Address Resolution Protocol              | `arp`                            |
| TCP source or destination port 443       | `tcp port 443`                   |
| UDP destination port 53                  | `udp dst port 53`                |
| TCP source port 80                       | `tcp src port 80`                |
| DNS over ordinary TCP/UDP port 53        | `udp port 53 or tcp port 53`     |
| HTTP's usual TCP port                    | `tcp port 80`                    |
| HTTPS TCP and common QUIC port           | `tcp port 443 or udp port 443`   |
| SSH's usual port                         | `tcp port 22`                    |
| DHCPv4's usual ports                     | `udp and (port 67 or port 68)`   |
| DHCPv6's usual ports                     | `udp and (port 546 or port 547)` |
| NTP's usual port                         | `udp port 123`                   |
| mDNS's usual port                        | `udp port 5353`                  |
| LLMNR's usual port                       | `udp port 5355 or tcp port 5355` |
| SMB's usual direct TCP port              | `tcp port 445`                   |
| TCP port range                           | `tcp portrange 8000-8100`        |
| Exclude SSH TCP traffic                  | `not tcp port 22`                |

A port number suggests a service; it does not prove which application is running.
Port 53 misses encrypted DNS over HTTPS or TLS. TCP port 443 misses QUIC over UDP.
Port filters can miss non-initial IP fragments because those lack port headers.

<a id="mac-addresses-and-vlans"></a>

## [MAC addresses and VLANs](#contents "Back to contents")

| Goal                                    | Filter                           |
| --------------------------------------- | -------------------------------- |
| Ethernet source or destination MAC      | `ether host 00:0c:29:b9:45:b2` |
| Ethernet source MAC                     | `ether src 00:0c:29:b9:45:b2`  |
| Ethernet destination MAC                | `ether dst 00:0c:29:b9:45:b2`  |
| Ethernet broadcast                      | `ether broadcast`              |
| Ethernet multicast, including broadcast | `ether multicast`              |
| VLAN-tagged traffic                     | `vlan`                         |
| VLAN ID 100                             | `vlan 100`                     |
| TCP port 443 inside VLAN 100            | `vlan 100 and tcp port 443`    |

Use `-e` to display Ethernet headers. MAC filters depend on the capture link type.
For routed WAN traffic, the local Ethernet frame usually identifies the gateway,
not the remote server's MAC.

`vlan` changes the offsets used by subsequent filter terms. Keep VLAN filters
simple and test them against your capture. NIC offloading can remove VLAN tags
before the capture sees them.

<a id="tcp-flags"></a>

## [TCP flags](#contents "Back to contents")

Use these to inspect connection setup, closure and resets in your own captures.

| Goal                                    | Filter                                                       |
| --------------------------------------- | ------------------------------------------------------------ |
| Any SYN, including SYN-ACK              | `tcp[tcpflags] & tcp-syn != 0`                             |
| SYN without ACK, usual connection start | `(tcp[tcpflags] & (tcp-syn\|tcp-ack)) == tcp-syn`           |
| Both SYN and ACK set                    | `(tcp[tcpflags] & (tcp-syn\|tcp-ack)) == (tcp-syn\|tcp-ack)` |
| Any reset                               | `tcp[tcpflags] & tcp-rst != 0`                             |
| Any FIN                                 | `tcp[tcpflags] & tcp-fin != 0`                             |
| Any PSH                                 | `tcp[tcpflags] & tcp-push != 0`                            |
| Any ACK                                 | `tcp[tcpflags] & tcp-ack != 0`                             |

The `\|` escapes in the Markdown table render as ordinary `|` characters.
Type `|`, not `\|`, inside the quoted filter. `&` is a bitwise mask here.
ACK does not mean an empty acknowledgement; data packets commonly have ACK set.
PSH does not identify every packet carrying data.

These transport-header byte-access expressions are IPv4-oriented in libpcap.
Do not rely on them for IPv6 TCP analysis. Use Wireshark/Tshark display filters
when you need reliable decoded IPv6 flag inspection.

<details>
<summary>Show TCP flag examples</summary>

```bash
# Initial connection attempts in a saved capture
tcpdump 
-nn -r capture.pcap '(tcp[tcpflags] & (tcp-syn|tcp-ack)) == tcp-syn'

# Resets involving one host
tcpdump -nn -r capture.pcap 'host 10.0.0.2 and (tcp[tcpflags] & tcp-rst != 0)'

# FIN or RST set
tcpdump -nn -r capture.pcap 'tcp[tcpflags] & (tcp-fin|tcp-rst) != 0'
```

</details>

<a id="packet-lengths-and-header-fields"></a>

## [Packet lengths and header fields](#contents "Back to contents")

| Goal                                         | Filter                               |
| -------------------------------------------- | ------------------------------------ |
| Packet length greater than 1000 bytes        | `len > 1000`                       |
| Packet length at most 128 bytes              | `len <= 128`                       |
| IPv4 TTL at most 1                           | `ip[8] <= 1`                       |
| IPv4 ICMP echo request                       | `icmp[icmptype] == icmp-echo`      |
| IPv4 ICMP echo reply                         | `icmp[icmptype] == icmp-echoreply` |
| IPv4 ICMP destination unreachable            | `icmp[icmptype] == icmp-unreach`   |
| IPv4 ICMP time exceeded                      | `icmp[icmptype] == icmp-timxceed`  |
| IPv4 fragments, including the first fragment | `(ip[6:2] & 0x3fff) != 0`          |
| Non-initial IPv4 fragments only              | `(ip[6:2] & 0x1fff) != 0`          |

`protocol[offset:size]` reads bytes from a header. Offsets start at zero; size
defaults to one byte. These expressions require knowledge of the packet format.
`len` is packet length, not HTTP file size or TCP payload length.
TCP and UDP header lengths vary, so a fixed offset is unreliable for general
application-content matching.

<a id="reading-packet-output"></a>

## [Reading packet output](#contents "Back to contents")

```text
12:34:56.123456 IP 10.0.0.2.49224 > 104.25.198.31.443: Flags [S], seq 1000, win 64240, length 0
```

| Part                  | Meaning                                                      |
| --------------------- | ------------------------------------------------------------ |
| `12:34:56.123456`   | Capture timestamp                                            |
| `IP`                | IPv4; IPv6 commonly appears as`IP6`                        |
| `10.0.0.2.49224`    | Source IP and TCP port                                       |
| `>`                 | Direction from source to destination                         |
| `104.25.198.31.443` | Destination IP and TCP port                                  |
| `Flags [S]`         | SYN flag                                                     |
| `seq`               | TCP sequence number; often relative after the initial packet |
| `ack`               | Next byte expected from the peer                             |
| `win`               | Advertised receive window field; scaling may also apply      |
| `length 0`          | No TCP payload in this packet, not a zero-byte frame         |

| Flag in output | Meaning |
| -------------- | ------- |
| `S`          | SYN     |
| `.`          | ACK     |
| `F`          | FIN     |
| `R`          | RST     |
| `P`          | PSH     |
| `U`          | URG     |
| `[S.]`       | SYN-ACK |
| `[P.]`       | PSH-ACK |
| `[F.]`       | FIN-ACK |

For DNS summaries, `A? example.com.` asks for IPv4 addresses and
`AAAA? example.com.` asks for IPv6 addresses. A DNS lookup alone does not prove
that a website loaded or a connection succeeded.

<a id="common-recipes"></a>

## [Common recipes](#contents "Back to contents")

<details>
<summary>Capture all traffic and local subnet traffic</summary>

```bash
# All visible packets on the selected interface
sudo tcpdump -i eth0 -nn -s 0 -w all-traffic.pcap

# Traffic with either endpoint in the local IPv4 subnet
sudo tcpdump -i eth0 -nn -w subnet.pcap 'net 192.168.30.0/24'
```

</details>

<details>
<summary>Investigate DNS and connectivity</summary>

```bash
# Ordinary DNS queries and responses
tcpdump -nn -vv -r capture.pcap 'udp port 53 or tcp port 53'

# ARP and IPv4 ping traffic
tcpdump -nn -e -r capture.pcap 'arp or icmp'

# IPv6 neighbour discovery and other ICMPv6
tcpdump -nn -vv -r capture.pcap 'icmp6'

# Usual DHCPv4 ports
tcpdump -nn -vv -r capture.pcap 'udp and (port 67 or port 68)'
```

</details>

<details>
<summary>Inspect one TCP conversation</summary>

```bash
# Both directions of a known connection
tcpdump -nn -tttt -r capture.pcap 'tcp and ((src host 10.0.0.2 and src port 49224 and dst host 104.25.198.31 and dst port 443) or (src host 104.25.198.31 and src port 443 and dst host 10.0.0.2 and dst port 49224))'
```

This selects a tuple, not a Wireshark stream ID. A reused tuple can match several
connections. Tcpdump does not provide Wireshark's Follow TCP Stream view.

</details>

<details>
<summary>Inspect unencrypted web traffic in a saved capture</summary>

```bash
tcpdump -nn -A -s 0 -r capture.pcap 'tcp port 80'
```

Text can be split across packets, compressed or encoded. This is a quick look,
not reliable HTTP object extraction. HTTPS content remains encrypted.

</details>

<details>
<summary>Capture while using SSH to your own machine</summary>

```bash
# Avoid capturing the SSH terminal session itself
sudo tcpdump -i eth0 -nn -w investigation.pcap 'not tcp port 22'
```

Use the actual SSH port if different. This exclusion also removes any other
traffic using that TCP port, so omit it if SSH itself is part of the investigation.

</details>

<a id="file-rotation-and-buffering"></a>

## [File rotation and buffering](#contents "Back to contents")

| Flag                        | Purpose                                                       |
| --------------------------- | ------------------------------------------------------------- |
| `-C 100`                  | Rotate near 100 million bytes per file; boundary is not exact |
| `-W 5` with `-C`        | Keep a five-file ring, overwriting older files                |
| `-G 60`                   | Rotate every 60 seconds                                       |
| `-W 10` with `-G` alone | Exit after creating ten files                                 |
| `-U` with `-w`          | Flush saved output after each packet                          |
| `-B 4096`                 | Request a 4096 KiB capture buffer                             |
| `--immediate-mode`        | Request packet delivery without normal buffering              |
| `-l`                      | Line-buffer terminal text output, useful with pipes           |

<details>
<summary>Show rotation examples</summary>

```bash
# Five-file size-based ring. OLD FILES WILL BE OVERWRITTEN.
sudo tcpdump -i eth0 -nn -C 100 -W 5 -w ring.pcap

# Ten time-based files, then stop
sudo tcpdump -i eth0 -nn -G 60 -W 10 -w 'capture-%Y%m%d-%H%M%S.pcap'

# Request a larger buffer and packet-buffered file output
sudo tcpdump -i eth0 -nn -B 4096 -U -w capture.pcap
```

</details>

Use timestamp placeholders in the `-G` filename to avoid replacing files.
Avoid combining `-C`, `-G` and `-W` until you have checked your version's manual;
their combined behaviour differs from the simple examples above.
Rotation directories must be writable by tcpdump's capture user after any
privilege drop. `-U` does not guarantee data survives a sudden power loss.

<a id="troubleshooting"></a>

## [Troubleshooting](#contents "Back to contents")

| Problem                             | What to check                                                                                |
| ----------------------------------- | -------------------------------------------------------------------------------------------- |
| `command not found`               | Install tcpdump and check`command -v tcpdump`                                              |
| Permission denied opening interface | Use approved capture permissions or`sudo`                                                  |
| No such device                      | List interfaces with`tcpdump -D`; check spelling                                           |
| No packets                          | Check the interface, generate your own test traffic, temporarily remove the filter           |
| Missing local process traffic       | Try Linux loopback`-i lo`                                                                  |
| Filter syntax error                 | Quote the expression; use capture syntax, not`ip.addr == ...`                              |
| `[                                  | tcp]` or similar truncation marker                                                           |
| Bad checksums on outgoing traffic   | NIC checksum offloading can cause apparent errors in local captures                          |
| Larger-than-expected packets        | Segmentation/coalescing offloads can change what the host capture sees                       |
| Kernel packet drops                 | Save with`-w`, reduce printed detail, narrow the filter, consider increasing `-B`        |
| Cannot write or rotate files        | Check path permissions, disk space and privilege-drop behaviour                              |
| A feature is unavailable            | Check`tcpdump --version` and the installed manual                                          |
| Binary output looks garbled         | Read the pcap with`tcpdump -r` or Wireshark                                                |
| Cannot read a pcapng file           | Support depends on libpcap and file contents; use Wireshark to export a compatible pcap copy |

At shutdown, tcpdump reports captured packets, packets received by the filter
and packets dropped by the kernel. These counters depend on the OS; the first
two are not always equal. Zero reported drops does not prove complete visibility.

<a id="wireshark-and-tshark"></a>

## [Wireshark and Tshark](#contents "Back to contents")

| Goal                 | Tcpdump / capture filter             | Wireshark display filter      |
| -------------------- | ------------------------------------ | ----------------------------- |
| One IPv4 host        | `host 10.0.0.2`                    | `ip.addr == 10.0.0.2`       |
| IPv4 source          | `src host 10.0.0.2`                | `ip.src == 10.0.0.2`        |
| IPv4 subnet          | `ip net 10.0.0.0/24`               | `ip.addr == 10.0.0.0/24`    |
| TCP port             | `tcp port 443`                     | `tcp.port == 443`           |
| Ordinary DNS ports   | `udp port 53 or tcp port 53`       | `dns` for decoded DNS       |
| TCP reset            | `tcp[tcpflags] & tcp-rst != 0`     | `tcp.flags.reset == 1`      |
| HTTP response        | No general decoded HTTP field filter | `http.response.code == 200` |
| A decoded TCP stream | Match endpoint addresses and ports   | `tcp.stream == 11`          |

These are practical comparisons, not exact equivalences for every packet type.
For example, unqualified `host` can also match ARP addresses, while `ip.addr`
matches IPv4. Decoded protocol filters are not just port checks.

In Tshark, `-f` takes a capture filter and `-Y` takes a display filter.
A capture filter discards nonmatching packets before saving. A display filter
selects from packets already available to the analyser.
Open tcpdump's `.pcap` output in Wireshark for stream reassembly, protocol
hierarchy, endpoint statistics and supported TLS decryption.

<a id="limits-and-safety"></a>

## [Limits and safety](#contents "Back to contents")

- Capture only on networks and systems you are authorised to inspect.
- Captures may contain credentials, session cookies and private messages.
  Store and share them carefully.
- Promiscuous mode does not make every packet on a switched LAN visible.
  You may need an authorised mirror port or network tap for wider visibility.
- Ordinary Wi-Fi capture is not the same as monitor mode. Monitor-mode support
  depends on the adapter, driver and platform; it can interrupt connectivity.
- A host capture sees traffic at that host's capture point. VPNs, VMs, routing,
  offloading and encryption affect what appears there.
- Tcpdump does not generally decrypt HTTPS using `SSLKEYLOGFILE`. Use Wireshark
  or Tshark with matching session secrets for that analysis.
- Tcpdump does not provide a general hostname, URL, downloaded-file hash or
  malware-verdict filter. Port and address matches are only clues.
- Missing traffic is not proof that a system is safe, especially with a short,
  filtered or late-starting capture.
- Preserve the original capture. Write filtered copies to new filenames.
- Ring capture deliberately replaces older evidence. Choose retention carefully.
- Filters and verbose terminal output can affect what evidence you retain.
  For an investigation, saving packets first often gives you more options later.

<a id="references"></a>

## [References](#contents "Back to contents")

Use the manuals installed with your version for platform-specific behaviour.

- [Official tcpdump manual](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Official libpcap filter manual](https://www.tcpdump.org/manpages/pcap-filter.7.html)
- [Tcpdump manual source](https://github.com/the-tcpdump-group/tcpdump/blob/master/tcpdump.1.in)
- [Libpcap filter manual source](https://github.com/the-tcpdump-group/libpcap/blob/master/pcap-filter.manmisc.in)

<details>
<summary>Show help commands</summary>

```bash
tcpdump -h
tcpdump --version
man tcpdump
man pcap-filter
```

</details>
