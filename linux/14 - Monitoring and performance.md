# Linux System Monitoring — Uptime, Memory and Processes

```text
uptime  → Is the server under load?
free    → Does the server have enough memory?
top     → Which process is causing the problem?
```

---

# 1. `uptime`

Run:

```bash
uptime
```

Example:

```text
19:15:42 up 3 days, 4:12, 2 users, load average: 0.42, 0.55, 0.61
```

This gives you three important pieces of information.

### 1. Current time

```text
19:15:42
```

The current server time.

### 2. How long the server has been running

```text
up 3 days, 4:12
```

The server has been running for 3 days and 4 hours 12 minutes since the last reboot.

Useful when someone asks:

> "When was this server last restarted?"

### 3. Load average

```text
0.42, 0.55, 0.61
```

This is the most important part.

---

# 2. What is Load Average?

The three numbers represent:

```text
0.42 → Last 1 minute
0.55 → Last 5 minutes
0.61 → Last 15 minutes
```

Load average basically tells you how much work is waiting to be processed.

But there is one very important point:

> **Always compare load average with the number of CPU cores.**

Check CPU cores using:

```bash
nproc
```

Example:

```text
4
```

That means the server has **4 CPU cores**.

---

# 3. Understanding Load with CPU Cores

Suppose:

```bash
nproc
```

returns:

```text
4
```

You have 4 CPU cores.

### Load = 1

```text
Load: 1
CPU cores: 4
```

There is plenty of capacity.

### Load = 4

```text
Load: 4
CPU cores: 4
```

The system is roughly fully occupied.

### Load = 8

```text
Load: 8
CPU cores: 4
```

There is more work than the available CPU capacity, so processes can be waiting.

---

# 4. Same Load Can Mean Different Things

Suppose load average is:

```text
4.0
```

On an **8-core server**:

```text
4 / 8 = 50%
```

There is still CPU capacity.

But on a **2-core server**:

```text
4 / 2 = 200%
```

There is significant contention.

So don't look at:

```text
load average = 4
```

and immediately say:

> "The server is overloaded."

First check:

```bash
nproc
```

---

# 5. Reading 1, 5 and 15 Minute Load

Suppose:

```text
load average: 5.0, 2.0, 1.0
```

Think:

```text
1 minute  → 5.0
5 minutes → 2.0
15 minutes → 1.0
```

The **1-minute value is much higher**.

That suggests something has recently become busy.

Maybe:

* A deployment started
* A batch job started
* Traffic increased
* CPU-intensive processing started

---

### Opposite situation

```text
load average: 1.0, 2.0, 5.0
```

The 15-minute value was high, but the current 1-minute value is low.

That suggests:

> The system was busy earlier but is now calming down.

---

### All three high

```text
load average: 5.0, 5.2, 5.1
```

This suggests the system has been under sustained load.

Now you should investigate.

---

# 6. Important: Load Average Doesn't Tell You the Cause

Suppose:

```bash
uptime
```

shows:

```text
load average: 7.5, 7.2, 7.0
```

You know:

> The system is experiencing significant load.

But you don't yet know **why**.

Possible causes could include:

* CPU-intensive process
* Too many processes
* Disk I/O activity
* Application workload
* Background jobs

So the next tool is:

```bash
top
```

---

# 7. `/proc/loadavg`

Linux maintains the load information in:

```text
/proc/loadavg
```

You can see it with:

```bash
cat /proc/loadavg
```

Example:

```text
0.42 0.55 0.61 2/312 4567
```

The first three numbers are:

```text
0.42 → 1 minute
0.55 → 5 minutes
0.61 → 15 minutes
```

The remaining information provides process-related details.

For normal troubleshooting, you don't need to memorize everything after the first three numbers.

---

# 8. Memory with `free`

Now suppose the problem isn't CPU.

Maybe the application is running out of memory.

Use:

```bash
free -h
```

Example:

```text
              total   used   free   shared   buff/cache   available
Mem:           2.0Gi   200Mi  1.2Gi   0.0Ki     500Mi        1.7Gi
Swap:             0B     0B     0B
```

---

# 9. Understanding `free`

The output has several columns.

The important ones are:

```text
total
used
free
buff/cache
available
```

But the most important number to focus on is:

> **available**

---

# 10. `used` vs `free` vs `available`

### `used`

Memory currently being used by processes and the system.

### `free`

Memory that is completely unused.

### `available`

Memory that Linux can make available to applications when required.

This is the important one.

---

# 11. Why Can `free` Be Low?

This confuses many beginners.

Suppose:

```text
total      8 GB
free       500 MB
available  5 GB
```

You might think:

> "Only 500 MB is free! The server is almost out of memory."

That's incorrect.

Linux intentionally uses unused RAM for things like **filesystem cache**.

So the operating system may have:

```text
Free memory → 500 MB
Cache       → 4.5 GB
Available   → 5 GB
```

If an application needs more memory, Linux can release some cache.

Therefore:

> **Don't panic just because `free` is low. Look at `available`.**

---

# 12. What is buff/cache?

Linux uses unused RAM to cache recently accessed data.

For example:

```text
Application
     ↓
Reads file
     ↓
Linux keeps frequently used data in RAM
     ↓
Future access can be faster
```

This improves performance.

So:

> **RAM being used for cache isn't necessarily a problem.**

Linux can reclaim that memory when applications need it.

---

# 13. What is Swap?

You may see:

```text
Swap:
```

Swap is disk space that can be used as memory when RAM becomes constrained.

Think:

