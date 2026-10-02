# What is Docker

## "But it works on my machine!"

Imagine you write a small program on your laptop. It runs perfectly. You send it to a friend to try, and on their computer it breaks.

Why? Because their computer is not set up exactly like yours. Maybe your program needs Python version 3.11 and they have 3.9. Maybe it needs a tool or a library that you installed months ago and forgot about, but they never installed. The program itself is fine; the computer around it is different.

This happens constantly in real work. Software that runs on a developer's laptop falls over on the test server, or on a teammate's machine, because each computer has slightly different versions and settings. People waste hours hunting down these differences.

## What Docker does

Docker fixes this by packing your program together with *everything* it needs to run - the code, the right version of the language, every library, every tool, every setting - into one sealed box.

That sealed box is called a **container**. And **Docker** is the software that builds and runs these boxes for you: you use Docker to create a container, start and stop it, and delete it when you are done.

Because the box already contains everything, it runs the same way no matter whose computer it is on. Your laptop, your friend's computer, the company server: same box, same result. Nobody has to install or configure anything by hand. They just run the box.

A good way to picture it: think of shipping goods across the world. A shipping container is a standard steel box. It does not matter what is inside it or which ship, train, or truck carries it - every port knows how to handle the box. Docker does the same for software. Your program is the goods; the container is the standard box that runs anywhere.

Here is the same idea as a picture. One container, built once, runs unchanged on three very different machines:

```

                             --------------------------------
                                Container
                                your program + its exact
                                versions, libraries, settings
                            -----------------------------------

runs the same                    runs the same                         runs the same

Your laptop                      A friend's computer                   Company server


```
## Why so many people use it

This one idea - "package it once, run it anywhere" - turns out to be useful everywhere in modern software:

- Developers build an app once and share it, knowing it will run the same for everyone.
- Servers run many of these boxes side by side without them interfering with each other.
- Automated systems build, test, and deploy software inside fresh, predictable containers.

You do not need to understand all of that yet. For now, just hold on to the core idea: **a container is a sealed box that holds a program plus everything it needs, so it runs the same on any machine.**

This skill builds up from there. First you will run containers other people made, then build your own, and finally connect several containers into one working application.

← Previous

# Container vs virtual machine

LessonImagine there is a server, and you want to run three apps on it. You do not want them mixed together, because one app's software could clash with another's. So you want to give each app its own isolated space.

There are two ways to do this: virtual machines, or containers. They both give each app its own space, but the way they do it is very different, and that difference is why Docker won.

## Way one: give each app a virtual machine

A **virtual machine** (VM) is a whole fake computer running inside your server. Each VM boots its own complete operating system - its own copy of Linux or Windows - on top of the server's.

So for three apps you now run three full operating systems, on top of the server's own. Each one brings gigabytes of files and takes a minute or two to start, just like a real computer booting up. It works, but it is heavy: most of that memory and disk is spent running operating systems, not your actual apps.

## Way two: give each app a container

A **container** does not boot its own operating system. All three containers share the one operating system already running on the server. Each container wraps just its own app and the files that app needs, and borrows the rest from the server underneath.

Because there is no extra operating system to boot, a container starts in well under a second and is measured in megabytes, not gigabytes. On the same server you could fit a couple of VMs, but dozens of containers.

And this is not just for big servers. Install Docker on your own laptop and you can run several containers right there, side by side, without spinning up a single heavy VM. In this course you will practise all of this in the hands-on labs, which already have Docker set up for you.

## A simple way to picture it

A VM is like building a separate house for every program: each one gets its own foundation, plumbing, and walls. A container is like renting a locked room inside one shared building: you get your own private space, but you share the building's foundation and plumbing with everyone else.
```
Containers: light, start instantly         Virtual machines: heavy, slow to boot          

app + app + app                           app + app + app

shared host OS                            a full OS per app  3 full OSes host OS + hypervisor

hardware                                  hardware
```
Both give a program its own isolated space. The container just does it without dragging a whole extra operating system along, which is why Docker took over.

## Terms this node introduces

- virtual machine (VM) - a full fake computer, with its own operating system, running inside a real one
- container - an isolated program that shares the host's operating system instead of booting its own

← Previous

# Images and containers

LessonYou will hear two words over and over in Docker: **image** and **container**. Beginners mix them up all the time, and it causes confusion later, so let's pin down the difference with a simple comparison.

Think of a recipe in a cookbook.

- The **recipe** is just instructions on paper. It lists everything you need and how to put it together. On its own it does nothing - it just sits in the book.
- The **dish** is what you actually get when you cook the recipe. It is real, it is finished, and you can eat it.

In Docker, the **image is the recipe** and the **container is the dish** you cook from it.

## An image is the recipe (the package)

An **image** is a ready-made package that sits on your computer. It bundles a program together with everything it needs, frozen together. Like a recipe, it does nothing by itself - it just waits until you tell Docker to run it.

Images have a name and a version, written as `name:tag`. For example `nginx:latest`, `python:3.12`, and `redis:alpine` are all images. The part after the colon (the **tag**) is usually the version. If you leave the tag off, Docker assumes `latest`.

## A container is the dish (the running program)

A **container** is what you get when you start an image. Docker takes the image and runs it, and that live, running thing is the container.

Just like you can cook the same recipe many times, you can start many containers from one image:

run

run

run

nginx image
one recipe on disk

container #1

container #2

container #3

Each container is separate from the others. Stopping or deleting one does not affect the image or the other containers - just like throwing away one dish does not touch the recipe or the other dishes.

## Where images come from

You do not have to build every image yourself. Ready-made images live in a **registry**, which is just an online store of images. The main public one is **Docker Hub**, and it already has official images for nginx, postgres, python, and thousands of other tools. You download ("pull") an image once, Docker keeps a copy on your machine, and every run after that is instant.

## Terms this node introduces

- image - the read-only package
- container - a running instance of an image
- tag - the version label after the colon in `name:tag`
- registry - where images are stored and pulled from, Docker Hub by default

← Previous

# Verify the engine

LessonDocker has two parts: a command line client you type into, and a background service called the daemon (or engine) that does the real work of pulling images and running containers. The client sends your commands to the daemon. On this machine the daemon is already running, so you can talk to it straight away.

## docker version

The first thing to check on any machine is whether the client can reach the daemon:

```
docker version
```

This prints two blocks, Client and Server. Seeing both means the client is installed and the daemon is up and answering. If you ever see "Cannot connect to the Docker daemon", the engine is not running - but here it is, so you will get both blocks.

## docker info

For a fuller picture of the engine's state, use:

```
docker info
```

It reports how many images and containers exist, the storage driver, the total memory, and more. It is the go-to command when you want to know what the engine currently holds.

## docker run hello-world

The classic smoke test pulls a tiny image and runs it:

```
docker run hello-world
```

The `hello-world` image exists for exactly this purpose. Docker pulls it from Docker Hub (the first time only), starts a container, and the container prints a short message confirming the whole pipeline works - client, daemon, pull, and run. The container prints its message and exits immediately, which is normal.

## Commands this node introduces

- `docker version` - show client and server versions
- `docker info` - show engine state and totals
- `docker run hello-world` - pull and run the test image end to end

← Previous
