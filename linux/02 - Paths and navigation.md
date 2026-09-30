# Linux Filesystem Tree — Notes

## 1. The filesystem tree

- Every file and folder exists inside one tree.
- The tree has one root: `/`.
- Linux does not use separate drive letters such as `C:` or `D:`.

### Common folders under `/`

| Folder | Purpose |
| --- | --- |
| `/etc` | System-wide configuration files |
| `/var` | Changing data such as logs, mail queues, and caches |
| `/home` | Home folders for regular users |
| `/root` | Home folder for the root administrator |
| `/tmp` | Temporary scratch space; files may be deleted on reboot |
| `/usr` | Installed software, libraries, and documentation |
| `/bin` | Essential command programs; often linked to `/usr/bin` |

Other important folders include `/dev`, `/proc`, `/sys`, `/opt`, `/srv`, `/mnt`, and `/media`.

## 2. Important folder details

- `/var/log` contains logs written by system services.
- A user named Alice normally has `/home/alice`.
- The root user’s home is `/root`, separate from `/home`.
- Do not store important files in `/tmp`.
- `/usr/bin` contains user programs.
- `/usr/lib` contains libraries.
- `/usr/share` contains shared documentation and other data.
- On modern Linux systems, `/bin` is usually a symbolic link to `/usr/bin`.

## 3. Paths

A path identifies the location of a file or folder.

### Absolute paths

- Begin with `/`.
- Describe the complete route from the filesystem root.
- Work from any current directory.

Example: `/etc/nginx/nginx.conf`

This means:

- `nginx.conf` is inside `nginx`
- `nginx` is inside `etc`
- `etc` is directly under `/`

Other examples:

- `/var/log/syslog`
- `/home/alice/notes.txt`

### Relative paths

- Do not begin with `/`.
- Start from the directory where you are currently located.
- Your current location can be checked with `pwd`.

If the current directory is `/home/alice`:

- `notes.txt` refers to `/home/alice/notes.txt`
- `photos/summer.jpg` refers to `/home/alice/photos/summer.jpg`
- `../bob/notes.txt` refers to `/home/bob/notes.txt`

## 4. Path shortcuts

- `.` means the current directory.
- `..` means the parent directory, one level higher.

## 5. Main rule

- Use an absolute path when a command must work from anywhere.
- Use a relative path when you are already near the target file or folder.

| Path type | Starts with | Best used when |
| --- | --- | --- |
| Absolute | `/` | You need an unambiguous location |
| Relative | Anything else | You are working from a known current directory |
