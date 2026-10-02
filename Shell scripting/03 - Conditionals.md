# if and exit codes

When you write scripts for real work, you constantly hit decisions: if the disk is more than 80% full, send an alert, otherwise carry on; if the config file exists, use it, otherwise create a default one. `if` is how you write that decision in a script.

## The shape of an if block

Here is the smallest `if`:

```
if grep -q root /etc/passwd; then
  echo "there is a root user"
fi
```

Read it out loud: "if this command works, then run the line inside." Three things to notice:

- It starts with `if` and ends with `fi` (that is `if` spelled backwards, and it just means "end of the if").
- The word `then` comes before the commands you want to run.
- The command right after `if` is a real command. Here it is `grep`, searching `/etc/passwd` for the word `root`. The `-q` flag just tells `grep` to stay quiet and not print what it finds.

## What if actually checks

This is the part that surprises people, so read it slowly.

`if` does not check whether something is "true" or "false". It runs the command you gave it and looks at one thing: did that command **succeed or fail**?

You saw this back in the exit-codes topic: every command ends with a number. `0` means it succeeded, any other number means it failed. `if` runs the command, and:

- if the command succeeded (exit code `0`), it runs the `then` part,
- if the command failed (any other code), it skips the `then` part.

`grep` succeeds when it finds a match and fails when it does not. So the block above prints its message only when `root` is actually in the file.

## Adding else and elif

To do something when the command fails, add an `else`:

```
if grep -q root /etc/passwd; then
  echo "found a root user"
else
  echo "no root user"
fi
# prints: found a root user
```

And to check a second command when the first one fails, add `elif` ("else if") in the middle:

```
if grep -q alice /etc/passwd; then
  echo "alice is here"
elif grep -q root /etc/passwd; then
  echo "no alice, but root is here"
else
  echo "neither one"
fi
```

`else` and `elif` are optional. Plenty of scripts use a plain `if ... then ... fi` with nothing after it.

Right now the only condition you can check is "did a command succeed?". The next  shows how to check values instead, like whether two words are equal or whether a file exists.



# The test command and [

Last , `if` ran a command like `grep` and checked whether it succeeded. But most of the time you do not want to run a program, you want to compare values: is this number bigger than that one, are these two words the same, does this file exist?

There is a command built for exactly that, and its name is `test`.

## test checks a condition and reports success or fail

`test` takes a condition, checks it, and (just like every command) ends with an exit code: `0` if the condition holds, `1` if it does not. It prints nothing.

```
test 5 -gt 3        # is 5 greater than 3?
echo $?             # prints: 0  (yes, so it succeeded)

test 5 -gt 30       # is 5 greater than 30?
echo $?             # prints: 1  (no, so it failed)
```

`-gt` means "greater than" (you will meet the full set of these in the next ). For now just notice the pattern: `test` turns a comparison into a success-or-fail exit code.

And success-or-fail is exactly what `if` reads. So you drop `test` straight after `if`:

```
if test "$USER" = "root"; then
  echo "hi root"
fi
```

Here `=` checks whether two strings are equal. If the value of `$USER` is `root`, `test` succeeds and the message prints.

## The [ shorthand

Writing `test` in full gets tiring, so  gives it a second name: a single square bracket, `[`. It is the very same command, just spelled differently. When you use `[`, you finish the condition with a closing `]`:

```
if [ "$USER" = "root" ]; then
  echo "hi root"
fi
# prints: hi root  (when you are logged in as root)
```

Read `[ ... ]` as "test whether ...". These two lines do exactly the same thing:

```
if test "$USER" = "root"; then ...
if [ "$USER" = "root" ]; then ...
```

You will see `[` far more often than `test` in real scripts, so the rest of these s use it.

## Two rules that trip up every beginner

Because `[` is really a command, it is fussy about spaces:

- **Put a space around everything.** A space after `[`, a space before `]`, and a space around the `=`. `["$USER"="root"]` fails because  cannot tell where the command and its arguments begin and end.
- **Never forget the closing `]`.** Leave it off and you get `[: missing ']'`.

