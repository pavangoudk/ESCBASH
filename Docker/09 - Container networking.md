# Docker networks

Real applications are rarely one container. A web app talks to a database, a cache, a queue. For that to work, containers need a way to reach each other. Docker networks provide it.

## Listing networks

Docker sets up a few networks out of the box. List them with:

```
docker network ls
```

You will see at least three: `bridge`, `host`, and `none`. The one that matters most is `bridge`, the default network every container joins unless you say otherwise.

A "bridge" here is just Docker's word for a private little network on your machine that containers plug into, a bit like a network switch in a server room that everything cables into so it can talk.

## The default bridge

When you run a container with no network options, it joins the default `bridge` network and gets a private IP address on it. Containers on the bridge can reach each other by IP address, and Docker connects the bridge to the outside so containers can reach the internet and you can publish ports to the host.

## The problem with the default bridge

There is a catch that trips people up. On the default bridge, containers can reach each other by IP address but **not** by name. Those IP addresses are assigned by Docker and change when containers restart, so hardcoding them is useless. That makes the default bridge awkward for connecting an app to its database - you have no stable address to point at.

## The fix: user-defined networks

The solution is to create your own network. On a user-defined network, Docker adds automatic name resolution: containers reach each other by container name, which is stable. That single feature is why nearly every multi-container setup uses a user-defined network, and it is the subject of the next node.

## Commands this node introduces

- `docker network ls` - list networks



# User-defined networks

A user-defined network is one you create yourself. Its headline feature is a built-in phone book: every container on the network can reach every other by container name, and Docker looks up the current address for you. No IP addresses to memorize, no addresses that break on restart.

## Creating a network

```
docker network create appnet
```

This creates a bridge network named `appnet`. `docker network ls` now lists it alongside the defaults.

## Joining containers to it

Attach a container to the network at run time with `--network`:

```
docker run -d --network appnet --name db postgres:16-alpine
docker run -d --network appnet --name api myapp
```

Both containers are now on `appnet`. Because they share a user-defined network, `api` can reach `db` using the name `db` - no IP addresses, no guessing.

## Connecting and disconnecting later

You do not have to decide at run time. You can attach a running container to a network, or detach it:

```
docker network connect appnet web
docker network disconnect appnet web
```

A container can even sit on more than one network at once, which is how you isolate groups of services - a database reachable only by the app tier, for instance.

## Inspecting a network

To see which containers are attached to a network and their addresses:

```
docker network inspect appnet
```

## Commands this node introduces

- `docker network create NAME` - create a user-defined network
- `docker run --network NAME ...` - attach a container at run time
- `docker network connect / disconnect NAME CONTAINER` - attach or detach a running container
- `docker network inspect NAME` - show a network's details



# Reaching a container by name

The payoff of a user-defined network is that a container name works as a hostname. This is how the app tier finds the database, and it is worth seeing concretely.

## The container name is the hostname

Put two containers on the same user-defined network and one can use the other's name anywhere it would use a hostname. Suppose `db` and `api` are both on `appnet`. Inside `api`, the address of the database is simply `db`:

```
postgres://shop:s3cret@db:5432/orders
```

Docker resolves `db` to the current IP of the `db` container automatically. If `db` restarts and gets a new IP, the name still resolves - that is exactly why you use the name and never a raw IP.

## Proving it works

You can test name resolution from inside one container. Attach two containers to a network, then from one, reach the other by name. For example, with a running `web` container reachable by that name, another container on the same network can fetch its page:

```
docker exec api wget -qO- http://web
```

The request goes out using the name `web`, Docker resolves it, and the page comes back. (`wget` is used because small images such as `alpine` and `busybox` include it; many do not include `curl`.) No published port is needed for this - port publishing (`-p`) is only for reaching a container from the host. Container to container traffic on a shared network flows directly.

## The key distinction

Two different connection paths, do not mix them up:

- **Host to container** - needs a published port (`-p`), addressed as `localhost:HOSTPORT`.
- **Container to container** - needs a shared user-defined network, addressed by container name on the service's real port. No `-p` required.

