# Linux File Listing and Help Commands — Notes

## 1. Listing folder contents with `ls`

# 

- `ls` lists files and folders in the current directory.
- `ls /etc` lists the contents of `/etc`.
- `ls -l /etc` displays a detailed listing.

Example:

```

ls
ls /etc
ls -l /etc
```

## 2. Understanding a long listing

# 
Example:

```
Plain textdrwxr-xr-x  2 root root  4096 Feb 10 14:22 nginx
```

Important fields:

- `d`: the item is a directory.
- `-`: the item is a regular file.
- `rwxr-xr-x`: permissions.
- `root`: owner.
- `root`: group.
- `4096`: size in bytes.
- `Feb 10 14:22`: last modification date and time.
- `nginx`: file or directory name.

For now, focus on:

1. Whether the item is a file or directory.
2. Its size.
3. Its modification date.
4. Its name.

## 3. Hidden files

# 
Files and folders whose names begin with `.` are hidden by default.

```

ls
ls -a
```

- `ls`: hides dotfiles.
- `ls -a`: shows all files, including hidden files.

Common dotfiles and folders:

| Item | Purpose |
| --- | --- |
| `.rc` | Shell startup configuration |
| `.ssh/` | SSH keys and known hosts |
| `.aws/` | AWS configuration and credentials |
| `.kube/` | Kubernetes configuration |
| `.gitignore` | Files Git should ignore |
| `.env` | Project environment configuration |

## 4. Useful `ls` options

# 

| Option | Meaning |
| --- | --- |
| `-a` | Show hidden files |
| `-l` | Use long format |
| `-h` | Show human-readable sizes |
| `-t` | Sort by modification time, newest first |

The commonly used command is:

```

ls -lah
```

This means:

- `-l`: detailed listing
- `-a`: include hidden files
- `-h`: use readable file sizes

To inspect your home directory:

```

ls -lah ~
```

Here, `~` represents your home directory.

## 5. Pipes

# 
A pipe, written as `|`, sends the output of one command into another command.

```

ls /etc | wc -l
```

This command:

1. Lists the contents of `/etc`.
2. Sends the output to `wc`.
3. Uses `-l` to count the lines.

`wc -l` is commonly used to count lines of command output.

## 6. Filtering output with `grep`

# 
`grep` keeps only lines that match a pattern.

```

ls -a ~ | grep '^\.'
```

This shows only entries beginning with a dot, which are hidden files.

The pattern `^\.` means:

- `^`: the beginning of a line
- `\.`: a literal period

## 7. Getting help

### Quick help

# 

```

ls --help
```

`--help` provides a short explanation of the command and its available options.

### Full manual

# 
```

man ls
```

Useful controls inside a manual page:

- Arrow keys: move through the document
- Page Up/Page Down: move faster
- `/`: search
- `q`: quit

Some commands use `-h` for help, but others use it for a different purpose. For example, `ls -h` means human-readable sizes. When unsure, try `--help` first.

## 8. Core workflow

# 
When working in the terminal:

1. Use `ls` to see what is present.
2. Use `ls -lah` to inspect details and hidden files.
3. Use `grep` to filter results.
4. Use `wc -l` to count results.
5. Use `--help` or `man` to learn command options.

## Key takeaway

# 
The `ls` command helps you inspect files and folders. Its flags provide more detail, pipes connect commands, `grep` filters output, and `wc -l` counts lines. When you are unsure how a command works, use `--help` first and `man` for the complete manual.
