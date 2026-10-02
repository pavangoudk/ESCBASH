# The ten characters

This topic is really about one string. When you run `ls -l`, every file answers with something like `-rw-r--r--`, and that string decides who is allowed to touch it.

The trick is to stop reading it as ten characters. It's **one, then three, then three, then three**:

```
-    rw-    r--    r--
↑    ↑      ↑      ↑
type owner  group  other
```

The first character is the file type — `-` for a regular file, `d` for a directory, `l` for a symlink. The nine after it are three identical triplets, one for each audience, and inside every triplet the same three slots appear in the same order: **r**ead, **w**rite, e**x**ecute, or a `-` where that permission is denied.

That's the whole system. Everything else in this topic is a consequence of it.

## The numbers are the same thing

Read is worth 4, write 2, execute 1. Add up one triplet and you get one digit, so three triplets become three digits — which is why `chmod 755` and `-rwxr-xr-x` are two spellings of the same thing.

Four modes cover almost everything you'll do:

| Mode | Bits | Use case |
| --- | --- | --- |
| 755 | `rwxr-xr-x` | Scripts and executables anyone can run |
| 644 | `rw-r--r--` | Configs and text files anyone can read |
| 700 | `rwx------` | A directory only the owner should enter |
| 600 | `rw-------` | Secrets: SSH keys, API tokens, .env files |

## Why it bites

SSH refuses to use a private key that anyone else on the machine can read. A key at `644` gets you `Permissions 0644 for '~/.ssh/id_rsa' are too open` and a rejected login — not because the key is wrong, but because four digits are. `chmod 600` on the key and `700` on `~/.ssh` fixes it.

The nodes that follow take each piece slowly: who users and groups are, how to read a long listing character by character, and how `chmod` and `chown` change what you just read.



# Users and groups

Every process on a Linux machine runs as some user. Every file belongs to a user. That's how the system decides who can read what.

## Who am I

You already met `whoami`. Its sibling `id` gives the fuller picture:

```
whoami          # just the username
id              # username, user ID, primary group, secondary groups
```

A typical `id` output looks like `uid=0(root) gid=0(root) groups=0(root)`.

## Users live in /etc/passwd

Each user has one line in `/etc/passwd`, colon-separated:

```
username:x:UID:GID:full name:home directory:login shell
root:x:0:0:root:/root:/bin/
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/
```

The `x` is a leftover from the days when the password hash lived there. Now hashes are in `/etc/shadow`, which only root can read.

## Two kinds of user

Regular users own their own home folder under `/home/<name>` and can run everyday commands. **root** is the superuser: UID 0, unrestricted, owns most of the system, and can undo anything.

On production servers you almost never log in directly as root. You log in as yourself and use `sudo` for the moments when root permissions are actually needed.

```
sudo apt update             # run this one command as root
sudo -i                     # start an interactive root shell
```

On this lab machine you're already root, so `sudo` is a no-op here. The habit still matters everywhere else.

## Groups

Groups let several users share access to a resource. Every user has one **primary group** (the GID in `/etc/passwd`) and can belong to any number of **secondary groups**, listed in `/etc/group`.

Each group has one line there, in a similar colon-separated shape:

```
groupname:x:GID:comma,separated,members
sudo:x:27:ubuntu
```

The last field lists the users who belong to the group as secondary members. The `ubuntu` user above is a member of `sudo`, and that membership is exactly what lets it run `sudo` commands.

When you look at file ownership in the next node, you'll see both a user and a group next to each file.



# Reading permissions in ls -l

Time to unpack the column you've been ignoring since topic 3. Run:

```
ls -l /etc/hosts
```

You'll see something like:

```
-rw-r--r-- 1 root root 178 Feb 10 14:22 /etc/hosts
```

Focus on the first field: `-rw-r--r--`. Ten characters, three groups.

## The first character: file type

