# for loops

Say you have five servers to ping, or a folder of `.log` files to compress. You do not want to write the same command five times. A `for` loop lets you describe the work once and run it for each item in a list, one after another.

## The basic shape

```
for user in alice bob charlie; do
  echo "hello, $user"
done
```

This prints:

```
hello, alice
hello, bob
hello, charlie
```

`for user in alice bob charlie` takes each word after `in`, one at a time, and stores it in the variable `user`. `do` starts the body that repeats, `done` ends it. On each pass `user` holds the next word; when the list runs out, the loop stops.

- `user` is just a name you picked. Read it back with `$user`.
- Words after `in` are split on spaces, so this is three items.
- The `;` before `do` lets them share a line. Put `do` on its own line and you can drop the `;`.

## Looping over files with a glob

A glob like `*.log` expands into all matching filenames before the command runs, which pairs perfectly with `for`. Seed a couple of files first:

```
mkdir -p /tmp/logs
echo "first" > /tmp/logs/app.log
echo "second" > /tmp/logs/db.log

for file in /tmp/logs/*.log; do
  echo "found: $file"
done
```

This prints:

```
found: /tmp/logs/app.log
found: /tmp/logs/db.log
```

The shell hands you each filename as a single item, so there are no word-splitting headaches even if a name contained a space.

## Looping over a script's arguments

`"$@"` holds all the arguments passed to a script. Loop over it to handle each one:

```
for arg in "$@"; do
  echo "processing $arg"
done
```

Run as `./myscript.sh one two three`, this prints `processing one`, `processing two`, `processing three`. Always keep the double quotes: quoted, `"$@"` keeps each argument separate even when it contains spaces.

## Counting with a range

When you just want to repeat something a fixed number of times, brace expansion gives you a numeric range:

```
for i in {1..5}; do
  echo "step $i"
done
```

This prints `step 1` through `step 5`. The shell turns `{1..5}` into `1 2 3 4 5` before the loop runs. Letters work too: `{a..e}` becomes `a b c d e`.

## The C-style loop

 also offers a counting form like the loops in C or Java, with a start value, a condition, and a step inside `(( ))`:

```
for (( i=0; i<5; i++ )); do
  echo "step $i"
done
```

This prints `step 0` through `step 4`. It starts at `i=0`, checks `i<5` before each pass, then does `i++`. Inside `(( ))` you write plain math: no `$` in front of variables. Reach for this when you need a start other than 1, want to count in steps, or compute the range from another variable.

## Looping over command output, with care

You can loop over a command's output with `$(...)`, but with a catch:

```
for word in $(ls /etc); do
  echo "entry: $word"
done
```

The shell splits that output on all whitespace, so any name containing a space would be broken into two items. For lines of a file or command, use `while read` (next ), which keeps each line whole. Use `for` for fixed lists, globs, arguments, and ranges.



# while, until, break, and continue

A `for` loop is perfect when you already know the list. But sometimes you just want to keep going as long as a condition holds and stop the moment it changes. That is what `while` and `until` are for:

- `while` runs its body as long as a command keeps succeeding.
- `until` runs its body as long as a command keeps failing, and stops once it finally succeeds.

Both build on exit status: `0` means success, anything else failure.

## while

A `while` loop checks a condition, runs the body if it is true, then checks again, stopping once it is false:

```
count=1
while (( count <= 5 )); do
  echo "step $count"
  (( count++ ))
done
```

This prints `step 1` through `step 5`. `(( count <= 5 ))` is  arithmetic asking "is `count` at most 5?" Without the `(( count++ ))` line, `count` would stay at 1 forever. That is the golden rule of `while`: something in the body must eventually make the condition false, or the loop never ends.

## until

`until` is the opposite: it runs while the condition is false and stops once it becomes true, which is handy for waiting on something. This version seeds a file after a moment so the loop finishes:

```
( sleep 4; touch /tmp/ready ) &

until [ -f /tmp/ready ]; do
  echo "waiting for /tmp/ready to appear..."
  sleep 1
done
echo "/tmp/ready is here"
```

The background job creates the file after four seconds. Meanwhile the loop keeps checking `[ -f /tmp/ready ]` ("does this file exist?"), printing and sleeping while it is missing. Once it appears, the test succeeds and the loop stops with `/tmp/ready is here`.

## Infinite loops on purpose

Sometimes you want a loop that never ends on its own, like a script that polls for work every minute:

```
while true; do
  echo "checking for work..."
  sleep 60
done
```

