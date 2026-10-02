# Arithmetic in 

Scripts often need a little math: count how many files you processed, work out how many megabytes are free, check if usage crossed 90%.

You might expect to just add two variables together. But  does not work like a calculator by default. Remember that  stores every value as text, so a variable holding `5` really holds the *character* `5`, not the number. Try to add to it the obvious way and  just sticks the characters side by side:

```
count=5
echo $count+1           # prints: 5+1   (glued together, not added)
```

It printed `5+1` because  never saw any math, just three characters in a row. To actually add, you have to tell  "treat this as a calculation," and there is a special syntax for that.

## The everyday form: `$(( ))`

Put an expression between double parentheses and  replaces the whole thing with the result:

```
count=5
total=$((count + 1))          # total is now 6
half=$((count / 2))           # half is now 2 (integer math, see next )
mod=$((count % 3))            # mod is now 2 (remainder of 5 / 3)
power=$((count ** 2))         # power is now 25 (5 squared)
echo "$total $half $mod $power"   # prints: 6 2 2 25
```

- Inside `$(( ))` you can write `count` without a `$`;  looks it up for you. Writing `$(( $count + 1 ))` works too.
- `5 / 2` gave `2`, not `2.5`:  only does whole-number math (next ).
- `%` is the remainder, used to test "is this even?" (`$((n % 2))` is `0` for even numbers) or "every 10th time round a loop."

## Doing math in place: `(( ))`

The cousin `(( ))` (no dollar sign) runs the same arithmetic but acts on the spot instead of handing back a value. Perfect for counters:

```
count=5
((count++))                   # add 1 to count, in place
echo "$count"                 # prints: 6
((count += 10))               # add 10 to count
echo "$count"                 # prints: 16
```

It also lets an `if` compare numbers with plain math symbols:

```
count=150
if (( count > 100 )); then
  echo "over the limit"
fi
# prints: over the limit
```

Compared with the `[ "$count" -gt 100 ]` style, `(( ))` is friendlier: write `>` instead of `-gt`, no `$` or quotes needed.

## The operators you can use

Both `$(( ))` and `(( ))` accept a familiar set:

- Arithmetic: `+`, `-`, `*`, `/`, `%`, and `**` (power).
- Assignment shorthands: `+=`, `-=`, `*=`, `/=`, `%=`, plus `++` and `--`.
- Comparison: `==`, `!=`, `<`, `<=`, `>`, `>=` (mostly inside `if (( ... ))`).

```
n=20
((n -= 5))              # n is now 15
((n *= 2))              # n is now 30
((n--))                 # n is now 29
echo "$n"              # prints: 29
```

Watch `=` versus `==`. A single `=` assigns; double `==` compares. Mixing them up is a classic bug:

```
x=5
(( x == 5 )) && echo "yes, x is 5"     # prints: yes, x is 5
(( x = 99 ))                            # this ASSIGNS 99 to x
echo "$x"                               # prints: 99
```

## The old ways: `let` and `expr`

You will meet these in other people's scripts:

```
let "count = 5 + 1"           # older style, same as $(( ))
count=$(expr 5 + 1)           # very old, runs a separate program each time
echo "$count"                 # prints: 6
```

Both are worse than `(( ))`: `let` is easier to get wrong, and `expr` launches a whole program per calculation. In new scripts reach for `$(( ))` for a result and `(( ))` to change or test a variable.



# Integer only, and how to get floats

Last  `$((5 / 2))` gave `2`, not `2.5`. That is a hard rule:  arithmetic is integer only, with no fractional numbers inside `$(( ))` at all. It was built to move files around, not to be a calculator. But most interesting numbers in scripts (percentages, averages, ratios) are fractions, so you need a tool to borrow.

## What "integer only" does

When a division is not even,  does not round. It chops off everything after the decimal point (truncation):

```
echo $((3 / 2))       # prints: 1   (1.5 chopped down to 1)
echo $((10 / 3))      # prints: 3   (3.33... chopped down to 3)
echo $((7 / 4))       # prints: 1   (1.75 chopped DOWN to 1, not rounded to 2)
```

Note `7 / 4` is `1.75` yet gives `1`:  always chops down.

Put a decimal into the expression and  stops with an error:

```
echo $((3.14))
# : 3.14: syntax error: invalid arithmetic operator (error token is ".14")
```

Seeing `invalid arithmetic operator` with a dot in the message almost always means a decimal point slipped inside `$(( ))`.

## The recommended tool: `awk`

The reliable way to get a decimal answer is `awk`, a small text program present on every fresh Ubuntu box. It does floating-point math and formats the result. Put the calculation in a `BEGIN` block ("do this before reading any input"):

```
result=$(awk 'BEGIN {print 3.14 * 2}')
echo "$result"        # prints: 6.28
```

`printf` inside awk controls the decimal places. The classic case, turning a fraction into a percentage with one decimal:

```
used=2
total=3
pct=$(awk -v a="$used" -v b="$total" 'BEGIN {printf "%.1f", (a/b)*100}')
echo "$pct%"          # prints: 66.7%
```

