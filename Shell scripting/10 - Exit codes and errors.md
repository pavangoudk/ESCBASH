# Exit codes

Every command leaves behind a single number when it finishes: its exit code. It answers one question, did I succeed or fail, and it is the foundation of every decision a script makes.

## The one rule: zero is success

This feels backwards but you must memorize it:

- `0` means success.
- Any nonzero value (`1`, `2`, `127`, ...) means failure.

There is only one way to succeed but many ways to fail, so failures get all the other numbers to describe what went wrong.

## Reading the exit code with `$?`

The shell stores the last command's exit code in the special variable `$?`. Expand it like any variable:

```
ls /etc
echo $?
```

```
(the listing of /etc prints here)
0
```

Now a directory that does not exist:

```
ls /nope
echo $?
```

```
ls: cannot access '/nope': No such file or directory
2
```

`ls` printed its error, then set `$?` to `2` to signal failure.

## `$?` changes after every command

`$?` only holds the result of the *most recent* command, and `echo` is a command too:

```
ls /nope
echo $?
echo $?
```

```
ls: cannot access '/nope': No such file or directory
2
0
```

The first `echo` printed `2` from `ls`, but that `echo` itself succeeded, so the second `echo` sees `0`. To keep a code, capture it immediately into an ordinary variable:

```
ls /nope
code=$?
echo "some other work here"
echo "the ls command exited with code $code"
```

```
ls: cannot access '/nope': No such file or directory
some other work here
the ls command exited with code 2
```

## Setting your own exit code with `exit`

In your own script, `exit` ends it immediately and returns the number you give: `exit 0` for success, `exit 1` for failure. Try it in a subshell so you don't close your session:

```
(exit 0); echo $?
(exit 1); echo $?
```

```
0
1
```

`exit` with no number reuses the last command's code, but writing the number makes your intent obvious.

## Why it matters

Every tool that runs your script judges it purely by its exit code. A cron job, a systemd service, a CI pipeline, a monitoring probe, none read your output text. Exit `0` and they believe everything worked; exit nonzero and they stop, retry, or alarm. A script that hits an error but still exits `0` is dangerous: the tools report "success" while your backup silently produced an empty file.

## Codes worth recognizing

- `0` success
- `1` general failure, the catch-all
- `2` misuse, often bad arguments or a missing file
- `126` found but not executable
- `127` command not found (usually a typo)
- `130` killed by Ctrl+C
- `137` killed hard, often out of memory
- `143` asked to shut down (SIGTERM)

Do not memorize the table, but recognizing `127` as "command not found" points you straight at a typo instead of leaving you guessing.



# stderr and stdout

When a command runs, it produces two different kinds of output:

- **normal results**, the thing you actually asked for, and
- **error messages**, when something goes wrong.

 keeps these on two separate **channels**, one for each kind. You do not usually notice, because both channels point at your terminal, so everything shows up mixed together on the same screen.

You can see both at once with a command that half-succeeds. Here `ls` is asked to list a folder that exists (`/etc`) and one that does not (`/nope`):

```
ls /etc /nope
```

```
ls: cannot access '/nope': No such file or directory
hosts   passwd   ssh   ...
```

The first line is the error message. The rest is the normal result (the real listing of `/etc`). It looks like one blob of text, but those two lines came out of two different channels that happened to land on the same screen.

Why does this matter? Because once you know they are separate, you can send each channel somewhere different: keep the real results, throw away or save the errors, or the other way round. The rest of this  shows how. First, the names.

## The two streams

- **stdout** (standard output), number `1`: the real results, the thing you asked for. `ls /etc` sends its file list here.
- **stderr** (standard error), number `2`: errors, warnings, and diagnostics. `ls /nope` sends `cannot access '/nope'` here, not stdout.

Those numbers `1` and `2` are called file descriptors, and you use them to steer each stream independently.

## Redirecting each stream to a file

Plain `>` redirects only stdout. To redirect stderr, name its number with `2>`. This `ls` lists one real directory and one fake one:

```
ls /etc /nope > out.txt 2> err.txt
```

Nothing prints, because both streams went to files:

```
cat out.txt
```

```
(the real listing of /etc, which is stdout)
```

```
cat err.txt
```

```
ls: cannot access '/nope': No such file or directory
```

### Sending both to the same place

`2>&1` means "send stream 2 wherever stream 1 is currently going." Place it after your `>`:

```
ls /etc /nope > combined.txt 2>&1
cat combined.txt
```

```
ls: cannot access '/nope': No such file or directory
(the real listing of /etc)
```

 offers a shorter spelling of the same thing, `&>`:

```
ls /etc /nope &> combined.txt
```

## Why the distinction matters: pipes

