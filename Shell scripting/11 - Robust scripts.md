# set -euo pipefail

LessonBy default  is dangerously forgiving: a command fails and it runs the next one anyway; you read a variable you never set and it hands you an empty string. That is a disaster in a script, which keeps marching forward on top of broken state.

`set -euo pipefail` is one line you put at the top of a script to close off three of these dangers at once: fail early, fail loudly, and refuse to run on top of a mistake. Here is the bug each part prevents, then the setting that fixes it.

## set -e: stop the moment a command fails

```
#!/bin/
cp /tmp/missing.db /root/backup.db    # this file does not exist
echo "backup complete"
```

```
cp: cannot stat '/tmp/missing.db': No such file or directory
backup complete
```

The copy failed, yet the script printed "backup complete" and exited with success. `set -e` tells : if any command fails (nonzero exit code), stop the whole script right there.

```
#!/bin/
set -e
cp /tmp/missing.db /root/backup.db    # fails, so the script stops
echo "backup complete"                # never runs
```

## set -u: treat unset variables as errors

```
#!/bin/
target_dir="/root/project"
rm -rf "$target_di/build"    # typo: target_di, missing the r
```

There is no `target_di`, so  expands it to empty and the command silently becomes `rm -rf "/build"`, a completely different path. `set -u` treats any unset variable as an error:

```
#!/bin/
set -u
target_dir="/root/project"
rm -rf "$target_di/build"
```

```
script.sh: line 4: target_di: unbound variable
```

The script stops before running `rm` and points at the exact line. When you genuinely want a variable to be allowed empty, ask for an empty default and `set -u` leaves it alone:

```
optional="${OPTIONAL:-}"    # empty if OPTIONAL is unset, no error
```

## set -o pipefail: catch failures inside a pipeline

In a pipeline,  only reports the exit code of the last command:

```
printf 'all systems normal\n' > /tmp/app.log
grep "ERROR" /tmp/app.log | wc -l
echo "exit code: $?"
```

```
0
exit code: 0
```

`grep` found nothing and failed (exit 1), but `wc` succeeded and it is last, so  calls the whole pipeline a success. `set -o pipefail` changes the rule: if any command in the pipeline fails, the pipeline fails.

```
set -o pipefail
printf 'all systems normal\n' > /tmp/app.log
grep "ERROR" /tmp/app.log | wc -l
echo "exit code: $?"
```

```
0
exit code: 1
```

## The one line to remember

Flip all three at once, right after the shebang, on every serious script you write:

```
#!/bin/
set -euo pipefail
```

- `-e` stops the script when a command fails.
- `-u` stops the script when you read an unset variable.
- `-o pipefail` makes a pipeline fail if any part of it fails.

## One quirk of set -e to know about

Some commands "fail" as a normal signal, and `set -e` treats that as real. The classic case is `grep`, which exits nonzero when it finds no matches:

```
#!/bin/
set -e
count=$(grep -c "ERROR" /tmp/app.log)    # no matches, grep exits 1
echo "errors: $count"                    # never runs
```

"Zero errors" is a valid answer, but `set -e` kills the script before the `echo`. When you expect a nonzero exit and that is fine, add `|| true` to accept it:

```
count=$(grep -c "ERROR" /tmp/app.log || true)
echo "errors: $count"                    # prints: errors: 0
```



# trap for cleanup

LessonScripts often create things meant to live only for the run: a temp directory, a lock file, a background process. When the script ends those should be tidied away. The trouble is a script can end in many ways, and most of them skip a cleanup line sitting at the bottom.

`trap` solves this. You register a cleanup command once, near the top, and  runs it whenever the script ends, whether it finished normally, hit an error, or someone pressed Ctrl-C. You write the cleanup in one place and stop worrying about which exit path was taken.

## The bug: cleanup that gets skipped

```
#!/bin/
tmp_dir=$(mktemp -d)
cp /tmp/missing.tgz "$tmp_dir/"    # this fails, file does not exist
echo "processing..."
rm -rf "$tmp_dir"                  # cleanup at the bottom
```

The `cp` fails. If anything makes the script exit before that last line, the `rm -rf` never runs and the temp directory is left behind. Run this a few hundred times and `/tmp` fills up with abandoned directories nobody cleans. The cleanup only happens on the one path where everything went perfectly.

## The fix: register cleanup with trap

```
#!/bin/
tmp_dir=$(mktemp -d)
trap 'rm -rf "$tmp_dir"' EXIT

cp /tmp/missing.tgz "$tmp_dir/"    # even if this fails...
echo "processing..."
```

`trap 'COMMAND' EXIT` runs `COMMAND` whenever the script exits, for any reason: reaching the end, hitting `exit`, dying on an error, or being interrupted. Register it once, right after you create the thing that needs cleaning, and never think about it again.

## Signals you can trap

`EXIT` covers most needs, but you can trap specific signals too:

- `EXIT` - the script exits for any reason.
- `INT` - the user pressed Ctrl+C.
- `TERM` - something ran `kill` on the script.

```
trap 'rm -rf "$tmp_dir"' EXIT
trap 'echo "interrupted, cleaning up"; exit 130' INT
```

The `EXIT` trap still fires on its way out, so the temp directory is removed even on Ctrl+C.

## Preserving the exit code in a bigger cleanup

When cleanup needs more than one step, move it into a function. But the cleanup runs commands, and those overwrite `$?`, the exit code of whatever made the script stop. If you want to report that original code (you usually do), capture it on the very first line:

