# Positional arguments

You have already used arguments, maybe without knowing the word for them. When you run `ls /home`, the `/home` part is an **argument**: it tells `ls` which folder to list. The command is `ls`, and the extra word after it is the argument telling it what to work on.

```
ls /home          # ls is the command, /home is the argument
cat notes.txt     # cat is the command, notes.txt is the argument
```

Scripts work the same way. If you write a backup script, someone can run it and pass the folder to back up as an argument:

```
backup.sh /home/data     # /home/data is the argument
```

So the question is: inside the script, how do you get hold of that `/home/data`?  hands it to you in a ready-made variable. In fact it puts every argument into its own variable, one per position:

- `$1` is the first argument,
- `$2` is the second,
- `$3` is the third, and so on.

That is why they are called **positional**: the number matches the position you typed. You do not create these variables yourself.  fills them in fresh every time the script runs, so whatever the user types after the script name lands in `$1`, `$2`, `$3`.

## Your first script that reads arguments

The `cat > file <<'EOF' ... EOF` block below writes everything between the two `EOF` markers into the file.

```
cat > /root/greet.sh <<'EOF'
#!/bin/
echo "first argument:  $1"
echo "second argument: $2"
echo "third argument:  $3"
EOF
```

Now make it executable and run it with three arguments:

```
chmod +x /root/greet.sh
/root/greet.sh alice bob charlie
```

```
first argument:  alice
second argument: bob
third argument:  charlie
```

The three words you typed landed in `$1`, `$2`, `$3` in order. Pass fewer arguments and the missing ones are simply empty:

```
/root/greet.sh alice
```

```
first argument:  alice
second argument:
third argument:
```

 never complains about missing arguments; checking is your job.

## Special variables that come with arguments

 also gives you a few built-ins describing the arguments as a whole:

- `$0` is the script's own name (with the path you typed), not one of your arguments. Handy in messages like `echo "usage: $0 ..."`.
- `$#` is the count of arguments, not counting `$0`. Check this to make sure the user gave you enough input.
- `$@` is every argument, in order: the easy way to say "all of them."

```
cat > /root/info.sh <<'EOF'
#!/bin/
echo "script name: $0"
echo "how many:    $#"
echo "all of them: $@"
EOF
```

Make it executable and run it with three arguments:

```
chmod +x /root/info.sh
/root/info.sh apple banana cherry
```

```
script name: /root/info.sh
how many:    3
all of them: apple banana cherry
```

One trap: `$1` through `$9` work as expected, but `$10` reads as `$1` followed by a literal `0`. For the tenth argument on, use braces: `${10}`, `${11}`.

## Looping over every argument

To handle each argument one at a time, use a `for` loop over `"$@"`:

```
cat > /root/listfiles.sh <<'EOF'
#!/bin/
for arg in "$@"; do
  echo "processing $arg"
done
EOF
```

Make it executable and run it with three arguments:

```
chmod +x /root/listfiles.sh
/root/listfiles.sh report.txt data.csv notes.md
```

```
processing report.txt
processing data.csv
processing notes.md
```

## Always write `"$@"` with quotes

 has two ways to say "all the arguments," `$@` and `$*`, and they differ the moment an argument contains a space:

```
cat > /root/showargs.sh <<'EOF'
#!/bin/
echo 'with "$@":'; for f in "$@"; do echo "[$f]"; done
echo 'with "$*":'; for f in "$*"; do echo "[$f]"; done
EOF
```

Make it executable and run it, passing an argument that has a space in it:

```
chmod +x /root/showargs.sh
/root/showargs.sh "monthly report.pdf" backup.tgz
```

```
with "$@":
[monthly report.pdf]
[backup.tgz]
with "$*":
[monthly report.pdf backup.tgz]
```

With `"$@"` each argument stays separate and spaces are preserved; with `"$*"` everything mashes into one string. The rule: whenever you mean "all the arguments," write `"$@"` with the double quotes.

## `shift`: dropping the first argument

Sometimes the first argument names an action (`deploy`, `rollback`) and the rest is data for it. `shift` throws away `$1` and slides every other argument down: `$2` becomes `$1`, and `$#` drops by one.

