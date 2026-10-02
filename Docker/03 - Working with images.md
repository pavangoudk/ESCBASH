# docker pull

`docker run` pulls an image automatically when it is missing, but often you want to download an image ahead of time without running it - to warm a cache before a demo, or to grab a specific version. That is what `docker pull` does:

```
docker pull redis
```

Docker contacts the registry, downloads the image, and stores it locally. Next time you run or pull it, nothing downloads.

## Images come in layers

Watch the output of a pull and you will see several lines, each with its own progress bar, ending in "Pull complete". Each line is a layer. An image is not one big file. It is built from a stack of pieces called layers, and Docker downloads them separately. Each layer is frozen once it is built - nothing running later can change it - so Docker can safely store it once and share it.

This sharing pays off. If two images share a base layer - say both build on `debian` - Docker stores that layer once and reuses it. Pull a second image that shares layers with one you already have and those layers show as "Already exists" instead of downloading again.

## Where images come from by default

When you pull `redis`, Docker expands it to `docker.io/library/redis` and fetches from Docker Hub. `docker.io` is the default registry and `library` is the namespace for official images. You only spell out the full path when pulling from a different registry, which comes up later in the skill.

## Commands this node introduces

- `docker pull IMAGE` - download an image without running it

# Tags and versions

An image reference is `name:tag`. The tag names a specific version of the image. Getting tags right is one of the most important habits in Docker, because it decides exactly what code runs.

## The default is latest

Leave the tag off and Docker assumes `:latest`:

```
docker pull nginx
```

is the same as:

```
docker pull nginx:latest
```

Despite the name, `latest` does not mean "the newest version forever". It is just the tag an image gets when none is specified. Whoever publishes the image decides what `latest` currently points at, and they can move it to a new version at any time.

## Why latest is risky in real work

Because `latest` can change under you, two machines that both pull `nginx:latest` a month apart can end up running different versions. That is the opposite of the reproducibility Docker is supposed to give you. In production and in automated build pipelines (CI) you pin a concrete version instead:

```
docker pull nginx:1.27
docker pull postgres:16
docker pull python:3.12-slim
```

Now every machine that pulls `nginx:1.27` gets the same image.

## Variant tags

Many official images publish variant tags built on smaller bases. You will see these constantly:

- `alpine` variants (like `python:3.12-alpine`) build on tiny Alpine Linux and produce much smaller images.
- `slim` variants (like `python:3.12-slim`) strip out extras to save space while staying on a Debian base.

Smaller images pull faster and have less inside them to worry about, which is why you will reach for `alpine` and `slim` tags often in this skill.


# docker images and inspect

After pulling a few images you will want to see what is on the machine and how much space it is using.

## Listing images

`docker images` lists every image stored locally:

```
docker images
```

Each row shows the repository (the name), the tag, a short image ID, when it was created, and its size. The same underlying image can appear under more than one tag - that is normal, and they share storage rather than doubling it.

## Reading the size column

The size column is where the `alpine` and `slim` tags prove their worth. A full `python:3.12` image is over 300 MB, while `python:3.12-slim` is a fraction of that and `python:3.12-alpine` smaller still. On a machine with limited disk, those differences add up quickly.

## Inspecting one image

For the full details of a single image - its layers, environment variables, default command, exposed ports, and more - use `docker inspect`:

```
docker inspect nginx
```

It prints a large block of structured JSON. You rarely need all of it, but it is the authoritative source when you need to know exactly how an image is configured, for example what command it runs by default.

## Commands this node introduces

- `docker images` - list images stored locally
- `docker inspect IMAGE` - show full details of an image as JSON


# Removing images

Images take disk space, and on a machine with a fixed disk you will eventually need to clear out ones you no longer use. This machine has 8 GB, so keeping the image list tidy matters.

## Removing a single image

`docker rmi` removes an image by name or ID:

```
docker rmi redis
```

Docker deletes the image and any of its layers that no other image needs. Shared layers stay if something else still uses them.

## The in-use error

You cannot remove an image while a container is using it, even a stopped container:

```
Error response from daemon: conflict: unable to remove repository
reference "redis" (must force) - container 3f2a is using its referenced
image
```

This is Docker protecting you. Remove or stop the containers built from the image first with `docker rm`, then remove the image. This is why the container and image cleanup order is: containers first, images second.

## Removing dangling images with prune

Over time you accumulate dangling images - layers left behind by rebuilds, shown as `<none>` in `docker images`. Clear them in one step:

```
docker image prune
```

It asks for confirmation, then deletes untagged, unused image layers. It is safe - it only removes images nothing references. A fuller cleanup command comes later in the skill.

## Commands this node introduces

- `docker rmi IMAGE` - remove an image
- `docker image prune` - remove dangling, unreferenced image layers

