# docker run

`docker run` is the command you will type more than any other. It takes an image, creates a container from it, and starts the process inside. The basic shape is:

```
docker run IMAGE [command]
```

If the image is not already on the machine, Docker pulls it from the registry first, then runs it.

## Running a command in a container

Most images have a default command, but you can override it by adding your own after the image name. This runs `echo` inside a fresh Alpine Linux container:

```
docker run alpine echo "hello from inside a container"
```

Docker pulls the tiny `alpine` image, starts a container, runs `echo`, and the moment `echo` finishes the container stops. A container lives exactly as long as its main process. When the process exits, so does the container.

## The foreground is the default

By default `docker run` attaches your terminal to the container and shows its output directly, then hands control back when it exits. Try a command that lists the container's own filesystem:

```
docker run alpine ls /
```

You see the root directory of the container, not your host. That is the isolation from the last topic in action - the process sees its own filesystem.

## Pulling happens once

The first `docker run alpine` downloads the image. Every run after that reuses the cached copy and starts almost instantly. You do not pull by hand for normal use - `docker run` handles it.

## Commands this node introduces

- `docker run IMAGE` - create and start a container from an image
- `docker run IMAGE command args` - override the default command



# docker ps

Once containers start running, you need to see them. `docker ps` lists containers.

## Running containers only

With no flags, `docker ps` shows only containers that are running right now:

```
docker ps
```

The output has one row per container with columns for the container ID, the image it came from, the command it is running, when it was created, its status, published ports, and its name.

## All containers, including stopped ones

A container that has exited does not show in plain `docker ps`. To see every container, running or stopped, add `-a` (short for `--all`):

```
docker ps -a
```

This matters because stopped containers do not disappear. Each one you ran and exited is still on the machine, taking up a little disk space, until you remove it. `docker ps -a` is how you find them.

## The container ID and name

Every container gets a long unique ID and a short readable name. If you do not name it yourself, Docker invents one like `dreamy_hopper`. You refer to a container by either its ID (the first few characters are enough) or its name in nearly every other command - `docker stop`, `docker logs`, `docker rm`, and so on.

## Commands this node introduces

- `docker ps` - list running containers
- `docker ps -a` - list all containers, including stopped ones



# Detached mode and naming

The `echo` and `ls` examples ran, printed, and stopped. But a web server or a database is meant to keep running. For those you want the container to run in the background and give it a name you can find it by.

## Running in the background with -d

The `-d` flag (short for `--detach`) starts the container in the background and immediately returns your prompt. Instead of attaching to the output, Docker prints the new container's ID and lets it run:

```
docker run -d nginx
```

This starts an nginx web server. It keeps running because nginx's main process does not exit. Run `docker ps` and you will see it listed as running, unlike the earlier foreground examples that had already stopped.

## Naming with --name

Relying on Docker's random names gets old fast. Give the container a name you choose with `--name`:

```
docker run -d --name web nginx
```

Now you can refer to it as `web` everywhere - `docker stop web`, `docker logs web`, `docker rm web`. Names must be unique. If a container called `web` already exists, even a stopped one, Docker refuses to reuse the name until you remove the old one.

## Foreground versus detached, when to use which

- Foreground (the default) suits short commands and anything whose output you want to watch live.
- Detached (`-d`) suits long-running services - web servers, databases, queues - that should keep running while you do other work.

## Commands this node introduces

- `docker run -d IMAGE` - run a container in the background
- `docker run --name NAME IMAGE` - give the container a chosen name



# Stopping and removing containers

Containers you start in the background keep running until you stop them, and stopped containers stick around until you remove them. Knowing how to clean up is part of running containers.

## Stopping a running container

`docker stop` asks a container's main process to shut down gracefully:

```
docker stop web
```

Docker sends the process a termination signal, waits a few seconds for it to exit cleanly, and forces it if it does not. The container moves from running to stopped. It is not gone - `docker ps -a` still shows it.

A stopped container can be started again with `docker start`:

```
docker start web
```

## Removing a container

To delete a stopped container and free the disk space it used, use `docker rm`:

```
docker rm web
```

You cannot remove a running container this way - stop it first, or force removal with `docker rm -f web`, which stops and deletes in one step.

## Removing automatically with --rm

For throwaway containers you never want to keep, add `--rm` to the run command. Docker deletes the container automatically the moment it exits:

```
docker run --rm alpine echo "gone when done"
```

This keeps `docker ps -a` from filling up with dozens of dead one-shot containers. Use it for quick commands; leave it off for services you may want to restart or inspect later.

## Commands this node introduces

- `docker stop NAME` - stop a running container gracefully
- `docker start NAME` - start a stopped container again
- `docker rm NAME` - remove a stopped container
- `docker rm -f NAME` - force-remove a running container
- `docker run --rm ...` - auto-remove the container when it exits