A pipe (`|`) only carries stdout. Anything on stderr skips the pipe and leaks to your screen:

```
find /nope | grep cannot
```

```
find: '/nope': No such file or directory
```

The error flashed on screen but `grep` never saw it, so `grep` printed nothing. Merge stderr into stdout first so the pipe can carry it:

```
find /nope 2>&1 | grep cannot
```

```
find: '/nope': No such file or directory
```

Now `grep` received the error through the pipe and matched it.

## Writing to stderr from your own script

In a script you choose each message's stream. `>&2` sends an `echo` to stderr. The convention: real results on stdout, everything else (status, warnings, errors) on stderr.

```
echo "starting the backup" >&2
echo "backup-2026.tar.gz"
```

This lets a caller capture your real output without the chatter, since `$(...)` captures only stdout:

```
echo "the captured result is: $(echo "starting the backup" >&2; echo "backup-2026.tar.gz")"
```

```
starting the backup
the captured result is: backup-2026.tar.gz
```

The status message still showed on screen through stderr, but it did not pollute the captured value.

## The die pattern

Combining exit codes and stderr gives a classic helper named `die`: print a message to stderr, then exit `1`.

```
die() {
  echo "$*" >&2
  exit 1
}
command -v jq > /dev/null || die "jq is required but not installed"
echo "jq is present, carrying on"
```

On a fresh Ubuntu system `jq` is not installed, so `command -v jq` fails, the `||` fires, and `die` runs:

```
jq is required but not installed
```

The message went to stderr, the script exited `1`, and the final `echo` never ran: a clear error on the right channel and a failing exit code.



# Checking command success

You know every command leaves an exit code. Now make your script react to it. Reading `$?` by hand works, but  gives cleaner ways to say "if this worked, do that" or "if it failed, bail out." Use `if` when the reaction is big or multi-step, and the `||` and `&&` shortcuts when it is a single quick action.

## The `if` pattern

`if` can test a command directly, not just brackets. It runs the command and treats exit `0` as true, nonzero as false:

```
if ls /etc > /dev/null; then
  echo "the directory exists and we can read it"
fi
```

```
the directory exists and we can read it
```

We sent output to `/dev/null` because we only care whether it worked. The failing case uses `else`:

```
if grep -q root /etc/passwd; then
  echo "root user found"
else
  echo "root user not found"
fi
```

```
root user found
```

`grep -q` searches quietly and exits `0` on a match. `root` is in `/etc/passwd`, so the `then` branch ran. Use `if` when the reaction is more than one line.

## The `||` pattern: do this, or else

`||` (read "or") runs the right side only if the left side failed:

```
ls /nope || echo "could not list that directory"
```

```
ls: cannot access '/nope': No such file or directory
could not list that directory
```

If the left side succeeds, the right side is skipped:

```
ls /etc > /dev/null || echo "this message never appears"
```

```
(no output, because ls /etc succeeded)
```

This is perfect for "do this, or bail," especially with `die`: `mkdir /some/dir || die "could not create the directory"`.

### When the fallback needs several steps

Group multiple commands in curly braces. The inner spaces and the semicolon before the closing brace are required:

```
ls /nope || {
  echo "listing failed, cleaning up" >&2
  echo "removing any leftover temp file"
  exit 1
}
```

Everything inside runs as a unit when `ls` fails.

## The `&&` pattern: do this, and then

`&&` (read "and") is the mirror image: it runs the right side only if the left side succeeded:

```
grep -q root /etc/passwd && echo "confirmed: root exists, continuing"
```

```
confirmed: root exists, continuing
```

Use `&&` for "only do the next thing if the setup worked."

## Combining `&&` and `||`

You can chain both on one line:

```
grep -q root /etc/passwd && echo "found it" || echo "not found"
```

```
found it
```

It reads like if-then-else and is fine for a simple message, but there is a trap: the `|| echo "not found"` fires if *either* the `grep` fails *or* the first `echo` fails. For anything beyond a trivial one-liner, use a proper `if/then/else` block.

## Checking exactly which failure happened

Sometimes you need to know *which* failure occurred. Save `$?` and use a `case`. `grep` is a great example: `0` found a match, `1` no match, `2` a real error like an unreadable file.

```
grep zzz /etc/passwd
code=$?
case $code in
  0) echo "found a match" ;;
  1) echo "no match, but the search itself was fine" ;;
  2) echo "grep hit a real error" ;;
  *) echo "some other exit code: $code" ;;
esac
```

```
no match, but the search itself was fine
```

`zzz` is not in `/etc/passwd`, so `grep` exited `1`. This distinguishes "searched fine but found nothing" from "grep genuinely broke," a difference a plain success-or-fail check would flatten.

