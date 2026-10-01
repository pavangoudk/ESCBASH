# Linux Notes: `systemd`

## 1. What is systemd?

`systemd` is the init system and service manager used by most modern Linux distributions.

It:

- Starts during system boot
- Usually runs as process ID `1`
- Starts and manages services
- Handles startup dependencies
- Restarts failed services
- Collects service logs
- Supports timers, sockets, mounts, and other unit types

Check whether systemd is PID 1 with `ps -p 1 -o pid,comm`.

## 2. Why use systemd instead of `nohup`?

`nohup` keeps a process running after logout, but it does not provide robust service management.

Systemd can:

- Start services automatically at boot
- Restart services after failure
- Start services in the correct dependency order
- Centralize logs in the system journal
- Track service status and process IDs

For a long-running production service, use a systemd unit instead of simply running a command with `&`.

## 3. Units

A **unit** is something managed by systemd.

Common unit types include:

| Unit type | Example | Purpose |
| --- | --- | --- |
| Service | `nginx.service` | Runs a long-lived program |
| Timer | `backup.timer` | Schedules a task |
| Socket | `app.socket` | Listens for network or local connections |
| Mount | `data.mount` | Manages a filesystem mount |

The two main tools are:

- `systemctl` — control and inspect units
- `journalctl` — read unit logs

The `.service` suffix is often optional, so `systemctl status nginx` generally refers to `nginx.service`.

## 4. Essential `systemctl` commands

### Check and control a service

| Command | Purpose |
| --- | --- |
| `systemctl status ssh` | Show service status |
| `systemctl start ssh` | Start the service now |
| `systemctl stop ssh` | Stop the service now |
| `systemctl restart ssh` | Stop and start the service |
| `systemctl reload ssh` | Reload configuration without stopping |
| `systemctl enable ssh` | Start automatically at boot |
| `systemctl disable ssh` | Do not start automatically at boot |
| `systemctl enable --now ssh` | Enable and start immediately |
| `systemctl disable --now ssh` | Disable and stop immediately |

`reload` only works when the service supports configuration reloading. Use `restart` when necessary, remembering that it may cause brief downtime.

## 5. `active` versus `enabled`

These are separate settings:

- **Active:** Is the service running now?
- **Enabled:** Will the service start automatically at boot?

Possible combinations:

| Active | Enabled | Meaning |
| --- | --- | --- |
| Yes | Yes | Running now and starts at boot |
| Yes | No | Running now but will not start after reboot |
| No | Yes | Not running now but configured for boot |
| No | No | Not running and not configured for boot |

## 6. Reading `systemctl status`

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

A healthy service often reports:

- **Loaded:** Unit file location and whether it is enabled
- **Active:** Current state, such as `active (running)`, `inactive (dead)`, or `failed`
- **Main PID:** Primary process ID
- **Tasks:** Number of processes or tasks
- **Memory:** Current memory usage
- **CGroup:** Process grouping for the service
- Recent journal entries

The most important line is usually **Active**.

List all services:

`systemctl list-units --type=service`

List failed services:

`systemctl list-units --type=service --state=failed`

On a healthy system, the failed-services command may return no results.

## 7. Creating a service unit

Local service unit files are commonly stored in `/etc/systemd/system/`.

A unit file normally contains three sections:

### `[Unit]`

Defines metadata and startup ordering.

- `Description=` — explains the service
- `After=network.target` — starts the service after the network target

### `[Service]`

Defines how the program runs.

- `ExecStart=` — full path to the executable or script
- `Type=simple` — normal foreground process
- `Restart=on-failure` — restart if the process exits unsuccessfully

### `[Install]`

Defines how the service connects to the boot process.

- `WantedBy=multi-user.target` — starts during the normal multi-user boot state

After adding or changing a unit file, reload systemd’s configuration with:

`sudo systemctl daemon-reload`

Then enable and start the service with:

`sudo systemctl enable --now myservice.service`

## 8. Viewing logs with `journalctl`

Systemd captures standard output and error output from services in the journal.

### Common commands

| Command | Purpose |
| --- | --- |
| `journalctl -u ssh` | Show all logs for a unit |
| `journalctl -u ssh --since today` | Show logs since midnight |
| `journalctl -u ssh --since '1 hour ago'` | Show recent logs |
| `journalctl -u ssh -n 100` | Show the last 100 entries |
| `journalctl -u ssh -f` | Follow logs in real time |
| `journalctl -u nginx -p err` | Show errors and more severe messages |
| `journalctl -u nginx -b` | Show messages from the current boot |
| `journalctl -u nginx -o cat` | Show only log messages |

A useful troubleshooting command is:

`journalctl -u nginx -p err --since '1 hour ago'`

## 9. Following logs in real time

Use `journalctl -u service -f` to follow a service’s logs as new entries arrive.

This is useful when:

1. Changing a service configuration
2. Reloading or restarting the service
3. Watching for errors or confirmation messages

It is similar to `tail -f`, but specifically filters logs for a systemd unit.

## 10. Where journal logs are stored

Persistent journal files are commonly stored under `/var/log/journal/`.

On some minimal systems, logs are stored only in memory. Those logs may disappear after reboot if persistent journaling is not configured.

Do not read journal files directly; use `journalctl`.

## Troubleshooting workflow

1. Check status: `systemctl status service`
2. Review recent errors: `journalctl -u service -p err --since '1 hour ago'`
3. Check whether the service is enabled: `systemctl is-enabled service`
4. Check whether it is running: `systemctl is-active service`
5. Reload systemd after unit-file changes: `systemctl daemon-reload`
6. Restart or reload the service as appropriate.
7. Follow the logs with `journalctl -u service -f`.

## Core mental model

- `systemd` manages services.
- `systemctl` controls services.
- `journalctl` reads service logs.
- `active` means running now.
- `enabled` means configured to start at boot.
- `daemon-reload` makes systemd recognize unit-file changes.
