# cron format and crontab

Lesson
`cron` is the classic Linux scheduler. It runs commands on a schedule you define. Every DevOps engineer meets it within their first month, and it hasn't fundamentally changed in decades.

## The five fields

A cron schedule is five space-separated fields:

```
┌───── minute (0 - 59)
│ ┌───── hour (0 - 23)
│ │ ┌───── day of month (1 - 31)
│ │ │ ┌───── month (1 - 12)
│ │ │ │ ┌───── day of week (0 - 6, Sunday is 0 and 7)
│ │ │ │ │
* * * * *  command to run
```

`*` means "every value". Combine them to express a schedule:

```
0 3 * * *          # every day at 03:00
*/5 * * * *        # every 5 minutes
0 * * * *          # top of every hour
0 0 * * 0          # every Sunday at midnight
30 2 1 * *         # 02:30 on the 1st of every month
```

When you're stuck writing a schedule, [crontab.guru](https://crontab.guru/) is a good sanity check.

## Reading a line field by field

Take this full crontab line:

```
30 2 1 * * /root/backup.sh
```

Read the five fields left to right:

- `30` - minute 30
- `2` - hour 2, so the time is 02:30
- `1` - the 1st day of the month
- `*` - every month
- `*` - every day of the week

Put it together: run `/root/backup.sh` at 02:30 on the 1st of every month. Whatever follows the fifth field is the command cron runs.

## Editing your crontab

Each user has their own crontab file. You never edit it directly. Instead:

```
crontab -e         # open your crontab in the default editor
crontab -l         # list your current crontab
crontab -r         # remove your entire crontab (be careful)
```

`crontab -e` opens `$EDITOR` (usually nano or vim). Save and quit, and cron picks up the change automatically. No daemon reload needed.

## cron runs with a bare environment

Here's the gotcha that catches everyone. When cron runs your job it does not load your shell's setup. No `.bashrc`, no `.profile`, and a very short `PATH` (usually just `/usr/bin` and `/bin`).

So a command that works when you type it by hand can still fail under cron, because cron can't find it. Tools like `docker`, `python3`, `aws`, and `node` often live in `/usr/local/bin`, which isn't on cron's `PATH`. The job dies with "command not found" and you never see the error.

Two ways to fix it:

- Use absolute paths inside your script, for example `/usr/local/bin/docker` instead of `docker`.
- Or set `PATH` yourself on the first line of your crontab:

```
PATH=/usr/local/bin:/usr/bin:/bin
```

# System-wide cron jobs

Lesson`crontab -e` sets up your own personal schedule. For system-level jobs (log rotation, package updates, backups), cron looks in a set of well-known folders under `/etc/`.

## The convenience folders

Drop an executable script into one of these and it runs on that cadence automatically:

- `/etc/cron.hourly/`
- `/etc/cron.daily/`
- `/etc/cron.weekly/`
- `/etc/cron.monthly/`

```
ls /etc/cron.daily/
```

You'll see the scripts your distribution already schedules for you. On Ubuntu that usually includes `apt-compat`, `logrotate`, and a few others.

Two rules trip people up here. The script must be marked executable with `chmod`, or cron skips it. And its filename must have no dot in it: cron runs these folders through `run-parts`, which ignores any file with an extension like `.sh`. So name the file `backup`, not `backup.sh`.

## /etc/cron.d for full-schedule files

When you need a custom schedule (not just "hourly"), drop a file into `/etc/cron.d/`. Each line has one extra field, the **user** to run as, between the schedule and the command:

```
# /etc/cron.d/nightly-backup
30 2 * * *  root  /usr/local/bin/backup.sh
```

These files behave like extra crontabs, except they specify which user runs the job. Handy for package-managed jobs that need to run as a specific user.

## /etc/crontab: the master crontab

There's also a single `/etc/crontab` file with the same format as `/etc/cron.d/*`. Modern practice is to leave `/etc/crontab` alone and put your jobs in `/etc/cron.d/`, one file per purpose. Easier to review, easier to remove.

← Previous
# systemd timers

Lessoncron works and it's everywhere, but it has real weaknesses: no proper logging, no dependency handling, and it silently discards output. systemd timers are the modern alternative, and they pair naturally with the service units from earlier topics.

## The pair: a service and a timer

A timer never runs a command directly. It triggers a matching service that describes what to run. The two files live side by side in `/etc/systemd/system/` and share a name:

```
/etc/systemd/system/backup.service
/etc/systemd/system/backup.timer
```

## Build one end to end

Start with a small script for the service to run. Build it with a couple of `echo` lines and mark it executable:

```
echo '#!/bin/bash' > /root/backup.sh
echo 'date >> /root/backup.log' >> /root/backup.sh
chmod +x /root/backup.sh
```

Now the service. It uses `Type=oneshot` because it runs once and exits, and its `ExecStart` points at the script you just made. Put this in `/etc/systemd/system/backup.service`:

```
[Unit]
Description=Nightly backup

[Service]
Type=oneshot
ExecStart=/root/backup.sh
```

Then the timer, in `/etc/systemd/system/backup.timer`:

```
[Unit]
Description=Run backup daily at 02:30

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true

[Install]
WantedBy=timers.target
```
`OnCalendar=` sets the schedule. `Persistent=true` runs a job that got missed (the machine was off at 02:30) as soon as it boots back up.

## Reading OnCalendar

The full format is `weekday year-month-day hour:minute:second`, and `*` means "every". So `*-*-* 02:30:00` reads as any year, any month, any day, at 02:30:00. systemd also accepts handy shorthands:

```
OnCalendar=hourly              # every hour, on the hour
OnCalendar=daily               # every day at 00:00
OnCalendar=weekly              # Monday at 00:00
OnCalendar=Mon *-*-* 09:00:00  # every Monday at 09:00

```

## Load, enable, and inspect

systemd doesn't notice new unit files on its own. Tell it to reread them, then enable the timer (not the service):

```
systemctl daemon-reload
systemctl enable --now backup.timer
```

`enable` starts the timer at every boot, and `--now` also starts it right away. Check what's scheduled:

```
systemctl list-timers
```

That prints every active timer, when it last ran, and when it fires next. Handy when you're auditing what a machine is set up to do.

## cron or systemd timer?

- cron - quick one-off schedules, personal automation, anything short and simple.
- systemd timer - production jobs, anything you want logged and monitored, anything that depends on other services.

You'll meet both. Older systems lean on cron; newer services often ship a `.timer` file next to their `.service`.

← Previous
