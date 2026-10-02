# cut, sort, and uniq

Three commands that show up in almost every shell pipeline. Build a tiny file first so you can see the exact output as you go:

```
echo "apple:3:red" > /tmp/fruit.txt
echo "banana:2:yellow" >> /tmp/fruit.txt
echo "cherry:1:red" >> /tmp/fruit.txt
```

Each line has three colon-separated fields: name, count, colour.

## cut: pull one column out of each line

`cut` slices a piece out of every line. Two flags do the work: `-d` sets the delimiter (the character between fields), and `-f` picks which field number you want.

```
cut -d ':' -f 1 /tmp/fruit.txt
```

```
apple
banana
cherry
```

Field 1 is everything before the first colon. Ask for two fields with a comma, and `cut` keeps the delimiter between them:

```
cut -d ':' -f 1,3 /tmp/fruit.txt
```

```
apple:red
banana:yellow
cherry:red
```

`/etc/passwd` is the classic real target. It's colon-separated as `username:x:uid:gid:...:home:shell`, so `cut -d ':' -f 1 /etc/passwd` prints every username on the machine, one per line.

There's also `-c` for character positions instead of fields: `cut -c 1-3 /tmp/fruit.txt` prints the first three characters of each line (`app`, `ban`, `che`).

## sort: put lines in order

Plain `sort` orders lines alphabetically:

```
cut -d ':' -f 1 /tmp/fruit.txt | sort
```

```
apple
banana
cherry
```

The catch: sort is alphabetical by default, so it compares numbers character by character. Watch what that does to a numeric column:

```
cut -d ':' -f 2 /tmp/fruit.txt | sort
```

```
1
2
3
```

That looks fine here, but with `10`, `2`, `1` alphabetical order gives `1`, `10`, `2` because it compares the `1` before the `2`. Add `-n` to sort numerically instead, and `-r` to reverse any sort:

```
cut -d ':' -f 2 /tmp/fruit.txt | sort -nr
```

```
3
2
1
```

When the fields live inside one line, `-k` picks the column to sort on and `-t` sets the delimiter. `sort -t ':' -k 2 -n /tmp/fruit.txt` orders the whole file by field 2 numerically.

## uniq: collapse repeated lines

`uniq` removes duplicates, but only when they sit next to each other. That's why it almost always runs after `sort`, which brings identical lines together first.

```
echo "red" > /tmp/colours.txt
echo "red" >> /tmp/colours.txt
echo "blue" >> /tmp/colours.txt
echo "red" >> /tmp/colours.txt
```

Run `uniq` alone and the last `red` survives, because it wasn't adjacent to the first two. Sort first and every duplicate collapses:

```
sort /tmp/colours.txt | uniq
```

```
blue
red
```

Add `-c` and `uniq` counts how many times each line appeared:

```
sort /tmp/colours.txt | uniq -c
```

```
   1 blue
   3 red
```

Pipe that into `sort -rn` and you get the busiest value on top. This count-then-rank pattern is one you'll type constantly:

```
sort /tmp/colours.txt | uniq -c | sort -rn
```

```
   3 red
   1 blue
```

Swap the input for the IPs in an access log, the error codes in a service log, or the shells in `/etc/passwd`, and the same three-stage pipeline answers "which value shows up most?" every time.



# tr, tee, and xargs

Three more pipe tools worth knowing well.

## tr: translate or delete characters

`tr` works character by character. Give it two sets and it maps every character in the first set to the one in the same position in the second:

```
echo "hello world" | tr 'a-z' 'A-Z'
```

```
HELLO WORLD
```

A common use is swapping one character for another, like turning spaces into underscores:

```
echo "my report file" | tr ' ' '_'
```

```
my_report_file
```

With `-d` it deletes characters instead of translating them:

```
echo "abc123def" | tr -d '0-9'
```

```
abcdef
```

`tr` only understands single characters, not words. When you need to replace a whole word or a pattern, reach for `sed` in the next topic.

## tee: send output to a file AND the screen

Redirecting with `> file` writes to the file and shows you nothing. `tee` does both at once: it prints to the screen and writes the same text to a file.

```
echo "deploy finished" | tee /tmp/run.log
```

