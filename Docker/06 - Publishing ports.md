# Why you cannot reach a container yet

Back in the running-containers topic you started `nginx` in the background. It was running - but if you tried to open it from the host, nothing answered. That is not a bug. It is isolation doing its job.

## A container has its own network

Just as a container has its own filesystem and process list, it has its own network. When nginx listens on port 80, it listens on port 80 *inside the container*, on the container's private network. That port is not automatically visible on the host machine.

So the service is up and reachable from inside its own network, but the host - and anything outside - has no path to it. The container is sealed by default, which is a sensible security stance: nothing is exposed unless you say so.

## The fix is to publish a port

To reach a container's service from the host, you tell Docker to publish the container's port onto a port on the host. Docker then forwards traffic that arrives at the host port into the container.

You set this up when you run the container, with the `-p` flag, which the next node covers. The mental model to hold now:

- Inside the container: the service listens on its own port.
- Publishing: Docker opens a host port and forwards it inward.
- From the host: you connect to the host port and reach the service.

Without publishing, a container's ports stay private. This is the single most common reason a beginner's "the server is running but I cannot connect" question has such a simple answer.

← Previous

# Publishing with -p

The `-p` flag (short for `--publish`) maps a host port to a container port. Its syntax is `-p HOST:CONTAINER`.

## Mapping a port

This runs nginx and forwards host port 8080 to the container's port 80:

```
docker run -d -p 8080:80 --name web nginx
```

Read the mapping left to right: traffic arriving at `8080` on the host goes to `80` inside the container, where nginx is listening. Now a request to the host on port 8080 reaches the web server.

<img width="413" height="62" alt="image" src="https://github.com/user-attachments/assets/b592bcf0-2299-4521-8408-734aadc13377" />

You can confirm it from the host with curl:

```
curl http://localhost:8080
```

You get back nginx's welcome page - proof the request travelled from the host port into the container.

## The order matters

It is always host first, container second. Mixing them up is a classic mistake. `-p 80:8080` would forward host port 80 to container port 8080, where nothing is listening, and the connection would fail. When a published service will not respond, checking the order of the two numbers is the first thing to do.

## Choosing the host port

The container port is fixed by whatever the service listens on - nginx is 80, Postgres is 5432, Redis is 6379. The host port is your choice. Pick any free one. If a host port is already taken, Docker refuses to start the container with a "port is already allocated" error, so use a different host port.

## Seeing published ports

`docker ps` shows active mappings in its PORTS column, like `0.0.0.0:8080->80/tcp`. That line confirms host 8080 is wired to container 80.

## Commands this node introduces

- `docker run -p HOST:CONTAINER ...` - publish a container port to a host port

← Previous

# EXPOSE versus publish

You will see the word "expose" around Docker and it is easy to assume it opens a port. It does not. Keeping `EXPOSE` and publishing straight saves a lot of confusion.

## EXPOSE is documentation

Many images declare an `EXPOSE` line in their build definition, for example nginx exposes 80. This is metadata - it records which port the service listens on, as a note to whoever runs the image. It does not open the port on the host and it does not publish anything. A container with an exposed but unpublished port is still unreachable from outside.

You can read an image's exposed ports with `docker inspect`, and `docker ps` marks a running container's exposed-but-unpublished ports too. Think of `EXPOSE` as a label saying "this service uses port 80", nothing more.

## Publishing is what actually opens a port

Only `-p` at run time actually forwards a host port into the container. If you want to reach the service, you publish; exposing alone never does it.

So the two ideas are:

- **EXPOSE** - a note in the image about which port the service uses. No effect on connectivity.
- **publish (-p)** - a real port forward set up when you run the container. This is what lets traffic in.

## Publishing all exposed ports with -P

There is a shortcut that ties the two together. Capital `-P` (short for `--publish-all`) publishes every port the image exposes, each to a random free host port:

```
docker run -d -P --name web nginx
```

Docker picks the host ports for you; `docker ps` shows which ones it chose. This is handy for quick tests, but for anything you care about you use lowercase `-p` with a specific host port so the mapping is predictable.

## Commands this node introduces

- `docker run -P ...` - publish all exposed ports to random host ports
