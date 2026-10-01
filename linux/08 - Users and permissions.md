# Linux Permissions and Ownership Notes

## 1. Reading the ten-character permission string

An `ls -l` entry may begin with:

`-rw-r--r--`

Read it as:

`type | owner | group | other`

| Section | Meaning |
| --- | --- |
| First character | File type |
| Next three characters | Owner permissions |
| Next three characters | Group permissions |
| Final three characters | Other users’ permissions |

### File types

| Character | Meaning |
| --- | --- |
| `-` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `c`, `b`, `s`, `p` | Special file types |

### Permission characters

| Character | File meaning | Directory meaning |
| --- | --- | --- |
| `r` | Read contents | List names |
| `w` | Modify contents | Create, rename, or delete entries |
| `x` | Execute | Enter/traverse the directory |
| `-` | Permission denied | Permission denied |

Example: `-rw-r--r--`

- Regular file
- Owner can read and write
- Group can read
- Others can read
- No one except the owner can write

## 2. Numeric permission values

Permissions have numeric values:

| Permission | Value |
| --- | --- |
| Read (`r`) | 4 |
| Write (`w`) | 2 |
| Execute (`x`) | 1 |

Add the values for each triplet:

| Number | Permissions |
| --- | --- |
| `7` | `rwx` |
| `6` | `rw-` |
| `5` | `r-x` |
| `4` | `r--` |
| `0` | `---` |

For example, `755` means:

- Owner: `7` → `rwx`
- Group: `5` → `r-x`
- Other: `5` → `r-x`

Therefore, `755` is equivalent to `-rwxr-xr-x`.

### Common modes

| Mode | Symbolic form | Typical use |
| --- | --- | --- |
| `755` | `rwxr-xr-x` | Scripts and executables |
| `644` | `rw-r--r--` | Text files and configuration files |
| `700` | `rwx------` | Private directories |
| `600` | `rw-------` | SSH keys, API tokens, and secrets |

The exact default mode depends on the system’s `umask`, but new regular files commonly begin with permissions equivalent to `644`.

## 3. Users and groups

Every Linux process runs as a user, and every file has an owner and group.

### Identify the current user

- `whoami` displays the current username.
- `id` displays the username, user ID, primary group, and secondary groups.

Typical `id` output may look like:

`uid=0(root) gid=0(root) groups=0(root)`

### User information

User accounts are listed in `/etc/passwd` using fields such as:

`username:x:UID:GID:full name:home directory:login shell`

Password hashes are stored separately in `/etc/shadow`, which is normally readable only by `root`.

### Root and sudo

- `root` is the superuser with UID `0`.
- Regular users typically work in `/home/<username>`.
- `sudo` runs a command with elevated privileges.
- `sudo -i` starts an interactive root shell.

On production systems, use a regular account and elevate only when necessary.

### Groups

Groups allow multiple users to share access to files and resources.

Group information is stored in `/etc/group`, with entries such as:

`groupname:x:GID:members`

A user can have:

- One primary group
- Multiple secondary groups

## 4. Reading ownership in `ls -l`

An entry such as:

`-rw-r--r-- 1 root root 178 Feb 10 14:22 /etc/hosts`

contains:

- Permissions: `-rw-r--r--`
- Owner: `root`
- Group: `root`
- File size: `178`
- Modification time
- File path

To determine your access:

1. Check whether you are the file owner.
2. If not, check whether you belong to the file’s group.
3. Otherwise, use the `other` permissions.

## 5. Directory permissions

Directory permissions behave differently from file permissions:

- `r`: list the directory’s names
- `w`: create, rename, or delete entries
- `x`: enter the directory and access items by name

A directory usually needs both `r` and `x` to be useful. A directory with `r` but no `x` may show filenames but prevent access to the files themselves.

Example: `drwxr-xr-x`

- It is a directory.
- The owner can list, modify, and enter it.
- The group and others can list and enter it but cannot modify its contents.

Use `ls -ld directory` to display the directory itself rather than its contents.

## 6. Changing permissions with `chmod`

`chmod` changes file permissions.

### Symbolic notation

Symbolic notation uses:

- `u`: owner
- `g`: group
- `o`: other
- `a`: all users

Operators:

- `+`: add permission
- `-`: remove permission
- `=`: set permissions exactly

Examples:

- `chmod u+x /root/report.txt` — add execute permission for the owner
- `chmod g-w /root/report.txt` — remove write permission from the group
- `chmod o=r /root/report.txt` — set others to read-only
- `chmod a+r /root/report.txt` — give all users read permission

### Octal notation

Octal notation is concise and commonly used:

- `chmod 755 file` → owner can fully access; group and others can read and execute
- `chmod 644 file` → owner can read/write; group and others can read
- `chmod 700 directory` → only the owner can access the directory
- `chmod 600 secret` → only the owner can read/write

## 7. Changing ownership with `chown`

`chown` changes a file’s owner and group.

Examples:

- `chown root file` — change the owner to `root`
- `chown root:root file` — change owner and group
- `chown :root file` — change only the group
- `chown -R user:group directory` — change ownership recursively

Use recursive ownership changes carefully because they affect every file and directory below the target path.

## Quick reference

| Goal | Command |
| --- | --- |
| Show detailed permissions | `ls -l file` |
| Show directory permissions | `ls -ld directory` |
| Identify current user | `whoami` |
| Show user and group membership | `id` |
| Add owner execute permission | `chmod u+x file` |
| Set standard executable permissions | `chmod 755 file` |
| Set standard file permissions | `chmod 644 file` |
| Protect a secret | `chmod 600 secret` |
| Change owner | `chown user file` |
| Change owner and group | `chown user:group file` |

**Core mental model:**

`chmod` controls **what** users can do.

`chown` controls **who** owns the file.

The permission string shows the result.