```text
RAM
 ↓
Fast
 ↓
Limited

Swap
 ↓
Disk
 ↓
Much slower
```

Some cloud VMs have no swap configured:

```text
Swap: 0B
```

That isn't automatically a problem.

But if a server is heavily using swap, you should investigate memory pressure and the system configuration.

---

# 14. `top`

Now imagine:

```text
uptime
```

tells you:

> The server is under heavy load.

And:

```bash
free -h
```

shows:

> Memory looks okay.

Now you want to know:

> **Which process is causing the load?**

Use:

```bash
top
```

`top` gives you a live view of the system.

It contains two major areas:

```text
Summary
   ↓
CPU / Memory / Load / Tasks

Processes
   ↓
Individual running processes
```

---

# 15. Understanding `top`

The process section contains information such as:

```text
PID
USER
%CPU
%MEM
COMMAND
```

The most useful ones initially are:

### PID

Process ID.

Every running process gets a PID.

### `%CPU`

How much CPU the process is consuming.

### `%MEM`

How much memory the process is consuming.

### COMMAND

The process/application name.

---

# 16. Important `top` Keys

While inside `top`:

### `P`

Sort by CPU usage.

```text
P → CPU
```

### `M`

Sort by memory usage.

```text
M → Memory
```

### `k`

Kill a process.

It asks you for the PID.

Be careful with this on production servers.

### `1`

Shows individual CPU information instead of just an aggregated CPU view.

### `h`

Shows help.

### `q`

Quit `top`.

---

# 17. `htop`

`htop` is an easier-to-use alternative to `top`.

Install it:

```bash
sudo apt install -y htop
```

Then:

```bash
htop
```

It provides a more interactive display.

You can:

* Scroll
* Sort processes
* See CPU usage
* See memory usage
* Select processes
* Kill processes

For example, `F9` can be used to kill a selected process.

### Important

`htop` may not be installed by default.

`top` is more universally available, so you should know **both**.

---

# 18. Running `top` in Scripts

Normally:

```bash
top
```

opens an interactive screen.

That's not useful inside a shell script.

Instead:

```bash
top -b -n1
```

### `-b`

Batch mode.

It prints normal text instead of the interactive screen.

### `-n1`

Run only one iteration.

So:

```bash
top -b -n1
```

means:

> Give me one snapshot of the current system state and exit.

You can combine it with:

```bash
top -b -n1 | head -20
```

This keeps only the first 20 lines.

Useful for:

* Health-check scripts
* Troubleshooting scripts
* Automation

---

# 19. `vmstat`

Another useful command is:

```bash
vmstat 1
```

It prints system statistics repeatedly.

The `1` means:

> Give me a new sample every 1 second.

It provides information about things like:

* Memory
* Processes
* CPU
* System activity

You don't need to master every column immediately.

For now, remember:

```text
vmstat → overall system activity over time
```

---

# 20. `iostat`

`iostat` is mainly useful for looking at **disk I/O**.

For example:

```text
Is the application slow because the disk is busy?
```

`iostat` can help answer that.

It may not be installed by default.

Install it with:

```bash
sudo apt install -y sysstat
```

Then:

```bash
iostat
```

Remember:

```text
top     → processes / CPU / memory
free    → memory
vmstat  → system activity
iostat  → disk I/O
```

---

# 21. Real-Time DevOps Troubleshooting Scenario

Imagine you receive an alert:

> "Production server is slow."

Don't randomly restart things.

Start with:

### Step 1 — Check uptime/load

```bash
uptime
```

Suppose:

```text
load average: 6.5, 6.2, 5.8
```

Now check CPU cores:

```bash
nproc
```

Suppose:

```text
4
```

Load is significantly above the core count.

So you investigate.

---

### Step 2 — Check memory

```bash
free -h
```

Suppose:

```text
available: 6G
```

Memory looks reasonably available.

So memory may not be the immediate problem.

---

### Step 3 — Find the process

```bash
top
```

Sort by CPU using:

```text
P
```

You discover:

```text
%CPU
95%
```

for an application process.

Now you know:

```text
Server slow
   ↓
High load
   ↓
CPU is likely involved
   ↓
top
   ↓
Application process consuming CPU
```

Now you investigate **why that application is consuming CPU**, instead of blindly rebooting the server.

That's the real DevOps mindset.

---

# ⭐ Final Mental Model

When troubleshooting a Linux server, remember:

```text
          SERVER PROBLEM
                ↓
          ┌─────┴─────┐
          ↓           ↓
        CPU          Memory
          ↓           ↓
       uptime       free -h
          ↓
         top
          ↓
   Find problematic process
```

And for the commands:

| Command             | Main purpose                          |
| ------------------- | ------------------------------------- |
| `uptime`            | Uptime + load average                 |
| `nproc`             | Number of CPU cores                   |
| `cat /proc/loadavg` | Raw load-average information          |
| `free -h`           | Memory usage                          |
| `top`               | Live process/system monitoring        |
| `htop`              | Easier interactive process monitoring |
| `vmstat 1`          | System activity over time             |
| `iostat`            | Disk I/O statistics                   |

## ⭐ The 5 commands I'd memorize first

```bash
uptime
```

**Is the server under load?**

```bash
nproc
```

**How many CPU cores do I have?**

```bash
free -h
```

**Do I have enough memory?**

```bash
top
```

**Which process is causing the problem?**

```bash
df -h
```

**Do I have enough disk space?**

Together, these commands give you a very solid **first-level Linux server troubleshooting toolkit**.
