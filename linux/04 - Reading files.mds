# Reading Text Files in Linux — Notes

## 1. `cat`: print an entire file

`cat` displays the complete contents of a text file in the terminal.

```
cat /etc/hostname
cat /etc/os-release
```

Use `cat` for short files. For long files, the output may fill the terminal too quickly.

## 2. `head`: view the beginning

`head` displays the first 10 lines of a file by default.

```


head /etc/os-release
head -n 3 /etc/os-release
```

- `head`: shows the first 10 lines.
- `head -n 3`: shows the first 3 lines.

This is useful for checking file headers, startup information, or the beginning of a log.

## 3. `tail`: view the end

`tail` displays the last 10 lines of a file by default.

```
tail /etc/passwd
tail -n 5 /etc/passwd
```

- `tail`: shows the last 10 lines.
- `tail -n 5`: shows the last 5 lines.

The end of a log often contains the newest events, so `tail` is especially useful for log investigation.

## 4. `wc`: count content

`wc` counts lines, words, and bytes. The option used most often is `-l`, which counts lines.

```
wc -l /etc/passwd
```

You can combine `wc -l` with another command using a pipe:

```


ls /etc | wc -l
```

This means:

1. List the entries in `/etc`.
2. Send the output to `wc`.
3. Count the lines.

## 5. `less`: read large files safely

`less` opens a file in a scrollable viewer. It is better than `cat` for large files because it does not flood the terminal with all the content at once.

```
less /etc/services
```

### Useful `less` controls

| Key | Action |
| --- | --- |
| `Space` or `f` | Move one page down |
| `b` | Move one page up |
| `g` | Go to the beginning |
| `G` | Go to the end |
| `/word` | Search forward for `word` |
| `n` | Find the next match |
| `?word` | Search backward for `word` |
| `q` | Quit and return to the shell |

To open a file directly at its end:

```
less +G /var/log/dpkg.log
```

Use `less` whenever you are unsure how large a file is.

## 6. `tail -f`: follow a live log

`tail -f` keeps watching a file and prints new lines as they are added. This is useful for monitoring service logs in real time.

First start the web server and generate a few requests:

```
systemctl start nginx
curl -s localhost > /dev/null
curl -s localhost > /dev/null
tail -f /var/log/nginx/access.log
```

- `systemctl start nginx`: starts the Nginx service.
- `curl -s localhost`: sends a quiet request to the local web server.
- `tail -f`: continuously follows the access log.

Starting an already-running service normally changes nothing, so running the start command first is generally safe.

Press `Ctrl+C` to stop following the file.

## 7. Useful `tail` variations

### Show recent lines, then continue following

```
tail -n 100 -f /var/log/nginx/access.log
```

This displays the last 100 lines and then waits for new entries.

### Continue across log rotation

```
tail -F /var/log/nginx/access.log
```

- `-f`: follows the current file.
- `-F`: follows the file even when it is renamed and replaced during log rotation.

## 8. Common system logs

Many systems use:

```
tail -f /var/log/syslog
```

On systems that use `systemd`, live system logs are often viewed with:

```
journalctl -f
```

## 9. Command selection guide

| Goal | Command |
| --- | --- |
| Print a short file completely | `cat file` |
| View the first lines | `head file` |
| View the last lines | `tail file` |
| Count lines | `wc -l file` |
| Read a large file interactively | `less file` |
| Watch a log update | `tail -f file` |
| Watch a rotated log | `tail -F file` |

## Key takeaway

Use `cat` for short files, `head` and `tail` to inspect either end, `wc -l` to count lines, and `less` for large files. Use `tail -f` when troubleshooting a service or watching a log in real time.