`true` always succeeds, so the condition is always met. Stop it with Ctrl+C in your terminal, or with `break` from inside a script.

## continue: skip to the next pass

`continue` abandons the rest of the current pass and jumps to the next item without leaving the loop. Here we skip files we can't read:

```
mkdir -p /tmp/logs
echo "hello" > /tmp/logs/a.log
echo "world" > /tmp/logs/b.log
chmod 000 /tmp/logs/b.log

for file in /tmp/logs/*.log; do
  if [ ! -r "$file" ]; then
    continue
  fi
  echo "reading: $file"
done
```

This prints:

```
reading: /tmp/logs/a.log
```

`[ ! -r "$file" ]` asks "is this file NOT readable?" That is true for the locked `b.log`, so `continue` fires and it is skipped.

## break: leave the loop entirely

`break` stops the loop completely and continues after `done`:

```
for i in {1..10}; do
  if (( i > 3 )); then
    break
  fi
  echo "number $i"
done
echo "done looping"
```

This prints:

```
number 1
number 2
number 3
done looping
```

As soon as `i` passes 3, `break` bails out of the whole loop. Both `break` and `continue` work the same way inside `for`, `while`, and `until`.

## Nested loops: break N and continue N

Inside nested loops, a plain `break` leaves only the inner loop. Add a number to jump out of more levels:

```
break 2      # break out of two levels of loop
continue 2   # continue at the next pass of the outer loop
```

You won't need these often, but they are exactly right when a pair of nested loops must bail out together.



# Reading a file line by line

A common job in scripts is to do something with each line of a file: read a list of servers and ping each, read a config and act on each entry. A `for` loop won't do, because it splits on every space, not just line breaks, so any line with a space gets torn apart. There is a dedicated pattern for this, and getting it exactly right matters.

Seed a small demo file to use throughout:

```
printf 'first line\nsecond line\nthird line\n' > /tmp/demo.txt
```

## The idiom

```
while IFS= read -r line; do
  echo "line: $line"
done < /tmp/demo.txt
```

This prints:

```
line: first line
line: second line
line: third line
```

`read` pulls one line into a variable. The `while` loop keeps pulling until the file ends, where `read` reports failure and stops the loop. Each line stays whole, spaces and all. The three pieces each earn their place:

- **`< /tmp/demo.txt`** redirects the file into the loop's input, so `read` gets its lines from the file instead of from you typing.
- **`-r`** stops `read` from eating backslashes, which would mangle lines with a backslash (like a Windows path). Use it on every read.
- **`IFS=`** turns off `read`'s default trimming of leading and trailing spaces, just for this command, so lines arrive untouched.

`while IFS= read -r line` is a fixed phrase worth memorizing. It is the correct, reliable way to read lines.

## Why not `for line in $(cat file)`

Beginners often write this, and it looks reasonable:

```
for line in $(cat /tmp/demo.txt); do   # DON'T do this
  echo "line: $line"
done
```

But it prints `line: first`, `line: line`, `line: second`, and so on, one word per line. The shell splits that output on every space, not on newlines, so `first line` became two items. The `while read` version streams one whole line at a time. Prefer it every time.

## Splitting each line into fields

Give `read` several variable names and it splits the line into them using `IFS` as the separator. `/etc/passwd` is a classic example, its lines made of colon-separated fields:

```
while IFS=: read -r username _ uid _ desc home shell; do
  echo "user=$username uid=$uid shell=$shell"
done < /etc/passwd
```

A few lines of output:

```
user=root uid=0 shell=/bin/
user=daemon uid=1 shell=/usr/sbin/nologin
user=bin uid=2 shell=/usr/sbin/nologin
```

`IFS=:` splits on colons. Each line has seven fields, so we give seven names. The `_` is an ordinary variable used by convention as a throwaway for fields we don't care about.

## Reading from a command instead of a file

You can pipe a command's output into the loop:

```
ls /etc | while IFS= read -r line; do
  echo "entry: $line"
done
```

This works, but piping into a loop runs it in a subshell (a child copy of your shell), so any variable set inside vanishes when the loop ends:

```
count=0
ls /etc | while IFS= read -r line; do
  (( count++ ))
done
echo "count is $count"   # prints: count is 0
```

If you need values to survive the loop, use process substitution instead of a pipe:

```
count=0
while IFS= read -r line; do
  (( count++ ))
done < <(ls /etc)
echo "count is $count"   # prints the real number of entries
```

`<(ls /etc)` produces the command's output and the leading `<` redirects it into the loop, just like a file, keeping the loop in your current shell so `count` keeps its value.

