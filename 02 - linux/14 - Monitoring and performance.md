# Uptime and load average

```
uptime
```

Prints one line:

```
 19:15:42 up 3 days,  4:12,  2 users,  load average: 0.42, 0.55, 0.61
```

Three pieces:

- **How long the machine has been up.** Handy when someone asks "when was this last rebooted?"
- **Users currently logged in.** Anyone besides you.
- **Load average.** The three numbers on the right, and the useful part.

## What load average means

The three numbers are the load average over the last **1 minute**, **5 minutes**, and **15 minutes**. Each number is roughly the average count of processes wanting the CPU during that window.

The trick is comparing to the number of CPU cores:

- Load **below** the core count means the machine has capacity to spare.
- Load **around** the core count means fully utilized.
- Load **above** the core count means processes are queuing for CPU.

A load of 4.0 on an 8-core box is 50% utilization, healthy. The same load on a 2-core box means it's overloaded. Always compare load against `nproc`:

```
nproc
```

Prints the core count in one number.

## Reading the three numbers together

- 1-min rising, 15-min low: something just started spiking. May resolve on its own.
- 1-min low, 15-min high: things were bad, calming down now.
- All three high: sustained overload, time to investigate.

Load average tells you the machine is busy. It doesn't tell you what's making it busy. That's what `top` is for, coming up next.

## The raw source: /proc/loadavg

The three numbers `uptime` prints come from a file the kernel keeps updated in real time:

```
cat /proc/loadavg
```

You'll see something like `0.42 0.55 0.61 2/312 4567` - the three load averages, then the count of runnable/total processes, then the last PID created. Scripts that need just the numbers (with no parsing) read `/proc/loadavg` directly instead of picking values out of the `uptime` line.



# Memory with free

```
free -h
```

`-h` prints human-readable units.

```
              total        used        free      shared  buff/cache   available
Mem:           2.0Gi       200Mi       1.2Gi       0.0Ki       500Mi       1.7Gi
Swap:             0B          0B          0B
```

Six columns, but only two really matter to you.

## used vs free vs available

- **used** - memory currently held by processes.
- **free** - memory the kernel hasn't touched at all. Usually a small number, which is fine.
- **available** - memory that could be given to a new process, including buff/cache that the kernel would release if needed.

Always look at "available", not "free". A machine with 200 MiB free but 1.7 GiB available has plenty of memory. Linux uses idle RAM for filesystem cache, and releases it the moment a process needs it.

## buff/cache: not lost, just borrowed

Linux happily uses spare memory to cache recently-read files. That's why "free" often looks tiny on an otherwise idle box. Unlike Windows or macOS, Linux treats unused RAM as wasted RAM.

## Swap

The `Swap:` row is disk-backed memory. On modern cloud VMs it's often zero or disabled. That's normal. If you see swap in active use on a box that has RAM available, something is misconfigured.

## Extracting the available memory value

For a health-check script, one line prints just the "available" value:

```
free -h | grep '^Mem:' | sed 's/.* //'
```

`grep '^Mem:'` keeps just the memory row, and `sed 's/.* //'` deletes everything up to the last space, leaving the final column, which is "available".



# top and htop

`top` is the interactive dashboard for what's running right now.

```
top
```

You get a live-updating screen with two sections. The **summary** block at the top shows load average, CPU%, memory usage, and task counts. The **process list** underneath shows one row per process, sorted by CPU by default. Press `q` to quit.

## Keys worth knowing inside top

- `M` - sort by memory
- `P` - sort by CPU (the default)
- `k` - kill a process (asks for PID)
- `1` - show per-CPU stats instead of aggregated
- `h` - built-in help

## htop: a better top

`htop` is a colored, scrollable, mouse-friendly version of top. It isn't installed on this machine, so you install it first:

```
sudo apt install -y htop
```

That command downloads the package, so it needs network access. If the install fails because the machine is offline, don't worry: everything below works with plain `top`, which is always there. Once htop is installed, run it:

```
htop
```

Same information as top, laid out more clearly. Sort by clicking column headers, kill with F9. Worth installing on any machine you plan to spend real time on.

## Running top in a script

top has a batch mode that prints one snapshot and exits, perfect for scripted health checks:

```
top -b -n1 | head -20
```

Two separate flags do the work here:

- `-b` is batch mode, so top prints plain text instead of drawing the interactive screen.
- `-n1` runs a single iteration and exits instead of updating forever.

Piping into `head -20` keeps just the first 20 lines, which is the summary block plus the busiest handful of processes.

## Two more tools worth knowing

Beyond top, two commands round out the basic toolkit:

- `vmstat` reports virtual memory and CPU activity over time. It ships with the same package as top and free, so it's already on this machine. Try `vmstat 1` to print a fresh line every second.
- `iostat` reports disk read and write stats. It comes from the `sysstat` package, which isn't installed here. Add it with `sudo apt install -y sysstat` (needs network) when you want it.

Both are worth learning once you're comfortable with top and free.

