# Archives and compression are two different jobs

Almost every file you download to a Linux server arrives as something like `nginx-1.27.tar.gz`. That name is two tools stacked on top of each other, and confusing them is the reason people guess at flags for years.

## Bundling: what tar does

**tar** takes many files and folders and writes them into one file, preserving names, permissions, timestamps, and the folder structure. It does **not** make anything smaller. The name is short for *tape archive* — it was invented to write a filesystem onto a tape in one long stream.

```
mkdir -p /root/demo/config
echo "print('hello')" > /root/demo/app.py
echo "port=8080" > /root/demo/config/settings.conf
tar -cf /root/demo.tar /root/demo
ls -l /root/demo.tar
```

One file now holds the whole folder. Its size is roughly the sum of what went in, plus a little bookkeeping.

## Squeezing: what gzip does

**gzip** takes one file and rewrites it smaller. It does **not** bundle. Point it at a file and it replaces that file with a `.gz` version:

```
gzip /root/demo.tar
ls -l /root/demo.tar.gz
```

Notice `demo.tar` is gone — gzip replaced it. `gunzip` reverses that:

```
gunzip /root/demo.tar.gz
ls -l /root/demo.tar
```

gzip only handles a single file. It has no idea what a folder is. That's exactly why the two tools are used together.

## Reading the file names

Now the extensions make sense:

| Name | What it is |
| --- | --- |
| `backup.tar` | A bundle, not compressed |
| `app.log.gz` | One compressed file, no bundle |
| `release.tar.gz` | A bundle that was then compressed |
| `release.tgz` | Same thing, shorter name |
| `source.tar.bz2`, `source.tar.xz` | Same idea, different compressor |

A `.tar.gz` is a folder tree turned into one file, then squeezed. Unpacking it is those two steps in reverse. In practice you never run them separately, because `tar` can call the compressor for you — that's the `z` flag you'll meet in the next node.

## Why DevOps work is full of these

- Release artifacts ship as one `.tar.gz` so a deploy downloads one thing.
- Backups are archived so a folder becomes one movable file.
- Rotated logs are gzipped in place — `app.log.1.gz` is the same log, just smaller, and `zcat` reads it without unpacking.
- Container image layers are tarballs underneath.

Compression matters here more than it looks: text compresses extremely well. A folder of logs or source code routinely shrinks to a fifth of its size, which is a fifth of the transfer time and a fifth of the storage bill.



# Creating and listing archives

`tar` has a reputation for cryptic flags. It's really four letters plus a couple of options, and once you can read them the commands stop looking like magic.

## The four verbs

Every tar command starts by picking exactly one job:

| Flag | Job |
| --- | --- |
| `-c` | **create** an archive |
| `-x` | **extract** an archive |
| `-t` | **list** (table of contents) without extracting |
| `-r` | append to an existing archive |

Then you add options:

| Flag | Meaning |
| --- | --- |
| `-f FILE` | the archive file to work on — nearly always required |
| `-z` | run it through gzip on the way in or out |
| `-v` | verbose: print each file as it's handled |
| `-C DIR` | change into DIR first |

`-f` must come last among the bundled letters, because the filename follows it. `tar -czf out.tar.gz src` works; `tar -cfz out.tar.gz src` does not — it would read `z` as the filename.

## Creating a compressed archive

Set up a folder to work with:

```
mkdir -p /root/site/css
echo "<h1>esc </h1>" > /root/site/index.html
echo "body { margin: 0 }" > /root/site/css/style.css
```

Bundle and compress it in one command:

```
tar -czvf /root/site.tar.gz -C /root site
```

```
site/
site/index.html
site/css/
site/css/style.css
```

Read it as: **c**reate, gzip (**z**), **v**erbosely, into the **f**ile `/root/site.tar.gz`, having first changed into `/root`, archiving `site`.

## Why -C matters when creating

That `-C /root site` is the difference between an archive that unpacks politely and one that doesn't. tar stores the paths exactly as you typed them. Archive it like this instead:

```
tar -czf /root/bad.tar.gz /root/site
tar -tzf /root/bad.tar.gz
```

```
root/site/
root/site/index.html
...
```

