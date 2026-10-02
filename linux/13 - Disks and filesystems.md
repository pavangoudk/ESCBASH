# Disks, partitions, and mount points

A Linux machine's storage stacks up in layers:

- A **physical disk** (or virtual disk on a VM) is the raw hardware. Examples: `/dev/sda`, `/dev/nvme0n1`, `/dev/vda`.
- A disk is carved into **partitions**. `/dev/sda1` is the first partition of `/dev/sda`.
- Each partition holds a **filesystem** (ext4, xfs, btrfs, etc). The filesystem is what actually understands "files" and "folders".
- The filesystem is **mounted** onto a folder in the tree. That folder is called a **mount point**. `/`, `/home`, `/var` are common ones.

When you write to `/var/log/app.log`, Linux checks which filesystem holds `/var`, sends the write to that filesystem, which stores it on its underlying partition on the disk.

## lsblk: see the whole stack

```
lsblk
```

Prints every block device the kernel knows about, as a tree:

```
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda       8:0    0  100G  0 disk
├─sda1    8:1    0   99G  0 part /
└─sda2    8:2    0    1G  0 part [SWAP]
```

Reading it:

- `sda` is the disk, 100 GB.
- `sda1` is the main partition, 99 GB, mounted at `/`.
- `sda2` is a small partition holding swap.

`lsblk` is your first stop when you want to know what storage exists before worrying about how full it is.

Your own output will vary. A small cloud VM often shows a single device like `vda` with just one partition (or none), not the tidy multi-partition tree above. That's normal.

## /etc/fstab, briefly

`fstab` tells Linux what to mount where at boot:

```
/dev/sda1  /       ext4  defaults  0 1
```

You'll edit it rarely. A broken `/etc/fstab` can prevent the machine from booting cleanly, so never touch it without checking your work twice.



# Disk usage with df and du

Two commands, two very different questions.

## df: how full each filesystem is

Run this to see free space on every mounted filesystem:

```
df -h
```

The `-h` flag means human-readable, so sizes come back as K, M, and G instead of raw block counts. You get one row per filesystem:

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        99G   14G   80G  15% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
```

Read the `/dev/sda1` row left to right:

- `Filesystem` is the device behind the mount, here `/dev/sda1`.
- `Size` is the total capacity, 99G.
- `Used` is what's taken, 14G.
- `Avail` is what's still free, 80G.
- `Use%` is used space as a percentage, 15%.
- `Mounted on` is the folder this filesystem lives at, `/`.

`Use%` is the number you watch. When any real filesystem creeps past 80%, start planning. Past 95%, act now. The `tmpfs` rows are backed by memory rather than disk, so they're usually not worth worrying about.

df tells you how much room is left per filesystem. It never tells you which folder ate the space. For that you need du.

## du: how big a folder is

df measures whole filesystems. du measures a single folder by adding up the sizes of everything inside it. Point it at a path:

```
du -sh /var/log
```

You get one line back, the total size of that folder:

```
28M  /var/log
```

Two flags did the work:

- `-s` summarizes the whole path into one number. Without it, du prints a separate line for every subfolder underneath.
- `-h` uses human units (K, M, G) instead of raw kilobyte counts.

Try it on a config folder to compare:

```
du -sh /etc
```

```
5.8M  /etc
```

Drop the `-s` when you want the breakdown instead of the total:

```
du -h /var/log
```

That recurses into every subfolder and prints a size for each one, which is how you find the specific folder that grew. The next node turns that into a repeatable search.

## Why df and du sometimes disagree

`df` reads filesystem-level accounting. `du` walks the tree and adds up file sizes. They can disagree when:

- A process still holds a **deleted file open**. df sees the space used, du doesn't see the file at all.
- **Sparse files** exist. du sees the logical size, df sees the physical size on disk.

That "df says 90% full, du says 30%" moment usually means someone deleted a huge log while an app was still writing to it. Restart the app and the space frees up.



# Finding large folders with du

`df` told you a filesystem is full. `du` will tell you which folder is at fault, if you point it at the right place.

## The top-level scan

Start at the mount that's full and drill down one level at a time:

```
sudo du -h --max-depth=1 /var 2>/dev/null | sort -h
```

Three flags earning their keep:

- `--max-depth=1` limits recursion to one level, so you get a line per direct subfolder instead of every file.
- `sort -h` sorts by human-readable size, smallest to largest.
- `2>/dev/null` throws away the "permission denied" noise from folders you can't read.

The biggest folder sits at the bottom. Repeat inside that folder:

```
sudo du -h --max-depth=1 /var/log 2>/dev/null | sort -h
```

Two or three iterations and you've found the guilty folder or file.

## ncdu: an interactive alternative

`ncdu` is a scrollable, interactive version of `du`. It isn't installed on this machine by default. On a box with package access you'd add it first:

```
sudo apt install -y ncdu
```

Then point it at a folder:

```
ncdu /var
```

Arrow keys drill into folders, and `d` deletes the selected entry. It's faster than repeated `du` calls when you're triaging a full disk under pressure, so it's worth installing on any box you'll be called to debug at odd hours. On an offline machine the install won't reach the package servers, so `du` piped into `sort -h` stays your reliable fallback.

## What NOT to do

Never `rm -rf` in `/var` (or anywhere else) based on a hunch. Confirm which files are safe to delete first. Active processes are writing to files right now, and deleting one of those triggers the "df and du disagree" mystery from the last node without actually freeing space.

