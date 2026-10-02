# Package Management in Linux

## 1. What is a Package?

A **package** is a ready-to-install software bundle.

It contains things such as:

* Program files
* Dependencies
* Configuration files
* Documentation/man pages

For example, if you want to install **Nginx**, instead of manually downloading everything it needs, you can install it using one command:

```bash
sudo apt install nginx
```

The package manager takes care of the dependencies and installation.

### Simple example

Think of a package like a **ready-made food kit**.

Instead of buying every ingredient separately and preparing everything yourself, the kit already contains what you need.

---

# 2. What is a Package Manager?

A **package manager** is a tool used to:

* Install software
* Update software
* Remove software
* Manage dependencies
* Keep track of installed software

For Ubuntu, the main package manager is:

```text
apt
```

Another lower-level tool is:

```text
dpkg
```

---

# 3. Why Do We Need a Package Manager?

Without a package manager, installing Nginx could involve:

1. Downloading the source code
2. Finding dependencies
3. Installing those dependencies
4. Compiling the application
5. Copying files to correct locations
6. Configuring the application
7. Repeating everything when an update is released

With a package manager:

```bash
sudo apt install nginx
```

That's it.

The package manager handles most of the work for us.

---

# 4. Two Major Package Management Families

Linux distributions mainly fall into two package-management families.

### Debian-based

Examples:

* Debian
* Ubuntu
* Linux Mint

Package format:

```text
.deb
```

Common tools:

```text
apt
dpkg
```

### Red Hat-based

Examples:

* RHEL
* Fedora
* CentOS
* Amazon Linux

Package format:

```text
.rpm
```

Common tools:

```text
dnf
yum
```

### Easy way to remember

```text
Ubuntu
  ↓
.deb
  ↓
apt / dpkg
```

```text
RHEL / Fedora / Amazon Linux
  ↓
.rpm
  ↓
dnf / yum
```

The commands are different, but the **concept is the same**.

---

# 5. Where Does apt Get Packages From?

`apt` gets packages from **repositories**.

A repository is basically a location/server containing software packages.

Ubuntu's repository configuration is mainly found in:

```text
/etc/apt/sources.list
```

and:

```text
/etc/apt/sources.list.d/
```

For example:

```text
Your Ubuntu Server
       ↓
      apt
       ↓
Repository
       ↓
Nginx package
       ↓
Dependencies
       ↓
Installed on server
```

Ubuntu normally comes with official repositories already configured.

---

# 6. apt update

Command:

```bash
sudo apt update
```

### What does it do?

It **refreshes the package list**.

It does NOT install anything.

Think of it like:

> "Check the repository and tell me what packages and versions are currently available."

Your server maintains a local copy of the package information.

That information can become outdated.

So we run:

```bash
sudo apt update
```

to refresh it.

### Important Interview Question

**Does `apt update` update installed packages?**

No.

It only updates the **package information/index**.

---

# 7. apt install

To install a package:

```bash
sudo apt install htop
```

What happens?

```text
apt
 ↓
Find htop
 ↓
Download htop
 ↓
Find dependencies
 ↓
Download dependencies
 ↓
Install everything
```

### Install without confirmation

Normally apt may ask:

```text
Do you want to continue? [Y/n]
```

Using:

```bash
sudo apt install -y htop
```

automatically answers **Yes**.

This is very important in:

* Shell scripts
* CI/CD pipelines
* Automation
* Dockerfiles
* Server provisioning

You can install multiple packages together:

```bash
sudo apt install -y htop curl vim
```

---

# 8. apt upgrade

To upgrade installed packages:

```bash
sudo apt upgrade
```

This applies available updates to installed packages.

You can also upgrade a specific package:

```bash
sudo apt upgrade htop
```

### DevOps point

On production servers, updates are normally handled through a controlled maintenance/patching process rather than randomly upgrading packages.

---

# 9. apt remove

To remove a package:

```bash
sudo apt remove htop
```

This removes the application but generally keeps its configuration files.

Why?

Because you may want to reinstall it later using the same configuration.

---

# 10. apt purge

If you want to remove the package **and its configuration files**:

```bash
sudo apt purge htop
```

Think:

```text
remove
→ Remove application

purge
→ Remove application + configuration
```

---

# 11. apt autoremove

Sometimes installing an application also installs dependencies.

Later, when you remove the application, some dependencies may no longer be needed.

You can clean them using:

```bash
sudo apt autoremove
```

Think:

```text
Application
   ↓
Dependencies
   ↓
Application removed
   ↓
Some dependencies unused
   ↓
apt autoremove
```

---

# 12. Most Common Pattern on a New Server

When you get a new Ubuntu server, a very common sequence is:

```bash
sudo apt update
sudo apt install -y htop
```