```
cat > /root/action.sh <<'EOF'
#!/bin/
echo "the action is: $1"
shift
echo "remaining arguments: $@"
EOF
```

Make it executable and run it with an action followed by three servers:

```
chmod +x /root/action.sh
/root/action.sh deploy server1 server2 server3
```

```
the action is: deploy
remaining arguments: server1 server2 server3
```



# Reading input

Arguments are set before a script starts. Sometimes instead you want the script to ask a question while running and wait for an answer, like "Are you sure? (y/n)". That's what `read` is for: it pulls a line from **standard input** (usually your keyboard) and stores it in a variable.

## The simplest read

```
cat > /root/hello.sh <<'EOF'
#!/bin/
echo "What is your name?"
read name
echo "Hello, $name"
EOF
```

Now make it executable and run it:

```
chmod +x /root/hello.sh
/root/hello.sh
```

The script prints the question and stops, waiting. You type a name and press Enter:

```
What is your name?
alice
Hello, alice
```

`read name` captured that line into `name`. To avoid typing interactively while testing, pipe the answer in and it behaves the same:

```
echo "alice" | /root/hello.sh
```

```
What is your name?
Hello, alice
```

## A cleaner prompt with `-p`

The `-p` option prints a message and waits on the same line, so the cursor sits right next to your question:

```
cat > /root/ask.sh <<'EOF'
#!/bin/
read -p "Enter your name: " name
echo "Hello, $name"
EOF
```

Make it executable, then run it (piping in an answer so it doesn't wait for typing):

```
chmod +x /root/ask.sh
echo "bob" | /root/ask.sh
```

```
Enter your name: Hello, bob
```

## Make `-r` your default habit

Almost always you want `read -r`. Without `-r`,  treats a backslash (`\`) in the input as special, so a Windows-style path like `C:\Users\alice` gets mangled. With `-r` the backslashes survive exactly as typed:

```
cat > /root/readpath.sh <<'EOF'
#!/bin/
read -r path
echo "you entered: $path"
EOF
```

Make it executable, then run it with a Windows-style path piped in:

```
chmod +x /root/readpath.sh
printf 'C:\\Users\\alice\n' | /root/readpath.sh
```

```
you entered: C:\Users\alice
```

There's almost no case where you want the old behavior, so always write `read -r`.

## Reading a file one line at a time

A very common job is going through a file line by line. The standard pattern combines `read` with a `while` loop:

```
cat > /root/servers.txt <<'EOF'
web-01
web-02
db-01
EOF
cat > /root/readlines.sh <<'EOF'
#!/bin/
while IFS= read -r line; do
  echo "found server: $line"
done < /root/servers.txt
EOF
```

Make it executable and run it:

```
chmod +x /root/readlines.sh
/root/readlines.sh
```

```
found server: web-01
found server: web-02
found server: db-01
```

The two pieces worth knowing:

- `< /root/servers.txt` redirects the file into the loop as standard input, so `read` grabs the next line each turn until none are left.
- `IFS=` turns off whitespace trimming for this command, so lines that start or end with spaces come through intact. Leave it out and that whitespace gets silently stripped.

Memorize the shape: `while IFS= read -r line; do ... done < file` is the reliable way to read a file line by line.

## Reading several values from one line

Give `read` several variable names and it splits the input on whitespace, filling each in order:

```
cat > /root/split.sh <<'EOF'
#!/bin/
read -r first last email <<< "Alice Smith alice@example.com"
echo "first: $first"
echo "last:  $last"
echo "email: $email"
EOF
```

Make it executable and run it:

```
chmod +x /root/split.sh
/root/split.sh
```

```
first: Alice
last:  Smith
email: alice@example.com
```

The `<<<` is a **here-string**: it hands the text on its right to the command as standard input, saving you a pipe or file for one line. If there are more words than variables, the last variable soaks up all the leftovers.



# Parsing flags with getopts

Positional arguments are useful, but they have one weakness: **order matters**. `$1` is always the first word, `$2` the second. So your script only works if the user passes things in the exact order you expect. Swap two arguments by accident and the script happily uses the wrong value in the wrong place.

Now think about the commands you use every day. They do not have this problem. With `grep`, both of these work the same way:

```
grep -i -v error log.txt
grep -v -i error log.txt
```

You can put `-i` and `-v` in any order and `grep` understands both. These dash-letter pieces are called **flags** (or **options**), and real tools let you pass them in whatever order you like. Some flags also carry a value, like `-o output.txt` meaning "write output to this file."

How does `grep` pull that off, when positional arguments cannot? It does not read `$1`, `$2` by position. It uses a helper that scans the flags wherever they appear.  gives you that same helper, and it is called `getopts`.

You tell `getopts` which flags your script accepts. It then walks through whatever the user typed, one flag at a time, and tells you which flag it found and (if that flag carries a value) what value came with it. Order stops mattering, because `getopts` looks for the flags instead of counting positions.

## A complete, runnable example

This tool accepts `-v` (an on/off flag) and `-o` (a flag that takes a filename value):

```
cat > /root/mytool.sh <<'EOF'
#!/bin/
verbose=false
output="default.txt"

while getopts "vo:" opt; do
  case $opt in
    v) verbose=true ;;
    o) output=$OPTARG ;;
    ?) echo "usage: $0 [-v] [-o output] files..." >&2; exit 1 ;;
  esac
