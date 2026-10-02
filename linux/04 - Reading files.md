# cat, head, tail, and wc

Four commands do most of the work when you need to look at a text file.

## cat: print the whole thing

`cat` prints a file's contents to the terminal.

```
cat /etc/hostname
cat /etc/os-release
```

Great for short files. For anything long, use one of the tools below, otherwise the terminal fills up faster than you can scroll.

## head and tail: peek at either end

`head` prints the first 10 lines by default. `tail` prints the last 10. Both take `-n` to change the count.

```
head /etc/os-release
head -n 3 /etc/os-release      # first 3 lines only
tail -n 5 /etc/passwd          # last 5 lines
```

These are perfect for log files where the interesting bit is usually near the top (startup) or bottom (most recent).

## wc: count lines, words, bytes

`wc` counts. Most of the time you'll use `-l` for line count.

```
wc -l /etc/passwd
```

Piped from another command, `wc -l` becomes "how many things did that produce":

```
ls /etc | wc -l           # how many entries live in /etc
```

You already used that trick in the previous topic.

---

# less for big files

`cat` prints everything at once, which is a problem when "everything" is a million lines of log. `less` opens the file in a scrollable viewer that only loads what you need.

```
less /etc/services
```

You're now inside the viewer, not back at the shell. The keys you'll actually use:

- **Space** or **f** - page down.
- **b** - page up.
- **g** - jump to the top.
- **G** (capital) - jump to the bottom.
- **/word** - search forward for `word`. Press **n** for the next match.
- **?word** - search backward.
- **q** - quit and return to the shell.

You can also jump straight to the bottom of a file when opening it:

```
less +G /var/log/dpkg.log
```

When you don't know whether a file is big, reach for `less` rather than `cat`. It'll open a 2 GB file instantly, whereas `cat` would spend minutes flooding your terminal.

---

# Following a log with tail -f

On any real server, watching a log update in real time is a daily activity. That's what `tail -f` is for.

This lab has a real nginx web server, and every request it serves lands in its access log. Nothing creates that log until nginx is running, and the server is still coming up in the first seconds of a session, so start it before you go looking. Without that, `tail` answers `No such file or directory`, which is the file telling you the truth: it is not there yet.

Start the server, give it a couple of requests so the log has lines, then follow it:

```
systemctl start nginx
curl -s localhost > /dev/null
curl -s localhost > /dev/null
tail -f /var/log/nginx/access.log
```

`systemctl start` on a server that is already running changes nothing, so it is always safe to run first.

`-f` (for "follow") keeps the command running and prints each new line the moment it's appended to the file. When something breaks in production, this is where the truth lives.

Two variations worth knowing:

```
# print the last 100 lines, then follow
tail -n 100 -f /var/log/nginx/access.log

# keep watching even if the file gets rotated
tail -F /var/log/nginx/access.log
```

`-n 100 -f` is useful when you want context before the live stream starts. `-F` (capital) keeps watching a file across log rotation, when the log gets renamed and a fresh one takes its place.

Press **Ctrl+C** to stop following and return to the shell.

On most servers the system-wide log is `/var/log/syslog` (or, on systemd machines, `journalctl -f`), and tailing it works the same way.

If you're ever debugging a service and someone says "just tail the log", this is what they mean.

---
