# grep for finding text in a file

`grep` searches for a pattern inside a file and prints every line that matches. It is one of the commands you will reach for most as a DevOps engineer. Log files run to thousands of lines, config files are full of settings you don't care about, and command output can scroll off the screen. `grep` pulls out only the lines that matter.

When a service breaks, the first move is almost always to `grep` the logs for the word `error`. When you need to know how a setting is configured, you `grep` the config file for it. Learning `grep` well pays off every single day.

## The basic form

```
grep pattern file
```

You give it a pattern (the text to look for) and a file (where to look). Try it on `/etc/passwd`, a file that lists the accounts on the system:

```
grep root /etc/passwd
```

That prints every line of `/etc/passwd` containing the word `root`. Usually more than one line comes back, because "root" appears in system paths too.

## Prepare a demo log file

Real logs are long and messy, so let's make a small one you can predict. Run these commands to create a log file with a few lines in it:

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

Now `app.log` has six lines. Search it for the word `error`:

```
grep error app.log
```

You get back only one line:

```
error disk space low
```

Notice the line with `ERROR` (in capitals) did **not** match. By default `grep` cares about upper- and lower-case. That is exactly the kind of thing the flags below fix.

## The four flags you'll use daily

### -i — ignore case

`-i` makes the search case-insensitive, so `error`, `Error`, and `ERROR` all match. Run:

```
grep -i error app.log
```

Now both error lines come back:

```
ERROR failed to connect to database
error disk space low
```

Almost always what you want when searching logs, because you rarely know how the message was capitalised.

### -n — show line numbers

`-n` prints the line number in front of each match, so you know where in the file it lives:

```
grep -in error app.log
```

```
3:ERROR failed to connect to database
5:error disk space low
```

The matches are on lines 3 and 5. Handy when you want to open the file at that spot in an editor.

### -c — count matches

`-c` skips the matching lines and just prints how many there were:

```
grep -ic error app.log
```

```
2
```

Two lines matched. Good for questions like "how many errors are in this log?" without scrolling through them all.

### -v — invert the match

`-v` flips the meaning: it prints every line that does **not** match the pattern. Print everything except the error lines:

```
grep -iv error app.log
```

```
Starting web service on port 8080
User alice logged in
Retrying database connection
Service healthy
```

A common use is stripping comment lines from a config file. Comments usually start with `#`, so this leaves only the settings that apply:

```
grep -v '^#' /etc/hosts
```

## Combining flags

Flags stack. You saw `-i` and `-n` together as `-in`, and `-i` with `-c` as `-ic`. Order doesn't matter, `grep -in`, `grep -ni`, and `grep -i -n` all do the same thing.

# grep across many files

Point grep at a folder with `-r` (recursive) and it walks every file underneath, searching each one.

```
grep -r "listen" /etc/nginx/
```

Every match prints as `filename:line`. That "which file mentioned this?" answer is why DevOps engineers reach for `grep -r` constantly.

## Narrowing the search

Two flags pay for themselves:

- `-l` prints only the filenames that had at least one match.
- `--include='*.conf'` restricts the search to files matching a glob.

```
grep -r -l "PermitRootLogin" /etc/           # which files mention this setting
grep -r --include='*.conf' "listen" /etc/    # only .conf files under /etc
```

## Piping into grep

grep isn't only for files. It filters any stream of text, which pairs perfectly with the commands you already know:

```
ps aux | grep nginx                 # any nginx processes running?
history | grep ssh                  # ssh commands from your history (empty on a brand-new shell)
ls /etc | grep -i network           # entries in /etc whose name mentions "network"
```

The pattern is always: something produces lines, `grep` keeps the ones that matter.

# find for locating files

`grep` searches inside files. `find` locates the files themselves.

## Basic shape

```
find <path> [conditions]
```

The conditions you'll use most often:

- `-name 'pattern'` - filename (case-sensitive). Wildcards work but you must quote them.
- `-iname 'pattern'` - filename (case-insensitive).
- `-type f` - regular files only.
- `-type d` - directories only.

## Everyday recipes

```
find /etc -name 'nginx.conf'             # a specific file under /etc
find /etc -name '*.conf'                 # every .conf file under /etc
find / -type d -name 'nginx'             # a directory anywhere on the machine
find /var/log -type f -name '*.log'      # regular log files under /var/log
```

## Always quote the pattern

If you leave the glob unquoted, the shell expands it before `find` sees it. You'll either miss matches or get an error.

```
find /etc -name '*.conf'      # correct
find /etc -name *.conf        # wrong, shell expanded *.conf first
```

## When to reach for find vs grep

- "Where is a file called X?" - `find`.
- "Which files contain the string X?" - `grep -r`.

You'll sometimes want both: `find` locates the candidates, `grep` searches inside them. That combination is worth adding to your muscle memory once each command feels natural on its own.

