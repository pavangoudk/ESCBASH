# Container states

A container is not simply on or off. It moves through a small set of states over its life, and knowing which state a container is in tells you what you can do with it.

## The main states

- **created** - the container exists but has never started. You get this from `docker create` (a rarely used cousin of `run`), or briefly before a container starts.
- **running** - the main process is alive. This is what `docker ps` shows.
- **exited** - the main process finished, cleanly or not. The container is stopped but still exists, with its writable layer, until removed.
- **paused** - the container's processes are frozen in place with `docker pause` and can be resumed with `docker unpause`. Used rarely.

## The normal path

<img width="608" height="302" alt="image" src="https://github.com/user-attachments/assets/70c6532d-abd4-432f-80f6-264a88e15709" />


Most containers walk a simple line: created, then running, then exited. A one-shot command like `docker run alpine echo hi` blinks through running and lands in exited almost instantly. A service like nginx sits in running until you stop it, then moves to exited.

## Status in docker ps

The STATUS column in `docker ps -a` spells the state out with detail:

```
Up 3 minutes
Exited (0) 10 seconds ago
Exited (137) 2 minutes ago
```

"Up" means running. "Exited (0)" means it stopped and the process returned exit code 0, a clean finish. "Exited (137)" means it stopped with a non-zero code, which usually signals a problem - the next node is about exactly that.

## Commands this node introduces

- `docker pause CONTAINER` / `docker unpause CONTAINER` - freeze and resume a container



# Why containers exit

A frequent beginner surprise is starting a container and finding it already stopped. There is one rule behind almost every case of this, and it is worth burning into memory.

## A container lives as long as its main process

A container runs exactly one main process, and the container exists only while that process runs. When the process ends, the container exits. There is nothing keeping it alive independently.

This is why `docker run alpine` seems to do nothing - Alpine's default command runs a shell that, with no terminal attached, has no input and exits immediately, so the container exits immediately too. And it is why `docker run -d nginx` keeps running - nginx's process stays up on purpose.

## Exit codes

When the main process ends it returns an exit code, and the container records it. You saw it in the STATUS column as `Exited (N)`:

- **0** - success. The process did its job and finished cleanly.
- **non-zero** - a failure or a signal. `1` and `2` are generic application errors. `137` means the process was killed (often out of memory, or a forced `docker stop`). `139` is a segmentation fault.

Reading the exit code is the first diagnostic step when a container you expected to stay up has exited. A `0` means it finished on purpose; a non-zero means something went wrong and the logs will usually say what.

## A common trap

People try to "keep a container running" by giving it a command that finishes, then wonder why it stops. If the process is meant to be a service, it should be a long-running process (a server, a worker). If you only need a container to sit idle for testing, a command like `sleep 300` keeps it alive for five minutes, because `sleep` does not return until its timer ends.

# Inspecting container state

`docker ps` gives a one-line summary, but when you need the precise state of a container - its exact exit code, when it started, why it stopped - `docker inspect` has the full record.

## Inspecting a container

```
docker inspect web
```

Like inspecting an image, this prints a large JSON document. The part you care about most is the `State` section, which holds the running flag, the exit code, the start and finish timestamps, and whether the container was killed.

## Pulling out one value with --format

Reading all that JSON to find one field is tedious. The `--format` flag extracts exactly the value you want using a small template:

```
docker inspect -f '{{.State.Status}}' web
docker inspect -f '{{.State.ExitCode}}' web
docker inspect -f '{{.State.Running}}' web
```

The first prints the status word (`running`, `exited`), the second the exit code as a number, the third `true` or `false`. This is the precise way to ask "is this container running?" or "what code did it exit with?" in a script, instead of grepping through `docker ps` output.

## Why this matters for automation

When you write scripts or health checks around Docker, you need exact, machine-readable answers. `docker inspect -f` gives them. A deploy script might check `{{.State.Running}}` is `true` before declaring a service healthy, or read `{{.State.ExitCode}}` to decide whether a job container succeeded.

## Commands this node introduces

- `docker inspect CONTAINER` - full container details as JSON
- `docker inspect -f '{{.State.Status}}' CONTAINER` - extract one field with a format template

# Restart policies

A service that crashes at 3am should come back on its own, not wait for someone to notice. A restart policy tells Docker what to do when a container's process exits, so the container can recover without you.

## Setting a policy with --restart

You set the policy when you run the container:

```
docker run -d --restart unless-stopped --name web nginx
```

There are four policies:

- **no** - the default. Docker never restarts the container. When it exits, it stays exited.
- **on-failure** - restart only if the process exits with a non-zero code. A clean exit (code 0) is left alone. You can cap the attempts with `on-failure:5`.
- **always** - restart whenever it stops, for any reason, and also start it again when the Docker daemon itself restarts.
- **unless-stopped** - like `always`, but if you deliberately `docker stop` it, Docker does not bring it back until you start it yourself.

## Which to choose

For a long-running service you want up whenever the machine is up, `unless-stopped` is the usual choice - it survives crashes and reboots but still respects a manual stop. `on-failure` fits a job that should retry a few times on error but not loop forever on a clean finish.

## Restart policies are not a bug fix

A policy restarts a crashing container, but if the app crashes on startup every time, `always` just loops it forever - a crash loop. The restart buys resilience against occasional failures; it does not fix a broken image. When you see a container restarting endlessly, read its logs and fix the cause.

## Commands this node introduces

- `docker run --restart POLICY ...` - set a restart policy (no, on-failure, always, unless-stopped)
