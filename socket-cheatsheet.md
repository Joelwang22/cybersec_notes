# Python `socket`: Quick Networking Guide

The standard-library `socket` module exposes the operating system's low-level
networking interface. It is most often used for TCP and UDP clients, servers,
small protocol tools, and learning how network applications work.

Only connect to, listen on, or test systems and networks you own or have
explicit permission to use. The examples use loopback and documentation-only
addresses so they do not target real external hosts.

```python
import socket

with socket.create_connection(("127.0.0.1", 8000), timeout=5) as sock:
    sock.sendall(b"hello\n")
    reply = sock.recv(4096)
```

- `socket` is included with Python; no package installation is required.
- Network data is `bytes`. Encode outgoing text and decode incoming text.
- TCP is a byte stream: one `sendall()` does not correspond to one `recv()`.
- UDP preserves datagram boundaries but does not guarantee delivery or order.
- Examples assume Python 3 and were checked against the Python 3.14 docs.

## Contents

- [Pick a workflow](#pick-a-workflow)
- [Core concepts](#core-concepts)
- [TCP client](#tcp-client)
- [TCP server](#tcp-server)
- [UDP client and server](#udp-client-and-server)
- [Addresses and name resolution](#addresses-and-name-resolution)
- [Sending, receiving, and framing](#sending-receiving-and-framing)
- [Blocking, timeouts, and non-blocking mode](#blocking-timeouts-and-non-blocking-mode)
- [Socket options](#socket-options)
- [Handling multiple clients](#handling-multiple-clients)
- [Binary protocols](#binary-protocols)
- [TLS](#tls)
- [Unix-domain sockets](#unix-domain-sockets)
- [Errors and cleanup](#errors-and-cleanup)
- [Inspection and debugging](#inspection-and-debugging)
- [Common recipes](#common-recipes)
- [Troubleshooting](#troubleshooting)
- [Security checklist](#security-checklist)
- [Quick reference](#quick-reference)

<a id="pick-a-workflow"></a>

## [Pick a workflow](#contents "Back to contents")

| Goal | Starting point |
|---|---|
| Connect to a TCP service | `socket.create_connection((host, port), timeout=5)` |
| Create an IPv4 TCP socket manually | `socket.socket(socket.AF_INET, socket.SOCK_STREAM)` |
| Create an IPv4 UDP socket | `socket.socket(socket.AF_INET, socket.SOCK_DGRAM)` |
| Listen for TCP connections | `bind()` -> `listen()` -> `accept()` |
| Send an entire TCP buffer | `sock.sendall(data)` |
| Receive up to 4096 bytes | `data = sock.recv(4096)` |
| Send one UDP datagram | `sock.sendto(data, (host, port))` |
| Receive one UDP datagram | `data, addr = sock.recvfrom(65535)` |
| Resolve IPv4 and IPv6 addresses | `socket.getaddrinfo(host, port, type=...)` |
| Add a five-second timeout | `sock.settimeout(5)` |
| Wait on many sockets | `selectors.DefaultSelector()` |
| Wrap a connected TCP socket in TLS | `ssl_context.wrap_socket(...)` |

Prefer `create_connection()` for ordinary TCP clients. It resolves both IPv4
and IPv6 addresses and tries returned addresses until one succeeds.

<a id="core-concepts"></a>

## [Core concepts](#contents "Back to contents")

### Family, type, and protocol

```python
sock = socket.socket(family, type, protocol)
```

| Value | Meaning | Typical use |
|---|---|---|
| `AF_INET` | IPv4 | Internet TCP or UDP |
| `AF_INET6` | IPv6 | Internet TCP or UDP |
| `AF_UNIX` | Local pathname/namespace | Local IPC; mainly Unix-like systems |
| `SOCK_STREAM` | Reliable byte stream | TCP |
| `SOCK_DGRAM` | Individual datagrams | UDP |
| `SOCK_RAW` | Raw protocol access | Specialized, privileged, OS-specific work |

The protocol argument is normally omitted or left as `0`; the operating system
selects TCP for `SOCK_STREAM` and UDP for `SOCK_DGRAM` in the Internet families.

### TCP and UDP at a glance

| Property | TCP / `SOCK_STREAM` | UDP / `SOCK_DGRAM` |
|---|---|---|
| Connection | Connect before exchanging data | Usually connectionless |
| Delivery | Reliable and ordered while connected | May be lost, duplicated, or reordered |
| Data model | Unstructured byte stream | Separate datagrams |
| Message boundaries | Application must define them | Preserved by the socket API |
| Typical methods | `connect`, `sendall`, `recv` | `sendto`, `recvfrom` |
| Peer shutdown | `recv()` returns `b""` | No equivalent stream EOF |

### Client and server lifecycles

```text
TCP server                            TCP client
socket()                              socket() / create_connection()
bind((host, port))                    connect((host, port))
listen()
accept()  <-------------------------> sendall() / recv()
recv() / sendall()
close()                               close()

UDP server                            UDP client
socket()                              socket()
bind((host, port))                    sendto(data, server)
recvfrom() <------------------------> recvfrom()
sendto(data, client)
close()                               close()
```

`accept()` returns a new connected socket. The original listening socket stays
open so it can accept more connections.

<a id="tcp-client"></a>

## [TCP client](#contents "Back to contents")

`create_connection()` is the simplest robust starting point for a TCP client.

<details>
<summary>Show a small TCP client</summary>

```python
import socket

HOST = "127.0.0.1"
PORT = 8000

try:
    with socket.create_connection((HOST, PORT), timeout=5) as sock:
        sock.sendall(b"hello\n")

        reply = sock.recv(4096)
        if reply == b"":
            raise ConnectionError("server closed the connection")

        print(reply.decode("utf-8", errors="replace"))
except socket.timeout:
    print("connection or I/O timed out")
except OSError as exc:
    print(f"network error: {exc}")
```

</details>

The timeout remains on the returned socket. Change it after connecting if the
connection phase and I/O phase need different limits:

```python
with socket.create_connection((HOST, PORT), timeout=3) as sock:
    sock.settimeout(10)
    sock.sendall(request)
    response = sock.recv(4096)
```

### Manual TCP client

Use the manual form when the address family or socket options must be chosen
before connecting.

```python
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
    sock.settimeout(5)
    sock.connect(("127.0.0.1", 8000))
    sock.sendall(b"hello\n")
    reply = sock.recv(4096)
```

<a id="tcp-server"></a>

## [TCP server](#contents "Back to contents")

Bind to loopback during development. Binding to `0.0.0.0` or `::` exposes the
service on every suitable interface unless a firewall blocks it.

<details>
<summary>Show a sequential echo server</summary>

```python
import socket

HOST = "127.0.0.1"
PORT = 8000

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server:
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind((HOST, PORT))
    server.listen()
    print(f"listening on {HOST}:{PORT}")

    while True:
        conn, addr = server.accept()
        with conn:
            print(f"connected: {addr}")
            conn.settimeout(30)

            while data := conn.recv(4096):
                conn.sendall(data)
```

</details>

This server handles one client at a time. A slow client delays all later
clients; use threads, processes, `selectors`, or `asyncio` when concurrency is
required.

### Binding to an automatically selected port

Port `0` asks the operating system to choose an available ephemeral port.

```python
server.bind(("127.0.0.1", 0))
host, port = server.getsockname()
print(f"chosen port: {port}")
```

This is especially useful for tests. Do not find a free port by briefly opening
and closing another socket; another process could claim it before the server.

### A connection handler

Keep per-client protocol logic separate from accepting connections:

```python
def handle_client(conn, addr):
    conn.settimeout(30)
    with conn:
        while data := conn.recv(4096):
            conn.sendall(data)


with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server:
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(("127.0.0.1", 8000))
    server.listen()

    while True:
        conn, addr = server.accept()
        handle_client(conn, addr)
```

<a id="udp-client-and-server"></a>

## [UDP client and server](#contents "Back to contents")

UDP sends complete datagrams. A single `recvfrom(size)` returns at most one
datagram and its sender address. If the buffer is too small, the excess may be
discarded, so choose a size suitable for the protocol.

<details>
<summary>Show a UDP echo server</summary>

```python
import socket

with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as server:
    server.bind(("127.0.0.1", 9999))

    while True:
        data, addr = server.recvfrom(65535)
        print(f"received {len(data)} bytes from {addr}")
        server.sendto(data, addr)
```

</details>

<details>
<summary>Show a UDP client</summary>

```python
import socket

server_addr = ("127.0.0.1", 9999)

with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as sock:
    sock.settimeout(2)
    sock.sendto(b"hello", server_addr)

    try:
        data, addr = sock.recvfrom(65535)
    except socket.timeout:
        print("no reply received")
    else:
        print(f"reply from {addr}: {data!r}")
```

</details>

Calling `connect()` on a UDP socket does not perform a TCP-style handshake. It
sets a default peer, allows `send()` and `recv()`, and normally filters incoming
datagrams to that peer:

```python
with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as sock:
    sock.connect(("127.0.0.1", 9999))
    sock.send(b"hello")
    data = sock.recv(65535)
```

If the application needs reliability, replay protection, ordering, retries, or
fragment handling over UDP, those rules must be designed into its protocol.

<a id="addresses-and-name-resolution"></a>

## [Addresses and name resolution](#contents "Back to contents")

| Family | Address passed to methods |
|---|---|
| IPv4 / `AF_INET` | `(host, port)` |
| IPv6 / `AF_INET6` | `(host, port, flowinfo, scope_id)`; last two often omitted |
| Unix / `AF_UNIX` | Filesystem path or platform-specific namespace |

Common bind hosts:

| Host | Meaning |
|---|---|
| `"127.0.0.1"` | IPv4 loopback only |
| `"::1"` | IPv6 loopback only |
| `"0.0.0.0"` | All local IPv4 interfaces |
| `"::"` | All local IPv6 interfaces; dual-stack behavior is OS/config dependent |

Use `getaddrinfo()` rather than `gethostbyname()` for new code because it can
return both IPv4 and IPv6 candidates.

```python
results = socket.getaddrinfo(
    "localhost",
    8000,
    family=socket.AF_UNSPEC,
    type=socket.SOCK_STREAM,
)

for family, socktype, proto, canonname, sockaddr in results:
    print(family, socktype, proto, sockaddr)
```

Each result is `(family, type, protocol, canonical_name, socket_address)`. Limit
the `type` and optionally `proto` so the results fit the application.

### Useful resolution helpers

| Goal | Function |
|---|---|
| Get local hostname | `socket.gethostname()` |
| Get a fully qualified name | `socket.getfqdn()` |
| Resolve IPv4 and IPv6 candidates | `socket.getaddrinfo(host, port, ...)` |
| Reverse-resolve an address | `socket.gethostbyaddr(address)` |
| Convert socket address to display names | `socket.getnameinfo(sockaddr, flags)` |
| Look up a service's port | `socket.getservbyname("https", "tcp")` |
| Look up a port's service | `socket.getservbyport(443, "tcp")` |

DNS results can contain multiple addresses and can change. Do not assume the
first result is always IPv4, stable, or reachable. `create_connection()` handles
the ordinary TCP-client case by trying candidates for you.

### IPv4 and IPv6 address conversion

```python
packed = socket.inet_pton(socket.AF_INET6, "2001:db8::1")
text = socket.inet_ntop(socket.AF_INET6, packed)

assert text == "2001:db8::1"
```

`inet_aton()` and `inet_ntoa()` are IPv4-only. Prefer `inet_pton()` and
`inet_ntop()` in code that may handle either address family.

<a id="sending-receiving-and-framing"></a>

## [Sending, receiving, and framing](#contents "Back to contents")

### Text must be encoded

```python
message = "hello"
sock.sendall(message.encode("utf-8"))

data = sock.recv(4096)
text = data.decode("utf-8")
```

Decode only at a protocol-defined text boundary. One TCP `recv()` may return
half a UTF-8 character, half a message, one message, or several messages.

### `send()` versus `sendall()`

| Method | Result |
|---|---|
| `send(data)` | Sends some bytes and returns the count; caller handles the rest |
| `sendall(data)` | Keeps sending until all bytes are accepted or an error occurs |
| `sendto(data, addr)` | Sends a datagram to an address and returns its byte count |

Use `sendall()` for most blocking TCP code. If using `send()`, account for
partial writes:

```python
view = memoryview(data)

while view:
    sent = sock.send(view)
    view = view[sent:]
```

### `recv()` semantics

```python
chunk = sock.recv(4096)

if chunk == b"":
    print("peer performed an orderly shutdown")
```

The argument is a maximum, not an exact requested length. TCP may return fewer
bytes even when the connection remains open.

### Read exactly N bytes

```python
def recv_exact(sock, size):
    chunks = bytearray()

    while len(chunks) < size:
        chunk = sock.recv(size - len(chunks))
        if not chunk:
            raise EOFError("connection closed before message was complete")
        chunks.extend(chunk)

    return bytes(chunks)
```

### Length-prefixed messages

A common protocol format is a fixed four-byte big-endian length followed by
that many payload bytes.

```python
import struct

MAX_MESSAGE = 1_048_576


def send_message(sock, payload):
    if len(payload) > MAX_MESSAGE:
        raise ValueError("message is too large")
    sock.sendall(struct.pack("!I", len(payload)) + payload)


def recv_message(sock):
    (length,) = struct.unpack("!I", recv_exact(sock, 4))
    if length > MAX_MESSAGE:
        raise ValueError("declared message is too large")
    return recv_exact(sock, length)
```

Always cap peer-controlled lengths before allocating or reading. Without a
limit, a tiny malicious header can make a server wait for or allocate enormous
messages.

### Newline-delimited messages

`makefile()` is convenient for line-oriented blocking protocols:

```python
with socket.create_connection(("127.0.0.1", 8000), timeout=5) as sock:
    with sock.makefile("rwb") as stream:
        stream.write(b"hello\n")
        stream.flush()
        reply = stream.readline(65537)

        if len(reply) > 65536 or not reply.endswith(b"\n"):
            raise ValueError("invalid or oversized line")
```

Remember to `flush()` buffered writes. The official documentation cautions
that a timeout can leave a `makefile()` buffer inconsistent; explicit framing
with `recv()` is often easier when timeouts and recovery matter.

### Half-closing a TCP connection

```python
sock.sendall(request)
sock.shutdown(socket.SHUT_WR)  # no more outgoing data

response = bytearray()
while chunk := sock.recv(4096):
    response.extend(chunk)
```

`shutdown()` disables communication directions; `close()` releases the socket.
Only use EOF as message framing when the protocol says the sender will send no
more messages on that connection.

<a id="blocking-timeouts-and-non-blocking-mode"></a>

## [Blocking, timeouts, and non-blocking mode](#contents "Back to contents")

| Mode | Set with | Behavior |
|---|---|---|
| Blocking | `sock.setblocking(True)` | Wait until an operation can proceed |
| Timeout | `sock.settimeout(seconds)` | Wait up to the given duration |
| Non-blocking | `sock.setblocking(False)` | Return immediately if not ready |

`setblocking(True)` is equivalent to `settimeout(None)`. `setblocking(False)`
is equivalent to `settimeout(0.0)`.

```python
sock.settimeout(5.0)

try:
    data = sock.recv(4096)
except socket.timeout:
    print("operation took too long")
```

Set finite timeouts on network-facing services so idle or stalled peers do not
hold resources forever. Treat a timeout as protocol state: retrying blindly can
duplicate an operation if the peer received the request but its reply was lost.

### Non-blocking operations

In non-blocking mode, `connect()`, `send()`, and `recv()` may not finish
immediately. Code normally waits for readiness using `selectors` rather than
repeatedly retrying in a busy loop.

```python
sock.setblocking(False)

try:
    data = sock.recv(4096)
except BlockingIOError:
    data = None  # no data is currently ready
```

`connect_ex()` returns an error code instead of raising for an in-progress or
failed connection. Completing a non-blocking connect correctly is subtle;
prefer `asyncio.open_connection()` or a tested event-loop abstraction unless
the low-level behavior is the point of the program.

<a id="socket-options"></a>

## [Socket options](#contents "Back to contents")

```python
sock.setsockopt(level, option, value)
value = sock.getsockopt(level, option)
```

| Option | Purpose | Notes |
|---|---|---|
| `SOL_SOCKET, SO_REUSEADDR` | Rebind a recently used server address | Semantics differ by OS; set before `bind()` |
| `SOL_SOCKET, SO_KEEPALIVE` | Enable TCP keepalive probes | Timings require OS-specific options |
| `IPPROTO_TCP, TCP_NODELAY` | Disable Nagle's algorithm | Consider only for latency-sensitive small writes |
| `SOL_SOCKET, SO_BROADCAST` | Permit IPv4 UDP broadcast sends | Broadcast is local-scope and network dependent |
| `IPPROTO_IPV6, IPV6_V6ONLY` | Control whether an IPv6 socket accepts IPv4-mapped traffic | Defaults and support vary |

`SO_REUSEADDR` is commonly used for TCP development servers:

```python
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(("127.0.0.1", 8000))
```

Do not copy socket options without understanding their platform semantics.
`SO_REUSEPORT`, raw sockets, multicast, keepalive tuning, and exclusive address
use are particularly OS-specific.

### Inspect endpoints

```python
local_addr = sock.getsockname()
peer_addr = sock.getpeername()  # connected sockets only

print(f"local={local_addr}, peer={peer_addr}")
```

<a id="handling-multiple-clients"></a>

## [Handling multiple clients](#contents "Back to contents")

| Model | Good fit | Main trade-off |
|---|---|---|
| Sequential loop | Tests and one-client tools | One slow client blocks others |
| Thread per client | Modest numbers of blocking connections | Threads and shared state need care |
| `selectors` | Many low-level sockets in one thread | Must manage buffers and state explicitly |
| `asyncio` | High-level asynchronous applications | Requires async design throughout |
| Process pool | CPU-heavy work | Higher IPC and process overhead |

### Thread-per-client server

<details>
<summary>Show a threaded echo server</summary>

```python
import socket
import threading


def handle_client(conn, addr):
    with conn:
        conn.settimeout(30)
        try:
            while data := conn.recv(4096):
                conn.sendall(data)
        except (ConnectionError, socket.timeout):
            pass


with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server:
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(("127.0.0.1", 8000))
    server.listen()

    while True:
        conn, addr = server.accept()
        thread = threading.Thread(
            target=handle_client,
            args=(conn, addr),
            daemon=True,
        )
        thread.start()
```

</details>

For a real service, also bound total connections, queues, message sizes, and
per-client work. Daemon threads are convenient for a demo but are stopped
abruptly when the process exits; production shutdown should signal and join
workers cleanly.

### `selectors` skeleton

`selectors.DefaultSelector` chooses the best readiness mechanism available on
the platform.

```python
import selectors
import socket

selector = selectors.DefaultSelector()

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(("127.0.0.1", 8000))
server.listen()
server.setblocking(False)
selector.register(server, selectors.EVENT_READ, data=None)

try:
    while True:
        for key, events in selector.select(timeout=1):
            if key.data is None:
                conn, addr = key.fileobj.accept()
                conn.setblocking(False)
                selector.register(conn, selectors.EVENT_READ, data=addr)
            else:
                conn = key.fileobj
                data = conn.recv(4096)
                if data:
                    conn.sendall(data)  # use output buffering in real non-blocking code
                else:
                    selector.unregister(conn)
                    conn.close()
finally:
    selector.close()
    server.close()
```

This is a learning skeleton, not a complete event-driven server. A real
non-blocking server must keep per-connection input/output buffers and register
for `EVENT_WRITE` while queued output remains; `sendall()` can still block or
raise `BlockingIOError` on a non-blocking socket.

<a id="binary-protocols"></a>

## [Binary protocols](#contents "Back to contents")

Use `struct` to encode fixed binary fields. Network protocols conventionally
use big-endian, called network byte order.

```python
import struct

packet = struct.pack("!IHB", 0x12345678, 443, 1)
identifier, port, enabled = struct.unpack("!IHB", packet)
```

| Prefix | Byte order and alignment |
|---|---|
| `!` | Network (big-endian), standard sizes |
| `>` | Big-endian, standard sizes |
| `<` | Little-endian, standard sizes |
| `@` | Native order, native sizes/alignment; usually wrong on the wire |

Useful integer helpers:

```python
wire = number.to_bytes(4, byteorder="big", signed=False)
number = int.from_bytes(wire, byteorder="big", signed=False)
```

Validate identifiers, versions, flags, reserved bits, lengths, and text
encodings before acting on decoded input.

<a id="tls"></a>

## [TLS](#contents "Back to contents")

Plain sockets do not provide encryption or peer authentication. Use the `ssl`
module for TLS; do not invent encryption around `send()` and `recv()`.

<details>
<summary>Show a verified TLS client</summary>

```python
import socket
import ssl

hostname = "example.com"
context = ssl.create_default_context()

with socket.create_connection((hostname, 443), timeout=5) as raw_sock:
    with context.wrap_socket(raw_sock, server_hostname=hostname) as tls_sock:
        print(tls_sock.version())
        print(tls_sock.getpeercert())
```

</details>

`create_default_context()` enables certificate verification and secure defaults.
Pass the expected DNS name as `server_hostname` so hostname verification and
Server Name Indication work. Avoid disabling verification outside a controlled,
purpose-built test.

For application protocols, use a maintained protocol library such as
`http.client`, `urllib`, or a suitable third-party client when possible. Those
libraries handle details beyond opening a socket.

<a id="unix-domain-sockets"></a>

## [Unix-domain sockets](#contents "Back to contents")

`AF_UNIX` sockets provide local inter-process communication without an IP port.
Availability and path rules vary; filesystem Unix sockets are common on Linux,
macOS, and modern Windows versions.

```python
import socket

address = "/tmp/example.sock"

with socket.socket(socket.AF_UNIX, socket.SOCK_STREAM) as sock:
    sock.connect(address)
    sock.sendall(b"hello\n")
```

A server calls `bind(address)`, `listen()`, and `accept()` as with TCP. For a
filesystem address, ensure the parent directory and socket-file permissions do
not allow unintended users to connect. Remove only the exact stale socket path
you own, and clean it up during orderly shutdown.

Linux also supports an abstract namespace represented by a leading null byte,
for example `b"\0example"`; it is not portable.

<a id="errors-and-cleanup"></a>

## [Errors and cleanup](#contents "Back to contents")

Most socket failures raise `OSError` or a subclass.

| Exception | Typical meaning |
|---|---|
| `socket.timeout` / `TimeoutError` | An operation exceeded its timeout |
| `ConnectionRefusedError` | No service accepted the TCP connection |
| `ConnectionResetError` | The peer or network reset the connection |
| `BrokenPipeError` | Writing after the peer closed its receive side |
| `socket.gaierror` | Name/address resolution failed |
| `BlockingIOError` | Non-blocking operation is not ready yet |
| `OSError` | Other socket or operating-system error |

Catch the narrowest exception that the program can meaningfully handle:

```python
try:
    with socket.create_connection((host, port), timeout=5) as sock:
        sock.sendall(payload)
except socket.gaierror as exc:
    print(f"name resolution failed: {exc}")
except ConnectionRefusedError:
    print("the host refused the connection")
except socket.timeout:
    print("the operation timed out")
except OSError as exc:
    print(f"network error: {exc}")
```

Use context managers so exceptions still close sockets:

```python
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
    sock.connect(("127.0.0.1", 8000))
    ...
```

Closing is idempotent, but a closed socket cannot be reused. A TCP connection
also cannot be disconnected and then connected to an unrelated peer; create a
new socket.

### Graceful server shutdown pattern

```python
server.settimeout(1)

while not stop_event.is_set():
    try:
        conn, addr = server.accept()
    except socket.timeout:
        continue
    handle_client(conn, addr)
```

The short timeout lets the loop periodically check a thread-safe stop signal.
Another common design uses `selectors` plus a wake-up socket.

<a id="inspection-and-debugging"></a>

## [Inspection and debugging](#contents "Back to contents")

```python
print(sock.family)         # e.g. AddressFamily.AF_INET
print(sock.type)           # e.g. SocketKind.SOCK_STREAM
print(sock.proto)          # protocol number
print(sock.gettimeout())   # None, 0.0, or timeout seconds
print(sock.getsockname())  # local endpoint
print(sock.getpeername())  # remote endpoint when connected
print(sock.fileno())       # OS descriptor/handle indicator
```

Safe local checks:

| Goal | Windows PowerShell | Linux/macOS |
|---|---|---|
| Test a local TCP port | `Test-NetConnection 127.0.0.1 -Port 8000` | `nc -vz 127.0.0.1 8000` |
| Show listening TCP sockets | `Get-NetTCPConnection -State Listen` | `ss -ltn` |
| Show UDP endpoints | `Get-NetUDPEndpoint` | `ss -lun` |

Log message sizes and sanitized endpoint metadata, not secrets or raw sensitive
payloads. Packet capture can expose credentials and private data; capture only
within authorization and store captures securely.

<a id="common-recipes"></a>

## [Common recipes](#contents "Back to contents")

### Receive until EOF with a size limit

```python
def recv_until_eof(sock, max_bytes=1_048_576):
    result = bytearray()

    while chunk := sock.recv(4096):
        result.extend(chunk)
        if len(result) > max_bytes:
            raise ValueError("response exceeded limit")

    return bytes(result)
```

This is appropriate only for protocols where EOF terminates the response.

### Find the local address selected for a destination

Connecting a UDP socket selects a route without needing to send application
data. Use a documentation address rather than a real target in examples:

```python
with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as sock:
    sock.connect(("192.0.2.1", 9))
    local_ip, local_port = sock.getsockname()
```

The result reflects routing configuration and does not prove the destination is
reachable.

### Dual-stack-friendly TCP client helper

```python
def exchange(host, port, request, timeout=5):
    with socket.create_connection((host, port), timeout=timeout) as sock:
        sock.sendall(request)
        return sock.recv(4096)
```

### Explicitly try `getaddrinfo()` results

Use this pattern only when `create_connection()` is not flexible enough:

```python
def open_tcp(host, port, timeout=5):
    last_error = None
    addresses = socket.getaddrinfo(
        host,
        port,
        family=socket.AF_UNSPEC,
        type=socket.SOCK_STREAM,
    )

    for family, socktype, proto, _, sockaddr in addresses:
        sock = socket.socket(family, socktype, proto)
        sock.settimeout(timeout)
        try:
            sock.connect(sockaddr)
            return sock
        except OSError as exc:
            last_error = exc
            sock.close()

    if last_error is not None:
        raise last_error
    raise OSError("no usable addresses returned")
```

### Read one bounded line without `makefile()`

```python
def recv_line(sock, max_bytes=65536):
    data = bytearray()

    while len(data) < max_bytes:
        byte = sock.recv(1)
        if not byte:
            raise EOFError("connection closed before newline")
        data += byte
        if byte == b"\n":
            return bytes(data)

    raise ValueError("line exceeded limit")
```

This simple version performs many small reads. Production code normally keeps
a reusable receive buffer and searches it for delimiters.

### Test a client and server locally

1. Start the TCP server in one terminal.
2. Run the TCP client in another terminal.
3. Keep `HOST = "127.0.0.1"` in both programs.
4. If the port is occupied, select another unprivileged port such as `18000`.
5. Stop the server with `Ctrl+C` after testing.

Loopback testing avoids exposing the unfinished service to the network.

<a id="troubleshooting"></a>

## [Troubleshooting](#contents "Back to contents")

### `ConnectionRefusedError`

- Confirm the server is running and has already called `listen()`.
- Confirm client and server use the same address and port.
- Check whether one side uses IPv4 (`127.0.0.1`) and the other IPv6 (`::1`).
- Check local firewall policy and container/virtual-machine port forwarding.

### `OSError: [Errno 98] Address already in use` or Windows error 10048

- Another process may already own the address.
- A previous TCP connection may still be in a closing state.
- Set `SO_REUSEADDR` before `bind()` where its platform semantics fit.
- For tests, bind port `0` and read the chosen port with `getsockname()`.

Do not assume `SO_REUSEADDR` allows multiple active servers to share a port on
every operating system.

### `recv()` hangs

- The peer may not have sent data yet.
- The protocol may be waiting for a delimiter, length, flush, or half-close.
- Add a timeout and log protocol state.
- Never use “read until EOF” if the peer expects to keep the connection open.

### `recv()` returns less data than expected

This is normal for TCP. Accumulate bytes until the protocol's delimiter,
declared length, or EOF condition is satisfied.

### `recv()` returns `b""`

The TCP peer closed its sending side. Break the receive loop. Repeated calls do
not restore the connection.

### `TypeError: a bytes-like object is required`

Encode text before sending:

```python
sock.sendall(text.encode("utf-8"))
```

Decode received bytes only after assembling a complete text unit:

```python
text = complete_message.decode("utf-8", errors="strict")
```

### UDP reply never arrives

- UDP delivery is not guaranteed; use a timeout and protocol-aware retry rule.
- Ensure the receiver bound the expected address and port before the send.
- Check firewall, NAT, routing, and datagram size.
- Verify the server replies to the source address returned by `recvfrom()`.

### Works on one OS but not another

Address reuse, dual-stack IPv6 behavior, Unix sockets, raw sockets, error codes,
and many socket options are platform dependent. Check the Python documentation
and the target operating system's socket documentation.

<a id="security-checklist"></a>

## [Security checklist](#contents "Back to contents")

- Bind to loopback unless remote access is intentionally required.
- Authenticate and authorize peers; an IP address alone is rarely identity.
- Use TLS for confidential or authenticated network traffic.
- Set connection, read, write, and idle deadlines appropriate to the protocol.
- Cap concurrent clients, queued work, message lengths, and total response size.
- Validate every peer-controlled length, type, state transition, and encoding.
- Do not unpickle network data or pass it directly to a shell, SQL query, path,
  template, or interpreter.
- Avoid logging credentials, tokens, private keys, or complete sensitive payloads.
- Treat DNS names, reverse DNS, peer-supplied hostnames, and headers as untrusted.
- For outbound services that accept user-provided destinations, defend against
  server-side request forgery and re-check resolved addresses as needed.
- Prefer higher-level, maintained protocol libraries over hand-written parsers.
- Test malformed, slow, truncated, oversized, duplicated, and out-of-order input.
- Keep active probing and packet transmission within documented authorization.

Raw sockets and packet capture/injection are privileged and platform specific.
Use Scapy or OS-specific facilities when raw packet work is genuinely required;
the ordinary TCP/UDP socket API is the right tool for application networking.

<a id="quick-reference"></a>

## [Quick reference](#contents "Back to contents")

### Module helpers

| API | Purpose |
|---|---|
| `socket.socket(family, type, proto=0)` | Create a socket |
| `socket.create_connection(address, timeout=...)` | Resolve and connect a TCP socket |
| `socket.socketpair()` | Create a connected pair where supported |
| `socket.getaddrinfo(host, port, ...)` | Resolve connection candidates |
| `socket.getnameinfo(sockaddr, flags)` | Convert a socket address to names/text |
| `socket.gethostname()` | Return the local hostname |
| `socket.getfqdn(name="")` | Return a fully qualified domain name |
| `socket.inet_pton(family, text)` | Convert an IP address to packed bytes |
| `socket.inet_ntop(family, packed)` | Convert packed bytes to IP text |
| `socket.htons()` / `ntohs()` | Convert 16-bit host/network byte order |
| `socket.htonl()` / `ntohl()` | Convert 32-bit host/network byte order |

### Socket methods

| Method | Purpose |
|---|---|
| `bind(address)` | Assign a local address |
| `listen(backlog=...)` | Mark a TCP socket as listening |
| `accept()` | Return `(connected_socket, peer_address)` |
| `connect(address)` | Connect to a peer |
| `connect_ex(address)` | Connect and return an error code instead of raising |
| `send(data)` | Send some connected-socket bytes |
| `sendall(data)` | Send all connected TCP bytes or raise |
| `sendto(data, address)` | Send a datagram |
| `recv(max_bytes)` | Receive connected-socket bytes |
| `recv_into(buffer)` | Receive into a writable buffer |
| `recvfrom(max_bytes)` | Return `(datagram, sender_address)` |
| `settimeout(seconds)` | Set a finite timeout; `None` means blocking |
| `setblocking(flag)` | Select blocking or non-blocking mode |
| `setsockopt(level, option, value)` | Set an OS socket option |
| `getsockname()` | Return the local endpoint |
| `getpeername()` | Return the connected peer endpoint |
| `shutdown(how)` | Disable reads, writes, or both |
| `makefile(mode)` | Create a buffered file wrapper |
| `close()` | Release the socket |

### Constants

| Constant | Meaning |
|---|---|
| `AF_INET` / `AF_INET6` | IPv4 / IPv6 family |
| `AF_UNIX` | Local Unix-domain family |
| `SOCK_STREAM` / `SOCK_DGRAM` | Stream / datagram type |
| `SHUT_RD` / `SHUT_WR` / `SHUT_RDWR` | Directions passed to `shutdown()` |
| `SOL_SOCKET` | General socket-option level |
| `SO_REUSEADDR` | Address-reuse option |
| `SO_KEEPALIVE` | TCP keepalive option |
| `IPPROTO_TCP` / `IPPROTO_UDP` | TCP / UDP protocol identifiers |
| `TCP_NODELAY` | Disable Nagle's algorithm on TCP |

## Further reading

- [Python `socket` module documentation](https://docs.python.org/3/library/socket.html)
- [Python Socket Programming HOWTO](https://docs.python.org/3/howto/sockets.html)
- [Python `selectors` documentation](https://docs.python.org/3/library/selectors.html)
- [Python `ssl` documentation](https://docs.python.org/3/library/ssl.html)
- [IANA IPv4 special-purpose address registry](https://www.iana.org/assignments/iana-ipv4-special-registry/iana-ipv4-special-registry.xhtml)