`-v a="$used"` hands your  variable into awk as `a`. `%.1f` prints one digit after the decimal; use `%.2f` for two. This one pattern covers almost every percentage or average you will need.

## The other tool: `bc`

Older scripts use `bc` ("basic calculator"), a dedicated calculator that reads an expression and prints the answer:

```
# only works if bc is installed - it is not on a bare Ubuntu system
result=$(echo "3.14 * 2" | bc)
echo "$result"        # would print: 6.28

precise=$(echo "scale=4; 10 / 3" | bc)
echo "$precise"       # would print: 3.3333
```

`scale=4` sets how many decimals to keep. The catch: `bc` is not always installed, so you may need `apt-get install -y bc` first. Prefer `awk` for portable scripts and treat `bc` as a nice-to-have.

## Which tool for which job

- Whole numbers (counters, indexes, thresholds): `$(( ))`. Fastest, needs nothing installed.
- Percentages, ratios, averages, currency: `awk`. Does the math and formatting in one step, always available.
- `bc`: convenient for arbitrary-precision work, but check it exists or install it first.

For most DevOps scripts `$(( ))` handles the counters and thresholds, and you reach for `awk` when a percentage or average shows up.



# String operations and printf

Most of what a script pushes around is text: usernames, paths, log lines, messages.  can measure, join, compare, and format text on its own, without calling any other program.

## Measuring length

Put a `#` after `$` and the opening brace to count characters:

```
name="alice"
echo ${#name}         # prints: 5
```

Read `#` as "count of." Handy for checking a password is long enough.

## Joining strings (concatenation)

 has no `+` for text. You join strings by writing them next to each other inside one set of double quotes;  expands each variable in place, and any plain text between them (the `, `below) appears exactly as typed:

```
greeting="hello"
name="alice"
full="$greeting, $name"
echo "$full"          # prints: hello, alice
```

If other letters follow a variable name directly, wrap it in braces so  sees where the name stops:

```
base="report"
full="${base}_final"
echo "$full"          # prints: report_final
```

Without braces, `$base_final` hunts for a nonexistent variable `base_final` and gives an empty result.

There is a shorthand for appending with `+=`:

```
greeting="hello"
greeting+=", world"                       # same as greeting="$greeting, world"
echo "$greeting"      # prints: hello, world
```

## Comparing strings

To compare strings or test for empty, use `[[ ]]` (the double-bracket test from conditionals):

```
a="apple"
b="banana"

if [[ "$a" == "$b" ]]; then echo "same"; fi          # equal
if [[ "$a" != "$b" ]]; then echo "different"; fi     # prints: different
if [[ "$a" < "$b" ]]; then echo "a comes first"; fi  # prints: a comes first
```

`==` tests equality, `!=` difference. `<` and `>` compare by character order (so `apple` before `banana`) and only work inside `[[ ]]`, not the older `[ ]`.

To check whether a variable is empty, use `-z` ("zero length") and `-n` ("non-empty"):

```
reply=""
if [[ -z "$reply" ]]; then echo "nothing entered"; fi   # prints: nothing entered
if [[ -n "$reply" ]]; then echo "got a value"; fi        # prints nothing
```

Keep the variable in double quotes so an empty value still counts as one thing.

## Formatted output with `printf`

`echo` cannot line up columns or control decimals. `printf` works from a template: a format string with placeholders, then the values to fill them:

```
user="alice"
count=42
printf "%s has %d files\n" "$user" "$count"
# prints: alice has 42 files
```

`%s` slots in a string, `%d` a whole number; values fill the slots in order. Unlike `echo`, `printf` adds no line break, so write `\n` yourself.

Placeholders can set width and alignment to build columns. `%-10s` is a string padded to 10 wide, left-aligned (the `-`); `%5d` a number padded to 5 wide, right-aligned. Same widths every row means columns line up:

```
printf "%-10s %5d\n" "alice" 42
printf "%-10s %5d\n" "bob" 7
# prints:
# alice         42
# bob            7
```

`printf` also shows fixed decimals. `%f` is a float, `%.2f` keeps two digits; write `%%` for a literal percent sign:

```
printf "%.2f%%\n" 66.667
# prints: 66.67%
```

Doing the math in `awk` and presenting it with `printf` is how you turn a raw ratio into a clean `66.67%` line for a report.



# Build a small calculator script

TaskWrite `/root/scripts/calc.sh` that takes two integer arguments and reports several calculations to a file.

## Arguments

- `$1` - first integer (call it `a`)
- `$2` - second integer (call it `b`)

Both arguments are required.

## What to compute and report

Write these four lines to `/root/answers/calc.txt` when the script is called with `20 6`:

- `sum:` followed by `a + b` (should be `26`)
- `diff:` followed by `a - b` (should be `14`)
- `product:` followed by `a * b` (should be `120`)
- `ratio:` followed by `(a / b) * 100`, as a decimal (should contain `333`)

Use  integer arithmetic for the first three. For the ratio,  integer math truncates, so reach for `awk` (which handles floats) to get a decimal answer.

## Run it

Call the script with the arguments `20 6`. When the report file contains all four expected values, press **Submit**.
