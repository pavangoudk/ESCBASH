# Linux Process Management Notes

## 1. What is a process?

A **process** is a running program.

Examples:

- `ls` runs briefly, prints output, and exits.
- `nginx`, `sshd`, and databases may run continuously.
- Every command you execute creates a process.

Each process has:

- An owner
- A unique process ID, or **PID**
- CPU and memory usage
- A command that started it
- A parent process

## 2. Listing processes

Use `ps aux` to display processes from across the system.

Important columns:

| Column | Meaning |
| --- | --- |
| `USER` | User who owns the process |
| `PID` | Unique process ID |
| `%CPU` | CPU usage |
| `%MEM` | Memory usage |
| `COMMAND` | Command that started the process |

PID `1` is special. It is the first userspace process started by the kernel and is the ancestor of most other processes.

```
ps aux
ps -ef
ps -ef | head
echo $$
```

`echo $$` displays the PID of the current shell.

## 3. PIDs and parent processes

Start a temporary process in the background:

```
sleep 300 &
```

The shell displays a job number and PID. Use the PID to target that specific process:

```
kill <PID>
```

Every process has a parent process. `ps -ef` includes a `PPID` column showing the parent PID.

Understanding parent-child relationships helps identify:

- Which service started a process
- Why a process keeps returning
- Which supervisor may be restarting a process

## 4. Finding processes

### Search with `ps` and `grep`

```
ps aux | grep nginx
ps aux | grep nginx | grep -v grep
```

The first command may show the `grep` command itself because it contains the search term.

### Use `pgrep`

```
pgrep nginx
pgrep -a nginx
```

- `pgrep nginx` displays matching PIDs.
- `pgrep -a nginx` displays PIDs and command lines.

### Use `pidof`

```
pidof nginx
```

`pidof` searches for the exact program name, while `pgrep` supports pattern matching.

## 5. Monitoring processes with `top`

`ps` gives a snapshot. `top` provides a continuously updating view.

```
top
```

Useful keys inside `top`:

- `q` — quit
- `M` — sort by memory usage
- `P` — sort by CPU usage

`htop` is a more user-friendly alternative when installed.

## 6. Stopping processes with signals

The `kill` command sends a signal to a process.

```
kill <PID>
kill -9 <PID>
```

### Important signals

| Signal | Number | Meaning |
| --- | --- | --- |
| `SIGTERM` | 15 | Request a clean shutdown |
| `SIGKILL` | 9 | Force immediate termination |
| `SIGHUP` | 1 | Often reloads configuration |
| `SIGINT` | 2 | Interrupt; sent by `Ctrl+C` |

### Recommended approach

1. Try `kill <PID>` first.
2. Wait briefly for graceful shutdown.
3. Use `kill -9 <PID>` only if the process is stuck.

`SIGTERM` gives the process time to close files and flush data. `SIGKILL` provides no cleanup opportunity.

## 7. Killing processes by name

```
pkill sleep
killall sleep
```

- `pkill` matches a pattern.
- `killall` matches the exact command name.

These commands may stop every matching process, including processes started by other users or scripts. Prefer an exact PID when possible.

## 8. Inspecting one process

Use `ps -p` when you already know the PID:

```
ps -p <PID>
ps -p "$(pgrep myapp)"
```

Command substitution uses `$(...)` to place one command’s output inside another command.

Examples:

```
kill "$(pgrep myapp)"
ps -p "$(cat /var/run/app.pid)"
```

Check the resulting PID before issuing destructive commands, especially when the search could return multiple processes.

## 9. Background jobs

Add `&` to run a command in the background:

```
sleep 300 &
jobs
fg %1
```

Key concepts:

- `&` starts a background job.
- `jobs` lists jobs belonging to the current shell.
- `fg %1` brings job 1 to the foreground.
- `fg` brings the most recent job forward.
- `Ctrl+Z` pauses a foreground process.
- `bg` resumes a paused process in the background.

A background job is still attached to the shell and may end when the shell exits.

## 10. Keeping jobs alive after logout

Use `nohup` for a basic task that should survive logout:

```
nohup sleep 300 &
```

`nohup`:

- Detaches the process from the shell
- Allows it to continue after logout
- Redirects terminal output to `nohup.out` by default

For automatic restarts, logging, dependency management, and startup ordering, use a service manager such as `systemd`.

## Quick reference

| Goal | Command |
| --- | --- |
| List all processes | `ps aux` |
| Show process hierarchy details | `ps -ef` |
| Find PIDs by pattern | `pgrep name` |
| Find exact program PIDs | `pidof name` |
| Monitor continuously | `top` |
| Gracefully stop a process | `kill PID` |
| Force-stop a process | `kill -9 PID` |
| Inspect one PID | `ps -p PID` |
| List shell jobs | `jobs` |
| Resume a job in foreground | `fg %job_number` |
| Resume a paused job in background | `bg` |
| Survive logout | `nohup command &` |

**Core mental model:**

- `PID` identifies a process.
- `ps` shows process details.
- `kill` sends signals.
- `&` runs a job in the background.
- `nohup` helps a job survive logout.

##### **Follow-Ups:**

- Give me a process-management quiz
- Explain systemd service management next