```
#!/bin/
cleanup() {
  local code=$?              # grab the exit code before anything else
  echo "cleaning up"
  rm -rf "$tmp_dir"
  return "$code"             # exit with the code we started with
}
tmp_dir=$(mktemp -d)
trap cleanup EXIT
```

Skip that step and a script that died on an error can end up reporting success, because the last thing that ran was a successful `rm`.



# Safe defaults for the script header

LessonYou already met `set -euo pipefail`, the one line that makes  fail loudly instead of limping along. A few more settings belong at the top of a serious script, each closing off a specific way  can surprise you. None are strictly required, but together they head off a whole category of bugs. As before: the bug first, then the setting.

## IFS: stop  splitting on spaces

`IFS`, the "internal field separator", is the set of characters  uses to chop an unquoted value into words. It includes the space by default, which causes a classic bug:

```
files="report 2024.txt"    # ONE filename that contains a space
for f in $files; do
  echo "handling: $f"
done
```

```
handling: report
handling: 2024.txt
```

One file, but the loop ran twice, because  split on the space. Setting `IFS` to just newline and tab stops that:

```
IFS=$'\n\t'
files="report 2024.txt"
for f in $files; do
  echo "handling: $f"
done
```

```
handling: report 2024.txt
```

This is not a cure for every quoting bug (quoting your expansions still matters), but it removes the most common source of them.

## LC_ALL: make sorting and formatting predictable

A script that sorts text or formats numbers can behave differently on two machines set to different languages: sort order, decimal separators, and month names all shift with the locale. Pinning it to `C` (the plain, language-neutral default) removes that variable:

```
LC_ALL=C
```

Now `sort`, `printf`, and friends produce the same output everywhere, which is what you want for reports or anything compared with `diff`.

## PATH: control which commands run

When your script runs `grep` or `ls`,  searches the directories in `PATH`. If someone slips an extra directory onto the front, they can put a malicious `grep` there and your script runs it instead of the real one. For anything security-sensitive, pin `PATH` to the standard system directories:

```
PATH=/usr/local/bin:/usr/bin:/bin
```

## The full header

Put together, a careful script often opens like this:

```
#!/usr/bin/env 
set -euo pipefail
IFS=$'\n\t'
LC_ALL=C
umask 077     # any files this script creates are readable by owner only
```

You do not need every line in every script; add them as a script grows into something others depend on. The one line that belongs in all of them, without exception, is `set -euo pipefail`.



# mktemp for safe temp files

LessonAlmost every script needs scratch space: a file to buffer output, a directory to unpack an archive. The obvious move is to pick a name in `/tmp` and use it, and that is where a surprising number of bugs and security holes come from, because `/tmp` is shared with every user and program on the machine.

`mktemp` does this safely: it hands you a temp file or directory with a name nobody can guess and nobody else is using. Here is the bug hardcoded names cause, then `mktemp` fixing it.

## The bug: a hardcoded temp name

```
tmpfile=/tmp/deploy.tmp    # looks harmless
echo "$data" > "$tmpfile"
```

A fixed name in `/tmp` causes three separate problems:

- Collisions. If the script runs twice at once, both runs write to `/tmp/deploy.tmp` and stamp on each other's data.
- Symlink attacks. Because the name is predictable, a hostile user can pre-create `/tmp/deploy.tmp` as a symlink pointing at a file they want overwritten. Your redirect then clobbers it with your privileges.
- Leftovers. If the script crashes before deleting it, the stale file sits there and confuses the next run.

## The fix: mktemp

`mktemp` creates a brand-new file with a random name and prints the path so you can capture it:

```
tmpfile=$(mktemp)
echo "$tmpfile"
```

```
/tmp/tmp.k9Xq2mVa7B
```

The name is random and the file is created in one atomic step, so there is nothing to collide with and nothing to pre-create. All three problems disappear at once. For a whole scratch directory, add `-d`:

```
tmp_dir=$(mktemp -d)
echo "$tmp_dir"
```

```
/tmp/tmp.Rf3Lp8Zq1C
```

## The pattern you will use everywhere

`mktemp` does not clean up after itself. Pair it with a `trap` (from the earlier lesson) so the directory is removed however the script ends:

```
#!/bin/
set -euo pipefail
tmp_dir=$(mktemp -d)
trap 'rm -rf "$tmp_dir"' EXIT

# now use $tmp_dir freely as scratch space
echo "step one output" > "$tmp_dir/step1.txt"
```

Create, register the cleanup, then use it. Those first two lines are the standard opening for any script that needs temporary space.

## Giving temp names a readable prefix

A name like `/tmp/tmp.k9Xq2mVa7B` is safe but tells you nothing about which script made it. Supply a template ending in at least three `X` characters, which `mktemp` replaces with random ones:

```
mktemp /tmp/deploy.XXXXXX
mktemp -d /tmp/build.XXXXXX
```

```
/tmp/deploy.a1B2c3
/tmp/build.d4E5f6
```

You still get an unguessable name, but now a glance at `/tmp` tells you which script each temp file belongs to.



# Write a robust script from scratch

TaskWrite `/root/scripts/robust.sh` that combines every robustness pattern from this topic in a working script.

## Required patterns

The script must use ALL of the following:

- `set -euo pipefail` near the top (fail early on errors, unset variables, and pipeline failures).
- A temporary directory created with `mktemp -d` (never a hardcoded /tmp path).
- A `trap` on `EXIT` that removes the temp directory when the script ends, regardless of how it ends.

## What the script must do

Do some "real work" inside the temp directory (write a file there, for example), then copy the result to `/root/answers/robust-output.txt`. The output file must contain the exact string `hello from a robust script`.

When the script exists, uses all three robustness patterns, and produces the output file, press **Submit**.
