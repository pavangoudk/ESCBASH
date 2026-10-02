
# Environment variables

Every shell session carries a set of **environment variables**, name-value pairs the shell (and any program it runs) can read. They carry configuration that changes per user, per machine, or per session, without touching any source code.

## Seeing them

```
env                # every exported variable in this session
printenv           # same, slightly different format
echo $HOME         # print one specific variable
echo $USER
echo $PATH
```

`$VAR` is how the shell expands a variable's value.

## Ones you'll meet daily

- `HOME` - your home folder path
- `USER` - your username
- `PWD` - the folder you're standing in
- `SHELL` - which shell is running
- `PATH` - where the shell looks for command binaries (its own topic in the next node)
- `LANG` - locale, controls sorting and message language

## Setting a variable

```
name=alice           # sets a shell variable, NOT exported
echo $name           # alice
```

That variable exists in your current shell, but any command the shell starts doesn't see it. You can prove this. ` -c '...'` runs a fresh child shell:

```
name=alice
 -c 'echo $name'
```

That prints a blank line. The child shell never received `name`. Now **export** it and run the same thing again:

```
export name=alice
 -c 'echo $name'
```

This time it prints:

```
alice
```

Exporting copies the variable into the environment, so every program you start from this shell can read it.

## The inline form

You can set a variable for a single command by writing it right in front of that command:

```
LOG_LEVEL=debug  -c 'echo $LOG_LEVEL'
```

That prints `debug`. The variable is exported to that one command only, then discarded the moment it finishes. You'll see this same pattern in front of real scripts, like `LOG_LEVEL=debug ./deploy.sh`, where a program reads its configuration from the environment.

## Removing a variable

```
unset name
```

Gone from the current session.



# PATH and command lookup

The most important environment variable you'll ever configure is `PATH`. It tells the shell where to look for command binaries.

## What's in PATH

```
echo $PATH
```

You'll see something like:

```
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

A colon-separated list of folders. When you type `ls`, the shell walks that list left to right and runs the first executable named `ls` it finds. To see which one that is:

```
type ls              # /usr/bin/ls
which ls             # /usr/bin/ls (older command, same idea)
command -v ls        # /usr/bin/ls (works inside scripts too)
```

All three answer the same question: which file runs when you type this name. `command -v` is the one to reach for in scripts, since it's built into every POSIX shell and doesn't depend on `which` being installed.

## Order matters

The first match wins. If both `/usr/local/bin/python` and `/usr/bin/python` exist, whichever comes first in PATH is what runs. This is why you'll see people prepend custom or newer tool folders to PATH:

```
export PATH="$HOME/bin:$PATH"
```

That puts `~/bin` at the front. Anything there overrides system defaults with the same name.

## When "command not found" hits

The classic frustration: you installed a tool, but the shell can't find it.

```
$ myapp
: myapp: command not found
```

Almost always one of two causes:

1. The tool was installed somewhere not in PATH.
2. The tool is in a folder that requires opening a new shell to pick up.

Find where it actually landed:

```
find / -name myapp 2>/dev/null
```

If the result is `/opt/myapp/bin/myapp`, that folder isn't in PATH. Add it (next node covers how to do that permanently).

## Absolute paths always work

If PATH is broken or a binary isn't in it, you can always call it with its full path:

```
/opt/myapp/bin/myapp
```

Absolute paths never depend on PATH. Handy fallback when troubleshooting.



# Startup files and aliases

Every time you log in or open a new shell,  reads a set of files that configure your environment. Editing them is how you make settings stick across sessions.

## The files that matter

- **`~/.rc`** - runs for interactive non-login shells (every terminal tab you open on an already-logged-in machine).
- **`~/.profile`** or **`~/._profile`** - runs at login (SSH into a fresh session).
- **`/etc/.rc`** - system-wide, runs for every user's interactive shells.
- **`/etc/profile`** - system-wide, runs at login for every user.

Rule of thumb on modern Ubuntu: **put your customizations in `~/.rc`**. `~/.profile` sources `.rc` for you, so any session picks up the changes.

## Making changes take effect

After editing `~/.rc`, you have two ways to apply the changes:

1. Open a new terminal or SSH session. The new shell reads `.rc` fresh.
2. Reload the file in your current session with `source`:

```
source ~/.rc
# or the one-character shorthand
. ~/.rc
```

`source` (or `.`) runs the file **in the current shell**. Running it directly with ` ~/.rc` would start a child shell, apply the changes there, and then exit, leaving your current session unchanged.

## Aliases: short names for common commands

An **alias** is a shortcut for a longer command. Define them once in `.rc` and they're always there:

```
alias ll='ls -lah'
alias ..='cd ..'
alias grep='grep --color=auto'
alias gs='git status'
```

List active aliases with `alias` alone. Remove one with `unalias ll`.

## Common additions to .rc

Drop these into `~/.rc` and reload:

```
# Prepend your own bin folder to PATH
export PATH="$HOME/bin:$PATH"

# Handy aliases
alias ll='ls -lah'
alias df='df -h'
alias free='free -h'

# Set a default editor for tools that read $EDITOR
export EDITOR=nano
```

Now `ll` gives a full listing, `df` is human-readable by default, and any tool that reads `$EDITOR` (git commits, `crontab -e`, `visudo`) opens nano.

