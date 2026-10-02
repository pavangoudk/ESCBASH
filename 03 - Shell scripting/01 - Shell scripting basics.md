# What is a shell script

LessonA shell script is a plain text file that holds a list of shell commands, one per line. Instead of typing each command at the terminal, you write them all in a file and run the file. The shell reads it top to bottom and runs each command in order, just as if you had typed them.

Every line is a normal command you already know. Why bother? You often need to run the same set of commands more than once, and typing them out each time is slow and error-prone. Put them in a script once, and run the whole thing any time with a single command.

## The shape of a script

```
#!/bin/bash
# print a friendly greeting
echo "hello, DevOps"
```

Three things to notice:

- The first line, starting with `#!`, is the **shebang**. It tells Linux which program should run this file. `#!/bin/bash` says "run this with bash."
- The line starting with `#` (any `#` other than the shebang) is a **comment**. Comments are little notes for humans reading the script. The shell ignores everything from a `#` to the end of that line, so a comment never runs as a command.
- `echo` prints text to the terminal. Here it prints `hello, DevOps`. You will use `echo` a lot in the coming lessons, so just remember for now: `echo` writes text to the screen.

## File extension is convention only

Scripts are usually named `something.sh`, but the `.sh` extension is decorative. Linux doesn't care about the extension, it cares about the shebang and whether the file is executable. You could name your script `deploy` (no extension) and it would work identically.

That's the whole reality: a shell script is a text file with a shebang that Linux is allowed to run.

← Previous

# Making a script executable and running it

LessonA file being full of commands and a file being allowed to run are two separate things. Saving a script is not enough; you have to grant permissions to make it executable. In this lesson you grant that permission, then see the ways to actually run it.

Create a small script to work with:

```
echo '#!/usr/bin/env bash' > hello.sh
echo 'echo "hello from the script"' >> hello.sh
```

That writes a two-line `hello.sh`: the shebang, then a line that prints a message. Nothing runs yet.

## Marking the file as executable with chmod +x

A brand new file has no permission to run, so running it directly is refused:

```
./hello.sh              # prints: bash: ./hello.sh: Permission denied
```

The error is about permissions, not contents. Linux keeps an "execute" flag on every file, and on a new text file it is off. You already learned about `chmod` back in the Linux skill. Use it to turn that flag on:

```
chmod +x hello.sh
```

`+x` means "add the execute permission." Nothing is printed, but the file is now runnable. Confirm with a long listing:

```
ls -l hello.sh         # prints: -rwxr-xr-x 1 root root 45 ... hello.sh
```

Those `x` letters are the execute flags your `chmod +x` set. Now it works:

```
./hello.sh             # prints: hello from the script
```

You only need `chmod +x` once per script. The flag stays set unless you remove it.

## The ways to run a script

Once executable, there is more than one way to run it. They run the same commands but differ in how Linux finds the file and which program reads it:

```
./hello.sh             # from the current folder; needs the execute flag
bash hello.sh          # hand it to bash; execute flag NOT needed
/root/hello.sh         # full path, works from anywhere; needs the flag
hello.sh               # prints: bash: hello.sh: command not found
```

- **`./hello.sh`** is the everyday way to run a script in the folder you are in.
- **`bash hello.sh`** hands the file to bash yourself, so it reads the file even without the execute flag. Handy for a quick test.
- **`/root/hello.sh`** gives the complete path, so Linux finds it no matter where you are.
- **`hello.sh`** (bare name) fails, because a bare name is only searched for in the folders on `PATH`, and your current folder is not one of them. Tools like `ls` and `git` work by bare name because they live in folders that are on `PATH`, like `/usr/local/bin`.

The common DevOps pattern is that split: run `./script.sh` while writing and testing, then copy it into a `PATH` folder (like `/usr/local/bin/`) once it is ready so anyone can call it by name.

## What the "./" is actually doing

`./` means "this folder, right here." So `./hello.sh` means "the file `hello.sh` in the folder I am standing in." By naming a location, even a tiny one, you tell Linux exactly where the file is so it never searches `PATH`. That is why `hello.sh` fails but `./hello.sh` works.

← Previous
