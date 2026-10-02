#  -x and set -x

When a script misbehaves, the hard part is you cannot see what the shell is actually doing. Tracing is the window: turn it on and  prints every command, with its variables already filled in, right before it runs. Instead of guessing what `$user` held, you see the real value. It is the single most useful debugging habit in shell, and it is built into , no extra tools needed.

## Running a whole script with tracing

Run the script with ` -x` instead of plain `` (`-x` stands for "execution trace"). Make a small script to see it for real:

```
cat > /root/greet.sh <<'EOF'
#!/bin/
name=alice
greeting="hello, $name"
echo "$greeting"
EOF
```

Now run it with tracing turned on:

```
 -x /root/greet.sh
```

Output:

```
+ name=alice
+ greeting='hello, alice'
+ echo 'hello, alice'
hello, alice
```

Lines starting with `+` are the trace: each command an instant before it ran. The line without a `+` (`hello, alice`) is the script's real output. Notice `greeting='hello, alice'` shows the value already substituted, so you are seeing what the shell saw *after* expanding the variables. If `$name` were empty, you would spot it here immediately.

## set -x inside the script

For a long script where you only care about one section, switch tracing on and off from *inside* using `set -x` (on) and `set +x` (off). The sign flip feels backwards but is the convention: `-x` turns an option **on**, `+x` turns it **off**.

```
cat > /root/report.sh <<'EOF'
#!/bin/
echo "setting up..."

set -x                 # trace ON from here
total=$((3 + 4))
echo "total is $total"
set +x                 # trace OFF from here

echo "all done"
EOF
```

Run it with plain ``:

```
 /root/report.sh
```

We ran it with plain ``; the tracing comes from inside the script. Output:

```
setting up...
+ total=7
+ echo 'total is 7'
total is 7
+ set +x
all done
```

Only the middle section got traced. The lines outside the `set -x` / `set +x` block ran silent, so you zoom in on exactly the part you suspect.

## Combining with -e (stop on first error)

The `-e` flag stops the script the moment any command fails. Paired with `-x` it becomes a precise bug finder: the trace shows every command, and `-e` halts at the one that broke, so the last `+` line is your culprit. Combine them as ` -xe`:

```
cat > /root/steps.sh <<'EOF'
#!/bin/
echo "step one"
cp /does/not/exist /tmp/    # this command fails
echo "step two"
EOF
```

Run it with `-xe`:

```
 -xe /root/steps.sh
```

Output:

```
+ echo 'step one'
step one
+ cp /does/not/exist /tmp/
cp: cannot stat '/does/not/exist': No such file or directory
```

`step two` never printed: `-e` stopped the script at the failed `cp`, and the trace makes it obvious that line was the last to run.

## Reading the trace output

- **A single `+`** is a top-level command, about to run at the main level of your script. This is what you see most of the time.
- **More `+` signs mean deeper nesting.** A line starting with `++` or `+++` runs one or two levels deep inside a subshell or command substitution. Read the count as "how nested am I right now."
- **Lines with no `+`** are the script's genuine output.
- **An empty value is the classic clue.** A trace line like `+ user=` with nothing after the `=` means that variable you were *sure* held a value is actually blank. A huge share of "why doesn't this work?" bugs come down to exactly that, and the trace shows it instantly.



# 
echo debugging and PS4

Full ` -x` tracing shows *everything*, and sometimes that is too much. When you just want to peek at one thing at one moment, plain `echo` debugging fits: drop a print at the spot, run, read the value, move on.

## The classic echo, sent to stderr

```
cat > /root/demo.sh <<'EOF'
#!/bin/
user=alice
echo "DEBUG: user=$user" >&2
echo "welcome, $user"
EOF
```

Run it:

```
 /root/demo.sh
```

Output:

```
DEBUG: user=alice
welcome, alice
```

The important detail is `>&2`: it sends the message to **stderr** instead of **stdout**. A real script's stdout often gets captured or piped onward, and debug prints there would corrupt that data. Stderr keeps them on your screen but out of the output. Prove it by redirecting stdout:

```
 /root/demo.sh > /tmp/output.txt
```

Only `DEBUG: user=alice` prints to your screen: `welcome, alice` went into the file (stdout), the `DEBUG:` line stayed on the terminal (stderr).

## Making debug output switchable

Rather than adding and deleting `echo` lines each time, write them once and control them with a switch: a helper function plus a variable.

```
cat > /root/deploy.sh <<'EOF'
#!/bin/
DEBUG=${DEBUG:-0}

debug() {
  (( DEBUG )) && echo "DEBUG: $*" >&2
}

user=alice
debug "starting up, user=$user"
echo "deploying for $user"
EOF
```

`DEBUG=${DEBUG:-0}` defaults `DEBUG` to `0` (off) if unset, and `debug` prints only when `(( DEBUG ))` is nonzero. Run it normally, then flipped:

```
 /root/deploy.sh
```

```
deploying for alice
```

```
DEBUG=1  /root/deploy.sh
```

```
DEBUG: starting up, user=alice
deploying for alice
```

The prints live in the code permanently and only speak up when you set `DEBUG=1`, so you never edit the script to toggle debugging.

## PS4: making the full trace readable

Every ` -x` trace line starts with `+`, which doesn't tell you *where* in the script you are. The prefix is the `PS4` variable, and you can redesign it to print useful context. Here is one with the line number:

