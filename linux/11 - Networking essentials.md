# Ps and interfaces

Every machine on a network has one or more **network interfaces**. An interface is a lane the machine uses to send and receive traffic. Most Linux servers have at least two:

- `lo` - the **loopback**, address `127.0.0.1`. Traffic that never leaves the machine (one process talking to another on the same box) goes here.
- `eth0` (or `ens3`, `enp0s3`, depends on the naming scheme) - the main outbound interface, with the machine's real network address.

## Listing interfaces

The modern command is `ip`:

```
ip addr show           # long form
ip a                   # short form, same result
```

You'll see one block per interface. The key line for each is `inet <address>/<mask>`, which is the IPv4 address:

```
inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
```

`10.0.2.15` is the machine's IP on that interface. `/24` is the size of the network it belongs to.

## Printing just the IP with hostname -I

If you only need the IP:

```
hostname -I
```

Prints only IP addresses, space-separated. Perfect for scripts.

## What about ifconfig?

The older command, still floating around in tutorials. Not installed by default on modern Ubuntu. Use `ip a` and don't look back.

# Ports and listening

An IP address gets you to a machine. A **port** gets you to a specific service on that machine. Ports are 16-bit numbers (0 to 65535). Common examples:

- **22** - SSH
- **80** - HTTP
- **443** - HTTPS
- **5432** - PostgreSQL
- **6379** - Redis

## TCP vs UDP

Two flavors of traffic. **TCP** is connection-based, ordered, and retries on packet loss. HTTP, SSH, and most databases use TCP. **UDP** is fire-and-forget. DNS, VoIP, and some game protocols use UDP. As a DevOps engineer you'll deal with TCP almost every time.

## Seeing what's listening

`ss` (socket statistics) is the modern tool.

```
ss -tlnp
```

Four flags you'll always use together:

- `-t` - TCP only
- `-l` - listening sockets only
- `-n` - don't resolve port numbers to service names (faster and clearer)
- `-p` - show which process owns the socket

This machine runs an nginx web server, so port 80 shows up when you run it yourself:

```
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port  Process
LISTEN 0      511    0.0.0.0:80          0.0.0.0:*          users:(("nginx",pid=1234,fd=6))
LISTEN 0      128    0.0.0.0:22          0.0.0.0:*          users:(("sshd",pid=1500,fd=3))
```

Read it one line at a time. The first line says a process named `nginx` is listening on port 80. The second says `sshd` is listening on port 22. The `Process` column ties each open port back to the program that opened it, which is exactly what you want when you're hunting down "what is using this port?"

## 0.0.0.0 vs 127.0.0.1 in the Local Address

The single most useful distinction to recognize:

- `0.0.0.0:80` means "listen on port 80 on every interface". Anyone who can reach the machine over the network can connect. That's why nginx shows up this way above.
- `127.0.0.1:5432` means "listen on the loopback only". Only processes on this same machine can connect. A database bound like this is reachable from the box itself but not from outside.

When a service is "unreachable from another machine," a common cause is that it bound to `127.0.0.1` when you needed `0.0.0.0`. `ss -tlnp` shows you that in one line.

## netstat, if you meet it

The older equivalent of `ss`. Same idea, older syntax (`netstat -tlnp`). Use `ss` unless you're on a system that lacks it, which is rare in 2026.


# Reaching other machines

Three commands cover almost every "can I reach that machine?" check: `ping` tests raw reachability, `curl` speaks HTTP, and `dig` (or `getent`) resolves names to IP addresses.

## ping: test raw reachability

Start with the machine itself. The loopback address always answers, so this works with no internet at all:

```
ping -c 4 127.0.0.1
```

`-c 4` sends four packets and stops. You'll see four reply lines and a summary that says `0% packet loss`.

To reach a remote machine, hand ping a name or an IP instead:

```
ping -c 4 example.com
```

A successful ping to a remote host proves three things at once: the name resolved to an IP, the network path works, and the remote machine is answering. That second example needs working internet and DNS, so it fails on a machine with no outside access. Some networks also block ping on purpose, so a failed ping by itself doesn't prove a machine is down.

## curl: talk HTTP

This machine runs an nginx web server on port 80, so you can talk to it with no internet at all:

```
curl http://localhost
```

That prints the HTML of the nginx welcome page. The same two flags cover most of the rest of your HTTP work:

```
curl -I http://localhost
curl -v http://localhost
```

- `-I` sends a HEAD request and prints only the response headers. Fastest way to answer whether the service is returning 200. The first line reads `HTTP/1.1 200 OK`.
- `-v` is verbose. It shows every step: the connection, the request headers you sent, and the response headers you got back. Reach for it when something is going wrong.

Point curl at a public URL like `https://example.com` the same way, but that one needs working internet.

If curl isn't installed, wget fetches the same page:

```
wget -qO- http://localhost
```

## Resolving names to IPs

`dig` asks a DNS server to turn a name into an IP. It queries that server over the network, so it needs working internet:

```
dig example.com
dig +short example.com
```

Plain `dig` prints a detailed answer; `+short` prints just the IPs. `dig` is the DevOps standard, but on a machine with no outside access it times out instead of answering.

`getent hosts` is the offline-friendly alternative. It goes through the system resolver, which checks `/etc/hosts` first and only then asks DNS, so a local name answers instantly with no network:

```
getent hosts localhost
```

That prints `127.0.0.1 localhost`. One important difference: `dig` only ever queries DNS servers, so it never sees names you put in `/etc/hosts`. `getent` does. When you want to confirm a `/etc/hosts` entry, `getent` is the tool.

## /etc/hosts

Before DNS existed, hostnames were mapped to IPs in a plain text file. That file still exists as `/etc/hosts`, and Linux checks it before asking any DNS server. It's handy for testing.

Add a line that points `api.local` at the loopback. Appending to `/etc/hosts` needs root, and you're logged in as root here:

```
echo "127.0.0.1  api.local" >> /etc/hosts
```

Confirm the resolver picked it up:

```
getent hosts api.local
```

That prints `127.0.0.1 api.local`. The name now points at this machine, where nginx is listening on port 80, so a request to it gets a real answer back:

```
curl http://api.local
```

Setting local overrides like this is a standard trick during development.

