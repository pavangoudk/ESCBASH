# What package management is

A **package** is a compiled program plus everything it needs (its files, dependencies, config, man page) bundled into one archive that a machine can install in one step. A **package manager** is the tool that installs, updates, and removes those packages, and keeps track of dependencies so you don't have to.

Without a package manager, installing nginx would mean downloading source code, its dependencies, compiling everything, copying files to the right places, and remembering to do it all again for the next security patch. With a package manager, it's one command.

## The two big families

Linux distros split into two main package management families:

- **Debian / Ubuntu / Mint** use `.deb` packages, managed by `apt` (or the lower-level `dpkg`).
- **RHEL / Fedora / Amazon Linux / CentOS** use `.rpm` packages, managed by `dnf` (or older `yum`).

Commands differ, concepts don't. If you know apt, you can learn dnf in an hour. This topic covers apt because Ubuntu is what most cloud VMs default to.

## Where packages come from

apt reads a list of **repositories** from `/etc/apt/sources.list` and files in `/etc/apt/sources.list.d/`. A repository is just a URL where packages live. Every `apt install` downloads from one of those URLs.

Ubuntu ships with the official Ubuntu repos already configured. Because a package like nginx sits in those default repos, `apt install` pulls it and every dependency in one command, with no manual downloads or extra setup.



# apt basics

Five commands cover almost everything you'll do with apt.

## apt update: refresh the package list

```
sudo apt update
```

This installs nothing. It contacts your configured repositories and downloads a fresh copy of their package index, the list of what's available and at which version. Your machine keeps a local copy of that index, and it drifts out of date as the repos publish newer versions.

Run `apt update` before any install on a box you haven't touched in a while. If the local index is stale, `apt install` can try to fetch a version that has already been replaced and fail with a 404. Downloading the index needs network access, so on a machine with no internet this step can fail on its own.

## apt install: add a package

```
sudo apt install htop
```

apt downloads htop and every package it depends on, then installs them. Because it pulls from the repos, installing a package you don't already have needs network access.

```
sudo apt install -y htop            # skip the "Y/n" confirmation
sudo apt install -y htop curl vim   # install several at once
```

The `-y` flag is critical in scripts. Without it, apt stops to ask for confirmation and waits for a keypress that a script has no way to provide.

## Upgrading

```
sudo apt upgrade                    # upgrade every installed package
sudo apt upgrade htop               # only htop
```

`apt upgrade` applies available updates. On a production box you'd usually schedule this, not run it ad-hoc.

## Removing

Three levels of removal:

```
sudo apt remove htop                # remove the package, keep its config files
sudo apt purge htop                 # remove package AND its config files
sudo apt autoremove                 # clean up unused dependencies left behind
```

`purge` is the one you want when you're truly done with something. `remove` leaves the config in place so you can reinstall with the same settings later.

## The update-then-install pattern

On any new server, you'll type this within the first five minutes:

```
sudo apt update
sudo apt install -y htop
```

Run the two lines in order: refresh the list first, then install against the fresh list. In scripts and docs you'll often see them joined on one line with `&&`, but two separate lines do the same job and are easier to read while you're learning.



# Searching and inspecting packages

Sometimes you know roughly what you want but not the exact package name. Sometimes a file shows up and you need to know which package put it there. apt and dpkg answer both.

One split to keep in mind: `apt search` and `apt show` look at the package list from the repos (what's available to install), while the `dpkg` commands look at what's actually installed on this machine right now.

## apt search: find a package by keyword

```
apt search nginx
```

Prints every package whose name or description mentions "nginx", each with a one-line summary. Handy when you don't know the exact name. It reads the same local package list that `apt update` refreshes, so if that list is empty or stale the results will be too.

## apt show: details on one package

```
apt show nginx
```

You get the description, version, dependencies, download size, and installed size. The two lines worth finding are `Version:` (what apt would install) and `Depends:` (what else comes along with it).

## dpkg -l: what's installed here

`dpkg` is the low-level tool apt sits on top of. `dpkg -l` on its own lists every installed package, one per line, which is a lot. Pipe it through `grep` to narrow it down:

```
dpkg -l | grep nginx
```

You'll see something like:

```
ii  nginx  1.24.0-...  amd64  small, powerful, scalable web server
```

The `ii` at the start means the package is installed and configured.

## dpkg -L: which files a package installed

```
dpkg -L nginx
```

Lists every file that package dropped on disk: binaries, config files, man pages. The output looks something like:

```
/usr/sbin/nginx
/etc/nginx/nginx.conf
/usr/share/nginx
```

It answers "where does this program's config live?" without guessing.

## dpkg -S: which package owns a file

```
dpkg -S /usr/sbin/nginx
```

The reverse lookup: give it a file path, it names the package that installed it. The output is the package name, a colon, then the path, something like:

```
nginx: /usr/sbin/nginx
```

Useful when a binary shows up and you want its origin, or when you're tracking down which package a config file belongs to.