Every entry now carries the `root/` prefix. Extract that anywhere and you get a stray `root/` folder instead of a `site/` folder. The habit worth building: `cd` (or `-C`) to the parent, then name the folder relatively, so the archive contains one clean top-level directory.

## Look before you leap: -t

Listing an archive is cheap, safe, and reads nothing onto disk:

```
tar -tzf /root/site.tar.gz
```

Do this every single time before extracting something you didn't create. It answers the only two questions that matter: is there one tidy top-level folder, and are any paths absolute or full of `../`? The next node covers what happens when the answer is bad.

## Excluding files

Logs, caches, and `.git` folders rarely belong in a release artifact:

```
tar -czf /root/site.tar.gz --exclude='*.log' --exclude='.git' -C /root site
```

Quote the pattern. Unquoted, the shell would expand `*.log` against your current directory before tar ever sees it.

## Compressing a single file

When there's only one file, skip tar entirely:

```
gzip /root/site/index.html      # becomes index.html.gz, original gone
gunzip /root/site/index.html.gz # back again
gzip -k /root/site/index.html   # -k keeps the original too
```

And you can read a gzipped text file without unpacking it, which is how you grep a rotated log:

```
zcat /var/log/app.log.1.gz | tail -20
zgrep "ERROR" /var/log/app.log.1.gz
```



# Extracting where you meant to

Creating an archive is forgiving. Extracting one is where people make a mess of a server, because `tar -xzf` writes into your **current directory** and it will happily scatter hundreds of files across it.

## The basic extract

```
tar -xzf /root/site.tar.gz
```

E**x**tract, gzip (**z**), from the **f**ile. Everything is written relative to the directory you are currently in. If you ran that from `/`, you just wrote into `/`.

## Always name the destination with -C

```
mkdir -p /var/www
tar -xzf /root/site.tar.gz -C /var/www
```

Now the contents land under `/var/www` no matter what your current directory is. `-C` means "change into this directory first", and the directory must already exist — tar will not create it for you.

Making `-C` a reflex removes the entire class of "where did all these files come from?" incidents.

## The tarball that has no top-level folder

A well-made archive holds one folder:

```
site/
site/index.html
site/css/style.css
```

Extract that anywhere and you get one tidy `site/` directory. A badly made one holds its files loose at the top:

```
index.html
css/style.css
README.md
```

Extract *that* into a folder you care about and its files mix straight in with what's already there, silently overwriting any name that collides. This is called a **tarball bomb**, and it's the reason `tar -tzf` before extracting is worth the two seconds.

The safe move when the listing looks loose is to extract into an empty directory of your own:

```
mkdir -p /tmp/unpack
tar -xzf /root/mystery.tar.gz -C /tmp/unpack
ls /tmp/unpack
```

Now look at what you got, then move it where it belongs.

## Dropping a wrapper folder with --strip-components

The opposite annoyance: an archive whose contents sit inside a version-stamped folder you don't want.

```
nginx-1.27.0/
nginx-1.27.0/conf/
nginx-1.27.0/src/
```

You want `conf/` and `src/` directly in `/opt/nginx`, not `/opt/nginx/nginx-1.27.0/`. `--strip-components=1` removes the first path element from every entry as it extracts:

```
tar -xzf nginx-1.27.0.tar.gz -C /opt/nginx --strip-components=1
```

This is extremely common in Dockerfiles and install scripts, where the version number in the folder name would otherwise leak into every path downstream.

## Extracting one file out of an archive

You don't have to unpack everything. Name the path exactly as it appears in the listing:

```
tar -tzf /root/site.tar.gz
tar -xzf /root/site.tar.gz -C /tmp site/index.html
```

Handy when you need one config file out of a 2 GB backup.

## Extracting doesn't consume the archive

Unlike bare `gzip`, which replaces its input, `tar -x` leaves the archive exactly where it was. After a successful extract the `.tar.gz` is still sitting there, and deleting it is a separate, deliberate decision.

## The habit, in three lines

```
tar -tzf archive.tar.gz | head        # 1. look inside
mkdir -p /path/to/destination         # 2. make the destination
tar -xzf archive.tar.gz -C /path/to/destination   # 3. extract there
```

Look, prepare, extract. That order is the whole .