- `-` regular file
- `d` directory
- `l` symbolic link
- `c`, `b`, `s`, `p` special files (you'll meet them later)

## The next nine characters: three triplets

Broken up into three groups of three:

```
rw- r-- r--
```

1. The **first** group is the **owner's** permissions.
2. The **second** group is the **group's** permissions.
3. The **third** group is **other** (everyone else's) permissions.

Each triplet is three letters, read in the same order:

- `r` - read the file (or list a directory)
- `w` - write to the file (or create/delete entries in a directory)
- `x` - execute the file (or `cd` into a directory)
- `-` - permission denied for that slot

So `-rw-r--r--` means: it's a regular file, owner can read and write, group can read only, everyone else can read only.

## r/w/x on directories, briefly

For directories, the letters mean something slightly different:

- `r` - see the list of names inside.
- `w` - create, rename, or delete entries.
- `x` - enter the folder (`cd` into it) and access files by name.

A directory with `r` but no `x` is a weird half-locked state where you can see the names but not actually reach the files. Almost every useful directory has both.

## A directory in a real listing

Make a folder and list it with the `-d` flag, which tells `ls` to describe the directory itself instead of its contents:

```
mkdir -p /root/demo
ls -ld /root/demo
```

```
drwxr-xr-x 2 root root 4096 Feb 10 14:22 /root/demo
```

Read `drwxr-xr-x` the same way, character by character. The first character is `d`, so it's a directory. Then `rwx` for the owner (root can list it, add or delete entries, and enter it), `r-x` for the group, and `r-x` for other. That `drwxr-xr-x` shape is what a normal directory looks like: the owner controls what's inside, and everyone else can look and enter but not change anything.

## The owner and group columns

Right after the permissions block, `ls -l` prints the owner and group:

```
-rw-r--r-- 1 root root 178 Feb 10 14:22 /etc/hosts
            ^^^^ ^^^^
            owner group
```

Reading permissions is now a two-step check: identify which triplet applies to you (are you the owner, in the group, or neither?), then look for r/w/x in that triplet.



# chmod and chown

Two commands let you change what you saw in the previous listing: `chmod` changes permissions, `chown` changes ownership.

## chmod, the symbolic way

Start with a file to practice on:

```
touch /root/report.txt
ls -l /root/report.txt
```

A freshly created file starts at mode `644`:

```
-rw-r--r-- 1 root root 0 Feb 10 14:22 /root/report.txt
```

Symbolic notation reads like a mini-sentence: **who**, **operator**, **which permission**.

Who: `u` (user/owner), `g` (group), `o` (other), `a` (all). Operator: `+` (add), `-` (remove), `=` (set exactly). Permission: `r`, `w`, `x`.

Give the owner execute, then look at the file again:

```
chmod u+x /root/report.txt
ls -l /root/report.txt
```

```
-rwxr--r-- 1 root root 0 Feb 10 14:22 /root/report.txt
```

The owner triplet went from `rw-` to `rwx`, and nothing else moved. A few more shapes work the same way:

```
chmod g-w /root/report.txt      # remove write from group
chmod o=r /root/report.txt      # set other to read only
chmod a+r /root/report.txt      # give everyone read
```

Fine for one-off tweaks, but most DevOps engineers reach for the octal form instead.

## chmod, the octal way

The three triplets can be written as three digits. Each digit is the sum of read (4), write (2), and execute (1):

| Digit | Meaning |
| --- | --- |
| 7 | rwx |
| 6 | rw- |
| 5 | r-x |
| 4 | r-- |
| 0 | --- |

So this command, run on the same file:

```
chmod 755 /root/report.txt
ls -l /root/report.txt
```

sets owner `rwx`, group `r-x`, other `r-x`:

```
-rwxr-xr-x 1 root root 0 Feb 10 14:22 /root/report.txt
```

Three digits, one call, unambiguous.

## Presets you'll type constantly

| Mode | Use case |
| --- | --- |
| 755 | Scripts and executables anyone can run |
| 644 | Configs and text files anyone can read |
| 700 | Directory only the owner should enter |
| 600 | Secrets: SSH keys, API tokens, .env files |

Learn these four and you can set the right mode on 90% of files without thinking.

## chown for ownership

`chown` sets who owns a file. The forms you'll use, shown here on the file you already created:

```
chown root /root/report.txt         # set the owner
chown root:root /root/report.txt    # set owner and group together
chown :root /root/report.txt        # set only the group
```

Add `-R` to change an entire folder and everything inside it in one call, which is what you reach for when handing a whole web root from one service to another.

Here every file already belongs to root, so these commands run without changing much. On a real server you'd name a real user and group, like `chown alice:developers deploy.sh`, and put `sudo` in front when moving files between services.