First:

```bash
sudo apt update
```

Refresh the package information.

Then:

```bash
sudo apt install -y htop
```

Install the package.

You can also write:

```bash
sudo apt update && sudo apt install -y htop
```

`&&` means:

> Run the second command only if the first command succeeds.

---

# 13. Searching for Packages

Sometimes you don't know the exact package name.

Use:

```bash
apt search nginx
```

It searches the available package information for packages related to `nginx`.

For example, you might find:

```text
nginx
nginx-common
nginx-core
```

If your package information is outdated, first run:

```bash
sudo apt update
```

---

# 14. apt show

Suppose you want more information about Nginx before installing it.

Use:

```bash
apt show nginx
```

It can show:

* Description
* Version
* Dependencies
* Download size
* Installed size

Two important fields are:

```text
Version:
Depends:
```

`Depends:` tells you about packages that this package requires.

---

# 15. What is dpkg?

`dpkg` is the **lower-level package management tool** for Debian-based systems.

Think of it like this:

```text
apt
 ↓
Higher-level package management
 ↓
dpkg
 ↓
Low-level package management
```

`apt` is commonly used for installing packages from repositories.

`dpkg` is useful for inspecting installed packages and working directly with `.deb` packages.

---

# 16. dpkg -l

To see installed packages:

```bash
dpkg -l
```

There can be a huge number of packages, so we can combine it with `grep`.

For example:

```bash
dpkg -l | grep nginx
```

You might see:

```text
ii  nginx  1.24.0-...  amd64  small, powerful, scalable web server
```

The `ii` means the package is installed and configured.

---

# 17. dpkg -L

Suppose Nginx is installed and you want to know:

> "Where are all the files installed by Nginx?"

Use:

```bash
dpkg -L nginx
```

You might see:

```text
/usr/sbin/nginx
/etc/nginx/nginx.conf
/usr/share/nginx
```

This is very useful in real-time troubleshooting.

For example, if you don't remember where the Nginx configuration file is, instead of guessing:

```bash
dpkg -L nginx
```

---

# 18. dpkg -S

This is the reverse of `dpkg -L`.

Suppose you find this file:

```text
/usr/sbin/nginx
```

and want to know:

> "Which package installed this file?"

Use:

```bash
dpkg -S /usr/sbin/nginx
```

Output:

```text
nginx: /usr/sbin/nginx
```

So you know the file belongs to the `nginx` package.

---

# 19. Very Important Difference

Remember this simple distinction:

### apt

Used mainly for:

```text
Install
Update
Upgrade
Remove
Search
Inspect packages from repositories
```

### dpkg

Used mainly for:

```text
Inspect installed packages
List package files
Find package ownership
Work with .deb packages
```

---

# 20. Commands You Should Remember

### Update package information

```bash
sudo apt update
```

### Install

```bash
sudo apt install nginx
```

### Install without confirmation

```bash
sudo apt install -y nginx
```

### Upgrade

```bash
sudo apt upgrade
```

### Remove

```bash
sudo apt remove nginx
```

### Remove + configuration

```bash
sudo apt purge nginx
```

### Remove unused dependencies

```bash
sudo apt autoremove
```

### Search

```bash
apt search nginx
```

### Package details

```bash
apt show nginx
```

### Check installed package

```bash
dpkg -l | grep nginx
```

### Find files installed by package

```bash
dpkg -L nginx
```

### Find which package owns a file

```bash
dpkg -S /usr/sbin/nginx
```

---

# 21. Real-Time DevOps Scenario

Imagine you get a **new Ubuntu VM**.

You need to troubleshoot the server, so you want `curl`, `vim`, and `htop`.

You would typically do:

```bash
sudo apt update
```

Then:

```bash
sudo apt install -y curl vim htop
```

Check whether `htop` was installed:

```bash
dpkg -l | grep htop
```

Later, you want to know where its files are:

```bash
dpkg -L htop
```

And if you find a particular file and want to know which package installed it:

```bash
dpkg -S /path/to/file
```

### The overall flow

```text
New Ubuntu VM
      ↓
apt update
      ↓
Refresh package information
      ↓
apt install
      ↓
Install application + dependencies
      ↓
dpkg -l
      ↓
Check installed package
      ↓
dpkg -L
      ↓
Find package files
      ↓
dpkg -S
      ↓
Find package owner
```

## ⭐ Interview Memory Trick

Just remember:

**`update` → information**

**`install` → install**

**`upgrade` → update installed software**

**`remove` → remove software**

**`purge` → remove software + config**

**`autoremove` → remove unused dependencies**

**`search` → find package**

**`show` → package details**

**`dpkg -l` → what's installed?**

**`dpkg -L` → what files did it install?**

**`dpkg -S` → who owns this file?**
