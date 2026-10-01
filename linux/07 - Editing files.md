# Linux Editing Notes: Vim, Nano, and Shell Editing

## 1. Vim basics

Vim uses modes:

- **Normal mode:** Keys perform commands.
- **Insert mode:** Keys type text like a standard editor.

### Basic editing workflow

```
vim /tmp/hello.txt
```

1. Press `i` to enter Insert mode.
2. Type your text.
3. Press `Esc` to return to Normal mode.
4. Type `:w`, then press Enter to save.
5. Type `:q`, then press Enter to quit.

### Essential Vim commands

| Command | Purpose |
| --- | --- |
| `i` | Enter Insert mode |
| `Esc` | Return to Normal mode |
| `:w` | Save the file |
| `:q` | Quit |
| `:wq` | Save and quit |
| `:q!` | Quit without saving |
| `dd` | Delete the current line |
| `u` | Undo the last change |

### Emergency exit

If you are stuck:

1. Press `Esc` several times.
2. Type `:q!`.
3. Press Enter.

Use `:wq` instead if you want to save your changes.

**Mental model:**

`i` lets you type; `Esc` lets you issue commands.

## 2. Nano basics

Nano behaves more like a traditional text editor. You can type immediately, use arrow keys, and see commands at the bottom of the screen.

Open a file with:

```
nano /root/notes.txt
```

### Essential Nano shortcuts

| Shortcut | Purpose |
| --- | --- |
| `Ctrl+O` | Save/write the file |
| `Ctrl+X` | Exit Nano |
| `Ctrl+K` | Cut the current line |
| `Ctrl+W` | Search for text |

In Nano, `^` means the **Ctrl** key. For example, `^X` means `Ctrl+X`.

If Nano is unavailable on Ubuntu, install it with:

```
apt install nano
```

Vim is often available when Nano is not.

## 3. Writing executable shell scripts

A shell script needs two things:

1. A shebang as the **first line**: `#!/bin/bash`
2. Executable permissions, typically added later with `chmod`

A minimal script should contain `#!/bin/bash` on line one, followed by `echo "hello from a script"`.

Important points:

- The shebang must be the first line.
- `#!/bin/bash` tells Linux to use Bash.
- Other lines beginning with `#` are comments.
- The file must be marked executable before it can run directly.

## 4. Editing without an interactive editor

### Append with `>>`

The `>>` operator adds output to the end of a file without overwriting existing content.

```
echo "127.0.0.1 example.local" >> /etc/hosts
```

- `>` overwrites a file.
- `>>` appends to a file.

### Heredocs

A heredoc writes multiple lines directly into a file.

```
cat << EOF > /root/hello.txt
first line
second line
third line
EOF
```

Everything between the opening and closing `EOF` becomes the file content.

### `sed -i` substitution

`sed -i` changes a file directly without opening an editor.

```
echo "mode: old" > /root/config.yaml
sed -i 's/old/new/' /root/config.yaml
```

The expression `s/old/new/` replaces the first occurrence of `old` on each line.

To replace every occurrence on each line, use the `g` flag:

```
sed -i 's/old/new/g' /root/config.yaml
```

### Create a backup before editing

Because `sed -i` has no built-in undo, use a backup suffix:

```
sed -i.bak 's/listen 80/listen 8080/' /etc/nginx/sites-available/default
```

This creates a backup file ending in `.bak`.

## Quick comparison

| Tool | Best for |
| --- | --- |
| Vim | Powerful editing on almost any Linux system |
| Nano | Simple interactive editing |
| `>>` | Appending one line |
| Heredoc | Creating a multi-line file |
| `sed -i` | Automated in-place replacements |
