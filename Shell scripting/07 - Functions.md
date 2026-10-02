# Defining and calling functions

A function is a name for a group of commands, so you can run them again just by saying the name. Write the steps once, give them a name, call it as many times as you like. Fix the steps in one spot and every caller gets the fix. That is the whole point: write once, use many times.

## Defining a function

Write a name, empty parentheses, then the commands in curly braces. The parentheses are always empty; they are just syntax, not a place for arguments:

```
greet() {
  echo "hello there"
}
```

This prints nothing yet.  just remembers there is a function called `greet`. The commands run later, when you call it.

 is picky about spaces around the braces: `{` needs a space or newline after it, and the last command needs a `;` or newline before `}`. The multi-line form above is easiest to get right. On one line you must add the separators yourself:

```
greet() { echo "hi" }    # WRONG: no ; before the }
greet() { echo "hi"; }   # correct: the ; ends the command
```

A missing `;` before `}` gives `syntax error: unexpected end of file`.

## Calling a function

Run it by writing its name, exactly like any normal command. No parentheses, no keyword:

```
greet() {
  echo "hello there"
}

greet          # prints: hello there
greet          # prints: hello there
```

## Passing arguments

List arguments after the name, separated by spaces, just like `cp` or `mkdir`. Inside the function you reach them through numbered variables: `$1` is the first, `$2` the second, and so on.

```
greet() {
  echo "hello, $1"
}

greet alice          # prints: hello, alice
greet bob            # prints: hello, bob
```

Wrap an argument with spaces in quotes so  keeps it as one piece; `greet "Bob Smith"` prints `hello, Bob Smith`, while without quotes `Bob Smith` splits into `$1` (`Bob`) and `$2` (`Smith`).

Two more special variables help when the count varies: `$#` is the number of arguments, and `"$@"` is the whole list, which you can loop over:

```
say_all() {
  echo "got $# args"
  for arg in "$@"; do
    echo "- $arg"
  done
}

say_all one two three
# prints:
# got 3 args
# - one
# - two
# - three
```

Always keep `"$@"` in double quotes so arguments with spaces stay intact. Note that inside a function, `$1`, `$#`, and `"$@"` refer to the function's own arguments, not the script's.

## Define before you call

 reads top to bottom and only knows a function after reading its definition. Calling it above the definition fails:

```
greet          # WRONG:  hasn't read the definition yet
greet() {
  echo "hello there"
}
```

That prints `greet: command not found`. Put the definition first:

```
greet() {
  echo "hello there"
}
greet          # prints: hello there
```

The usual habit is to define all functions at the top and put the main logic below, so every function is known by the time you use it.



# Return codes vs echoed output

In most languages `return` hands a value back to the caller.  is different, and this trips up nearly everyone: a function's `return` gives back only a small whole number meaning "did this succeed or fail." It cannot hand back a name or a sentence.

 functions have two separate channels, and keeping them straight is the whole :

- The **exit code** answers "did it work?" A number from 0 to 255, where 0 means success and anything else means failure. You set it with `return`. These are the same exit codes you saw with commands.
- The **output** is any text the function prints with `echo`. To give back an actual value like a filename or count, you print it and the caller catches what was printed.

## `return` is for the exit code only

`return` ends the function right away and sets its exit code:

```
is_ready() {
  if [ -f /tmp/app.pid ]; then
    return 0        # success: the file is there
  else
    return 1        # failure: the file is missing
  fi
}

touch /tmp/app.pid
if is_ready; then
  echo "app is ready"
fi
# prints: app is ready
```

Putting `if` on the call works because a `0` exit code counts as "true," so the body runs. A `1` would skip it.

### You often don't need `return`

If you never write `return`, a function hands back the exit code of its **last command**. So you can often drop `return` entirely:

```
is_ready() {
  [ -f /tmp/app.pid ]     # its exit code becomes the function's
}

rm -f /tmp/app.pid
if is_ready; then
  echo "ready"
else
  echo "not ready"
fi
# prints: not ready
```

The test failed (the file was deleted), so the function's exit code was `1` and the `else` branch ran. To inspect an exit code directly, `$?` holds the most recent one:

```
is_ready() { [ -f /tmp/nope ]; }

is_ready
echo "$?"    # prints: 1
```