```
deploy finished
```

The line appears on screen, and `/tmp/run.log` now holds the same text. Check it with `cat /tmp/run.log`. Use `-a` to append to the file instead of overwriting it:

```
echo "second run" | tee -a /tmp/run.log
```

It's handy when you're watching a long command run but also want a copy of its output saved for later.

## xargs: turn input into command arguments

Some commands read from a pipe. Others, like `rm` or `mkdir`, expect their input as arguments on the command line, not on stdin. `xargs` takes text from a pipe and hands it over as arguments.

```
echo "apple banana cherry" | xargs -n1 echo
```

```
apple
banana
cherry
```

Here `xargs` fed each word to `echo`. The `-n1` means "one argument per run", which is why each word landed on its own line. Without it, all three words would go to a single `echo` and print on one line.

A real example: find files and delete them. `find` prints the paths, `xargs` passes them to `rm`.

```
find /var/log -name '*.gz' | xargs -r rm
```

The `-r` flag tells `xargs` to do nothing when `find` prints nothing, which avoids running `rm` with no arguments.

### Two habits that save you

Filenames with spaces confuse plain `xargs`, because it splits on spaces. Pair `find -print0` with `xargs -0` so they split on a null character instead:

```
find /var/log -name '*.gz' -print0 | xargs -0 -r rm
```

Before any destructive `xargs` command, preview it by putting `echo` in front. That prints the command it would run without running it:

```
find /var/log -name '*.gz' | xargs echo rm
```

Once the preview looks right, drop the `echo` and run it for real.



# awk and sed basics

`awk` and `sed` are two small languages built for text. You could spend years on either. Here's the slice that covers most day-to-day work.

## awk: work with fields

`awk` splits each line into fields. By default it splits on whitespace, and you refer to the fields as `$1`, `$2`, `$3`, and so on. `$0` is the whole line, and `$NF` is always the last field, however many there are.

Build a small file to follow along:

```
echo "alice 90 " > /tmp/scores.txt
echo "bob 75 zsh" >> /tmp/scores.txt
echo "carol 82 " >> /tmp/scores.txt
```

Print the first field of every line:

```
awk '{print $1}' /tmp/scores.txt
```

```
alice
bob
carol
```

`$NF` grabs the last field without counting columns:

```
awk '{print $NF}' /tmp/scores.txt
```

```

zsh

```

Print two fields together, and awk joins them with a space:

```
awk '{print $1, $3}' /tmp/scores.txt
```

```
alice 
bob zsh
carol 
```

When fields are separated by something other than whitespace, `-F` sets the separator. `/etc/passwd` uses colons, so this prints every username on the machine:

```
awk -F: '{print $1}' /etc/passwd
```

The real power is putting a condition in front of the action. The action `{print $1}` only runs on lines that match the condition:

```
awk '$2 > 80 {print $1}' /tmp/scores.txt
```

```
alice
carol
```

One `awk` call filtered on field 2 and printed field 1, replacing a whole chain of pipes.

## sed: substitute text

The everyday job for `sed` is find-and-replace, written as `s/old/new/`. Read it as "substitute old with new".

```
echo "hello world" | sed 's/world/DevOps/'
```

```
hello DevOps
```

By default `sed` changes only the first match on each line. Add the `g` flag on the end to change every match:

```
echo "a-b-c" | sed 's/-/_/'
```

```
a_b-c
```

```
echo "a-b-c" | sed 's/-/_/g'
```

```
a_b_c
```

It works on files the same way. Build one and rewrite the scheme in every line:

```
echo "http://a.com" > /tmp/urls.txt
echo "http://b.com" >> /tmp/urls.txt
sed 's/http:/https:/g' /tmp/urls.txt
```

```
https://a.com
https://b.com
```

That prints the result without touching the file. Redirect it with `>` to save a new copy, or use `sed -i` to edit the file in place.

## When to reach for which

- Replacing or rewriting text? `sed`.
- Picking out columns or filtering by a condition? `awk`.
- Both at once? Common: `... | awk '{print $2}' | sed 's/old/new/'`.

You don't have to master either one. Recognise what they do, and you can copy a pattern from documentation, adapt it, and read it when it turns up in someone else's script.

