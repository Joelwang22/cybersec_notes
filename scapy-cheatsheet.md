# Scapy: Quick Packet Manipulation Guide

Scapy is a Python library and interactive shell for constructing, sending,
capturing, dissecting, and saving network packets.

Only capture or transmit traffic on systems and networks you own or have
explicit permission to test. Confirm the allowed interfaces, addresses,
protocols, packet rates, and testing window before sending packets.

```python
from scapy.all import IP, ICMP, sr1

reply = sr1(IP(dst="192.0.2.10") / ICMP(), timeout=2, verbose=False)
```

- Examples follow the Scapy 2.7.0 stable documentation and use Python 3.
- Replace documentation-only addresses, interfaces, and filenames before use.
- Sending and live capture normally require `sudo` on Unix-like systems or an
  Administrator terminal with Npcap on Windows.

## Contents

- [Pick a workflow](#pick-a-workflow)
- [Setup and help](#setup-and-help)
- [Packets, layers, and fields](#packets-layers-and-fields)
- [Layer and field reference](#layer-and-field-reference)
- [Inspecting and dissecting packets](#inspecting-and-dissecting-packets)
- [Layer 3 and Layer 2 sending](#layer-3-and-layer-2-sending)
- [Sending and receiving](#sending-and-receiving)
- [Live capture](#live-capture)
- [Capture filters and callbacks](#capture-filters-and-callbacks)
- [PCAP files](#pcap-files)
- [Common protocol examples](#common-protocol-examples)
- [Interfaces, routes, and configuration](#interfaces-routes-and-configuration)
- [Writing reusable scripts](#writing-reusable-scripts)
- [Understanding results](#understanding-results)
- [Troubleshooting](#troubleshooting)
- [Common recipes](#common-recipes)
- [Safety and scope](#safety-and-scope)

<a id="pick-a-workflow"></a>

## [Pick a workflow](#contents "Back to contents")

| Goal | Starting point |
|---|---|
| Open the interactive Scapy shell | `sudo scapy` |
| Construct a packet without sending it | `pkt = IP(dst="192.0.2.10") / ICMP()` |
| Inspect every decoded field | `pkt.show()` |
| Send at Layer 3 without waiting | `send(pkt)` |
| Send at Layer 3 and receive answers | `ans, unans = sr(pkt, timeout=2)` |
| Return only the first Layer 3 answer | `reply = sr1(pkt, timeout=2)` |
| Send an Ethernet frame and receive answers | `ans, unans = srp(frame, timeout=2)` |
| Capture five packets | `pkts = sniff(count=5)` |
| Read a capture file | `pkts = rdpcap("capture.pcap")` |
| Write packets to a capture file | `wrpcap("capture.pcap", pkts)` |

Scapy is a packet toolkit rather than a single-purpose scanner. Constructing
and displaying a packet is local and passive. `sniff()` observes traffic
visible to the selected interface, while `send()`, `sendp()`, `sr()`, and
related functions actively transmit traffic.

<a id="setup-and-help"></a>

## [Setup and help](#contents "Back to contents")

| Goal | Command |
|---|---|
| Install the stable release | `python -m pip install scapy` |
| Install command-line extras | `python -m pip install "scapy[cli]"` |
| Show the installed version | `python -c "import scapy; print(scapy.__version__)"` |
| Start the interactive shell | `sudo scapy` |
| Show Scapy shell commands | `lsc()` |
| Search known packet layers by name | `ls("dns")` |
| Show fields for one layer | `ls(IP)` |
| Open Python help for a function | `help(sniff)` |

On Linux, Scapy can use native packet sockets; libpcap is still useful for
compiling BPF capture filters. On Windows, install Npcap in its WinPcap
API-compatible mode. Use a virtual environment when adding Scapy to a Python
project so its dependencies remain isolated.

<details>
<summary>Show a quick preflight</summary>

```bash
python -m venv .venv

# Linux / macOS
source .venv/bin/activate

# PowerShell
.venv\Scripts\Activate.ps1

python -m pip install scapy
python -c "import scapy; print(scapy.__version__)"
sudo scapy
```

Inside the Scapy shell:

```python
conf.iface
get_if_list()
ls(IP)
```

</details>

Importing from `scapy.all` is convenient and is used throughout the official
tutorial. In larger applications, explicit imports make dependencies clearer
and avoid placing hundreds of names in the module namespace.

<a id="packets-layers-and-fields"></a>

## [Packets, layers, and fields](#contents "Back to contents")

Scapy uses `/` to stack protocol layers. A Python `bytes` value or string at
the top becomes a `Raw` payload.

```python
from scapy.all import Ether, IP, TCP, Raw

pkt = (
    Ether(dst="00:11:22:33:44:55")
    / IP(dst="192.0.2.10", ttl=32)
    / TCP(dport=443, flags="S")
    / Raw(load=b"test")
)
```

| Goal | Example |
|---|---|
| Create an IPv4 layer | `IP(dst="192.0.2.10")` |
| Stack IPv4 and ICMP | `IP(dst="192.0.2.10") / ICMP()` |
| Add a byte payload | `IP() / UDP() / Raw(load=b"hello")` |
| Read a field | `pkt[IP].dst` |
| Change a field | `pkt[IP].ttl = 64` |
| Test for a layer | `if TCP in pkt:` |
| Get a layer or `None` | `layer = pkt.getlayer(TCP)` |
| List the packet's layer classes | `pkt.layers()` |
| Make an independent copy | `copy = pkt.copy()` |
| Serialize the packet | `wire_bytes = bytes(pkt)` |

Many fields such as source addresses, lengths, protocol identifiers, and
checksums have automatic defaults. Scapy calculates them when it assembles the
packet. `pkt.show()` can therefore display `None` for an automatic checksum;
`pkt.show2()` displays an assembled view with calculated fields.

<details>
<summary>Show packet-building examples</summary>

```python
# ICMP echo request
echo = IP(dst="192.0.2.10") / ICMP(id=0x1234, seq=1)

# UDP payload; Raw.load is bytes
message = IP(dst="192.0.2.20") / UDP(dport=9999) / b"lab message"

# One template that expands to three TCP packets
probes = IP(dst="192.0.2.10") / TCP(dport=[22, 80, 443], flags="S")
for probe in probes:
    print(probe.summary())

# Recalculate dependent values after editing a dissected packet
edited = pkt.copy()
edited[IP].ttl = 48
del edited[IP].len
del edited[IP].chksum
del edited[TCP].chksum
rebuilt = IP(bytes(edited[IP]))
```

</details>

A list, range, network, or Scapy generator placed in a field may expand one
template into many packets. Inspect the expansion with `list(template)` before
sending it; combinations across multiple fields form a Cartesian product and
can grow unexpectedly.

<a id="layer-and-field-reference"></a>

## [Layer and field reference](#contents "Back to contents")

Scapy has hundreds of built-in packet classes and an extensible `contrib`
catalog, so "all layers" is not one fixed list. It depends on the installed
Scapy version, platform, optional dependencies, and modules loaded during the
current session. Use Scapy's live registry as the authoritative reference.

### Discover and inspect any Scapy layer

| Goal | Command |
|---|---|
| List all currently loaded layer classes | `ls()` |
| Search loaded layers by name | `ls("icmp")` |
| Show every field on one class | `ls(IP)` |
| Show a layer instance and current values | `IP().show()` |
| Show calculated automatic values | `IP(dst="192.0.2.10").show2()` |
| Draw the layer's bit layout | `rfc(IP)` |
| Browse a protocol module | `explore("dns")` |
| Inspect the registered class list | `conf.layers` |
| Count registered layer classes | `len(conf.layers)` |
| Load an additional built-in module | `load_layer("http")` |
| Load an optional contributed module | `load_contrib("mqtt")` |

`ls(Layer)` shows the field name, Scapy field type, and default value. It does
not send or capture traffic.

```python
>>> ls(IP)
version    : BitField             = 4
ihl        : BitField             = None
tos        : XByteField           = 0
len        : ShortField           = None
id         : ShortField           = 1
flags      : FlagsField           = <Flag 0 ()>
frag       : BitField             = 0
ttl        : ByteField            = 64
proto      : ByteEnumField        = 0
chksum     : XShortField          = None
src        : SourceIPField        = None
dst        : DestIPField          = None
options    : PacketListField      = []
```

Fields whose default is `None` are often inferred from the route, surrounding
layers, or final packet size when Scapy assembles the packet. `None` does not
necessarily mean that zero or an empty value will appear on the wire.

<details>
<summary>Generate a field table for every registered layer</summary>

The following prints Markdown for every class in the live `conf.layers`
registry. Run it after importing or loading every protocol module you need.

```python
from scapy.all import conf


def markdown_cell(value):
    text = repr(value).replace("|", "\\|")
    return text.replace("\r", " ").replace("\n", " ")


layers = sorted(
    set(conf.layers),
    key=lambda cls: (cls.__module__.lower(), cls.__name__.lower()),
)

for layer_cls in layers:
    qualified_name = f"{layer_cls.__module__}.{layer_cls.__name__}"
    print(f"\n### `{qualified_name}`\n")

    fields = getattr(layer_cls, "fields_desc", ())
    if not fields:
        print("_No declared packet fields._")
        continue

    print("| Field | Scapy field type | Default |")
    print("|---|---|---|")
    for field in fields:
        name = getattr(field, "name", "<unnamed>")
        field_type = type(field).__name__
        default = markdown_cell(getattr(field, "default", None))
        print(f"| `{name}` | `{field_type}` | `{default}` |")
```

Save the output from a Python script to build a version-specific companion
reference:

```bash
python scapy-layer-catalog.py > scapy-layer-fields.md
```

The registry contains only classes loaded in that Python process. `scapy.all`
loads Scapy's normal built-in set; optional `contrib` modules appear after the
corresponding `load_contrib()` or import. Some contributed modules require
extra dependencies and may not be available on every installation.

</details>

<details>
<summary>Inspect fields programmatically</summary>

```python
def describe_layer(layer_cls):
    print(f"{layer_cls.__module__}.{layer_cls.__name__}")
    for field in layer_cls.fields_desc:
        print(
            f"{field.name:16} "
            f"type={type(field).__name__:24} "
            f"default={field.default!r}"
        )


describe_layer(Ether)
describe_layer(IP)
describe_layer(TCP)
```

For one constructed packet, use `pkt.fields` to see values explicitly assigned
or decoded on that instance. Use `pkt.getfieldval("name")` to retrieve the
effective value, including defaults and overloaded values.

```python
pkt = IP(dst="192.0.2.10") / TCP(dport=443)
print(pkt[IP].fields)
print(pkt[IP].getfieldval("ttl"))
print(pkt[TCP].getfieldval("flags"))
```

</details>

> **Link-layer protocols:** Ethernet framing, VLAN tagging, and local address
> resolution.

### Ethernet frames: `Ether()`

| Field | Type | Default | Meaning |
|---|---|---|---|
| `dst` | `DestMACField` | `None` | Destination MAC address; may be resolved automatically |
| `src` | `SourceMACField` | `None` | Source MAC address; normally selected from the interface |
| `type` | `XShortEnumField` | `0x9000` | EtherType; stacking a known upper layer normally overloads it |

```python
frame = Ether(
    dst="00:11:22:33:44:55",
    src="66:77:88:99:aa:bb",
    type=0x0800,
)
```

### VLAN tags: `Dot1Q()` and `Dot1AD()`

| Field | Type | Default | Meaning |
|---|---|---|---|
| `prio` | `BitField` (3 bits) | `0` | IEEE 802.1p priority code point |
| `dei` | `BitField` (1 bit) | `0` | Drop eligible indicator |
| `vlan` | `BitField` (12 bits) | `1` | VLAN identifier |
| `type` | `XShortEnumField` | `0` | Encapsulated EtherType |

`Dot1AD` uses the same field layout for provider bridging. Construct VLAN tags
only inside an authorized lab; switch configuration determines their effect.

### Address Resolution Protocol: `ARP()`

| Field | Type | Default | Meaning |
|---|---|---|---|
| `hwtype` | `XShortEnumField` | `1` | Hardware type; `1` is Ethernet |
| `ptype` | `XShortEnumField` | `0x0800` | Protocol type; `0x0800` is IPv4 |
| `hwlen` | `FieldLenField` | `None` | Hardware-address length, calculated when possible |
| `plen` | `FieldLenField` | `None` | Protocol-address length, calculated when possible |
| `op` | `ShortEnumField` | `1` | Operation; `1` request, `2` reply |
| `hwsrc` | `MultipleTypeField` | `None` | Sender hardware address |
| `psrc` | `MultipleTypeField` | `None` | Sender protocol address |
| `hwdst` | `MultipleTypeField` | `None` | Target hardware address |
| `pdst` | `MultipleTypeField` | `None` | Target protocol address |

```python
request = ARP(op="who-has", pdst="192.0.2.10")
reply = ARP(
    op="is-at",
    hwsrc="00:11:22:33:44:55",
    psrc="192.0.2.10",
    hwdst="66:77:88:99:aa:bb",
    pdst="192.0.2.20",
)
```

> **Network-layer protocols:** IPv4 and IPv6 addressing and delivery.

### Internet Protocol version 4: `IP()`

| Field | Type | Default | Meaning |
|---|---|---|---|
| `version` | `BitField` (4 bits) | `4` | IP version |
| `ihl` | `BitField` (4 bits) | `None` | Header length in 32-bit words |
| `tos` | `XByteField` | `0` | DSCP and ECN byte |
| `len` | `ShortField` | `None` | Total IPv4 packet length |
| `id` | `ShortField` | `1` | Fragment identification value |
| `flags` | `FlagsField` | empty | IPv4 flags: `DF`, `MF`, or reserved |
| `frag` | `BitField` (13 bits) | `0` | Fragment offset in eight-byte units |
| `ttl` | `ByteField` | `64` | Time to live |
| `proto` | `ByteEnumField` | `0` | Upper-layer protocol number |
| `chksum` | `XShortField` | `None` | IPv4 header checksum |
| `src` | `SourceIPField` | `None` | Source IPv4 address, normally route-derived |
| `dst` | `DestIPField` | `None` | Destination IPv4 address |
| `options` | `PacketListField` | `[]` | IPv4 options |

```python
packet = IP(
    src="192.0.2.20",
    dst="192.0.2.10",
    ttl=32,
    flags="DF",
    id=1234,
)
```

### Internet Protocol version 6: `IPv6()`

| Field | Type | Default | Meaning |
|---|---|---|---|
| `version` | `BitField` (4 bits) | `6` | IP version |
| `tc` | `BitField` (8 bits) | `0` | Traffic class: DSCP and ECN |
| `fl` | `BitField` (20 bits) | `0` | Flow label |
| `plen` | `ShortField` | `None` | Payload length after the base header |
| `nh` | `ByteEnumField` | `59` | Next-header protocol; overloaded by stacked layers |
| `hlim` | `ByteField` | `64` | Hop limit |
| `src` | `SourceIP6Field` | `None` | Source IPv6 address, normally route-derived |
| `dst` | `DestIP6Field` | `None` | Destination IPv6 address |

IPv6 extension headers are separate layers stacked between `IPv6()` and the
transport layer. Use `ls(IPv6ExtHdrHopByHop)`, `ls(IPv6ExtHdrRouting)`,
`ls(IPv6ExtHdrFragment)`, or `ls(IPv6ExtHdrDestOpt)` for their exact fields.

> **Transport-layer protocols:** TCP connections and UDP datagrams.

### Transmission Control Protocol: `TCP()`

| Field | Type | Default | Meaning |
|---|---|---|---|
| `sport` | `ShortEnumField` | `20` | Source port |
| `dport` | `ShortEnumField` | `80` | Destination port |
| `seq` | `IntField` | `0` | Sequence number |
| `ack` | `IntField` | `0` | Acknowledgment number |
| `dataofs` | `BitField` (4 bits) | `None` | Header size in 32-bit words |
| `reserved` | `BitField` (3 bits) | `0` | Reserved bits |
| `flags` | `FlagsField` | `S` | TCP control flags |
| `window` | `ShortField` | `8192` | Receive-window value |
| `chksum` | `XShortField` | `None` | TCP checksum |
| `urgptr` | `ShortField` | `0` | Urgent pointer |
| `options` | `TCPOptionsField` | `b''` | TCP options |

TCP flag letters are `F` FIN, `S` SYN, `R` RST, `P` PSH, `A` ACK, `U` URG,
`E` ECE, `C` CWR, and `N` NS. Combine them as a string such as `"SA"`.

```python
segment = TCP(
    sport=RandShort(),
    dport=443,
    seq=1000,
    flags="S",
    window=64240,
    options=[("MSS", 1460), ("WScale", 7)],
)
```

### User Datagram Protocol: `UDP()`

| Field | Type | Default | Meaning |
|---|---|---|---|
| `sport` | `ShortEnumField` | `53` | Source port |
| `dport` | `ShortEnumField` | `53` | Destination port |
| `len` | `ShortField` | `None` | UDP header plus payload length |
| `chksum` | `XShortField` | `None` | UDP checksum |

```python
datagram = UDP(sport=RandShort(), dport=53) / b"payload"
```

> **Control-message protocols:** IPv4 and IPv6 diagnostic and error messages.

### Internet Control Message Protocol: `ICMP()`

| Field | Type | Default | Meaning |
|---|---|---|---|
| `type` | `ByteEnumField` | `8` | ICMP message type; `8` is echo request |
| `code` | `MultiEnumField` | `0` | Meaning depends on `type` |
| `chksum` | `XShortField` | `None` | ICMP checksum |
| `id` | conditional `XShortField` | `0` | Identifier for echo and related messages |
| `seq` | conditional `XShortField` | `0` | Sequence for echo and related messages |
| `ts_ori` | conditional timestamp field | automatic | Originate timestamp |
| `ts_rx` | conditional timestamp field | automatic | Receive timestamp |
| `ts_tx` | conditional timestamp field | automatic | Transmit timestamp |
| `gw` | conditional `IPField` | `0.0.0.0` | Gateway address for redirect messages |
| `ptr` | conditional `ByteField` | `0` | Problem pointer for parameter errors |
| `reserved` | conditional `ByteField` | `0` | Reserved value for applicable messages |
| `length` | conditional `ByteField` | `0` | Extension length for applicable messages |
| `addr_mask` | conditional `IPField` | `0.0.0.0` | Address mask for mask messages |
| `nexthopmtu` | conditional `ShortField` | `0` | Reported next-hop MTU |
| `unused` | `MultipleTypeField` | `b''` | Type-specific unused data |
| `extpad` | conditional extension pad | `b''` | RFC 4884 extension padding |
| `ext` | conditional extension field | `None` | RFC 4884 extension object |

Most conditional fields appear only for the selected `type` and `code`.
Inspect the configured instance to see the applicable shape:

```python
ICMP(type="echo-request", id=0x1234, seq=1).show()
ICMP(type=3, code=4).show()  # destination unreachable: fragmentation needed
```

### ICMPv6 echo: `ICMPv6EchoRequest()` and `ICMPv6EchoReply()`

| Field | Request default | Reply default | Meaning |
|---|---|---|---|
| `type` | `128` | `129` | ICMPv6 message type |
| `code` | `0` | `0` | ICMPv6 code |
| `cksum` | `None` | `None` | ICMPv6 checksum |
| `id` | `0` | `0` | Echo identifier |
| `seq` | `0` | `0` | Echo sequence number |
| `data` | `b''` | `b''` | Echo payload bytes |

Other ICMPv6 message types are separate classes rather than one conditional
class. Search them with `ls("ICMPv6")`, then use `ls(TheClass)` for the exact
router-discovery, neighbor-discovery, error, or multicast fields.

> **Application-layer protocols:** DNS messages, questions, and resource
> records.

### Domain Name System message: `DNS()`

| Field | Type | Default | Meaning |
|---|---|---|---|
| `length` | conditional `ShortField` | `None` | TCP DNS message length; absent for UDP |
| `id` | `ShortField` | `0` | Transaction identifier |
| `qr` | `BitField` | `0` | `0` query, `1` response |
| `opcode` | `BitEnumField` | `0` | Operation code |
| `aa` | `BitField` | `0` | Authoritative answer |
| `tc` | `BitField` | `0` | Truncated response |
| `rd` | `BitField` | `1` | Recursion desired |
| `ra` | `BitField` | `0` | Recursion available |
| `z` | `BitField` | `0` | Reserved bit |
| `ad` | `BitField` | `0` | Authentic data |
| `cd` | `BitField` | `0` | Checking disabled |
| `rcode` | `BitEnumField` | `0` | Response code |
| `qdcount` | `FieldLenField` | `None` | Question count |
| `ancount` | `FieldLenField` | `None` | Answer-record count |
| `nscount` | `FieldLenField` | `None` | Authority-record count |
| `arcount` | `FieldLenField` | `None` | Additional-record count |
| `qd` | packet-list field | `[DNSQR()]` | Questions |
| `an` | packet-list field | `[]` | Answers |
| `ns` | packet-list field | `[]` | Authority records |
| `ar` | packet-list field | `[]` | Additional records |

### DNS question record: `DNSQR()`

| Field | Type | Default | Meaning |
|---|---|---|---|
| `qname` | `DNSStrField` | `b'www.example.com.'` | Queried domain name |
| `qtype` | `ShortEnumField` | `1` | Record type; `1` is A |
| `unicastresponse` | `BitField` | `0` | mDNS unicast-response request bit |
| `qclass` | `BitEnumField` | `1` | Query class; `1` is IN |

### DNS resource record: `DNSRR()`

| Field | Type | Default | Meaning |
|---|---|---|---|
| `rrname` | `DNSStrField` | `b'.'` | Owner name |
| `type` | `ShortEnumField` | `1` | Resource-record type |
| `cacheflush` | `BitField` | `0` | mDNS cache-flush bit |
| `rclass` | `BitEnumField` | `1` | Resource-record class |
| `ttl` | `IntField` | `0` | Record lifetime in seconds |
| `rdlen` | `FieldLenField` | `None` | Encoded RDATA length |
| `rdata` | `MultipleTypeField` | `b''` | Type-specific record data |

Specialized resource records such as `DNSRRMX`, `DNSRRSOA`, `DNSRRSRV`,
`DNSRRDNSKEY`, and `DNSRRRSIG` add type-specific fields. Find the appropriate
class with `ls("DNSRR")` and inspect it with `ls(DNSRRMX)`.

> **Payload layers:** undecoded application bytes and link-layer padding.

### Raw payload and padding: `Raw()` and `Padding()`

| Layer | Field | Type | Default | Meaning |
|---|---|---|---|---|
| `Raw` | `load` | `StrField` | `b''` | Undissected or explicitly supplied payload bytes |
| `Padding` | `load` | `StrField` | `b''` | Link-layer padding left after protocol dissection |

```python
payload = Raw(load=b"hello")
packet = IP(dst="192.0.2.10") / UDP(dport=9999) / payload
```

These tables cover every packet layer used directly in this guide. For every
other built-in, optional, or contributed Scapy class, `ls(TheLayer)` and the
registry generator above expose the complete field list for the exact version
installed locally.

<a id="inspecting-and-dissecting-packets"></a>

## [Inspecting and dissecting packets](#contents "Back to contents")

| Goal | Method |
|---|---|
| Print a one-line description | `pkt.summary()` |
| Print all decoded layers and fields | `pkt.show()` |
| Show calculated automatic fields | `pkt.show2()` |
| Display hexadecimal bytes | `hexdump(pkt)` |
| Return raw wire bytes | `raw(pkt)` or `bytes(pkt)` |
| Show a packet list with indices | `pkts.nsummary()` |
| Show every packet in a list | `pkts.show()` |
| Recreate an Ethernet frame from bytes | `Ether(data)` |
| Recreate an IPv4 packet from bytes | `IP(data)` |
| Produce a generating Scapy expression | `pkt.command()` |

```python
if IP in pkt:
    print(f"{pkt[IP].src} -> {pkt[IP].dst}")

if Raw in pkt:
    print(bytes(pkt[Raw].load))
```

<details>
<summary>Show inspection examples</summary>

```python
# Return the formatted output instead of printing it
details = pkt.show(dump=True)

# Select packets from a PacketList
tcp_packets = pkts.filter(lambda p: TCP in p)

# Python list comprehension is often clearer for complex conditions
syn_packets = [
    p for p in pkts
    if TCP in p and p[TCP].flags & 0x02
]

# Format fields safely after checking that the layers exist
for p in pkts:
    if IP in p and TCP in p:
        print(p.sprintf("%IP.src%:%TCP.sport% -> %IP.dst%:%TCP.dport%"))
```

</details>

Dissection is an interpretation of bytes based on link type, protocol fields,
ports, and Scapy's registered layer bindings. An unfamiliar or nonstandard
protocol may remain as `Raw`, and traffic on a familiar port is not proof that
Scapy chose the correct application-layer decoder.

<a id="layer-3-and-layer-2-sending"></a>

## [Layer 3 and Layer 2 sending](#contents "Back to contents")

| Function family | Starts with | Routing behavior |
|---|---|---|
| `send`, `sr`, `sr1` | `IP(...)` or `IPv6(...)` | Scapy selects a route and builds the link layer |
| `sendp`, `srp`, `srp1` | Usually `Ether(...)` | Caller supplies the Layer 2 frame and interface |

Use Layer 3 functions for normal IP routing. Use Layer 2 functions when the
Ethernet destination, broadcast behavior, VLAN tag, or other link-layer field
is part of the experiment.

```python
# Layer 3: Scapy resolves the route and next hop
send(IP(dst="192.0.2.10") / ICMP(), verbose=False)

# Layer 2: the caller supplies Ethernet addressing and interface
sendp(
    Ether(dst="ff:ff:ff:ff:ff:ff") / ARP(pdst="192.0.2.10"),
    iface="eth0",
    verbose=False,
)
```

Do not add `Ether()` to a packet passed to `send()` merely to silence a
warning. Pick the function that matches the layer you intend to control. A
Layer 2 frame is limited to its local link unless a network device forwards
the enclosed Layer 3 packet.

<a id="sending-and-receiving"></a>

## [Sending and receiving](#contents "Back to contents")

| Goal | Function |
|---|---|
| Send Layer 3 packets without collecting replies | `send()` |
| Send Layer 2 frames without collecting replies | `sendp()` |
| Send Layer 3 packets and return answered/unanswered sets | `sr()` |
| Return only the first Layer 3 answer | `sr1()` |
| Send Layer 2 frames and return answered/unanswered sets | `srp()` |
| Return only the first Layer 2 answer | `srp1()` |

Always give request/response experiments a finite `timeout`. Use `retry` only
after confirming that retransmission is allowed and that the target can handle
it.

```python
ans, unans = sr(
    IP(dst="192.0.2.10") / TCP(dport=[22, 80, 443], flags="S"),
    timeout=2,
    retry=0,
    inter=0.2,
    verbose=False,
)

for sent, received in ans:
    print(sent[TCP].dport, received.summary())

print(f"answered={len(ans)} unanswered={len(unans)}")
```

<details>
<summary>Show request and response examples</summary>

```python
# One response or None
reply = sr1(
    IP(dst="192.0.2.10") / ICMP(),
    timeout=2,
    verbose=False,
)

if reply is None:
    print("no response before timeout")
else:
    reply.show()

# ARP is a Layer 2 exchange
answered, unanswered = srp(
    Ether(dst="ff:ff:ff:ff:ff:ff") / ARP(pdst="192.0.2.0/29"),
    iface="eth0",
    timeout=2,
    inter=0.1,
    retry=0,
    verbose=False,
)

for request, response in answered:
    print(response.psrc, response.hwsrc)
```

</details>

An unanswered probe is not evidence that a host or port is absent. Firewalls,
routing, capture limitations, packet loss, protocol behavior, and timeouts can
all produce silence.

<a id="live-capture"></a>

## [Live capture](#contents "Back to contents")

```python
pkts = sniff(iface="eth0", count=10, timeout=30)
pkts.nsummary()
```

| Goal | Argument |
|---|---|
| Select an interface | `iface="eth0"` |
| Capture from several interfaces | `iface=["eth0", "eth1"]` |
| Stop after a packet count | `count=100` |
| Stop after elapsed seconds | `timeout=30` |
| Use a BPF capture filter | `filter="tcp port 443"` |
| Apply a Python packet predicate | `lfilter=lambda p: IP in p` |
| Run a function for each packet | `prn=handle_packet` |
| Avoid retaining packets in memory | `store=False` |
| Stop when a callback returns true | `stop_filter=should_stop` |

If no interface is given, Scapy captures on `conf.iface`. Promiscuous mode can
make additional frames visible on a suitable network, but it cannot bypass
switch isolation, encryption, operating-system restrictions, or traffic that
never reaches the interface.

<details>
<summary>Show synchronous and asynchronous capture</summary>

```python
# Print summaries without retaining packets
sniff(
    iface="eth0",
    filter="icmp",
    prn=lambda p: p.summary(),
    store=False,
    timeout=30,
)

# Start and stop a capture from program logic
from scapy.all import AsyncSniffer

sniffer = AsyncSniffer(
    iface="eth0",
    filter="tcp",
    store=True,
)
sniffer.start()

# Perform the authorized lab action here.

captured = sniffer.stop()
print(f"captured={len(captured)}")
```

</details>

`AsyncSniffer` improves control flow, not capture performance. A single
`sniff()` or `AsyncSniffer` can accept a list of interfaces when simultaneous
capture is needed.

<a id="capture-filters-and-callbacks"></a>

## [Capture filters and callbacks](#contents "Back to contents")

`filter` and `lfilter` operate at different stages:

| Filter | Syntax | Behavior |
|---|---|---|
| `filter` | BPF text such as `tcp port 443` | Narrows traffic before normal Scapy dissection; usually more efficient |
| `lfilter` | Python callable | Receives dissected packets and keeps those returning true |

Common BPF examples:

| Goal | Filter |
|---|---|
| ICMP traffic | `icmp` |
| DNS over TCP or UDP port 53 | `port 53` |
| Traffic involving one host | `host 192.0.2.10` |
| TCP to a destination port | `tcp dst port 443` |
| Exclude SSH management traffic | `not port 22` |

```python
def describe_dns(pkt):
    if DNSQR in pkt:
        name = pkt[DNSQR].qname.decode(errors="replace")
        return f"DNS query: {name}"
    return None

sniff(
    iface="eth0",
    filter="udp port 53",
    prn=describe_dns,
    store=False,
    timeout=30,
)
```

`prn` should return the value to print; it does not decide whether the packet
is stored. Keep callbacks quick so packet processing does not fall behind.
`stop_filter` is evaluated after a packet has been received, so the triggering
packet is included in the returned list when storage is enabled.

<a id="pcap-files"></a>

## [PCAP files](#contents "Back to contents")

| Goal | Function or class |
|---|---|
| Load an entire PCAP or PCAPNG | `rdpcap("capture.pcap")` |
| Process a capture through `sniff` | `sniff(offline="capture.pcap")` |
| Write packets to PCAP | `wrpcap("output.pcap", packets)` |
| Write packets to PCAPNG | `wrpcapng("output.pcapng", packets)` |
| Stream a large capture | `PcapReader("capture.pcap")` |
| Append or stream output | `PcapWriter("output.pcap", append=True, sync=True)` |

```python
from scapy.all import IP, PcapReader, PcapWriter

with PcapReader("input.pcap") as reader, PcapWriter(
    "ip-only.pcap", sync=True
) as writer:
    for pkt in reader:
        if IP in pkt:
            writer.write(pkt)
```

<details>
<summary>Show offline analysis examples</summary>

```python
# Simple whole-file workflow
pkts = rdpcap("capture.pcap")
print(f"packets={len(pkts)}")
pkts.nsummary()

# Use Python filtering without a live-capture dependency
dns_packets = sniff(
    offline="capture.pcap",
    lfilter=lambda p: DNS in p,
)
wrpcap("dns-only.pcap", dns_packets)

# Preserve the original and edit copies
edited = []
for original in pkts:
    clone = original.copy()
    if IP in clone:
        clone[IP].ttl = 64
        del clone[IP].chksum
    edited.append(clone)
wrpcap("edited-copy.pcap", edited)
```

</details>

`rdpcap()` loads the whole file and can exhaust memory on a large capture.
Use `PcapReader` for streaming. Capture files may contain credentials, tokens,
personal data, internal addresses, and payloads; store and share them as
sensitive evidence.

<a id="common-protocol-examples"></a>

## [Common protocol examples](#contents "Back to contents")

### ICMP and IPv6 echo

```python
# IPv4 echo request
reply4 = sr1(
    IP(dst="192.0.2.10") / ICMP(),
    timeout=2,
    verbose=False,
)

# IPv6 echo request
reply6 = sr1(
    IPv6(dst="2001:db8::10") / ICMPv6EchoRequest(),
    timeout=2,
    verbose=False,
)
```

### ARP discovery on a local lab link

```python
request = Ether(dst="ff:ff:ff:ff:ff:ff") / ARP(
    pdst="192.0.2.0/29"
)
answered, _ = srp(
    request,
    iface="eth0",
    timeout=2,
    inter=0.1,
    retry=0,
    verbose=False,
)

for _, response in answered:
    print(f"{response.psrc}\t{response.hwsrc}")
```

ARP does not cross a router. Limit `pdst` to the approved local subnet and
preview the generated request count before sending.

### DNS query to an approved resolver

```python
query = (
    IP(dst="192.0.2.53")
    / UDP(sport=RandShort(), dport=53)
    / DNS(rd=1, qd=DNSQR(qname="example.com", qtype="A"))
)
response = sr1(query, timeout=2, verbose=False)

if response is not None and DNS in response:
    print(response[DNS].summary())
    for answer in response[DNS].an:
        print(answer.rrname, answer.rdata)
```

### Focused TCP SYN observation

```python
probe = IP(dst="192.0.2.10") / TCP(
    sport=RandShort(),
    dport=443,
    flags="S",
)
reply = sr1(probe, timeout=2, verbose=False)

if reply is None:
    print("no response")
elif TCP in reply and reply[TCP].flags & 0x12 == 0x12:
    print("received SYN-ACK")
elif TCP in reply and reply[TCP].flags & 0x04:
    print("received RST")
elif ICMP in reply:
    print(f"received ICMP type={reply[ICMP].type} code={reply[ICMP].code}")
else:
    print(reply.summary())
```

A SYN-ACK, RST, ICMP error, and silence are observations, not vulnerability
findings. The local operating system may send a RST for unsolicited SYN-ACKs
because it does not own Scapy's handcrafted connection state.

<a id="interfaces-routes-and-configuration"></a>

## [Interfaces, routes, and configuration](#contents "Back to contents")

| Goal | Example |
|---|---|
| Show the default interface | `conf.iface` |
| List interface names | `get_if_list()` |
| Show Scapy's interface table | `conf.ifaces.show()` |
| Set the default interface | `conf.iface = "eth0"` |
| Show the IPv4 route table | `conf.route` |
| Look up the route for a destination | `conf.route.route("192.0.2.10")` |
| Show the IPv6 route table | `conf.route6` |
| Disable global interactive verbosity | `conf.verb = 0` |
| Enable libpcap-backed sockets | `conf.use_pcap = True` |

```python
iface, source, gateway = conf.route.route("192.0.2.10")
print(f"iface={iface} source={source} gateway={gateway}")
```

Scapy's route table is separate from simply choosing a capture interface.
Layer 3 sends normally follow `conf.route`; Layer 2 sends use the interface
provided to `sendp()` or `srp()`. On Windows, Scapy may display a friendly
description, a device name, and a GUID for the same adapter; use a value shown
by Scapy rather than guessing it.

<a id="writing-reusable-scripts"></a>

## [Writing reusable scripts](#contents "Back to contents")

Avoid hard-coded production targets, unbounded loops, and import-time packet
transmission. Validate input, use finite timeouts, return meaningful exit
codes, and put active behavior behind `main()`.

```python
#!/usr/bin/env python3
import argparse

from scapy.all import ICMP, IP, conf, sr1


def parse_args():
    parser = argparse.ArgumentParser(
        description="Send one ICMP echo request to an authorized target."
    )
    parser.add_argument("target")
    parser.add_argument("--timeout", type=float, default=2.0)
    return parser.parse_args()


def main():
    args = parse_args()
    if args.timeout <= 0:
        raise SystemExit("--timeout must be greater than zero")

    conf.verb = 0
    request = IP(dst=args.target) / ICMP()
    response = sr1(request, timeout=args.timeout, retry=0)

    if response is None:
        print("No response before timeout")
        return 1

    print(response.summary())
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

<details>
<summary>Show a capture callback pattern</summary>

```python
from scapy.all import IP, TCP, sniff


def summarize_tcp(pkt):
    if IP not in pkt or TCP not in pkt:
        return None
    return (
        f"{pkt[IP].src}:{pkt[TCP].sport} -> "
        f"{pkt[IP].dst}:{pkt[TCP].dport} "
        f"flags={pkt[TCP].flags}"
    )


sniff(
    iface="eth0",
    filter="tcp",
    prn=summarize_tcp,
    store=False,
    timeout=30,
)
```

</details>

For automated tests, construct packets and feed known byte strings or sample
PCAPs into parsing functions. Keep live transmission in a small boundary that
can be replaced or mocked.

<a id="understanding-results"></a>

## [Understanding results](#contents "Back to contents")

| Object | Meaning |
|---|---|
| `Packet` | One constructed, captured, or dissected packet |
| `PacketList` | An ordered collection returned by capture/file helpers |
| `SndRcvList` | Matched `(sent, received)` pairs returned as `ans` |
| `PacketList` returned as `unans` | Sent packets for which Scapy matched no answer |
| `None` from `sr1()` | No matching answer arrived before the function returned |

Scapy matches replies using protocol-specific logic, but a matched packet is
still evidence to interpret. Check addresses, ports, identifiers, sequence
values, flags, ICMP types and codes, and the capture position.

<details>
<summary>Show a response-reading checklist</summary>

```text
SCOPE       Are the source, destination, interface, and protocol authorized?
ROUTE       Did the packet leave through the expected interface and gateway?
PAIRING     Does the response actually correspond to the sent packet?
FLAGS       Interpret TCP flags together, not as isolated characters.
ICMP        Read both type and code; an ICMP error may quote the original packet.
TIMEOUT     Silence means only that no matched response arrived in time.
CAPTURE     Could filtering, loss, offloading, or visibility explain the result?
```

</details>

Validate important findings with a standard protocol client or another packet
analyzer. Scapy exposes low-level evidence but does not automatically determine
whether a service is correctly configured or vulnerable.

<a id="troubleshooting"></a>

## [Troubleshooting](#contents "Back to contents")

| Symptom | Likely cause | What to check |
|---|---|---|
| Permission or raw-socket error | Insufficient packet privileges | Use `sudo`/Administrator in the authorized environment |
| No packets are captured | Wrong interface, filter, or visibility | Check `conf.iface`, remove filters briefly, and generate known lab traffic |
| BPF filter cannot compile | Missing libpcap/Npcap or invalid syntax | Install the capture dependency and test a simple filter such as `icmp` |
| `sr1()` returns `None` | Timeout, filtering, routing, or no protocol reply | Check route, target, timeout, and a simultaneous authorized capture |
| Warning about Layer 2 destination | Wrong send function for packet shape | Use `send`/`sr` for IP or `sendp`/`srp` for `Ether` frames |
| Checksum displays as `None` | Packet has not been assembled | Use `show2()` or serialize and re-dissect the packet |
| Edited packet keeps old checksum | Cached dependent fields remain | Delete length/checksum fields before rebuilding |
| Capture consumes too much memory | Every packet is stored | Use `store=False`, finite limits, or stream to a `PcapWriter` |
| Windows shows no usable adapters | Npcap installation or privilege problem | Install Npcap API-compatible mode and use an Administrator terminal |
| Packets look truncated or malformed | Snap length, link type, or decoder mismatch | Inspect the PCAP metadata and raw bytes in a second analyzer |
| Duplicate or unexpected packets | Multiple interfaces, retries, or local capture behavior | Record `sniffed_on`, disable retries, and narrow the interface list |

Use `conf.verb = 2` or a function's `verbose=True` on a tiny experiment when
diagnosing behavior. Avoid verbose logs around sensitive packet contents, and
restore quiet output after finding the issue.

<a id="common-recipes"></a>

## [Common recipes](#contents "Back to contents")

<details>
<summary>Show recipes</summary>

```python
# 1. Build and inspect locally; this sends nothing
pkt = IP(dst="192.0.2.10") / ICMP()
pkt.show2()
hexdump(pkt)

# 2. Send one bounded ICMP request in an authorized lab
reply = sr1(pkt, timeout=2, retry=0, verbose=False)
print(reply.summary() if reply else "no response")

# 3. Discover hosts on one small, approved local Ethernet range
request = Ether(dst="ff:ff:ff:ff:ff:ff") / ARP(pdst="192.0.2.0/29")
ans, _ = srp(
    request, iface="eth0", timeout=2, inter=0.1,
    retry=0, verbose=False,
)
for _, response in ans:
    print(response.psrc, response.hwsrc)

# 4. Capture bounded DNS metadata without retaining packets
sniff(
    iface="eth0", filter="port 53", timeout=30,
    prn=lambda p: p.summary(), store=False,
)

# 5. Read a PCAP and count common network layers without sending traffic
pkts = rdpcap("capture.pcap")
for layer in (ARP, IP, IPv6, TCP, UDP, ICMP, DNS):
    print(layer.__name__, sum(layer in p for p in pkts))

# 6. Stream only TCP packets into a new PCAP
with PcapReader("input.pcap") as reader, PcapWriter(
    "tcp-only.pcap", sync=True
) as writer:
    for packet in reader:
        if TCP in packet:
            writer.write(packet)
```

</details>

Start by constructing, expanding, and displaying packets without sending.
Then run one bounded exchange, confirm the result and target behavior, and
expand only when the authorization and evidence justify it.

<a id="safety-and-scope"></a>

## [Safety and scope](#contents "Back to contents")

- Obtain explicit authorization for the source system, interfaces, target
  addresses, protocols, packet types, rates, payloads, and testing window.
- Inspect generated packet sets with `list()` or a loop before transmission;
  field combinations can silently multiply the number of packets.
- Use `timeout`, `count`, `inter`, and `retry=0` to make experiments bounded.
  Avoid `loop`, flood helpers, and high-rate replay unless separately approved.
- Prefer offline PCAP analysis when live capture or transmission is unnecessary.
- Do not use source spoofing, ARP cache manipulation, malformed traffic,
  protocol fuzzing, or denial-of-service techniques without specific approval
  and an isolated environment designed for them.
- Stop if a service degrades, monitoring teams request it, or packets leave the
  approved route or target range.
- Treat captured payloads and PCAP files as sensitive. Minimize collection,
  redact before sharing, restrict access, and follow the agreed retention plan.
- Verify conclusions with another tool or protocol client; silence, decoded
  fields, and response flags are observations rather than proof of a finding.

<details>
<summary>Show the minimum help commands</summary>

```python
import scapy
print(scapy.__version__)

from scapy.all import *
lsc()
ls(IP)
help(send)
help(sr1)
help(sniff)
help(rdpcap)
```

</details>

Official Scapy 2.7.0 reference and project:

- <https://scapy.readthedocs.io/en/stable/>
- <https://scapy.readthedocs.io/en/stable/usage.html>
- <https://scapy.readthedocs.io/en/stable/api/scapy.sendrecv.html>
- <https://github.com/secdev/scapy>