done

shift $((OPTIND - 1))

echo "verbose: $verbose"
echo "output:  $output"
echo "files:   $@"
EOF
```

Make it executable and run it with both flags and two files:

```
chmod +x /root/mytool.sh
/root/mytool.sh -v -o report.txt file1 file2
```

```
verbose: true
output:  report.txt
files:   file1 file2
```

It picked out the flags, grabbed `report.txt` for `-o`, and left `file1 file2` as plain arguments. Because `getopts` handles order, swapping the flags gives the same result:

```
/root/mytool.sh -o report.txt -v file1 file2
```

```
verbose: true
output:  report.txt
files:   file1 file2
```

## The option string `"vo:"`

The string lists which flags your script accepts. Each letter is one flag; a colon **after** a letter means that flag requires a value.

```
"vo:"
 ^^^
 | \_ the colon says: -o must be followed by a value
 \___ v is a plain on/off flag, no value
```

A third flag `-n` that also takes a value would make it `"vo:n:"`. The colon is the only thing distinguishing a switch from a flag-with-value.

## The loop and the `case`

`while getopts "vo:" opt` runs once per flag, storing the letter it found in `opt` (a name you chose). When the flags run out, `getopts` returns false and the loop ends. The `case` decides what to do:

- `v)` sets `verbose=true`, just recording the switch was seen.
- `o)` reads its value from `$OPTARG`. Whenever a flag takes a value, `getopts` puts it in `$OPTARG`; for `-o report.txt` that's `report.txt`.
- `?)` is the catch-all for a flag you didn't declare. Here it prints a usage message to standard error (`>&2`, the right place for errors) and exits.

## Defaults and clearing the flags away

`getopts` sets nothing for flags the user leaves out, so the two lines at the top set sensible defaults; the loop overwrites them only when a flag actually appears. Run with no flags to see them:

```
/root/mytool.sh file1
```

```
verbose: false
output:  default.txt
files:   file1
```

The line `shift $((OPTIND - 1))` leaves `$@` holding only the non-flag arguments. As `getopts` works, it counts consumed words in `$OPTIND`; `shift` drops that many from the front. Treat it as boilerplate: put it right after the loop.

## What `getopts` will not do

- **No long options.** It only understands short flags like `-v`, not `--verbose`. For long forms you'd use the separate `getopt` tool or hand-roll a loop. Short flags cover most scripts.
- **It stops at the first non-flag argument**, so flags must come before plain arguments:

```
/root/mytool.sh file1 -v
```

```
verbose: false
output:  default.txt
files:   file1 -v
```

`getopts` saw `file1` first, decided the flags were over, and left `-v` unparsed among the files. This is standard behavior, so put your flags first. For everyday scripts, `getopts` covers what you need with very little code.

