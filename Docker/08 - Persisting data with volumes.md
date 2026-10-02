# Why container data disappears

Picture this. You run a database in a container, use it for a week, then during cleanup you remove that container with `docker rm`. Every row, every table, gone. Not corrupted, not hidden somewhere - genuinely deleted. This surprises almost everyone the first time, and understanding why it happens points straight at the fix.

## Where a container keeps the files it writes

Think of the image as a sealed, ready-made package - the app and everything it needs, locked so nothing can change it. When you start a container from that image, the container cannot write into the sealed package. So Docker gives it a small scratch area of its own, sitting on top of the package, where all new writes go.

Every file the running container creates or changes goes into that scratch area: log files, uploaded images, database rows, everything. It is the container's private workspace.

## The workspace belongs to the container

That scratch workspace is part of the container itself. Stop the container and the workspace is still there, so `docker start` brings your data back. But `docker rm` deletes the container - and the workspace goes with it. The files are destroyed for good.

## Why Docker does this on purpose

This is by design, not a bug. Containers are meant to be disposable. You should be able to kill one and start a fresh copy without a second thought - that is what makes them easy to replace and scale. If a container quietly hoarded important data, throwing it away would be dangerous. So Docker keeps each container's own storage throwaway on purpose, and gives you a separate place to put data you actually want to keep.

## The fix: store data outside the container

Anything that must outlive the container has to live outside that scratch workspace, on storage the container only borrows. Docker gives you two ways to do this:

- **bind mounts** - map a folder from the host machine into the container.
- **named volumes** - a storage area Docker creates and manages for you, kept separate from any single container.

Both point a location inside the container at storage that lives on its own, so removing the container leaves the data untouched. The next two nodes cover each in turn. The rule to carry with you: never keep data you care about inside a container.

# Bind mounts

Sometimes the data you care about already lives in a folder on the host machine, and you want the container to work with it directly - the code you are editing, or a config file you maintain by hand. A bind mount does exactly that.

A bind mount maps a folder on the host straight into a container. The container sees that host folder as one of its own directories, and both sides see the same files at the same time. Change a file on the host and the container sees the change instantly, and the other way around.

## Mounting a host folder

You set a bind mount with `-v HOST_PATH:CONTAINER_PATH`, using absolute paths:

```
docker run -d -p 8080:80 \
  -v /root/site:/usr/share/nginx/html \
  --name web nginx
```

This points the container's web root at the host folder `/root/site`. Whatever files sit in `/root/site` on the host are what nginx serves. Edit a file in `/root/site` with any host editor and the change is live in the container immediately - no rebuild, no copy.

## What it is good for

Bind mounts shine during development. You keep your source code on the host, mount it into the container, and edit with your normal tools while the containerised app picks up the changes. They are also how you inject a config file from the host into a container.

## The catches

A bind mount ties the container to a specific host path, so it is not portable - the exact folder must exist on whatever machine runs it. And the mount hides whatever was at the container path before: if the host folder is empty, the container sees an empty directory there, even if the image shipped files at that location. The host side always wins.

For data you want Docker to manage and keep portable - a database's files, for instance - a named volume is usually the better fit, which is the next node.

## Commands this node introduces

- `docker run -v /host/path:/container/path ...` - bind-mount a host folder into the container


# Named volumes

A named volume is storage Docker creates and manages for you, with a name you choose. Unlike a bind mount, you do not pick a host path - Docker keeps the data in its own area and you refer to it by name.

## Creating and using one

You can create a volume explicitly:

```
docker volume create appdata
```

Then mount it into a container with the same `-v` flag, but using the volume's name instead of a host path:

```
docker run -d \
  -v appdata:/var/lib/postgresql/data \
  --name db postgres:16-alpine
```

Docker points the container's data directory at the `appdata` volume. If the volume does not exist yet, Docker creates it automatically on first use, so the explicit `create` step is often optional.

## Why the data now survives

The volume lives independently of the container. Remove the `db` container with `docker rm` and the `appdata` volume stays, holding all the database files. Start a new Postgres container mounting the same `appdata` volume and the data is right there - same rows, same tables. The container was disposable; the data was not.

## Named volume or bind mount

A rough guide:

- **Named volume** - for data Docker should own and manage: databases, application state, anything you just want to persist and not think about the host path for. Portable and the default choice for real data.
- **Bind mount** - for when you specifically need a known host folder: live-editing source code in development, or feeding in a config file.

## Commands this node introduces

- `docker volume create NAME` - create a named volume
- `docker run -v NAME:/container/path ...` - mount a named volume


# Managing volumes

Named volumes stick around by design, which means you also need to see them, inspect them, and clean them up.

## Listing volumes

```
docker volume ls
```

This lists every named volume on the machine. You will see ones you created by name, plus long random-named volumes that Docker created automatically for images that declare a volume but were run without an explicit `-v`.

## Inspecting a volume

```
docker volume inspect appdata
```

This shows the volume's details, including `Mountpoint` - the actual path on the host where Docker stores the data. You rarely touch that path directly, but it is there when you need it.

## Removing a volume

```
docker volume rm appdata
```

This deletes the volume and everything in it, permanently. Docker refuses if a container is still using the volume - remove the container first. Because a volume often holds the only copy of real data, deleting one is a step to take deliberately.

## Cleaning up unused volumes

Volumes that no container references pile up over time and quietly eat disk. Clear the unused ones with:

```
docker volume prune
```

It asks for confirmation, then removes every volume no container is using. Be careful - "unused" includes a database volume whose container you removed but whose data you meant to keep. Check `docker volume ls` before pruning so you do not throw away data you still need.

## Commands this node introduces

- `docker volume ls` - list volumes
- `docker volume inspect NAME` - show a volume's details
- `docker volume rm NAME` - remove a volume
- `docker volume prune` - remove all unused volumes

