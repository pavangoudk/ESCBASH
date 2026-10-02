# Creating, copying, and moving files

Three commands cover most of the "put a file here" moves you'll make.

## touch

`touch` creates an empty file.

```
touch /root/notes.txt
```

If the file already exists, `touch` updates its modification time and leaves the contents alone. That second behaviour is useful sometimes; usually you just want a new empty file.

## cp

`cp` copies a file from one place to another.

```
cp /etc/hostname /tmp/hostname-backup
```

If the destination is a folder, the file is placed inside it with the same name:

```
cp /etc/hostname /tmp/
```

Copying a whole folder needs `-r` (recursive):

```
cp -r /etc/apt /tmp/apt-backup
```

Without `-r`, `cp` refuses to copy directories.

## mv

`mv` is both "move to a different folder" and "rename in place". Linux doesn't distinguish, renaming a file is just moving it to a new name in the same folder.

```
mv /tmp/hostname-backup /tmp/hostname.txt      # rename
mv /tmp/hostname.txt /root/                    # move to /root
```

Unlike `cp`, `mv` handles folders without needing `-r`. Moving a folder just updates a pointer, so it's fast even for huge trees.


# Removing files safely

`rm` deletes files. It's the simplest command in this topic, and the most dangerous.

## The basics

In the previous  you created a file `/root/notes.txt` with `touch`. Let's delete it:

```
rm /root/notes.txt             # delete a file
```

You also created `/tmp/hostname` and `/root/hostname.txt` in that . You can delete several at once by listing them:

```
rm /tmp/hostname /root/hostname.txt   # delete several
```

`rm` refuses to delete folders unless you tell it to recurse. In the previous  you created the folder `/tmp/apt-backup` with `cp -r`. Let's delete it:

```
rm -r /tmp/apt-backup          # -r walks into subfolders
```

`-f` forces the delete without prompting, even for write-protected files. Combined with `-r`, it removes a whole folder without asking. Make a throwaway folder and delete it this way:

```
mkdir /tmp/scratch
rm -rf /tmp/scratch            # recursive AND forced
```

## The rm -rf trap

`rm -rf` on a real server is the mistake that gets people fired. Two things make it deadly:

1. There's no trash. Files are gone the second `rm` finishes.
2. It's typo-friendly. A stray space between `/` and the rest of the path turns a targeted delete into a delete of your entire operating system. These two commands are one keystroke apart:

```
rm -rf /root/oldproject
rm -rf / root/oldproject
```

Habits worth building before you press Enter on any `rm`:

- Read the path twice, especially the leading `/`.
- If you're deleting a folder, run `ls` on it first to see what's inside.
- Prefer specific paths over wildcards. Deleting one named log file is safer than deleting everything a wildcard happens to match:

```
rm /var/log/oldapp/2023.log      # one exact file
rm /var/log/oldapp/*             # every file in the folder
```

(The `*` wildcard is covered in the next node. Here it just means "every file in the folder".)

## -i for a safety net

`-i` (interactive) asks for confirmation before each delete. Make a file and try it, it will ask before removing:

```
touch /tmp/scratch.txt
rm -i /tmp/scratch.txt
```

Annoying for batch operations, but a reasonable default when you're new or when the delete is important.

# Wildcards and globs

Globs let you name many files at once with a pattern. Instead of typing out every filename, you describe them with a pattern, and the shell turns that pattern into the list of matching filenames before the command runs.

Imagine a folder full of files and you want to list only the ones that end in `.log`. Typing each name one by one would be tedious. A glob like `*.log` says "every file ending in `.log`" and does it in one shot.

## The two patterns you'll actually use

- `*` matches any number of characters (including zero).
- `?` matches exactly one character.

## Run the below commands to prepare your lab environment

```
mkdir -p /tmp/globs
cd /tmp/globs
touch a.log app.log notes.txt
```

Now you are in the `/tmp/globs` folder. It has three files: `a.log`, `app.log`, and `notes.txt`. Run `ls *.log` and you should see the two `.log` files:

```
ls *.log       # a.log app.log   (any name ending in .log)
```

Now try `?.log`. The `?` matches exactly one character, so it only matches `a.log` (a single `a` before `.log`). `app.log` has three characters before `.log`, so it is left out:

```
ls ?.log       # a.log           (one character, then .log)
```

And `*` on its own matches everything in the folder:

```
ls *           # a.log app.log notes.txt
```

## Common uses

Add a few more files to `/tmp/globs` so you can try the patterns you'll reach for most often:

```
touch server.conf db.conf
touch old_data.txt old_report.txt
touch access.log.gz error.log.gz
```

**Copy every `.conf` file into another folder.** `*.conf` matches `server.conf` and `db.conf`, so both get copied:

```
mkdir /tmp/globs/backup
cp *.conf /tmp/globs/backup/
ls /tmp/globs/backup/            # db.conf server.conf
```

**Move every file whose name starts with `old_`.** `old_*` matches `old_data.txt` and `old_report.txt`:

```
mkdir /tmp/globs/archive
mv old_* /tmp/globs/archive/
ls /tmp/globs/archive/          # old_data.txt old_report.txt
```

**List every gzipped file.** `*.gz` matches `access.log.gz` and `error.log.gz`:

```
ls -la *.gz                     # access.log.gz error.log.gz
```

## The rm + glob trap

The shell expands globs before the command sees them. A mistake in the pattern can widen the match without any warning:

```
rm *.tmp        # removes .tmp files. Good.
rm * .tmp       # removes EVERYTHING in the folder AND tries to remove ".tmp"
```

That extra space is all it takes. When deleting with a glob, always preview first:

```
ls *.tmp        # what will get matched?
rm *.tmp        # now delete
```

The preview costs nothing and catches mistakes before they matter.

← Previous
