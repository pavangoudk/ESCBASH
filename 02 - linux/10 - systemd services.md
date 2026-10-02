# What is systemd

Every long-running program on a modern Linux server is a "service", and almost every service is managed by **systemd**. When the machine boots, systemd is the first process to start, and it launches everything else: network, SSH, cron, your database, your web server.

## The first process

When Linux boots, the kernel starts exactly one program, and that program is responsible for starting everything else. That program is the **init system**, and on almost every modern distribution the init system is systemd.

Because it starts first, it gets process ID 1. You can see it:

```
ps -p 1 -o pid,comm
```

```
  PID COMMAND
    1 systemd
```

Every other process is started by systemd, or by something systemd started, so PID 1 sits at the root of the whole process tree. If it ever exits, the machine goes down, which is why it's built to keep running no matter what.

## Why not just run things with nohup?

You met `nohup` in the previous topic. It keeps a command alive after you log out, but it doesn't handle any of:

- Starting automatically when the machine boots.
- Restarting if the program crashes.
- Making sure dependencies (like the network) are up first.
- Collecting the program's logs where the rest of the system's logs live.

systemd does all of these. When you deploy a service to a Linux box, you don't run it with `&`, you write a systemd unit for it.

## Units

A **unit** is systemd's word for "a thing it manages". The most common type is `service`, but systemd also manages timers, sockets, mounts, and more. Every unit has a name that ends in its type:

- `ssh.service` - the SSH daemon
- `cron.service` - the cron scheduler
- `nginx.service` - the nginx web server
- `backup.timer` - a scheduled job

You control units with one command, `systemctl`. You read their logs with `journalctl`. That's the whole daily surface.



# systemctl basics

`systemctl` is how you talk to systemd. A handful of subcommands cover almost all your daily work.

## Life cycle

```
systemctl status ssh              # what's going on with ssh?
systemctl start ssh               # start it now
systemctl stop ssh                # stop it now
systemctl restart ssh             # stop then start
systemctl reload ssh              # re-read config, no downtime
```

`reload` only works if the service supports it. If in doubt, `restart` always works, at the cost of a brief downtime.

## enabled vs active

Two independent settings you'll always be checking:

- **active** - is the service running right now?
- **enabled** - will it auto-start when the machine boots?

A service can be enabled but not active (will start on boot but currently off), active but not enabled (running right now but won't come back after a reboot), both, or neither.

```
systemctl enable ssh              # auto-start on next boot
systemctl disable ssh             # don't start on boot
systemctl enable --now ssh        # enable AND start immediately
systemctl disable --now ssh       # disable AND stop immediately
```

`--now` is the shortcut you'll type most often when setting up (or removing) a service.

## Reading status output

`systemctl status` is the command you'll run most, so it's worth learning to read line by line:

```
systemctl status nginx
```

A healthy service prints something like this:

```
● nginx.service - A high performance web server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-07-28 09:14:02 UTC; 2h ago
   Main PID: 431 (nginx)
      Tasks: 2 (limit: 1109)
     Memory: 3.2M
     CGroup: /system.slice/nginx.service
             ├─431 nginx: master process /usr/sbin/nginx
             └─432 nginx: worker process

Jul 28 09:14:02 host systemd[1]: Started nginx.service.
```

Read it top to bottom:

- The dot on the first line is green when the service is healthy and red when it has failed.
- **Loaded** shows the unit file's path and whether it's `enabled`, so you know if it starts on boot.
- **Active** is the line to check first. `active (running)` means it's up. You'll also see `inactive (dead)`, `failed`, or `activating`, and the `since ...` tells you how long it's held that state.
- **Main PID** is the process ID of the service, handy if you want to look it up with the process tools.
- The line at the bottom is the most recent journal entry for the service, often enough to spot a problem without opening the journal yourself.

## Listing units

```
systemctl list-units --type=service          # only services
systemctl list-units --type=service --state=failed   # only broken ones
```

On a healthy machine, `--state=failed` prints nothing. On a broken one, it prints exactly what you need to look at.

## Unit file structure

So far you've only used systemd to control units that already exist. When you want to add your own service, you write a **unit file**.

Unit files for local services live under `/etc/systemd/system/`. Any file ending in `.service` is a service unit. A minimal one looks like this:

```
[Unit]
Description=My tiny service
After=network.target

[Service]
ExecStart=/usr/local/bin/mytool
Restart=on-failure
Type=simple

[Install]
WantedBy=multi-user.target
```

Three sections:

- `[Unit]` - description and startup ordering. `After=` says "start this after the network is up".
- `[Service]` - what to run (`ExecStart=` with the full path to the command or script), how to run it (`Type=simple` for a normal foreground program), what to do if it crashes (`Restart=on-failure`).
- `[Install]` - how to hook it into "start on boot". `WantedBy=multi-user.target` is systemd's shorthand for "the normal running state".

Whenever you add or change a unit file, systemd doesn't notice on its own. You have to tell it:

```
sudo systemctl daemon-reload
```

After that, `systemctl enable --now myservice.service` starts it and wires it up for future boots.



# journalctl

systemd captures everything a service prints - its normal output and its error messages - and stores it in the **journal**. `journalctl` is how you read it.

## Per-service logs

```
journalctl -u ssh                  # everything ssh has ever logged
journalctl -u ssh --since today    # since midnight today
journalctl -u ssh --since '1 hour ago'
journalctl -u ssh -n 100           # last 100 lines
```

`-u` stands for "unit", the same names you'd pass to `systemctl`.

A couple of lines of output look like this:

```
Jul 28 09:14:01 host sshd[512]: Server listening on 0.0.0.0 port 22.
Jul 28 09:20:44 host sshd[788]: Accepted password for root from 10.0.0.5
```

Each line is a timestamp, the hostname, the program with its PID in brackets, then the message the program logged. The newest entries sit at the bottom, and `journalctl` jumps straight there by default.

## Follow in real time

```
journalctl -u ssh -f
```

Same idea as `tail -f` from the reading-files topic, but scoped to a single service. This is the command you leave running in another terminal while you change a config and watch the service react to it.

## Useful filters

- `-p err` - errors and worse only
- `--since '2 hours ago'` - time window
- `-b` - only messages from the current boot
- `-o cat` - strip the timestamp/hostname prefix, print only the message

Combine them freely:

```
journalctl -u nginx -p err --since '1 hour ago'
```

That's "nginx errors from the last hour", in one command. If something broke recently on a production server, this is usually the first thing you type.

## Where the journal lives

Under `/var/log/journal/` as binary files. Don't try to read them directly, `journalctl` parses them for you. On some minimal systems `/var/log/journal/` doesn't exist and the journal lives in memory only, wiped on reboot. Worth knowing when you can't find yesterday's logs.