```
cat > /root/trace.sh <<'EOF'
#!/bin/
user=alice
greeting="hi $user"
echo "$greeting"
EOF
```

Set the custom `PS4`, then run it with tracing on:

```
export PS4='+ ${LINENO}: '
 -x /root/trace.sh
```

Output:

```
+ 2: user=alice
+ 3: greeting='hi alice'
+ 4: echo 'hi alice'
hi alice
```

`LINENO` is a built-in holding the current line number, so every trace line now points at exactly where it came from.

For scripts that source other files (where two files can share a line number), add the file name and function name too:

```
export PS4='+ ${_SOURCE}:${LINENO}: ${FUNCNAME[0]:-main}(): '
```

- `${_SOURCE}`: the script file the current line came from.
- `${LINENO}`: the line number.
- `${FUNCNAME[0]:-main}`: the running function's name, or `main` if none.

A trace line then reads `+ /root/trace.sh:4: main(): echo 'hi alice'`: gold for a big script spread across files.

## Dumping the whole variable state

When you want the whole picture at once, `declare -p` prints variables with their exact stored values:

```
cat > /root/state.sh <<'EOF'
#!/bin/
user=alice
count=5
declare -p user count
EOF
```

Run it:

```
 /root/state.sh
```

Output:

```
declare -- user="alice"
declare -- count="5"
```

Naming variables shows just those; with no names it dumps every shell variable. The definitive answer to "is this really what I think it is?"



# 
ShellCheck

Everything so far has found bugs *after* you run a script. ShellCheck catches them *before*: it is a spell-checker for shell scripts. As a "static analysis" tool it studies your code as text without running it, and it knows the hundreds of traps that bite shell scripters (the unquoted variable that breaks on a filename with a space, the comparison written the wrong way, the typo that silently does nothing), flagging each with an explanation and a suggested fix.

## Installing ShellCheck

ShellCheck is not preinstalled on Ubuntu. This command downloads it from Ubuntu's servers, so it needs an internet connection:

```
sudo apt install -y shellcheck
```

With no internet access this step fails, there is no way around the download. On a connected machine it finishes in a few seconds.

## Running it on a script

Write a small buggy script on purpose. This one copies a file using an unquoted variable, one of the most common shell bugs there is:

```
cat > /root/backup.sh <<'EOF'
#!/bin/
source=/etc/hostname
cp $source /var/backups/
EOF
```

Now run ShellCheck on it:

```
shellcheck /root/backup.sh
```

Output:

```
In /root/backup.sh line 3:
cp $source /var/backups/
   ^-----^ SC2086: Double quote to prevent globbing and word splitting.

Did you mean:
cp "$source" /var/backups/
```

Every finding follows this same shape:

- **Location and offending code:** the file, the line number (`line 3`), the line itself, with `^-----^` pointing at the exact problem.
- **The SC code:** an identifier like `SC2086`. Search it online for a page explaining the rule in depth. `SC2086` (unquoted variables) is the one you'll meet most.
- **The suggested fix:** the `Did you mean:` section shows corrected code, here adding the quotes: `cp "$source" /var/backups/`.

## What ShellCheck catches

That unquoted-variable warning is one of hundreds of rules. The common ones:

- **Unquoted variables:** `$var` where you meant `"$var"`. Without quotes, a value with a space or wildcard splits into multiple words or expands to filenames. The single most frequent shell bug.
- **Comparison and test mistakes:** mixing up `[ ]` and `[[ ]]`, or the wrong operator inside them. These often pass your quick test and fail on real input.
- **Useless patterns:** `cat file | grep foo` when `grep foo file` would do. Harmless, but it nudges you to the simpler form.
- **Typos and misused options:** small slips that make a command silently do the wrong thing, plus non-portable syntax.

## Ignoring a warning on purpose

When ShellCheck flags something you did deliberately, silence that one rule with a comment directly above the line:

```
# shellcheck disable=SC2086
echo $var    # word-splitting is intended here
```

The comment skips the rule for the next line only. Use it sparingly: every silenced warning is a promise you understand why the "bug" is correct here. When in doubt, fix the code instead.

## Fitting it into your workflow

- Run it before you consider a script finished, like spell-checking a document before sending.
- Add it to CI so any new warning fails the build before it merges.
- Install the editor plugin (Vim, VS Code, and others) so problems get underlined as you type.



# 
Add selective tracing to a script

TaskWrite `/root/scripts/trace-demo.sh` that turns  bash tracing on around a specific block and off again afterwards. Then run it in a way that captures stdout and stderr into separate files.

## The script

The script should:

- Have a section (before the traced block) that prints a status message to stderr.
- Have a **traced** section wrapped in `set -x` ... `set +x` that loops over three names (`alice`, `bob`, `charlie`) and prints `hello, <name>` for each on stdout.
- Have a section (after the traced block) that prints another status message to stderr.

Only the loop should be traced.

## Run and capture separately

Run the script once with stdout and stderr redirected to different files:

- `/root/answers/trace-stdout.txt` - script's stdout
- `/root/answers/trace-stderr.txt` - script's stderr

When the checks pass, stdout will hold the three clean greetings (no `+` prefixes) and stderr will hold the trace lines.

Press **Submit** when both files have the expected contents.
