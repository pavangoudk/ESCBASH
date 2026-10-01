# Linux Notes: `grep` and `find`

## 1. `grep`: Search inside files

`grep` searches for a pattern inside a file and prints every matching line.

### Basic syntax

```
grep pattern file
```

Example:

```
grep root /etc/passwd
```

This prints every line in `/etc/passwd` containing `root`.

## 2. Creating a demo log file

```
mkdir -p /tmp/grep-demo
cd /tmp/grep-demo
echo "Starting web service on port 8080" > app.log
echo "User alice logged in" >> app.log
echo "ERROR failed to connect to database" >> app.log
echo "Retrying database connection" >> app.log
echo "error disk space low" >> app.log
echo "Service healthy" >> app.log
```

The file contains six lines. Search for lowercase `error`:

```
grep error app.log
```

Only the lowercase match is returned because `grep` is case-sensitive by default.

## 3. Important `grep` options

| Option | Purpose | Example |
| --- | --- | --- |
| `-i` | Ignore uppercase/lowercase differences | `grep -i error app.log` |
| `-n` | Show matching line numbers | `grep -n error app.log` |
| `-c` | Count matching lines | `grep -c error app.log` |
| `-v` | Show lines that do not match | `grep -v error app.log` |
| `-r` | Search recursively through directories | `grep -r "listen" /etc/nginx/` |
| `-l` | Print only filenames with matches | `grep -r -l "PermitRootLogin" /etc/` |
| `--include` | Restrict recursive search to matching filenames | `grep -r --include='*.conf' "listen" /etc/` |

### Common examples

Case-insensitive search:

```
grep -i error app.log
```

Show line numbers:

```
grep -in error app.log
```

Expected output:

```
3:ERROR failed to connect to database
5:error disk space low
```

Count matches:

```
grep -ic error app.log
```

Expected result: `2`

Show everything except matching lines:

```
grep -iv error app.log
```

## 4. Combining `grep` options

Options can be combined:

```
grep -in error app.log
grep -ni error app.log
grep -i -n error app.log
```

These commands are equivalent. Option order generally does not matter.

## 5. Searching across multiple files

Use `-r` to search recursively through a directory:

```
grep -r "listen" /etc/nginx/
```

Results typically appear in this format:

```
Plain textfilename:line containing the match
```

Search only configuration files:

```
grep -r --include='*.conf' "listen" /etc/
```

Show only files containing a match:

```
grep -r -l "PermitRootLogin" /etc/
```

## 6. Using `grep` with pipes

`grep` can filter the output of another command.

```
ps aux | grep nginx
history | grep ssh
ls /etc | grep -i network
```

The general pattern is:

```
Plain textcommand that produces output | grep pattern
```

This is useful for filtering processes, command history, directory listings, and logs.

# 7. `find`: Locate files and directories

`find` searches for files and directories based on conditions.

### Basic syntax

```
find <path> [conditions]
```

Unlike `grep`, which searches inside files, `find` locates the files themselves.

## 8. Common `find` conditions

| Condition | Purpose |
| --- | --- |
| `-name 'pattern'` | Find names using case-sensitive matching |
| `-iname 'pattern'` | Find names using case-insensitive matching |
| `-type f` | Search for regular files |
| `-type d` | Search for directories |

### Examples

Find a specific file:

```
find /etc -name 'nginx.conf'
```

Find all `.conf` files:

```
find /etc -name '*.conf'
```

Find a directory:

```
find / -type d -name 'nginx'
```

Find regular log files:

```
find /var/log -type f -name '*.log'
```

## 9. Always quote wildcard patterns

Quote patterns containing wildcards such as `*`.

Correct:

```
find /etc -name '*.conf'
```

Incorrect:

```
find /etc -name *.conf
```

Without quotes, the shell may expand `*.conf` before `find` receives it, causing errors or unexpected results.

## 10. `grep` versus `find`

| Question | Command |
| --- | --- |
| Where is a file named `nginx.conf`? | `find` |
| Which files contain the word `listen`? | `grep -r` |
| Which `.conf` files contain `listen`? | `grep -r --include='*.conf'` |

### Key distinction

- **`find` locates files based on their names, types, or paths.**
- **`grep` searches for text inside files or command output.**

A useful mental model:

```
Plain textfind = Which files?
grep = Which lines?
```

## Quick reference

```
grep pattern file
grep -i pattern file
grep -n pattern file
grep -c pattern file
grep -v pattern file
grep -r pattern directory
grep -r -l pattern directory
find path -name 'pattern'
find path -iname 'pattern'
find path -type f
find path -type d
```

### Practical takeaway

Start with `find` when you need to locate a file. Start with `grep` when you need to inspect its contents. In day-to-day DevOps work, combining both tools helps you quickly locate configuration settings, investigate logs, and troubleshoot services.
