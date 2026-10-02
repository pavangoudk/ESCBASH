# What is Linux

Linux is an operating system, same category as Windows and macOS but built with a different philosophy. It's free, open source, and designed for servers first. Launch a VM on AWS, Azure, or Google Cloud and Linux is the default. Docker, Kubernetes, most CI runners, most production databases, all Linux.

That's why every DevOps job description says "comfortable in Linux." You don't need to be a kernel developer. You need to sit at a Linux machine, look around, read a config, restart a service, and get out. That's the skill.

The good news: the commands you'll use day to day number in the dozens, not thousands. You'll meet most of them in this skill, one at a time.

## Distributions

Linux isn't one thing, it's a family. Ubuntu, Debian, CentOS, Red Hat, Amazon Linux, Alpine, all called **distributions**. They mostly behave the same. This course uses Ubuntu, which is what most cloud VMs default to.

## The shell

You interact with Linux through the **shell**, a text program that reads what you type and prints the result. No mouse. On a server, the shell IS the interface.

Next up, meeting your first shell.


# Your first commands

The **shell** reads your commands and runs them. On Ubuntu it's called **bash**. When you land in a shell, it greets you with a prompt:

```
root@abc123:~#
```

Read the prompt left to right: `root` is your user, `abc123` is the machine name, `~` is your current folder (shorthand for home), and `#` is the prompt symbol (`#` for root, `$` for a regular user).

## Safe read-only commands

These print information. Nothing to break, so try each one now:

```
whoami          # which user are you?
hostname        # what's this machine called?
date            # what time does it think it is?
uname -a        # kernel and architecture details
echo hello      # print "hello" back at you
```

## Flags change what a command does

Most commands accept **flags** (also called options), single letters prefixed with `-`, that change what gets printed:

```
uname           # kernel name only (short)
uname -s        # same as above, explicit form
uname -a        # everything: kernel, host, version, architecture
```

You'll meet flags on almost every command. When you don't know which flag to use, `--help` after any command usually prints the list.

## Keyboard shortcuts

Three keystrokes worth learning right now:

- **Ctrl+C** - cancel whatever the shell is running.
- **Ctrl+L** (or type `clear`) - wipe the screen.
- **Up arrow** - bring back your previous command. Keep pressing to walk further back through history.

Nobody retypes long commands. Everyone reaches for **Up arrow**.


# pwd, ls, and cat

Three commands answer the three most common questions about a Linux machine: where am I, what's in this folder, and what's inside this file?

## pwd: print the current folder

When you log in, the shell drops you into a folder called your **home folder**. For the root user that's `/root`; for a user named `alice` it's `/home/alice`. The shell always keeps track of which folder you are standing in, called your **current folder**.

```
pwd
```

`pwd` prints the current folder as a full path. Run it now and you will see something like `/root`.

Any command you type acts on files in your current folder unless you give it a full path (starting with `/`).

## ls: list a folder's contents

`ls` lists the entries in a folder.

```
ls              # entries in the current folder
ls /            # entries at the top of the filesystem
ls /etc         # entries in /etc
```

Give `ls` no argument and you get the current folder. Give it a path and you get the contents of that folder.

## cat: print a file's contents

`cat` prints a file's contents to the screen.

```
cat /etc/hostname          # the machine's name, as stored on disk
cat /etc/os-release        # which Linux distribution and version
```

Great for short text files. Longer files have better tools, covered in a later topic.

## Tab completion

Type part of a command or path, press **Tab**, and the shell finishes it for you. Typing `who` and pressing Tab becomes `whoami`. Typing `cat /et` and pressing Tab becomes `cat /etc/`. Press Tab twice when multiple things match to see the options.

Learn this now. It saves keystrokes and prevents typos in long paths.



# mkdir and cd

Two commands you will use many times a day: `mkdir` creates a directory, `cd` moves you into the directory.

## mkdir: create a directory

```
mkdir notes                     # create a directory in the current directory
mkdir /root/projects            # create at an absolute path
```

Plain `mkdir` fails if the parent folder doesn't already exist. `mkdir -p` creates every missing parent along the way:

```
mkdir -p /root/projects/api/src
```

That creates `projects`, then `projects/api`, then `projects/api/src`, even if none of them existed before. `-p` also stays quiet if the directory is already there, which makes it safe to run twice.

## cd: change your current directory

```
cd /etc                # jump to /etc using an absolute path
cd notes               # step into the `notes` directory in your current folder
cd ..                  # move up one level to the parent directory
cd ~                   # jump to your home folder
cd                     # bare cd is the same as cd ~
```

After `cd`, `pwd` will show the new location, and any command you run from there acts on that folder unless you say otherwise. `cd` is the single most-used navigation command in Linux.

## The typical workflow

Create a folder, move into it, work there:

```
mkdir notes
cd notes
# ...do things here...
```

A bare `cd` takes you back home:

```
cd
```


# Redirecting output with >

By default, a command prints to your screen. The `>` operator sends that output into a file instead.

## The > operator

```
whoami > user.txt         # write your username into user.txt
hostname > host.txt       # write the machine name into host.txt
date > when.txt           # write the current date and time
```

If the file doesn't exist, `>` creates it. If the file already exists, `>` **overwrites** it (the previous contents are gone).

## Reading the file back

Use `cat` to see what got saved:

```
cat user.txt
```

You should see exactly what the original command printed.

## Relative names after cd

Once you `cd` into a folder, a plain filename like `user.txt` lands in that folder. Full paths always work too, regardless of where you are standing:

```
whoami > /root/answers/user.txt
```

Both styles produce the same result. The relative name is shorter when you are already in the right folder; the full path is safer when you are not sure where you are.

