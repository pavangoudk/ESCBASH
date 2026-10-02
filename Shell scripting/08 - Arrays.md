# Indexed arrays

A plain variable holds one value. But real scripts often deal with a *list* of things: three web servers to restart, ten files to back up, every username in a report.

With plain variables, a list gets awkward fast. You end up with separate names:

```
server1="web-01"
server2="web-02"
server3="web-03"
```

Now they are not really connected. To do something to all of them you must repeat the work for each name, and adding a fourth server means editing the script in several places.

An **array** solves this. It holds many values under one name, and each value has a position number so you can still reach any single one:

```
servers=("web-01" "web-02" "web-03")
```

One name, `servers`, now holds the whole list. You can loop over all of them at once, add more, or count them, without inventing a new variable each time.

This  covers the **indexed array**, where each value sits under a number (its position, called its index). A later  covers the kind that uses words as keys instead of numbers.

## Creating an array

List the values inside parentheses, separated by spaces (no commas):

```
fruits=(apple banana cherry)
```

 numbers them starting at zero: `apple` is 0, `banana` is 1, `cherry` is 2. Nothing prints; the array just exists now. If a value needs a space inside it, quote that one value:

```
names=("Alice Smith" "Bob Jones")
```

`names` has two elements, not four, because the quotes hold each name together.

## Reaching a single element

Write the name and position in square brackets, wrapped in `${...}`. Counting starts at 0, and a negative number counts back from the end:

```
fruits=(apple banana cherry)
echo "${fruits[0]}"     # prints: apple
echo "${fruits[1]}"     # prints: banana
echo "${fruits[-1]}"    # prints: cherry
```

The `${...}` braces are required with arrays. Drop them and  misreads you:

```
fruits=(apple banana cherry)
echo "$fruits[1]"       # prints: apple[1]
echo "${fruits[1]}"     # prints: banana
```

`$fruits` gives element 0 (`apple`) and `[1]` sticks on as plain text. Always use the braces.

## Getting every element and counting

Use `@` as the index to grab all the values, and a `#` before the name to count them:

```
fruits=(apple banana cherry)
echo "${fruits[@]}"     # prints: apple banana cherry
echo "${#fruits[@]}"    # prints: 3  (element count)
echo "${#fruits[0]}"    # prints: 5  (length of "apple")
```

Note that `[@]` counts slots while `[0]` measures the string in one slot. There is also `${fruits[*]}`: the difference from `[@]` only shows with quotes and spaces, where `"${fruits[@]}"` keeps each element separate and `"${fruits[*]}"` glues them into one string. Reach for `[@]` almost every time.

## Adding to an array later

Set a specific slot by number, or append to the end with `+=`:

```
fruits=(apple banana cherry)
fruits[3]="date"
fruits+=(elderberry)
echo "${fruits[@]}"     # prints: apple banana cherry date elderberry
```

`+=` is the handy one when building a list as you go, since you don't have to know the next free position.

## Looping over an array

A `for` loop over `"${fruits[@]}"` hands you one element at a time:

```
fruits=(apple banana cherry)
for fruit in "${fruits[@]}"; do
  echo "processing $fruit"
done
```

```
processing apple
processing banana
processing cherry
```

Always double-quote the expansion. The quotes keep a value with a space (like `"Alice Smith"`) as one item instead of splitting it in two.



# Working with array contents

Once you can create an array, the next step is shaping the data: take a slice, change an entry, turn a line of text into a list, or a list back into a line. Each example builds its own array and prints the result.

## Taking a slice

Carve out a range with `${array[@]:start:count}`, where `start` is the index to begin at and `count` is how many to take. Leave off `count` to run to the end:

```
letters=(a b c d e f g)
echo "${letters[@]:2}"     # prints: c d e f g
echo "${letters[@]:2:3}"   # prints: c d e
echo "${letters[@]: -2}"   # prints: f g
```

The last line counts in from the end, and the space before the minus is required: without it  reads `:-` as a different feature (a default-value expansion).

## Changing and removing elements

Replace an element by assigning to its index. Remove one with `unset`:

```
letters=(a b c d e f g)
letters[3]="D"
echo "${letters[@]}"       # prints: a b c D e f g
unset 'letters[3]'
echo "${letters[@]}"       # prints: a b c e f g
```

Quote `'letters[3]'` in the `unset`: without quotes  may treat `[3]` as a filename pattern and misfire. Also note that removing element 3 leaves a hole; the elements after it do not shift down, so what was at index 4 stays at index 4.

## Listing the actual indices

After an `unset` the indices can have gaps, so don't assume they still run `0, 1, 2, ...`. Put a `!` before the name to see the indices that actually exist:

```
letters=(a b c d e f g)
unset 'letters[3]'
echo "${!letters[@]}"      # prints: 0 1 2 4 5 6
```

