# Linux — Notes

## What is Linux

Linux is an operating system, just like Windows and macOS, but it is built around a different philosophy.

It is:

- Free
- Open source
- Designed primarily for servers

Launch a virtual machine on AWS, Azure, or Google Cloud, and Linux is usually the default. Docker, Kubernetes, most CI runners, and many production databases also run on Linux.

That is why Linux appears in almost every DevOps job description.

You do not need to become a kernel developer. You need to be able to sit at a Linux machine, look around, read a configuration file, restart a service, and move on.

The good news is that you only need a few dozen commands for most day-to-day work—not thousands.

## Distributions

Linux is not one single operating system. It is a family of operating systems called **distributions**, or **distros**.

Common distributions include:

- Ubuntu
- Debian
- CentOS
- Red Hat
- Amazon Linux
- Alpine

They mostly work the same way. This course uses Ubuntu because it is a common default for cloud virtual machines.

## The Shell

You interact with Linux through the **shell**.

The shell is a text-based program that:

1. Reads what you type
2. Runs the command
3. Prints the result

There is no mouse involved. On a server, the shell is the interface.

On Ubuntu, the shell you will commonly use is called **Bash**.

## Your First Commands

When you enter a shell, it greets you with a prompt that may look like this:

`root@abc123:~#`

Read it from left to right:

- `root` is the user
- `abc123` is the machine name
- `~` is the current folder, usually the user’s home folder
- `#` is the prompt symbol for the root user

A regular user normally sees `$` instead of `#`.

### Safe, read-only commands

These commands print information. They do not normally change anything, so they are safe to try:

- `whoami` — shows which user you are
- `hostname` — shows the machine name
- `date` — shows the current date and time
- `uname -a` — shows kernel and architecture details
- `echo hello` — prints `hello`

These are useful when you first connect to a machine and need to understand where you are.

## Flags Change What a Command Does

Most commands accept flags, also called options.

Flags usually begin with a hyphen and change what the command displays or how it behaves.

For example:

- `uname` — prints the kernel name
- `uname -s` — explicitly asks for the kernel name
- `uname -a` — prints everything: kernel, host, version, and architecture

You will use flags with almost every command.

When you do not know which flags are available, add `--help` after the command. It usually prints the command’s usage information.

## Keyboard Shortcuts

There are three keyboard shortcuts worth learning immediately.

### `Ctrl+C`

Cancels whatever the shell is currently running.

### `Ctrl+L`

Clears the screen.

You can also type `clear`.

### Up Arrow

Brings back your previous command.

Keep pressing Up to move further back through your command history.

Nobody retypes long commands if they can avoid it. Everyone uses the Up arrow.

## `pwd`, `ls`, and `cat`

These three commands answer three common questions:

1. Where am I?
2. What is in this folder?
3. What is inside this file?

## `pwd`: Print the Current Folder

`pwd` means **print working directory**.

When you log in, the shell places you in a folder called your home folder:

- For the root user: `/root`
- For a user named Alice: `/home/alice`

Run:

`pwd`

It prints the full path of the folder you are currently standing in.

Any command you run acts on files in your current folder unless you provide a full path.

## `ls`: List a Folder’s Contents

`ls` lists the files and folders in a directory.

Examples:

- `ls` — lists the current folder
- `ls /` — lists the top level of the filesystem
- `ls /etc` — lists the contents of `/etc`

Give `ls` no argument and it lists the current folder. Give it a path and it lists that folder instead.

## `cat`: Print a File’s Contents

`cat` prints the contents of a file to the screen.

Examples:

- `cat /etc/hostname` — shows the machine name stored on disk
- `cat /etc/os-release` — shows the Linux distribution and version

`cat` is useful for short text files. Longer files need better tools, which you will meet later.

## Tab Completion

Type part of a command or path and press Tab. The shell will try to finish it for you.

For example:

- Type `who` and press Tab. It may become `whoami`.
- Type `cat /et` and press Tab. It may become `cat /etc/`.

If more than one option matches, press Tab twice to see the available choices.

Learn this early. Tab completion saves keystrokes and prevents typos in long paths.

## `mkdir` and `cd`

Two commands you will use many times a day are:

- `mkdir` — creates a directory
- `cd` — moves into a directory

## `mkdir`: Create a Directory

Use `mkdir` to create a new directory.

Examples:

- `mkdir notes` — creates a directory called `notes` in the current folder
- `mkdir /root/projects` — creates a directory at an absolute path

A normal `mkdir` command fails if the parent folder does not already exist.

Use `mkdir -p` to create every missing parent directory along the way.

For example:

`mkdir -p /root/projects/api/src`

This creates:

1. `/root/projects`
2. `/root/projects/api`
3. `/root/projects/api/src`

The `-p` flag also stays quiet if the directory already exists, so it is safe to run the command twice.

## `cd`: Change Your Current Directory

Use `cd` to move between directories.

Examples:

- `cd /etc` — moves to `/etc`
- `cd notes` — moves into the `notes` folder in the current directory
- `cd ..` — moves up one level
- `cd ~` — moves to your home folder
- `cd` — also moves to your home folder

After using `cd`, run `pwd` to confirm your new location.

From that point on, commands act on the new current folder unless you provide another path.

`cd` is the single most-used navigation command in Linux.

## The Typical Workflow

A common Linux workflow looks like this:

1. Create a folder.
2. Move into it.
3. Work there.
4. Return home when finished.

For example:

`mkdir notes`

`cd notes`

Do your work inside the folder.

To return home, run:

`cd`

## Redirecting Output with `>`

By default, a command prints its output to the screen.

The `>` operator sends that output into a file instead.

Examples:

- `whoami > user.txt`
- `hostname > host.txt`
- `date > when.txt`

If the file does not exist, Linux creates it.

If the file already exists, `>` overwrites it. The previous contents are gone.

That makes `>` useful, but something to use carefully.

## Reading the File Back

Use `cat` to see what was saved:

`cat user.txt`

You should see exactly what the original command printed.

For example, if you ran:

`whoami > user.txt`

then `cat user.txt` displays the username.

## Relative Names After `cd`

Once you move into a folder, a plain filename is created in that folder.

For example, if you are in `/root/answers` and run:

`whoami > user.txt`

the file is created at:

`/root/answers/user.txt`

You can also use the full path:

`whoami > /root/answers/user.txt`

Both styles work.

- A relative filename is shorter when you are already in the right folder.
- A full path is safer when you are not sure where you are.

## The Main Idea

Linux work begins with a few basic questions:

- Who am I?
- What machine am I on?
- Where am I?
- What is in this folder?
- What is inside this file?
- Where should I create the next file or directory?

The commands in this lesson answer those questions:

- `whoami`
- `hostname`
- `pwd`
- `ls`
- `cat`
- `mkdir`
- `cd`
- `>`

Learn these commands well. They are the foundation for everything that comes next.

##### **Follow-Ups:**

- Turn these notes into a quick revision sheet
- Create practice exercises for each Linux command
