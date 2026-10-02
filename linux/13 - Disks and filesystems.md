# 1. Disks, Partitions and Mount Points

Imagine you have an Ubuntu server with a **100 GB disk**.

At the physical/virtual level, you have a disk:

```text
Disk
/dev/sda
100 GB
```

But Linux doesn't directly treat the entire disk as your normal file/folder structure.

There are multiple layers.

### The storage layers

```text
Physical / Virtual Disk
        ↓
    Partition
        ↓
   Filesystem
        ↓
   Mount Point
        ↓
 Linux directories/files
```

Let's understand each one.

---

## 1.1 Disk

A disk is the actual storage device.

On Linux you might see:

```text
/dev/sda
/dev/vda
/dev/nvme0n1
```

In a cloud environment, this is usually a **virtual disk**, even though Linux presents it like a block device.

For example:

```text
/dev/sda
   100 GB
```

Think:

> **Disk = the complete storage device**

---

# 1.2 Partition

A disk can be divided into smaller sections called **partitions**.

For example:

```text
/dev/sda
100 GB
│
├── /dev/sda1 → 99 GB
└── /dev/sda2 → 1 GB
```

Here:

* `/dev/sda` = entire disk
* `/dev/sda1` = first partition
* `/dev/sda2` = second partition

Think of it like a 100-acre piece of land divided into two plots.

---

# 1.3 Filesystem

A partition by itself doesn't understand things like:

```text
file
folder
permissions
directories
```

A **filesystem** provides that structure.

Common Linux filesystems include:

```text
ext4
xfs
btrfs
```

For example:

```text
/dev/sda1
    ↓
   ext4
```

Now Linux can store files and directories on it.

Think:

```text
Partition = empty plot
Filesystem = system that organizes the plot
```

---

# 1.4 Mount Point

Now we need to make that filesystem available somewhere in Linux's directory tree.

That's where **mounting** comes in.

For example:

```text
/dev/sda1
   ↓
  ext4
   ↓
   /
```

`/` is the **mount point**.

You can also have:

```text
/dev/sdb1
   ↓
  ext4
   ↓
 /data
```

Now everything stored under `/data` is stored on that filesystem.

---

# 1.5 Why Is This Important?

Suppose your application writes:

```text
/var/log/app.log
```

Linux needs to determine:

> Which filesystem contains `/var`?

Suppose `/var` belongs to the root filesystem:

```text
/dev/sda1
    ↓
    /
    ↓
  /var
    ↓
app.log
```

The data ultimately gets stored on `/dev/sda1`.

This becomes **very important in production troubleshooting**.

For example:

> "My application can't write logs."

One possible reason is that the filesystem containing `/var` is full.

---

# 2. `lsblk` — See Your Storage Structure

The first command you should remember is:

```bash
lsblk
```

It shows block devices in a tree structure.

Example:

```text
NAME    SIZE TYPE MOUNTPOINTS
sda     100G disk
├─sda1   99G part /
└─sda2    1G part [SWAP]
```

Read it like this:

```text
sda
100 GB disk
│
├── sda1
│   99 GB partition
│   mounted at /
│
└── sda2
    1 GB partition
    used for swap
```

### Very important

Don't expect your server to look exactly like this.

A cloud VM might simply show:

```text
vda
└── vda1
```

or even have a different layout.

That's normal.

---

# 3. What is `/etc/fstab`?

There is a configuration file:

```text
/etc/fstab
```

It tells Linux:

> "When the machine starts, mount these filesystems at these locations."

For example:

```text
/dev/sda1    /    ext4    defaults    0 1
```

This means roughly:

```text
/dev/sda1
   ↓
mount it at
   ↓
/
```

### Why should you be careful?

`/etc/fstab` is involved during boot.

If you put an incorrect entry there, the machine may have problems mounting filesystems during startup.

So:

