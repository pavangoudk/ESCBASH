# vim basics

`vim` is one of the most popular choices for editing files on Linux, and it comes installed on almost every system. That means sooner or later you will work on a server where vim is the editor you have to use.

Beginners often find vim confusing at first, many people get stuck and can't even figure out how to close it. But that is only because of one idea nobody explains up front. Once you understand it, vim becomes easy.

This  gives you just enough to open a file, change it, save, and quit with confidence.

## Why vim feels weird: modes

In an editor like Notepad, the keyboard always does one thing, you press `a` and an `a` appears on screen. vim is different. vim has **modes**, and the same key does different things depending on which mode you are in.

There are two modes you need to know:

- **Normal mode** — the mode vim starts in. Here the keys are *commands*, not text. Pressing `i`, `d`, or `x` does something like insert, delete, or move, it does **not** type those letters into the file. This is the part that surprises beginners: you start typing and nothing appears the way you expect.
- **Insert mode** — the mode where the keyboard behaves like Notepad. Whatever you type shows up in the file. You switch into insert mode on purpose, edit your text, then switch back out.

The whole trick to vim is knowing which mode you are in and how to move between the two.

## Step by step: edit a file

Let's walk through it slowly. Open (and create) a file:

```
vim /tmp/hello.txt
```

vim opens with an empty screen. **You are in normal mode.** If you start typing now, you will get strange behaviour, that is expected, don't panic. First switch to insert mode.

**1. Press `i`.** This enters insert mode. Look at the bottom-left of the screen: it now says `-- INSERT --`. That is vim telling you the keyboard will now type text.

**2. Type some text**, for example:

```
Hello from vim
This is my first edit
```

The letters appear as you type, just like a normal editor.

**3. Press `Esc`.** This leaves insert mode and returns to normal mode. The `-- INSERT --` label disappears. You are now back in "commands" mode, ready to save.

**4. Save the file.** In normal mode, type `:w` and press `Enter`. The `:` moves your cursor to the bottom line, `w` means "write" (save). vim confirms the file was written.

**5. Quit.** Type `:q` and press `Enter`. You are back at the shell.

That is a full edit: open, `i` to insert, type, `Esc`, `:w`, `:q`.

## The commands worth memorising

All of these are typed in **normal mode** (press `Esc` first if you're not sure):

- **`i`** — enter insert mode, so you can type.
- **`Esc`** — leave insert mode, back to normal mode.
- **`:w`** — write (save) the file. Press `Enter` after.
- **`:q`** — quit. Won't work if you have unsaved changes.
- **`:wq`** — save and quit in one step.
- **`:q!`** — quit and throw away unsaved changes. Your escape hatch when you opened something by accident or made a mess.

Two editing commands that pay off often, both in normal mode:

- **`dd`** — delete the whole current line.
- **`u`** — undo the last change.

## Stuck? How to get out of vim

This is the moment everyone hits. You're in vim, you don't know what mode you're in, and nothing works. Do this:

1. Press `Esc` two or three times. No matter what mode you were in, you are now in normal mode.
2. Type `:q!` and press `Enter`.

`:q!` quits and discards any changes, so it always works, even if vim is refusing to quit because of unsaved edits. If you *do* want to keep your changes, use `:wq` instead.

## The mental model to keep

- Can't type text? You're in normal mode, press `i`.
- Typed a command and it went into the file as letters? You were in insert mode, press `Esc` first, then the command.
- Want out no matter what? `Esc`, then `:q!`.

That's enough vim to edit configs on any Linux box in the world. Come back to it properly later if you want more, but this is the floor, and it is genuinely all you need to survive.



# nano

`nano` is a text editor that behaves the way most people expect. Arrow keys move the cursor, typing types, and every keystroke that isn't obvious is listed at the bottom of the screen.

## Opening a file

```
nano /root/notes.txt
```

If the file doesn't exist, nano opens an empty buffer and creates the file when you save. If it exists, nano loads it and you can edit.

## The two keystrokes you need

Look at the bottom of the nano screen. Every option is listed as `^X`, where `^` means the Ctrl key.

- **`Ctrl+O`** - write (save) the current buffer. nano asks for the filename; press Enter to confirm.
- **`Ctrl+X`** - exit. If you have unsaved changes, nano offers to save them first.

Two more worth knowing:

- **`Ctrl+K`** - cut the current line.
- **`Ctrl+W`** - search for text. Press Enter and nano jumps to the next match.

## When nano isn't installed

Some minimal container images and older server images ship without nano. If `nano` returns "command not found", either fall back to `vim` (which is on almost every Linux system by default) or install nano on Ubuntu with:

```
apt install nano
```

(You'll learn `apt`, the package installer, in a later topic. For now just know this is how you'd add nano if it's missing.)

The previous node covered just enough vim to survive that situation.

## Writing an executable script

Once you can edit files, you'll eventually write a **shell script** - a text file that Linux can execute like any other command. Two things make a text file a runnable script:

- The very first line is a **shebang**, written `#!` followed by the path to the interpreter. `#!/bin/` tells Linux "run this file with ". The line must be the first line, with no blank line above it and no space after `#!`.
- The file must be marked **executable** (a permission mode you'll set in the next topic with `chmod`).

A minimal runnable script:

```
#!/bin/
echo "hello from a script"
```

Save that in nano, mark it executable, and Linux runs it like any other command. Any other `#` on any other line is a plain comment ( ignores everything from `#` to the end of that line). Only the very first line, starting with `#!`, is the shebang.



# Edits without opening an editor

Not every file edit needs an interactive editor. Three tools cover in-place changes without opening nano or vim: the append operator `>>`, heredocs, and `sed -i`.

## Append with >>

You already know `>` overwrites a file with new output. Its two- arrow sibling `>>` appends instead:

```
echo "127.0.0.1 example.local" >> /etc/hosts
```

That adds a new line at the end of `/etc/hosts` without touching the existing contents. Great for adding entries. Useless for changing something already in the file.

## Multiple lines with a heredoc

A heredoc lets you write a block of text straight into a file:

```
cat << EOF > /root/hello.txt
first line
second line
third line
EOF
```

Everything between `<< EOF` and the closing `EOF` becomes the file's contents. Handy for generating small config files from a shell script.

## sed -i for in-place substitution

`sed` is a stream editor. With `-i` (in-place) it edits a file directly, no editor needed:

```
echo "mode: old" > /root/config.yaml
sed -i 's/old/new/' /root/config.yaml
```

That reads: substitute the first occurrence of `old` with `new` on every line. Add `g` at the end (`s/old/new/g`) to replace every occurrence on each line, not just the first.

## Always write a backup

`sed -i` has no undo. Use `-i.bak` to keep the original around:

```
sed -i.bak 's/listen 80/listen 8080/' /etc/nginx/sites-available/default
```

That writes the change to the file AND saves the original as `default.bak` in the same folder. If your search pattern matches more than you meant, `mv /etc/nginx/sites-available/default.bak /etc/nginx/sites-available/default` puts things back.

