# Listing a folder

`ls` prints what's inside a folder. Given no arguments, it lists your current folder. Given a path, it lists that folder.

```
ls              # current folder
ls /etc         # specifically /etc
```

On a fresh machine your home folder can look empty, because its only files are hidden (their names start with a dot). A populated folder like `/etc` shows the difference right away.

The plain output shows names only. Useful, but not the whole picture. Add `-l` (lowercase L, for "long") and every entry expands into a row with metadata:

```
ls -l /etc
```

A single row looks like this:

```
drwxr-xr-x  2 root root   4096 Feb 10 14:22 nginx
```

Seven columns worth reading:

1. `d` says this is a directory. A `-` in that position means a regular file.
2. `rwxr-xr-x` is the permissions preview. You will learn this in a later topic.
3. `2` is the link count. Ignore it for now.
4. `root` is the owner.
5. `root` is the group.
6. `4096` is the size in bytes.
7. The date and time (in the example, `Feb 10` at `14:22`) show when the file was last modified.
8. `nginx` is the name.

For now, focus on the first character (file or folder?) and the last few columns (size, date, name). The rest becomes useful once you learn about permissions and ownership.

---

# Hidden files and useful flags

## Hidden files

Any file or folder whose name starts with a dot is hidden from plain `ls`. These are called **dotfiles**.

```
ls           # normal listing, no dotfiles
ls -a        # include hidden entries
```

Dotfiles hold configuration. On any Linux machine you'll meet:

- `.rc` - shell startup config
- `.ssh/` - SSH keys and known hosts
- `.aws/`, `.kube/` - cloud and cluster credentials
- `.gitignore`, `.env` - project-level config

As a DevOps engineer you'll edit dotfiles constantly. Getting comfortable with `-a` early saves a lot of "where did that file go?" moments.

## Human sizes and time sorting

Two more flags earn their keep every day:

- `-h` prints file sizes in human units (`4.0K`, `12M`, `3.2G`) instead of raw bytes.
- `-t` sorts newest first, which is exactly what you want when hunting for the file that just changed.

## The ls -lah combo

Most people run `ls` with a combination of flags. The one you'll type hundreds of times a week:

```
ls -lah
```

That's long listing, all files (hidden included), human-readable sizes. Type it now against your home folder:

```
ls -lah ~
```

You'll see files like `.rc` and `.profile` that plain `ls` was hiding from you.

## Pipes and counting lines

A **pipe** (`|`) sends the output of one command as the input of another. It's the single most useful operator in the shell.

```
ls /etc | wc -l           # how many entries live in /etc
```

Read that as: run `ls /etc`, then send its output into `wc -l`, which counts lines. Because `ls` prints one entry per line when its output isn't a terminal, this gives you the count in one shot.

`wc` is short for "word count". You'll use the `-l` flag (count lines) far more often than the others. A full topic on reading files covers `wc` in more detail; for now, `wc -l` is the count tool.

You can also filter output with `grep`, which keeps only the lines matching a pattern:

```
ls -a ~ | grep '^\.'      # only entries starting with a dot (dotfiles)
```

`grep` gets its own topic later. Recognise the shape for now: some command produces lines, `grep` keeps the ones that match.

---

# Getting help

Nobody memorises every flag of every command. You look them up. Linux ships with three ways to do that, all on the machine you're already logged into.

## `--help`

Almost every command accepts `--help`. It prints a short summary of what the command does and every flag it takes.

```
ls --help
```

Fast, keyword-searchable when piped through `grep`, and never lies about the version you have installed.

## `man`

`man` opens the full manual page for a command. Use it when `--help` is too terse.

```
man ls
```

Navigate with the arrow keys or Page Up / Page Down. Press `/` to search, and `q` to quit. Every core Linux command has a man page.

## `-h`

Some commands use `-h` as a shortcut for help (`docker -h`, `curl -h`). Others use `-h` for something completely different, like "human-readable sizes" in `ls -h`. When in doubt, try `--help` first, it's the convention.

As a rule of thumb: reach for `--help` first because it's fast, and fall back to `man` when you need the full picture.

---
