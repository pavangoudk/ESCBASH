# Linux Networking Notes

## 1. IP addresses and interfaces

A **network interface** is a connection used by a machine to send and receive traffic.

Common interfaces:

- `lo` — loopback interface, usually `127.0.0.1`; traffic stays on the local machine.
- `eth0`, `ens3`, or `enp0s3` — physical or virtual network interface used for external traffic.

Useful commands:

- `ip addr show` or `ip a` — display interfaces and addresses.
- `hostname -I` — print the machine’s IP addresses only.

An address such as `10.0.2.15/24` means:

- `10.0.2.15` is the IPv4 address.
- `/24` identifies the network size.

`ifconfig` is an older tool. Prefer `ip` on modern Linux systems.

## 2. Ports and listening services

An IP address identifies a machine. A **port** identifies a service on that machine.

Common ports:

| Port | Typical service |
| --- | --- |
| `22` | SSH |
| `80` | HTTP |
| `443` | HTTPS |
| `5432` | PostgreSQL |
| `6379` | Redis |

Ports range from `0` through `65535`.

### TCP and UDP

- **TCP:** Connection-oriented, ordered, and reliable; commonly used by HTTP, SSH, and databases.
- **UDP:** Connectionless and lightweight; commonly used by DNS, VoIP, and some games.

### Find listening sockets

Use `ss -tlnp`.

Options:

- `-t` — TCP only
- `-l` — listening sockets only
- `-n` — show numeric ports and addresses
- `-p` — show the owning process

This helps answer: **Which process is using this port?**

### Understanding bind addresses

- `0.0.0.0:80` — listens on port 80 on all IPv4 interfaces.
- `127.0.0.1:5432` — listens only on the local machine.

If a service cannot be reached remotely, check whether it is bound to `127.0.0.1` instead of an external interface or `0.0.0.0`.

`netstat -tlnp` is the older equivalent of `ss -tlnp`.

## 3. Testing connectivity with `ping`

`ping` tests basic network reachability.

- `ping -c 4 127.0.0.1` — send four packets to the local machine.
- `ping -c 4 example.com` — test reachability to a remote host.

A successful remote ping generally shows that:

1. The hostname resolved.
2. A network route exists.
3. The remote host responded.

A failed ping does **not** always mean the host is down. Firewalls and networks may intentionally block ICMP traffic.

## 4. Testing HTTP with `curl`

`curl` communicates with HTTP services.

Common commands:

- `curl http://localhost` — retrieve the response body.
- `curl -I http://localhost` — retrieve response headers only.
- `curl -v http://localhost` — display detailed connection and request information.

`curl -I` is useful for quickly checking whether a web service returns a status such as `HTTP/1.1 200 OK`.

`curl -v` helps troubleshoot connection, request, and response problems.

If `curl` is unavailable, `wget -qO- http://localhost` can retrieve the page content.

## 5. Resolving hostnames

### `dig`

`dig` queries DNS directly.

- `dig example.com` — detailed DNS response.
- `dig +short example.com` — display only the returned IP addresses.

`dig` requires access to a DNS server and does not normally check `/etc/hosts`.

### `getent hosts`

`getent hosts` uses the system’s configured resolver.

- It checks `/etc/hosts`.
- It can query DNS when necessary.
- It is useful for confirming local hostname mappings.

Example: `getent hosts localhost` typically returns `127.0.0.1 localhost`.

### Key difference

| Tool | Main behavior |
| --- | --- |
| `dig` | Queries DNS directly |
| `getent hosts` | Uses the system resolver, including `/etc/hosts` |

## 6. The `/etc/hosts` file

`/etc/hosts` provides local hostname-to-IP mappings. Linux commonly checks it before DNS.

Example entry:

`127.0.0.1 api.local`

Add an entry with:

`echo "127.0.0.1 api.local" >> /etc/hosts`

Confirm it with:

`getent hosts api.local`

Then test the local web service with:

`curl http://api.local`

This technique is useful for local development and testing hostname-based configurations.

## Troubleshooting workflow

1. Check interfaces and addresses with `ip a`.
2. Confirm the local IP with `hostname -I`.
3. Check whether the service is listening with `ss -tlnp`.
4. Verify the service is bound to the correct address.
5. Test basic reachability with `ping`.
6. Test HTTP behavior with `curl -I` or `curl -v`.
7. Resolve names with `dig` or `getent hosts`.
8. Check `/etc/hosts` when testing local hostname overrides.

## Core mental model

- **Interface:** Where traffic enters or leaves.
- **IP address:** Which machine to reach.
- **Port:** Which service to reach.
- **`ss`:** What is listening.
- **`ping`:** Can the host respond?
- **`curl`:** Does the HTTP service work?
- **`dig` / `getent`:** What IP does the hostname resolve to?
