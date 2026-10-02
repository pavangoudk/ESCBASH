# What is a process

LessonA process is a running program. Every command you type spawns one. Every service on the machine is one. Some are short (`ls` starts, prints, exits, gone), some run for weeks (nginx, sshd, a database).

## Listing processes

```
ps aux
```

Prints every process on the machine. A couple of rows look like this:

```
USER   PID %CPU %MEM    VSZ   RSS TTY   STAT START   TIME COMMAND
root     1  0.0  0.4 168944 12016 ?     Ss   09:12   0:01 /sbin/init
root   533  0.0  0.2  55208  5400 ?     Ss   09:12   0:00 nginx: master process
```

The columns that matter to a DevOps engineer:

- `USER` - who owns the process
- `PID` - process ID, unique per process
- `%CPU` and `%MEM` - resources it's using
- `COMMAND` - the command line that launched it

Read the second row left to right: the user `root` owns it, its PID is `1`, it's barely touching CPU or memory, and it was launched from `/sbin/init`. That process with PID `1` is special, it's the first thing the kernel starts and the ancestor of everything else.

## Everything is referred to by PID

Every process has a PID, and that number is how you refer to it later. Start a throwaway process so you have a real PID to work with:

```
sleep 300 &
```

The shell prints something like `[1] 4821`. That second number is the PID. Now you can stop that exact process by its number:

```
kill 4821
```

Use whatever number your shell actually printed, not `4821`. Your own shell has a PID too. Run `echo $$` to see it.

## Parent processes

Every process was started by another process, called its **parent**. `ps -ef` shows a `PPID` (parent PID) column:

```
ps -ef | head
```

When you run a command in ,  is the parent, your command is the child. This matters when you're chasing which service started which helper, or when a process seems to keep coming back (its parent is respawning it).



# 
Finding and stopping processes

Lesson## Finding by name

Two patterns cover most cases.

The everyday move:

```
ps aux | grep nginx
```

You'll see one line per matching process plus the grep itself (because your grep also has "nginx" in it). Add `| grep -v grep` to drop the noise:

```
ps aux | grep nginx | grep -v grep
```

Cleaner alternative:

```
pgrep nginx              # just PIDs
pgrep -a nginx           # PIDs plus command lines
```

`pgrep` prints only the PIDs, which pairs nicely with commands that take PIDs as input.

`pidof` does nearly the same job, matching on the exact program name rather than a pattern:

```
pidof nginx
```

It prints the matching PIDs on a single line. Reach for `pgrep` when you want pattern matching, `pidof` when you know the exact name.

## The live view: top

`ps` is a snapshot. `top` updates continuously, sorted by CPU by default:

```
top
```

Useful keys inside `top`: `q` to quit, `M` to sort by memory, `P` to sort by CPU. `htop` is a nicer, colored version, worth installing on any machine you spend real time on.

## Stopping a process

`kill` sends a **signal** to a process. Different signals mean different things.

```
kill 1234                # SIGTERM: "please stop"
kill -9 1234             # SIGKILL: "stop now, no cleanup"
```

## Signals worth knowing

| Signal | Number | What it means |
| --- | --- | --- |
| SIGTERM | 15 | Please shut down cleanly. This is the default. |
| SIGKILL | 9 | Force immediate stop. No cleanup possible. |
| SIGHUP | 1 | Reload config (many daemons treat it that way). |
| SIGINT | 2 | Interrupt. This is what Ctrl+C sends. |

Rule of thumb: try plain `kill PID` first (SIGTERM), wait a couple of seconds, only reach for `kill -9` if the process is truly stuck. A SIGTERM lets the process close its files and flush its buffers before exiting. SIGKILL gives it no chance, which can leave half-written files behind.

## Killing by name

Looking up a PID just to kill it gets tedious. `pkill` and `killall` match by name and signal every process that matches, in one step. Start a couple of throwaway processes so you can watch it work:

```
sleep 300 &
sleep 300 &
```

`pkill` matches on a pattern, the same way `pgrep` does:

```
pkill sleep
```

`killall` matches the exact command name instead:

```
killall sleep
```