## Getting a real value out: echo and capture

The exit code cannot carry text like a timestamp. For that, the function prints the value with `echo` and the caller grabs it with `$(...)`, the command substitution you have seen before:

```
timestamp() {
  date +%Y%m%d-%H%M%S
}

ts=$(timestamp)
echo "backup-$ts.tgz"
# prints something like: backup-20260728-142530.tgz
```

`$(timestamp)` runs the function, catches what it printed, and hands that text to `ts`. This "print it, then capture it" pattern is how  functions return real values. One warning: only printed output gets captured, so an extra `echo "starting..."` inside the function gets captured too and corrupts your value. When a function returns a value, print exactly that value and nothing else.

## Using both channels together

Print the answer on success, and set a failure exit code when there is no answer. The caller checks the code and grabs the value in one line:

```
find_user() {
  local name=$1
  local line
  line=$(grep "^${name}:" /etc/passwd) || return 1
  echo "$line"
}

if info=$(find_user root); then
  echo "found: $info"
else
  echo "no such user"
fi
# prints: found: root:x:0:0:root:/root:/bin/
```

`grep` found the line, so it was echoed into `info` and the exit code was success, running the `if`. For a missing name, `grep` fails, `|| return 1` fires, and the `else` branch runs. One call, both channels checked. (Capturing with `$(...)` holds all the output in memory, so for output that could be megabytes, have the function write to a file the caller reads instead.)



# Local variables

A surprise for newcomers: by default a variable created inside a function is **global**, shared with the whole script. If a function sets `count` and the script already had a `count`, the function quietly overwrites it, and the change sticks around after the function is done. The `local` keyword gives the function its own private copy instead.

## The problem: variables leak out

Here a script sets `status`, then a function uses the same name:

```
status=unknown

set_status() {
  status=ready        # this touches the SAME status
}

echo "before: $status"
set_status
echo "after:  $status"
# prints:
# before: unknown
# after:  ready
```

The function changed the script's `status` even though we probably wanted a throwaway value. Now imagine two functions that both use `i` or `tmp` or `count`: they trample each other's values, and those bugs are miserable to track down.

## The fix: declare it `local`

Put `local` in front of a variable and it becomes private, existing only while the function runs and vanishing when it returns:

```
count=0

count_lines() {
  local count            # a private count, separate from the outer one
  count=$(wc -l < "$1")
  echo "$count"
}

echo "outer before: $count"
count_lines /etc/passwd    # prints the line count of /etc/passwd
echo "outer after:  $count"
# prints:
# outer before: 0
# count_lines output above shows the file's line count
# outer after:  0
```

The outer `count` stayed `0`. Inside, `local count` shadowed it, so the function's work left the script's `count` untouched. Build the habit: unless you specifically mean to change an outer variable, declare function variables `local`.

## Declare each variable at the top

A clean style is to `local`-declare every variable near the top, then do the work below. A value on the `local` line is fine for simple cases:

```
describe_file() {
  local path=$1
  local line_count
  local size

  line_count=$(wc -l < "$path")
  size=$(stat -c '%s' "$path")

  echo "$path: $line_count lines, $size bytes"
}

describe_file /etc/hostname
# prints something like: /etc/hostname: 1 lines, 10 bytes
```

Declaring `path=$1` on one line is fine because `$1` is plain text. But mixing `local` with a captured command hides a trap.

## The `local` plus command-substitution trap

Combine `local` with `$(...)` on one line and the exit code you get back is `local`'s, not the command's. Since `local` almost always succeeds, a failure inside `$(...)` gets swallowed:

```
check() {
  local x=$(false)     # 'false' fails, but...
  echo "exit code was: $?"
}

check
# prints: exit code was: 0
```

`false` failed, yet `$?` reported `0`, because bash reported `local`'s success and threw away the failure. The fix: split the declaration and assignment onto two lines:

```
check() {
  local x
  x=$(false)           # now the failure is not hidden
  echo "exit code was: $?"
}

check
# prints: exit code was: 1
```

Now `$?` reports `1`, because the assignment line carries the command's real exit code. Small habit, declare `local` on one line and capture on the next, but it saves you from a nasty class of bugs.

