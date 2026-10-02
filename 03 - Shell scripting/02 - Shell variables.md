# Declaring and expanding variables

Scripts often use the same value in several places: a folder path, a filename, a server address. Instead of typing it out each time (and fixing every copy when it changes), you store it once in a variable and refer to it by name. In bash the value is treated as plain text by default. Even `5` is stored as the text `5`.

## Creating a variable

Write the name, an `=`, and the value:

```
name=alice
count=5
message="hello world"
```

Three rules confuse almost every beginner:

- **No spaces around `=`.** `name = alice` fails, because bash reads the first word as a command and tries to run `name`. Keep it tight: `name=alice`.
- **Quote values with spaces.** `message=hello world` breaks (bash thinks the line ends at `hello`). Wrap it: `message="hello world"`.
- **Use lowercase names.** UPPERCASE is by convention reserved for system variables like `PATH` and `HOME`. Naming your own variable `PATH` can overwrite the real one and break commands.

## Using a variable's value

Put a `$` in front of the name to get the value back out (this is called **expanding** it):

```
name=alice
echo $name              # prints: alice
echo "hello, $name"     # prints: hello, alice
```

### When you need `${name}`

Use the longer `${name}` form when the name touches other letters, so bash knows where it ends:

```
file=report
echo $file_final        # prints nothing: no variable named file_final
echo ${file}_final      # prints: report_final
```

## No declaring first

Unlike some languages, bash has no separate declare step and no types. Assign a value and the variable exists:

```
first_name=Alice
last_name=Smith
full_name="$first_name $last_name"
echo "$full_name"       # prints: Alice Smith
```

← Previous

# The quoting rules

The same variable can behave three different ways depending on how you quote it. The reason: after bash replaces `$name` with its value, it may chop the result into pieces wherever it finds spaces. Quotes control that second step.

## The same variable, three ways

```
name="Alice Smith"

echo $name        # prints: Alice Smith
echo "$name"      # prints: Alice Smith
echo '$name'      # prints: $name
```

The first two look identical here, but they are not the same:

- **Unquoted `$name`** first becomes `Alice Smith`. Then bash sees the space in the middle and chops it into two pieces, `Alice` and `Smith`. It sends those to `echo` as two separate words. `echo` just prints whatever words it gets, with a space between each, so you see `Alice Smith` and everything looks fine. But behind the scenes bash handed over two words, not one. This chopping is called **word splitting**, and it causes most quoting bugs.
- **Double-quoted `"$name"`** also becomes `Alice Smith`, but the quotes tell bash "keep this as one piece, do not chop it." `echo` gets one word, `Alice Smith`. This is what you want almost every time.
- **Single-quoted `'$name'`** turns off expansion completely. Bash prints the literal four characters `$name`. Use it when you want a dollar sign to stay a dollar sign.

You cannot see the chopping in the example above because `echo` glues the pieces back with a space, so both lines look the same. Add extra spaces to the value and the difference shows up:

```
name="Alice   Smith"   # three spaces in the middle

echo $name        # prints: Alice Smith     (bash chopped it, extra spaces gone)
echo "$name"      # prints: Alice   Smith   (kept as one piece, spaces stay)
```

Unquoted, bash chopped the value into `Alice` and `Smith` and threw the in-between spaces away. Quoted, the whole value stayed intact. Same variable, different result, and the only difference is the quotes.

## The safe default: double-quote your variables

**Put double quotes around every variable expansion, by default.** They let the value expand but stop bash from splitting it apart.

```
source_file="/etc/hostname"
destination="/tmp/hostname.bak"
cp "$source_file" "$destination"
```

## Why it matters: a real example

```
cd /tmp
filename="my report.txt"
touch "$filename"

rm $filename       # danger: deletes 'my' and 'report.txt'
rm "$filename"     # safe: removes the one file
```

With `rm $filename`, bash splits the value and hands `rm` two arguments, `my` and `report.txt`, deleting files you never meant. With `rm "$filename"` it passes the single name `my report.txt`. Filenames with spaces are real, so quoting here prevents real damage.

## What stays special inside double quotes

Double quotes are not a total shutdown like single quotes. Inside `"..."` a few characters keep their meaning:

- `$` still triggers variable expansion.
- `$(...)` still runs a command (next ).
- `\` still escapes the characters above.

So to get a literal dollar sign inside double quotes, escape it:

```
price=5
echo "it costs \$$price"   # prints: it costs $5
```

The `\$` becomes a plain `$`, and `$price` right after it expands to `5`.

← Previous

# Command substitution

So far you have put fixed text into variables, like `name="Alice"`. But often the value you want is not fixed. It is something a command already knows: today's date, the machine's name, how many files are in a folder. You could run the command, read the output with your eyes, and type it into the variable yourself. But that value would be stale the next day, and a script cannot read with its eyes anyway.

Command substitution solves this. It runs a command for you and drops whatever the command prints straight into a variable. No copying by hand: bash grabs the output and stores it.

## The `$(...)` syntax

Wrap the command in `$(` and `)`:

```
today=$(date +%Y-%m-%d)
echo "$today"           # prints: 2026-07-28

host=$(hostname)
echo "$host"            # prints your machine's name
```

Bash runs `date +%Y-%m-%d`, grabs what it prints, and stores it in `today`. The command runs at the moment of assignment, so `today` holds the date as it was then, not a live clock. Bash also trims the trailing newline for you, so you get a clean `2026-07-28`.

## Combining it with text

A captured value is plain text, so you can drop it into the middle of a string, mixed with your own text and other substitutions:

```
echo "log-$(date +%Y%m%d)-$(hostname).txt"
# prints something like: log-20260728-myhost.txt
```

## Command substitution and quoting

Last 's quoting rules apply. Double quotes let the substitution run; single quotes turn it off and print it literally:

```
report="report-$(date +%Y%m%d).csv"
echo "$report"    # prints: report-20260728.csv

literal='report-$(date +%Y%m%d).csv'
echo "$literal"   # prints: report-$(date +%Y%m%d).csv
```

## Backticks: the old way

Older scripts use backticks for the same job:

```
today=`date +%Y-%m-%d`
```

They are easy to confuse with ordinary quotes and awkward to nest. Recognise them when reading old scripts, but always write `$(...)` yourself.

## Why you will use this constantly

Grab a value from a command, store it, then use it. A large share of everyday automation is exactly this shape:

```
file_count=$(ls /etc | wc -l)
echo "there are $file_count entries in /etc"

kernel=$(uname -r)
echo "running kernel $kernel"
```

← Previous
