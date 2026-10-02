# Default values for variables

Sometimes a variable has no value, because it was never set. Using it as-is just gives you an empty string:

```
echo "hello $name"    # prints: hello
```

`name` was never set, so you get `hello` and a blank where the name should be.

Usually you would rather have a backup value ready: use `name` if it has a value, otherwise use `guest`. Parameter expansion lets you write that in one short expression, with no `if` statement. You put the variable inside `${...}` with a special symbol in the middle, and that symbol supplies the backup value when the variable is empty.

## `${var:-default}` gives a fallback without changing anything

The one you will reach for most: if `var` has a value, use it, otherwise use `default` this one time. It does NOT change `var` itself.

```
name="alice"
echo "${name:-guest}"      # prints: alice   (name has a value)

name=""                    # now change name to an empty string
echo "${name:-guest}"      # prints: guest   (unset, so use the fallback)
echo "$name"               # prints:         (empty)
```

`name` is still empty afterward; the fallback applied only to that one `echo`. To keep the value, assign it back:

```
port="${port:-8080}"
echo "$port"               # prints: 8080    (if port was never set)
```

## `${var:=default}` fills in the blank permanently

Almost identical, but the `=` also ASSIGNS the default back into the variable, so afterward it is guaranteed to hold a value:

```
unset greeting             # unset greeting is same as greeting=""
echo "${greeting:=hello}"  # prints: hello   (greeting was unset)
echo "$greeting"           # prints: hello   (and now greeting holds it too)
```

If the variable already has a value, the default is ignored:

```
greeting="hi there"
echo "${greeting:=hello}"  # prints: hi there   (already set, default ignored)
```

## `${var:?message}` stops the script when something required is missing

For a value you cannot paper over with a default. If `var` is unset or empty, `:?` prints your message to stderr and exits the script:

```
unset api_key
echo "${api_key:?api_key must be set before running this script}"
# prints to stderr: : api_key: api_key must be set before running this script
# and the script stops right here
```

If `api_key` has a value, the expression gives it back and the script continues. It replaces writing `if [ -z "$api_key" ]; then ... exit 1; fi` by hand.

## `${var:+alternate}` uses a value only when set

This flips the logic: if `var` IS set and non-empty, use `alternate` instead of the real value; if `var` is unset or empty, the whole thing becomes empty.

```
debug="yes"
echo "${debug:+--verbose}"   # prints: --verbose   (debug is set)

unset debug
echo "${debug:+--verbose}"   # prints:              (empty - debug not set)
```

Handy for optional flags. You do not care what `debug` holds, only whether it is set at all:

```
unset debug
echo "run${debug:+ in debug mode}"   # prints: run
debug="1"
echo "run${debug:+ in debug mode}"   # prints: run in debug mode
```

## Why the colon matters

Every operator above has a colon: `:-`, `:=`, `:?`, `:+`. The colon means "treat an empty value the same as unset." Drop the colon and the operator triggers only when the variable is truly unset:

```
name=""
echo "${name:-guest}"      # prints: guest   (colon: empty counts as missing)
echo "${name-guest}"       # prints:         (no colon: empty counts as set)
```

Usually empty and missing mean the same thing to you, so keep the colon when in doubt.



# Length and substrings

Sometimes you want to know how long a value is, or pull out just a piece of it.  builds both directly into parameter expansion, so you can measure and slice a string without piping it through `wc`, `cut`, or `awk`.

## Length with `${#var}`

Put a `#` right after the opening brace, and  gives you the number of characters, spaces included:

```
name="alice"
echo "${#name}"        # prints: 5

empty=""
echo "${#empty}"       # prints: 0

greeting="hi there"
echo "${#greeting}"    # prints: 8   (2 + 1 space + 5)
```

A common use is checking a minimum length. The `(( ... ))` below is 's way of comparing numbers (covered properly in a later ); read it as "if the length is less than 8":

```
password="hunter2"
if (( ${#password} < 8 )); then
  echo "password too short"    # prints: password too short   (it is 7 characters)
fi
```

## Substrings with `${var:offset:length}`

Give  where to start and how many characters to take. Counting starts at 0, not 1: the first character is at position 0, the second at position 1, and so on.

```
text="Hello, DevOps World"
#      position 0 is 'H', position 7 is 'D'
echo "${text:0:5}"     # prints: Hello    (start at 0, take 5 characters)
echo "${text:7:6}"     # prints: DevOps   (start at 7, take 6 characters)
```

Leave off the length and  takes everything from that position to the end:

```
echo "${text:7}"       # prints: DevOps World   (from position 7 to the end)
```

A negative offset counts backwards from the end. One catch: put a space before the minus sign, or  reads `${text:-5}` as the default-value operator from the previous :

```
echo "${text: -5}"     # prints: World   (last 5 characters - note the space)
```

A real use is shortening a git commit hash to its first 7 characters:

```
git_sha="1a2b3c4d5e6f7g8h9i0j"
echo "${git_sha:0:7}"  # prints: 1a2b3c4   (the short commit hash)
```

You can slice any fixed-format string this way - dates, IDs, codes - without an external tool.