So the safe habit is: `[` space, condition with spaces, space `]`.



# Comparing strings, numbers, and files

Now that you can write `[ ... ]`, here are the conditions you will actually put inside it. They come in three groups: comparing text, comparing numbers, and checking files. Skim them now and come back when you need a specific one.

## Comparing strings (text)

- `=` equal, `!=` not equal
- `-z` the string is empty, `-n` the string is not empty

```
answer=""
if [ -z "$answer" ]; then
  echo "you didn't answer"
fi
# prints: you didn't answer
```

## Comparing numbers

Numbers do **not** use `=`, `<`, or `>`. They use two-letter codes:

- `-eq` equal, `-ne` not equal
- `-lt` less than, `-le` less than or equal
- `-gt` greater than, `-ge` greater than or equal

```
age=20
if [ "$age" -ge 18 ]; then
  echo "adult"
fi
# prints: adult
```

Why not just `>` for numbers? Because `>` already means "send output to a file". Writing `[ 5 > 3 ]` would not compare anything, it would quietly create a file named `3`. The two-letter codes avoid that trap.

## Checking files

- `-e` the file exists
- `-f` it is a regular file, `-d` it is a directory
- `-r` readable, `-w` writable, `-x` executable

```
if [ -f /etc/hostname ]; then
  echo "the hostname file is here"
fi
# prints: the hostname file is here
```

## Always quote your variables

You may have noticed every example wraps its variable in double quotes, like `"$age"` and `"$answer"`. That is not for looks, it prevents a crash.

Remember that  replaces `$name` with its value *before* `[` runs. If the value is empty and you did not quote it, the value simply disappears:

```
name=""
if [ $name = "alice" ]; then    # WRONG: no quotes
  echo "hi alice"
fi
# prints: : [: =: unary operator expected
```

After  swaps in the empty value, `[` sees `[ = "alice" ]`, with nothing on the left of `=`, and errors out. Quote the variable and the empty value stays put as an empty string:

```
name=""
if [ "$name" = "alice" ]; then    # correct
  echo "hi alice"
fi
# prints nothing, and no error
```

The rule is simple: **always double-quote variables inside `[ ]`.** The next  shows a newer form, `[[ ]]`, that removes this class of bug for good.



# The [[ ]] alternative

`[ ]` works but has sharp edges: forget to quote a variable and your script crashes.  ships a newer version, `[[ ]]` (two brackets), that does everything `[ ]` does, avoids most crashes, and adds a few tricks. In a  script today, this is the one to reach for.

## It looks almost the same

Swap single brackets for double and you are basically done:

```
if [[ "$USER" == "root" ]]; then
  echo "hi root"
fi
# prints: hi root
```

Same spacing, same `if ... then ... fi` shape. `==` is the usual string-equality operator here (a single `=` works too). `[[ ]]` behaves better because it is a  keyword, not a plain command, so  can handle it more carefully.

## Why it is safer: empty variables

Recall the crash from last  - an empty variable breaks `[ ]`. `[[ ]]` handles it fine:

```
name=""
[ $name = "alice" ]      # prints: : [: =: unary operator expected
[[ $name == "alice" ]]   # no error, quietly false
```

Inside `[[ ]]`, an empty variable stays as one empty piece instead of vanishing, so the comparison is still well formed. Quoting is still good style, but a forgotten quote is far less likely to crash.

## It can match patterns

On the right of `==` you can use a glob pattern instead of an exact string, the same `*` and `?` wildcards you know from filenames:

```
filename="app.log"
if [[ "$filename" == *.log ]]; then
  echo "it's a log file"
fi
# prints: it's a log file
```

Leave the right side unquoted: quoting `*.log` would turn it back into a literal string.

For heavier matching, `=~` runs a regular expression:

```
version="v1.2"
if [[ "$version" =~ ^v[0-9]+\.[0-9]+ ]]; then
  echo "looks like a version tag"
fi
# prints: looks like a version tag
```