> **Never modify `/etc/fstab casually on a production server.**

---

# 4. Now the Real Troubleshooting Part: `df` vs `du`

Imagine your server suddenly reports:

> Disk space is almost full!

You need to answer two different questions.

### Question 1

**How full is the filesystem?**

Use:

```bash
df -h
```

### Question 2

**Which folder is consuming the space?**

Use:

```bash
du
```

This distinction is extremely important.

---

# 5. `df -h`

Run:

```bash
df -h
```

Example:

```text
Filesystem      Size  Used  Avail  Use%  Mounted on
/dev/sda1        99G   14G    80G   15%  /
```

Let's understand it:

### Filesystem

```text
/dev/sda1
```

The device/filesystem being reported.

### Size

```text
99G
```

Total capacity.

### Used

```text
14G
```

Space currently used.

### Avail

```text
80G
```

Available space.

### Use%

```text
15%
```

Percentage used.

### Mounted on

```text
/
```

Where that filesystem is mounted.

---

# 6. Why `-h`?

Without `-h`, the output may use less friendly units.

With:

```bash
df -h
```

you get human-readable values like:

```text
99G
14G
80G
```

instead of difficult-to-read block values.

---

# 7. What Should You Watch?

The most important column is:

```text
Use%
```

For example:

```text
/dev/sda1   99G   90G   9G   91%   /
```

Now you know:

> The filesystem is 91% full.

You need to investigate what is consuming the space.

And this is where `du` comes in.

---

# 8. `du` — Find Folder Size

`du` means **disk usage**.

For example:

```bash
du -sh /var/log
```

You might get:

```text
28M    /var/log
```

This means:

> `/var/log` is consuming approximately 28 MB.

---

# 9. Understanding `du -sh`

There are two important options.

### `-s`

Means:

**summary**

Instead of showing every file and subdirectory, give me the total.

### `-h`

Means:

**human-readable**

So:

```bash
du -sh /var/log
```

means:

> Give me a human-readable summary of the total disk usage of `/var/log`.

---

# 10. `df` vs `du`

This is one of the most important concepts from this lesson.

| Command  | Question it answers         |
| -------- | --------------------------- |
| `df -h`  | How full is the filesystem? |
| `du -sh` | How big is this folder?     |

Remember:

```text
df → Filesystem level
du → Directory/file level
```

### Example

You run:

```bash
df -h
```

and discover:

```text
/dev/sda1   100G   95G   5G   95%   /
```

You know:

> Something is consuming 95 GB.

But `df` doesn't tell you **what**.

So you start checking:

```bash
du -sh /var
du -sh /home
du -sh /opt
du -sh /tmp
```

Now you can identify where the space is going.

---

# 11. Finding the Exact Folder

Suppose:

```bash
du -sh /var
```

returns:

```text
80G    /var
```

Now `/var` looks suspicious.

Don't immediately delete anything.

Go one level deeper:

```bash
sudo du -h --max-depth=1 /var 2>/dev/null | sort -h
```

You might get:

```text
100M    /var/cache
2G      /var/lib
75G     /var/log
80G     /var
```

Now you know:

```text
/var
  ↓
/var/log
  ↓
