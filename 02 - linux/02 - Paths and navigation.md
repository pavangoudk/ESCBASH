# The Linux filesystem tree

Every file and folder on a Linux machine lives inside one tree. The tree has exactly one root, written `/`. No `C:`, no `D:`, no separate drives.

```
/
├── etc/       # system-wide configuration
├── var/       # variable data: logs, spools, caches
├── home/      # regular users' home folders
├── root/      # the root user's home
├── tmp/       # temporary files, often wiped on reboot
├── usr/       # installed software
└── bin/       # essential command binaries
```

These are the most common folders that you would see directly under `/`. Each one has a specific job.

- `/etc` holds system-wide configuration.
- `/var` holds data that changes while the system runs, like logs, mail queues, and package caches. The one you'll open most is `/var/log`, which is where every service writes its log files.
- `/home` holds one folder per regular user account. A user named `alice` gets `/home/alice` as her home folder, and she typically owns everything inside it.
- `/root` is the home folder of the root (administrator) user. It's kept separate from `/home` so the root account can still work even if `/home` is on a broken disk.
- `/tmp` is scratch space. Any process can write files here, and the contents are typically wiped on reboot. Never store anything you want to keep.
- `/usr` holds installed software: user programs in `/usr/bin`, libraries in `/usr/lib`, documentation in `/usr/share`. Package managers put most of their files here.
- `/bin` holds essential command binaries needed early during boot and system recovery, like `ls`, `cp`, and `bash`. On modern Linux it's usually a symlink to `/usr/bin`.

The others exist too (`/dev`, `/proc`, `/sys`, `/opt`, `/srv`, `/mnt`, `/media`), and you'll meet them as they come up.

## Where you are in the tree

You met `pwd` in the previous topic. Run it now and you'll see where in this tree the shell has placed you (probably `/root`, the root user's home). Every path you write is either an address starting from `/` (the top of this tree) or a shortcut from wherever `pwd` currently reports. The next node covers those two styles in detail.

# Absolute and relative paths

A **path** is an address for a file or folder. If you want to access a file or a folder in Linux, you can use either its **absolute path** or its **relative path**.

## Absolute paths

Imagine there is a file `nginx.conf` inside the `nginx` folder, which is inside the `etc` folder:

```
/
└── etc
    └── nginx
        └── nginx.conf
```

An absolute path starts with `/` and describes the full route from the top of the tree down to the file. For the file above, that is:

```
/etc/nginx/nginx.conf
```

Read the slashes as "inside". `/etc/nginx/nginx.conf` means the file `nginx.conf` inside the folder `nginx`, which is inside the folder `etc` at the top of the tree.

A few more examples:

```
/var/log/syslog
/home/alice/notes.txt
```

Absolute paths work from anywhere. Standing in `/tmp`, you can still say `/etc/hosts` and the shell knows exactly which file you mean.

## Relative paths

A relative path doesn't start with `/`. It's read starting from wherever you're currently standing.

If your current folder is `/home/alice`, then:

- `notes.txt` means `/home/alice/notes.txt`
- `photos/summer.jpg` means `/home/alice/photos/summer.jpg`

Two shortcuts turn up in relative paths: `.` means the folder you are in now, and `..` means the folder one level up. So from `/home/alice`, `../bob/notes.txt` means `/home/bob/notes.txt`.

Both styles name the same file. Use an absolute path when the command has to work from anywhere, and a relative path when you are already standing next to the file.
