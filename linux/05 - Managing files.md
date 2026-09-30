#Creating, Copying, Moving, and Removing Files — Notes

## 1. Creating files with `touch`

`touch` creates an empty file.

- `touch /root/notes.txt`

If the file already exists, `touch` updates its modification time without changing its contents.

## 2. Copying files with `cp`

`cp` copies a file while leaving the original unchanged.

- `cp /etc/hostname /tmp/hostname-backup`

To copy a file into a folder, specify the folder path:

- `cp /etc/hostname /tmp/`

To copy an entire folder, use `-r` for recursive copying:

- `cp -r /etc/apt /tmp/apt-backup`

Without `-r`, `cp` will not copy directories.

## 3. Moving and renaming with `mv`

`mv` can move a file or folder, or rename it.

Rename a file:

- `mv /tmp/hostname-backup /tmp/hostname.txt`

Move a file:

- `mv /tmp/hostname.txt /root/`

Unlike `cp`, `mv` does not require `-r` to move folders. Moving is usually fast because Linux changes the file’s location reference rather than copying all its contents.

## 4. Removing files with `rm`

`rm` permanently deletes files.

Delete one file:

- `rm /root/notes.txt`

Delete multiple files:

- `rm /tmp/hostname /root/hostname.txt`

Linux does not normally provide a recoverable trash step for `rm`.

## 5. Removing folders

Use `-r` to delete a folder and its contents recursively:

- `rm -r /tmp/apt-backup`

Use `-f` to force deletion without confirmation:

- `rm -rf /tmp/scratch`

Meaning:

- `-r`: recursive
- `-f`: forced
- `-rf`: recursive and forced

### The `rm -rf` warning

`rm -rf` is extremely powerful and dangerous:

- Deleted files are not moved to a trash folder.
- A typo can delete far more than intended.
- Always verify the path before pressing Enter.
- Pay close attention to the leading `/`.
- Avoid unnecessary wildcards.

For example, `rm -rf /root/oldproject` targets one directory, while an accidental space in `rm -rf / root/oldproject` can cause a destructive command targeting `/`.

Before deleting a folder, inspect it first with `ls`.

## 6. Interactive deletion with `rm -i`

`-i` asks for confirmation before deleting.

- First create a test file with `touch /tmp/scratch.txt`.
- Then use `rm -i /tmp/scratch.txt`.

This is useful when learning Linux or deleting important files.

## 7. Wildcards and globs

Globs are patterns that the shell expands into matching filenames before running a command.

### Main wildcard characters

| Pattern | Meaning |
| --- | --- |
| `*` | Matches any number of characters, including zero |
| `?` | Matches exactly one character |

Example files:

- `a.log`
- `app.log`
- `notes.txt`

Patterns:

- `*.log` matches `a.log` and `app.log`.
- `?.log` matches only `a.log`.
- `*` matches all files in the directory.

## 8. Preparing a glob practice folder

Useful setup commands include:

- `mkdir -p /tmp/globs`
- `cd /tmp/globs`
- `touch a.log app.log notes.txt`

Additional examples:

- `touch server.conf db.conf`
- `touch old_data.txt old_report.txt`
- `touch access.log.gz error.log.gz`

`mkdir -p` creates the directory and any missing parent directories.

## 9. Common glob uses

Copy all `.conf` files:

- `mkdir /tmp/globs/backup`
- `cp *.conf /tmp/globs/backup/`

Move files beginning with `old_`:

- `mkdir /tmp/globs/archive`
- `mv old_* /tmp/globs/archive/`

List compressed `.gz` files:

- `ls -la *.gz`

## 10. The wildcard deletion trap

The shell expands the wildcard before `rm` runs.

- `rm *.tmp` deletes files ending in `.tmp`.
- `rm * .tmp` may attempt to delete everything in the current directory and then process `.tmp` separately.

Always preview a wildcard before deleting:

1. Run `ls *.tmp`.
2. Confirm the results.
3. Run `rm *.tmp`.

## Command selection guide

| Goal | Command |
| --- | --- |
| Create an empty file | `touch file` |
| Copy a file | `cp source destination` |
| Copy a folder | `cp -r source destination` |
| Move a file or folder | `mv source destination` |
| Rename a file | `mv old-name new-name` |
| Delete a file | `rm file` |
| Delete a folder | `rm -r folder` |
| Force deletion | `rm -f file` |
| Delete recursively without prompts | `rm -rf folder` |
| Confirm each deletion | `rm -i file` |
| Match many filenames | Use `*` or `?` |

## Key takeaway

Use `touch` to create files, `cp` to preserve the original while making a copy, and `mv` to move or rename items. Treat `rm`, especially `rm -rf`, with extreme caution. Before using a wildcard with deletion, preview exactly what it matches.