75 GB
```

So you investigate `/var/log`.

```bash
sudo du -h --max-depth=1 /var/log 2>/dev/null | sort -h
```

Maybe:

```text
500M    /var/log/nginx
2G      /var/log/app
72G     /var/log/myapplication
75G     /var/log
```

Now you've found the problem.

---

# 12. Understanding the Command

This command looks complicated:

```bash
sudo du -h --max-depth=1 /var 2>/dev/null | sort -h
```

But break it into pieces.

### `sudo`

Run with elevated permissions.

Some directories cannot be completely read by normal users.

### `du`

Check disk usage.

### `-h`

Human-readable sizes.

### `--max-depth=1`

Only go **one level deep**.

Without it, you could get thousands of lines.

### `2>/dev/null`

Hide permission-denied errors.

### `|`

Pipe the output to another command.

### `sort -h`

Sort the sizes using human-readable values.

So the overall meaning is:

> "Show me the size of each direct folder under `/var`, sort them by size, and hide permission errors."

---

# 13. `ncdu`

There is another useful tool:

```bash
ncdu
```

It provides an interactive way to explore disk usage.

Install it:

```bash
sudo apt install -y ncdu
```

Then:

```bash
ncdu /var
```

You can navigate through directories using the keyboard.

It's useful during production troubleshooting because instead of repeatedly typing `du`, you can interactively drill down.

But remember:

> If you don't have internet/package access, `ncdu` may not be available.

In that situation:

```bash
du
```

is your reliable option.

---

# 14. Why Can `df` and `du` Show Different Numbers?

This is a **very important real-world troubleshooting scenario**.

Imagine:

```bash
df -h
```

says:

```text
/dev/sda1   100G   90G   10G   90%   /
```

But:

```bash
du -sh /
```

shows something much smaller.

You think:

> "Where did the missing space go?"

One common reason is a **deleted file that is still open by a running process**.

---

# 15. Deleted File Still Open

Imagine your application is writing to:

```text
/var/log/app.log
```

The file becomes huge:

```text
app.log → 20 GB
```

Someone deletes it:

```bash
rm app.log
```

You might expect the 20 GB to immediately become available.

But the application still has the file open.

So:

```text
Directory
   ↓
File deleted
   ↓
File no longer visible
   ↓
Application still has it open
   ↓
Disk space still occupied
```

Therefore:

```text
df → still sees the 20 GB
du → cannot see the deleted file
```

This creates the classic:

> **"df says 90%, but du says 50%" problem.**

This is a very useful production troubleshooting concept to remember.

---

# 16. Complete DevOps Troubleshooting Flow

Imagine you receive an alert:

> **Disk usage is 95%**

Don't immediately delete files.

Follow this process.

### Step 1 — Check filesystem usage

```bash
df -h
```

Find the filesystem that is full.

Example:

```text
/dev/sda1   100G   95G   5G   95%   /
```

### Step 2 — Find large top-level directories

```bash
sudo du -h --max-depth=1 / 2>/dev/null | sort -h
```

Maybe:

```text
5G     /home
10G    /opt
75G    /var
95G    /
```

Now investigate `/var`.

### Step 3 — Drill down

```bash
sudo du -h --max-depth=1 /var 2>/dev/null | sort -h
```

Maybe:

```text
2G     /var/lib
70G    /var/log
75G    /var
```

Now investigate `/var/log`.

### Step 4 — Find the actual problem

```bash
sudo du -h --max-depth=1 /var/log 2>/dev/null | sort -h
```

Now you might discover that one application's logs are consuming most of the space.

### Step 5 — Don't blindly delete

First understand:

* What generated the files?
* Is the application still using them?
* Is log rotation configured?
* Are the files safe to remove?
* Is there a retention requirement?

---

# ⭐ The Mental Model You Should Remember

### Storage structure

```text
Disk
 ↓
Partition
 ↓
Filesystem
 ↓
Mount Point
 ↓
Directories
 ↓
Files
```

### Disk troubleshooting

```text
df
 ↓
Which filesystem is full?
 ↓
du
 ↓
Which directory is consuming space?
 ↓
du --max-depth=1
 ↓
Drill down
 ↓
Find the actual large files/folders
```

### Three commands to remember first

```bash
lsblk
```

**What storage devices/partitions do I have?**

```bash
df -h
```

**How full is each filesystem?**

```bash
du -sh /var/log
```

**How much space is this directory using?**

And for real troubleshooting:

```bash
sudo du -h --max-depth=1 /var 2>/dev/null | sort -h
```

**Which subdirectory is consuming the space?**

If you remember just **`lsblk → df → du`**, you've captured the core of this entire lesson.
