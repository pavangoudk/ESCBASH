# Why containers use environment variables

You built one image, and now you want to run it in three places: your laptop for testing, a staging server, and production. The trouble is that each place needs slightly different settings - a different database password, a different log level, a different API URL to talk to.

You do not want to build three separate images for this. That would be three things to keep in sync every time the app changes. You want one image that behaves differently depending on where it runs. So where do those per-place settings come from? Environment variables.

## The same env vars you already know from Linux

On a Linux server you have set environment variables many times - things like `PATH` or a value you `export` in your shell. They are just named values a program can read while it runs: a name, an equals sign, and a value, like `LOG_LEVEL=debug`.

Docker uses the exact same idea. When you start a container you hand it a few of these `KEY=value` pairs, and the program inside can read them.

## Keep settings out of the image, hand them in at start

A good image is built once and then left alone. Anything that changes from one place to another - the password, the log level, the URL - is not baked into the image. Instead you supply it from the outside each time you start the container.

That keeps the image generic, so the same image runs anywhere, and the environment variables are what make each run specific to its place.

## How images use them

Official images lean on this heavily. The Postgres image reads `POSTGRES_PASSWORD` to set the database password. The MySQL image reads `MYSQL_ROOT_PASSWORD`. Application images read variables for their log level, port, or connection strings. The image's documentation lists which variables it understands.

## What they are good for and not

Environment variables suit small pieces of configuration - a hostname, a port, a feature flag, a mode. They are the right tool for most settings.

They are not meant for large data, and they are a weak spot for secrets, since anything passed as an environment variable can be read back with `docker inspect`. That trade-off comes up again in this topic's scenario. For now, the takeaway is that environment variables are how you configure a container from the outside without touching the image.



▶Explain this · 3:44Englishहिन्दीతెలుగుதமிழ்ಕನ್ನಡമലയാളംবাংলাमराठी# Passing variables with -e

The `-e` flag (short for `--env`) sets an environment variable inside the container. You add one `-e KEY=value` for each variable.

## Setting a variable

```
docker run -e LOG_LEVEL=debug --name app alpine env
```

Here `alpine` runs the `env` command, which prints all environment variables, and you will see `LOG_LEVEL=debug` among them. The process inside the container can read it exactly like any other environment variable.

## Setting several

Repeat the flag for each value:

```
docker run -d \
  -e POSTGRES_USER=shop \
  -e POSTGRES_PASSWORD=s3cret \
  -e POSTGRES_DB=orders \
  --name db postgres:16
```

The Postgres image reads those three variables on first startup and creates a database `orders` owned by user `shop` with the given password. You configured a database without editing a single file inside the image.

## Passing a variable through from your shell

If a variable already exists in your shell, you can pass just its name and Docker forwards the current value:

```
export API_TOKEN=abc123
docker run -e API_TOKEN --name app alpine env
```

With no `=value`, `-e API_TOKEN` tells Docker to copy `API_TOKEN` from your shell into the container. This keeps the actual value off the command line.

## Checking what a container has

To see the variables a running container was given, inspect its config:

```
docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' db
```

## Commands this node introduces

- `docker run -e KEY=value ...` - set an environment variable in the container
- `docker run -e KEY ...` - pass a variable through from your shell



# 
Env files with --env-file

Once a container needs more than two or three variables, a row of `-e` flags gets long and easy to mistype. An env file collects them in one place.

## The file format

An env file is a plain  file with one `KEY=value` per line:

```
POSTGRES_USER=shop
POSTGRES_PASSWORD=s3cret
POSTGRES_DB=orders
```

No `export`, no quotes needed around simple values, no spaces around the `=`. Lines starting with `#` are comments. Save it as something like `db.env`.

## Loading it with --env-file

Point the container at the file with `--env-file`:

```
docker run -d --env-file db.env --name db postgres:16
```

Every line in the file becomes an environment variable inside the container, exactly as if you had passed each one with `-e`. One flag replaces the whole stack.

## Why this is the better habit

An env file keeps configuration out of your shell history and out of the long command line, and it groups related settings so they are easy to review and reuse. Different env files - `dev.env`, `prod.env` - let you run the same image against different settings by swapping one flag.

Keep env files that hold secrets out of version control. Adding them to `.gitignore` is standard, so a password never lands in a Git repository.

## Mixing with -e

You can combine both. Anything you also pass with `-e` on the command line overrides the same key from the file, which is useful for a one-off change without editing the file.

## Commands this node introduces

- `docker run --env-file FILE ...` - load environment variables from a file