There is no `3`. `${!letters[@]}` gives the keys (index numbers) rather than the values, which is the safe way to loop over an array with gaps.

## Turning a string into an array

Assigning an unquoted variable into parentheses splits it on spaces into elements:

```
line="one two three"
words=($line)
echo "${words[0]}"         # prints: one
echo "${#words[@]}"        # prints: 3
```

This is handy but risky: it splits on any whitespace and expands filename patterns like `*`. For anything beyond simple, known input, use the safer tool below.

## Splitting on a chosen separator

Real data often uses another separator, like a comma in a CSV line. `read -a` splits a line into an array, and `IFS` sets the separator:

```
IFS=',' read -ra parts <<< "web,db,cache"
echo "${parts[0]}"         # prints: web
echo "${parts[2]}"         # prints: cache
echo "${#parts[@]}"        # prints: 3
```

`IFS=','` breaks on commas, `-a parts` fills the array, `-r` keeps backslashes literal, and `<<<` feeds the string in. This does not expand `*`, so it is the reliable choice for parsing.

## Joining an array into a string

To glue elements together with a separator, set `IFS` to it and use the `[*]` form, which joins on the first character of `IFS`:

```
list=(alpha beta gamma)
joined=$(IFS=,; echo "${list[*]}")
echo "$joined"             # prints: alpha,beta,gamma
```

Running this inside `$(...)` is a subshell, so the change to `IFS` disappears when it finishes and your main script keeps its normal `IFS`. Setting `IFS` globally and forgetting to reset it is a classic source of hard-to-find bugs.



# Associative arrays

Indexed arrays store values under numbers. But often the natural label for a value is a word, not a number: the IP of a server called `web1` is easier to find by the name `web1` than by index 2. An **associative array** stores each value under a word of your choosing, called the **key**. If you have used a dictionary, map, or hash in another language, this is the same idea: look something up by name, get its value back.

## Declaring one first

Unlike indexed arrays, you must announce an associative array with `declare -A` before using it:

```
declare -A servers
```

The capital `-A` is what makes it associative. This is not optional: if you skip it and assign with word keys,  treats the array as indexed and your keys all collapse to index 0, silently overwriting each other.

## Storing and reading values

Set a value by putting the key in the brackets, and read it back the same way. A key that was never set expands to an empty string rather than an error:

```
declare -A servers
servers[web1]="10.0.0.1"
servers[db1]="10.1.0.1"

echo "${servers[web1]}"              # prints: 10.0.0.1
echo "start[${servers[missing]}]end" # prints: start[]end
```

That empty result matters because "the key is missing" and "the key exists but its value is empty" look identical. You will tell them apart below.

## Getting all keys and all values

As with indexed arrays, `${array[@]}` gives every value and `${!array[@]}` gives every key:

```
declare -A servers
servers[web1]="10.0.0.1"
servers[db1]="10.1.0.1"

echo "${!servers[@]}"   # prints the keys:   web1 db1
echo "${servers[@]}"    # prints the values: 10.0.0.1 10.1.0.1
```

Warning: associative arrays do not keep keys in insertion order.  stores them for fast lookup, so they may come back in any order.

## Looping over the pairs

Loop over the keys, then look up each value inside the loop. Here `name` holds one key each time and `${servers[$name]}` looks up its value:

```
declare -A servers
servers[web1]="10.0.0.1"
servers[db1]="10.1.0.1"

for name in "${!servers[@]}"; do
  echo "$name is at ${servers[$name]}"
done
```

```
web1 is at 10.0.0.1
db1 is at 10.1.0.1
```

## Checking whether a key exists

Since a missing key and an empty value both expand to nothing, use the `-v` test inside `[[ ]]` to ask whether a key exists at all. It is true even when the stored value is an empty string:

```
declare -A servers
servers[web1]="10.0.0.1"

if [[ -v servers[web1] ]]; then
  echo "web1 is registered"     # prints (web1 exists)
fi
if [[ -v servers[web9] ]]; then
  echo "web9 is registered"
else
  echo "web9 is not registered" # prints (web9 does not exist)
fi
```

## Counting occurrences

A neat use is tallying. An unseen key reads as empty, and empty behaves as zero in arithmetic, so you can count with no per-key setup:

```
declare -A counts
for user in alice bob alice carol alice bob; do
  counts[$user]=$(( counts[$user] + 1 ))
done

for user in "${!counts[@]}"; do
  echo "$user logged in ${counts[$user]} times"
done
```

```
alice logged in 3 times
bob logged in 2 times
carol logged in 1 times
```

The first `alice` makes the sum `0 + 1 = 1`, and each later `alice` adds one. Reach for an associative array whenever you want to find a value by a word rather than a position: hostnames to IPs, usernames to home directories, or config keyed by environment.

f
