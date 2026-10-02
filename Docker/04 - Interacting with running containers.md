# An interactive shell with -it

So far every container ran a command and exited. Often you want to poke around inside one interactively - open a shell, look at the filesystem, try a few commands. That is what the `-it` flags are for.

## Running a shell

Start a container and give it a shell as its command:

```
docker run -it alpine sh
```

You land at a prompt inside the container. Commands you type run inside it - `ls`, `cat`, `whoami` - all against the container's own filesystem, isolated from the host. Type `exit` to leave, and because the shell was the main process, the container stops when you do.

## What -it actually means

`-it` is two flags combined:

- `-i` (`--interactive`) keeps the input stream open, so what you type reaches the container.
- `-t` (`--tty`) allocates a terminal, giving you a proper prompt with line editing.

You almost always use them together as `-it` when you want an interactive session. Leave them off and the shell would start with no way to type into it and exit immediately.

## Which shell to ask for

Not every image has the same shell. Small images like `alpine` ship only `sh`, not ``. Larger ones like `ubuntu` and `debian` include ``. If `` is not found, fall back to `sh`:

```
docker run -it ubuntu 
docker run -it alpine sh
```

## Commands this node introduces

- `docker run -it IMAGE sh` - start a container with an interactive shell
- `-i` and `-t` - keep input open and allocate a terminal

# docker exec

`docker run` always creates a new container. But when a service is already running - an nginx container that has been up for an hour - you do not want a new one. You want to run a command inside the one that is already there. That is `docker exec`.

## Running a command in a running container

Given a running container named `web`, run a command inside it:

```
docker exec web ls /usr/share/nginx/html
```

Docker runs `ls` inside the existing `web` container and shows the result. The container keeps running the whole time - `exec` starts an extra process alongside the main one, it does not restart anything.

## Getting a shell in a running container

Combine `exec` with `-it` to open an interactive shell inside a live container. This is the single most common way to debug a running service:

```
docker exec -it web sh
```

Now you are inside the running `web` container. You can check config files, look at what the process sees, test network calls from its point of view, then `exit` - and the container keeps running, because you only exited your extra shell, not the main process.

## run versus exec

This distinction trips up beginners, so hold onto it:

- `docker run` - create and start a brand new container from an image.
- `docker exec` - run a command inside a container that is already running.

× docker run IMAGE- Builds a brand new container from an image and starts it

× docker exec CONTAINER- Runs a command inside a container that is already running

If a container is not running, `exec` has nothing to attach to and fails. Start it first.

## Commands this node introduces

- `docker exec CONTAINER command` - run a command in a running container
- `docker exec -it CONTAINER sh` - open a shell in a running container


# docker logs

When something goes wrong with a container, the first question is always the same: what did it print? A container's output does not vanish when it runs detached - Docker captures it, and `docker logs` shows it.

## Reading a container's output

```
docker logs web
```

This prints everything the container's main process has written to its standard output and standard error since it started. For a web server, that is the access and error log. For an app, it is whatever the app prints. This is how you read logs from a detached container you never attached to.

## Following logs live

Add `-f` (short for `--follow`) to stream new log lines as they happen, the way `tail -f` works on a file:

```
docker logs -f web
```

The command keeps running and prints each new line as the container produces it. Press Ctrl and C to stop following - that only stops the log stream, it does not stop the container.

## Limiting the output

A container that has run for a while can have a huge log. Show only the last lines with `--tail`, or only recent entries with `--since`:

```
docker logs --tail 20 web
docker logs --since 5m web
```

`--tail 20` shows the last twenty lines; `--since 5m` shows everything from the last five minutes. Both keep you from scrolling through thousands of old lines when you only care about what just happened.

## Commands this node introduces

- `docker logs CONTAINER` - show a container's output
- `docker logs -f CONTAINER` - follow the log stream live
- `docker logs --tail N CONTAINER` - show only the last N lines
- `docker logs --since TIME CONTAINER` - show only recent lines

# docker cp

Sometimes you need to move a file between your machine and a container - pull out a log file to attach to a ticket, or drop a config file in to test a change. `docker cp` copies files in either direction.

## Copying out of a container

The form is `docker cp SOURCE DESTINATION`, where a container path is written as `container:/path`. To copy a file out of the `web` container onto the host:

```
docker cp web:/etc/nginx/nginx.conf ./nginx.conf
```

Now you have the container's nginx config as a file on the host, where you can read or edit it with normal tools.

## Copying into a container

Swap the order to copy from the host into the container:

```
docker cp ./index.html web:/usr/share/nginx/html/index.html
```

This drops your local `index.html` into the container's web root. It is handy for a quick test, though for anything permanent you would bake the file into an image or mount it as a volume, both covered later.

## When to reach for it

`docker cp` is a debugging and one-off tool. It is perfect for extracting a crash log or a generated report from a running container. It is not how you manage a container's real data over time - that is what volumes are for, a couple of topics ahead.

## Commands this node introduces

- `docker cp CONTAINER:/path ./local` - copy a file out of a container
- `docker cp ./local CONTAINER:/path` - copy a file into a container