Both take the same signal flags as `kill`, so `pkill -9 sleep` force-kills every match. The convenience has a downside. `pkill sleep` stops every sleep on the machine, including ones another user or a script started, so when you already know the exact PID, prefer that. Reach for `pkill` and `killall` when you genuinely want every process of a given name gone.

## Inspecting a single process with ps -p

`ps aux` prints every process. If you already have a PID and want details about only that one, use `ps -p`:

```
ps -p 1234                    # show only PID 1234
```

Combine it with `pgrep` to look up a name and inspect the match in one line, using the command-substitution pattern below.

## Command substitution with $(...)

`$(command)` runs `command` in a subshell and substitutes its output into the surrounding line before the shell runs it. This is how you chain commands that don't accept pipes as arguments (`kill` and `ps -p` both take a PID as an argument, not on stdin).

```
kill $(pgrep myapp)              # kill the PID that pgrep printed
ps -p $(cat /var/run/app.pid)    # inspect the PID stored in a file
```

You'll see `$(...)` everywhere. Anywhere a command needs the OUTPUT of another command as an ARGUMENT, this is the pattern.



# 
Background and persistent jobs

LessonSometimes you want a command to keep running while you get your shell back. Two mechanisms cover almost every case.

## Ampersand: run in the background

Append `&` to any command:

```
sleep 300 &
```

The command starts, and the shell immediately hands you the prompt back. You'll see something like `[1] 12345`, which is the job number and the PID.

### A never-ending sleep

`sleep` also accepts the word `infinity` in place of a number:

```
sleep infinity
```

That call never returns on its own, so press Ctrl+C to stop it and get your prompt back. It's the standard way to keep a script or a container running until something else stops it. You'll see it inside minimal Docker images and in placeholder systemd services.

See your background jobs:

```
jobs
```

Bring one back to the foreground:

```
fg %1              # by job number
fg                 # most recent one
```

Pause a foreground command and push it to the background:

```
# while the command is running:
Ctrl+Z             # pauses it
bg                 # resumes it in the background
```

## Background jobs die with your shell

A background job is still a child of your shell. Close the SSH session and the shell exits, taking its children with it. That's fine for a `sleep` while you work, disastrous for a two-hour backup.

## nohup: survive logout

`nohup` (short for "no hangup") lets a command outlive the shell:

```
nohup sleep 300 &        # stand-in for a long-running task
```

Two things happen. The process is detached from your shell, and output that would have gone to the terminal is redirected to a file called `nohup.out` in the current directory. You can log out and the task keeps going.

Find it again later:

```
ps aux | grep sleep
```

For anything more sophisticated (auto-restart, log rotation, startup ordering), you'll want `systemd`, which gets its own topic later.



# Background and persistent jobs

LessonSometimes you want a command to keep running while you get your shell back. Two mechanisms cover almost every case.

## Ampersand: run in the background

Append `&` to any command:

```
sleep 300 &
```

The command starts, and the shell immediately hands you the prompt back. You'll see something like `[1] 12345`, which is the job number and the PID.

### A never-ending sleep

`sleep` also accepts the word `infinity` in place of a number:

```
sleep infinity
```

That call never returns on its own, so press Ctrl+C to stop it and get your prompt back. It's the standard way to keep a script or a container running until something else stops it. You'll see it inside minimal Docker images and in placeholder systemd services.

See your background jobs:

```
jobs
```

Bring one back to the foreground:

```
fg %1              # by job number
fg                 # most recent one
```

Pause a foreground command and push it to the background:

```
# while the command is running:
Ctrl+Z             # pauses it
bg                 # resumes it in the background
```

## Background jobs die with your shell

A background job is still a child of your shell. Close the SSH session and the shell exits, taking its children with it. That's fine for a `sleep` while you work, disastrous for a two-hour backup.

## nohup: survive logout

`nohup` (short for "no hangup") lets a command outlive the shell:

```
nohup sleep 300 &        # stand-in for a long-running task
```

Two things happen. The process is detached from your shell, and output that would have gone to the terminal is redirected to a file called `nohup.out` in the current directory. You can log out and the task keeps going.

Find it again later:

```
ps aux | grep sleep
```

For anything more sophisticated (auto-restart, log rotation, startup ordering), you'll want `systemd`, which gets its own topic later.

