# What a Dockerfile is

So far you have run images other people built. Now you build your own. A Dockerfile is the recipe for an image - a plain text file listing the steps to assemble it, one instruction per line.

## From recipe to image to container

Three things, in order:

- The **Dockerfile** is the written recipe.
- `docker build` reads the recipe and produces an **image**.
- `docker run` starts a **container** from that image.

<img width="600" height="83" alt="image" src="https://github.com/user-attachments/assets/438a4c8f-a806-4313-a209-c0bd43a389cf" />


You write the Dockerfile once. Anyone can build the same image from it, and run identical containers anywhere. The Dockerfile is what makes an image reproducible - it is the source code of the image.

## What it looks like

A Dockerfile is a list of instructions, each an uppercase keyword followed by its arguments:

```
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
RUN pip install requests
CMD ["python", "app.py"]
```

Read top to bottom, this says: start from the Python slim image, work in `/app`, copy `app.py` in, install a library, and by default run the app. Each instruction adds to the image.

## The file is named Dockerfile

By convention the file is named exactly `Dockerfile`, with no extension, in the root of your project. `docker build` looks for that name automatically. You can point at a differently named file when you need to, but the default is `Dockerfile`.

## The instructions this topic covers

The next nodes take the core instructions one at a time:

- `FROM` - the base image to build on
- `RUN` - run a command while building the image
- `COPY` - copy files from your project into the image
- `WORKDIR` - set the working directory
- `CMD` - the default command a container runs

These five are enough to build a real image, which you will do by the end of the topic.



# FROM and RUN

Every image builds on another image, and most images need a few setup commands run while they are built. `FROM` and `RUN` handle those two jobs.

## FROM sets the base

`FROM` must be the first real instruction in a Dockerfile. It names the base image you build on top of:

```
FROM python:3.12-slim
```

You are not starting from nothing - you start from an existing image that already has an operating system and, here, Python installed. Your instructions add to it. Choosing a good base matters: a slim or alpine base keeps your final image small, as you saw in the images topic.

## RUN executes a command at build time

`RUN` runs a command inside the image while it is being built, and the result becomes part of the image. It is how you install packages or prepare the filesystem:

```
RUN pip install requests
```

When `docker build` reaches this line, it runs `pip install requests` in the half-built image and saves the outcome. Everything `RUN` installs is baked in, so the container does not install anything at startup - it is already there.

## Build time versus run time

This is a distinction to hold firmly. `RUN` happens once, while the image is being built. It does not run when you later start a container. Installing dependencies, creating folders, compiling code - all of that is build-time work and belongs in `RUN`. What the container does each time it starts is a different instruction, `CMD`, covered shortly.

## Combining RUN commands

Each `RUN` creates a new image layer. Chaining related commands with `&&` in one `RUN` keeps the layer count down:

```
RUN apt-get update && apt-get install -y curl
```

You will see why layers matter for build speed and image size in a later topic.

## Instructions this node introduces

- `FROM image` - set the base image (first instruction)
- `RUN command` - run a command at build time and bake in the result



# COPY and WORKDIR

An image needs your actual project files inside it, and a sensible place to put them. `COPY` brings files in; `WORKDIR` sets where you are working.

## WORKDIR sets the working directory

`WORKDIR` sets the directory that later instructions run in, and that a container starts in. If the directory does not exist, Docker creates it:

```
WORKDIR /app
```

After this line, a `COPY` with a relative destination lands in `/app`, a `RUN` executes from `/app`, and a container starts there. It replaces the awkward alternative of writing `/app/...` on every path, and it is cleaner than running `cd` inside `RUN` commands, which would not stick between instructions.

## COPY brings files into the image

`COPY SOURCE DESTINATION` copies files from your project folder on the host into the image:

```
COPY app.py .
```

The source is a path relative to the build context - your project folder, the subject of the build-and-run node. The destination is a path inside the image; the `.` here means "the current working directory", so with `WORKDIR /app` set, `app.py` lands at `/app/app.py`.

You can copy a whole folder too:

```
COPY . .
```

This copies everything in the build context into the working directory. It is common, but it also copies things you may not want in the image - a concern the `.dockerignore` file solves in a later topic.

## COPY versus the host

`COPY` only sees files inside the build context you pass to `docker build`. It cannot reach arbitrary paths on your host - that isolation is deliberate and keeps builds reproducible. Put the files you need to copy inside your project folder.

## Instructions this node introduces

- `WORKDIR /path` - set the working directory for later instructions and the container
- `COPY src dest` - copy files from the build context into the image


# CMD

`CMD` sets the default command a container runs when it starts. It is what turns an image full of files into an image that actually does something.

## The default command

```
CMD ["python", "app.py"]
```

When you `docker run` this image with no command of your own, the container runs `python app.py`. This is the run-time counterpart to `RUN`: `RUN` did its work once at build time, while `CMD` runs every time a container starts. It is the process whose lifetime is the container's lifetime, the idea from the lifecycle topic.

## The exec form

Write `CMD` as a JSON array of strings - the command, then each argument as its own element:

```
CMD ["nginx", "-g", "daemon off;"]
```

This is called the exec form, and it is the one to use. It runs the command directly, so the process receives stop signals properly and shuts down cleanly. Double quotes are required around each element - it is real JSON.

There is also a shell form, `CMD nginx -g "daemon off;"`, which runs the command through a shell. It looks simpler but wraps your process in a shell that can swallow stop signals, so prefer the exec form.

## Only one CMD wins

A Dockerfile can list several `CMD` lines but only the last one takes effect. There is one default command per image. If you need setup work before it, that is `RUN` at build time, not extra `CMD` lines.

## CMD is a default you can override

`CMD` is only the default. Anything you type after the image name on `docker run` replaces it:

```
docker run myimage echo "different command"
```

This runs `echo` instead of the `CMD`. That flexibility is useful, and a later topic contrasts `CMD` with `ENTRYPOINT`, which changes how overriding works.

## Instructions this node introduces

- `CMD ["exe", "arg"]` - set the default command, exec form preferred



# docker build

With a Dockerfile written, `docker build` turns it into an image. Then you run that image like any other.

## Building an image

From the folder that holds your Dockerfile:

```
docker build -t myapp:1.0 .
```

Two parts matter here:

- `-t myapp:1.0` tags the resulting image with a name and version, so you can refer to it later. Without a tag you get an image with only an ID, which is awkward to use.
- The `.` at the end is the build context - the folder Docker sends to the build so that `COPY` can find files. The dot means "this folder".

Docker reads the Dockerfile, runs each instruction in order, and produces the tagged image. `docker images` then lists `myapp:1.0`.

## The build context

That trailing `.` is more important than it looks. Docker packages up the whole context folder and hands it to the build engine, and `COPY` can only reach files inside it. This is why your Dockerfile and the files it copies live together in one project folder. Point the build at the folder that contains them.

## Reading the build output

Each instruction shows as a step in the output. Docker caches the result of each step, so the next build reuses unchanged steps and only reruns from the first thing that changed. That caching is a topic of its own shortly; for now, notice that a second build of the same files is much faster.

## Running your image

Once built, it is just an image:

```
docker run --rm myapp:1.0
```

The container starts, runs your `CMD`, and behaves exactly like the official images you ran earlier - because it is the same kind of thing, only this one is yours.

## Commands this node introduces

- `docker build -t NAME:TAG .` - build an image from a Dockerfile in the given context