That regex means "a `v`, then digits, a dot, then digits". Regex is a big topic; treat this as a taste of what `=~` can do.

## It can combine conditions

Join conditions with `&&` (and) and `||` (or):

```
file="/etc/hostname"
if [[ -f "$file" && -r "$file" ]]; then
  echo "the file exists and I can read it"
fi
# prints: the file exists and I can read it
```

The old `[ ]` needed awkward `-a`/`-o` flags with surprising edge cases; `[[ ]]` uses the `&&` and `||` you already know.

## When to still use [ ]

Portability. `[[ ]]` is a  feature. Minimal systems run scripts under a plainer shell like `/bin/sh` (often `dash`), where `[[ ]]` errors out:

- If your script starts with `#!/bin/`, use `[[ ]]`.
- If it must run under `#!/bin/sh`, stick to `[ ]`.

Almost everything here begins with `#!/bin/`, so `[[ ]]` is your default.

## A note on numbers: (( ))

For numbers, `(( ... ))` reads the most naturally. Use ordinary math symbols and skip the `$` on variable names:

```
count=15
if (( count > 10 )); then
  echo "more than ten"
fi
# prints: more than ten
```

Compare `[ "$count" -gt 10 ]`: `(( count > 10 ))` is shorter and looks like school math. Use `(( ))` for pure numbers, `[[ ]]` for the rest.



# Short-circuit and case

# Short-circuit && and ||, and case

This  adds two tools scripts lean on constantly: a way to make a quick decision on one line without a full `if` block, and `case`, the clean way to check one value against many possibilities.

## Chaining commands with && and ||

Inside `[[ ]]`, `&&` and `||` combined conditions. Between two commands they decide whether the second command runs at all, based on whether the first succeeded (exit code `0`):

```
command_one && command_two    # run two ONLY if one succeeded
command_one || command_two    # run two ONLY if one failed
```

A memory hook: `&&` is "and then", `||` is "or else".

Use `&&` when the second step only makes sense if the first worked:

```
mkdir -p /tmp/backups && cp /etc/hostname /tmp/backups/
# prints nothing; both succeed, so the copy happens
```

If `mkdir` had failed,  would skip the `cp`. Use `||` to react when a command fails:

```
grep -q ERROR /etc/hostname || echo "no errors in that file"
# prints: no errors in that file
```

`/etc/hostname` has no `ERROR`, so `grep` fails and the `||` fires.

### Running a whole block with { ... }

To do more than one thing on failure, wrap commands in `{ }` so they run as a group:

```
command -v jq >/dev/null || { echo "jq is not installed"; exit 1; }
# prints: jq is not installed   (jq isn't on this machine)
```

When `command -v jq` fails, the block prints a message then `exit 1`. Mind the syntax: a space after `{`, and a semicolon (or newline) after every command including the last. Miss those and  errors.

## The quick exit trick

A very common one-liner combines `||` with `exit`:

```
cd /tmp || exit 1
# cd succeeds here, so the script keeps going
```

"Change into the directory, or else exit." A compact safety net: if the `cd` fails, you stop instead of running the rest in the wrong place.

## case for many possibilities

When you check one value against a long list, stacked `if`/`elif` gets repetitive. `case` takes one value and runs the first pattern that matches:

```
env="prod"

case "$env" in
  prod|production)
    echo "using production settings"
    ;;
  staging)
    echo "using staging settings"
    ;;
  *)
    echo "using development settings (default)"
    ;;
esac
# prints: using production settings
```

The pieces:

- **Each branch** is `pattern)`, then commands, then `;;` to end it. The `;;` is required; forget it and  reads into the next branch and errors.
- **`|`** lets one branch match several values: `prod|production)` fires for either.
- **`*)`** is a shell glob matching anything. Put it last as a catch-all default, the `case` version of a final `else`. Good practice to include one.
- The block opens with `case "$value" in` and closes with `esac` (`case` backwards, like `fi` closes `if`).

The moment you are choosing between three or more options on one value, `case` reads cleaner than a long `if`/`elif` chain.

