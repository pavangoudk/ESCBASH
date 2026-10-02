# What are links

A **link** is an alternate name for a file. Linux has two flavors, and they behave very differently.

## Hard links

Every file on a filesystem is stored at an **inode**, a numeric ID. The "name" of a file is a directory entry that points at that inode. A hard link is just another directory entry pointing at the same inode.

```
echo hi > original.txt          # a file to link to
ln original.txt copy.txt        # creates a hard link
ls -li                          # -i shows the inode number
```

Both `original.txt` and `copy.txt` show the same inode. They're two names for one file. Delete one, the file is still there under the other name. Edit one, both "see" the change because there's only one file underneath.

Limitations:

- Hard links can't cross filesystems (an inode is only meaningful within one filesystem).
- Hard links can't point at directories (with rare exceptions).

Hard links are useful for deduplication and backup tools. In daily DevOps work, you'll rarely create one on purpose.

## Symbolic links (symlinks)

A **symlink** is a small file that holds a path pointing at another file or folder. Following a symlink is like following a shortcut: you land on whatever that path names.

Make a folder with a file in it, then point a symlink at the folder:

```
mkdir -p /root/releases/v2
echo "v2" > /root/releases/v2/version.txt
ln -s /root/releases/v2 /root/current
```

Now `/root/current` points at `/root/releases/v2`. Reading through the link reaches the real file underneath:

```
cat /root/current/version.txt
```

```
v2
```

The important detail: a symlink stores the target *path*, not the inode. It's a signpost, not a second name for the file. That's the key difference from a hard link.

## Telling them apart in a listing

```
ls -l /root/current
```

A symlink shows an `l` as the first character and an arrow to its target:

```
lrwxrwxrwx 1 root root 17 Jul 28 10:30 /root/current -> /root/releases/v2
```

Regular files show `-`, directories show `d`, and hard links look exactly like regular files (because they ARE regular files, just under a different name).

## When the target is deleted or moved

Because a symlink only remembers a path, it breaks the moment the thing at that path goes away. Delete the target and read through the link again:

```
rm -r /root/releases/v2
cat /root/current/version.txt
```

```
cat: /root/current/version.txt: No such file or directory
```

The link still exists, but it now points at nothing. A hard link would have survived this, because it names the inode directly and the file's data stays until the last name is gone. A symlink just remembers where the target used to be. Moving the target somewhere else breaks it the same way.



# Using symlinks

`ln -s target link_name` creates a symlink. Read it left to right as "make link_name a symbolic link to target". The first path is what already exists; the second is the new name you are creating.

The argument order trips up almost everyone at first. `ln -s` writes the first path into the new link exactly as you typed it, so swapping the two arguments makes a link with the wrong name pointing at the wrong place, and it usually doesn't error. Say it the same way every time: target first, then the name of the link.

```
mkdir -p /root/releases/v2
echo "v2" > /root/releases/v2/version.txt
ln -s /root/releases/v2 /root/current
readlink /root/current
```

```
/root/releases/v2
```

## The "current release" pattern

This is the single most common use of symlinks in modern DevOps. You deploy versioned releases to their own folders, and a symlink points at whichever one is live:

```
/var/www/releases/2026-07-01/
/var/www/releases/2026-07-14/
/var/www/current -> /var/www/releases/2026-07-14
```

Deploys become: extract the new release, update the symlink, reload nginx. Rollbacks become: point the symlink at the previous release. No file copies, no downtime.

## Pointing at a specific version of a tool

Ubuntu's `python3` is a symlink to `python3.12`. Same trick, same benefit:

```
ls -l /usr/bin/python3
# lrwxrwxrwx 1 root root 10 ... /usr/bin/python3 -> python3.12
```

Swap the symlink to change which version `python3` resolves to.

## Replacing an existing symlink

The classic gotcha. Say a new release lands and you want `/root/current` to point at it. Create the new folder and try a plain `ln -s`:

```
mkdir -p /root/releases/v3
echo "v3" > /root/releases/v3/version.txt
ln -s /root/releases/v3 /root/current
readlink /root/current
```

```
/root/releases/v2
```

The link still points at v2. Plain `ln -s` did not overwrite it. Because `/root/current` already points at a directory, `ln` followed it and quietly created a stray `v3` link *inside* the v2 folder instead. Not what you meant.

The fix is `ln -sfn`:

```
ln -sfn /root/releases/v3 /root/current
readlink /root/current
```

```
/root/releases/v3
```

- `-s` symbolic
- `-f` force (overwrite the link if one already exists)
- `-n` treat the existing link as a plain file, don't follow it into the directory it points at

`ln -sfn` is the exact one-liner every zero-downtime deploy script uses to swap the live release.

## Finding what a symlink points at

`readlink` prints the target path a symlink holds:

```
readlink /root/current
```

To follow the whole chain down to the real file, add `-f`:

```
readlink -f /root/current
```

## Broken symlinks

If the target no longer exists, the symlink is called **broken**. Listings still show the link, but reading through it fails. Delete the target and try:

```
rm -r /root/releases/v3
cat /root/current/version.txt
```

```
cat: /root/current/version.txt: No such file or directory
```

Find broken symlinks under a folder:

```
find /root -xtype l
```

`-xtype l` matches symlinks whose target doesn't exist. Very useful during a bad-deploy cleanup.

