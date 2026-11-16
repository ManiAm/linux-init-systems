# Interacting with systemd

`systemctl` is the primary command-line tool for interacting with systemd. It allows administrators to start, stop, enable, disable, reload, restart, and query the status of units. It also provides commands to manage system states, reload configuration files, and control system behavior such as shutting down or rebooting.

## Checking Unit Status

| Command                      | Description                                       |
|------------------------------|---------------------------------------------------|
| `systemctl status <unit>`    | Display the status of a specific unit             |
| `systemctl is-enabled <unit>`| Check if a unit is enabled to start at boot       |
| `systemctl is-active <unit>` | Check if a unit is currently active               |
| `systemctl is-failed <unit>` | Check if a unit has failed                        |

For example, to check the status of the `sshd` service:

    $ systemctl status sshd

    ● ssh.service - OpenBSD Secure Shell server
        Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enabled)
        Active: active (running) since Wed 2025-01-08 14:05:24 PST; 2 days ago
        Docs: man:sshd(8)
              man:sshd_config(5)
    Main PID: 1047 (sshd)
        Tasks: 1 (limit: 11658)
        Memory: 3.8M
        CGroup: /system.slice/ssh.service
                └─1047 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups

Here is what each part of the output means:

- **`● ssh.service`** — the unit name. The `●` symbol is color-coded: green means active, red means failed, and yellow indicates a warning or degraded state.

- **`OpenBSD Secure Shell server`** — the service description, taken from the `Description=` field in the unit file.

- **`Loaded: loaded (...; enabled; vendor preset: enabled)`** — the unit's [load state](04_systemd.md#load-state). `loaded` confirms systemd successfully read the unit file. `enabled` means the service is configured to start automatically at boot. `vendor preset` shows the distribution's default setting.

- **`Active: active (running)`** — the unit's [active state](04_systemd.md#active-state) and [substate](04_systemd.md#unit-substates). Here, the active state is `active` and the substate is `running`. The timestamp shows when the service last started and how long it has been running.

- **`Docs`** — documentation references for the service.

- **`Main PID: 1047 (sshd)`** — the process ID of the service's main process.

- **`Tasks: 1 (limit: 11658)`** — the number of active tasks (threads or processes) under this service, and the maximum allowed by cgroup settings.

- **`Memory: 3.8M`** — memory consumed by the service.

- **`CGroup: /system.slice/ssh.service`** — the control group (cgroup) the service runs in. `system.slice` is the default slice for system services.

## Starting, Stopping, and Restarting Units

| Command                                  | Description                                                       |
|------------------------------------------|-------------------------------------------------------------------|
| `systemctl start <unit>`                 | Start the specified unit immediately                              |
| `systemctl stop <unit>`                  | Stop the specified unit immediately                               |
| `systemctl restart <unit>`               | Restart the specified unit                                        |
| `systemctl reload <unit>`                | Reload the configuration of a running unit without restarting it  |
| `systemctl reload-or-restart <unit>`     | Reload if possible; otherwise, restart the unit                   |
| `systemctl try-restart <unit>`           | Restart the unit only if it is currently running                  |
| `systemctl try-reload-or-restart <unit>` | If running: reload if supported, otherwise restart. Does nothing if not running |
| `systemctl clean <unit>`                 | Remove runtime, cache, state, logs, or configuration of a unit    |
| `systemctl kill <unit>`                  | Send a signal to processes of a unit                              |
| `systemctl isolate <unit>`               | Start a unit and its dependencies, stop all others                |

## Enabling and Disabling Units

| Command                         | Description                                                          |
|---------------------------------|----------------------------------------------------------------------|
| `systemctl enable <unit>`       | Enable a unit to start automatically at boot                         |
| `systemctl disable <unit>`      | Disable a unit from starting at boot                                 |
| `systemctl enable --now <unit>` | Enable and start the unit immediately                                |
| `systemctl disable --now <unit>`| Disable and stop the unit immediately                                |
| `systemctl revert <unit>`       | Remove local overrides and drop-ins, reverting to the vendor version |
| `systemctl reenable <unit>`     | Re-enable a unit (disable then enable, refreshing symlinks)          |

`systemctl reenable` removes the unit's existing symlinks and creates new ones based on the current `[Install]` section. This is useful after editing a unit file's `[Install]` directives to ensure the symlinks reflect the updated configuration.

## Masking and Unmasking Units

| Command                         | Description                                       |
|---------------------------------|---------------------------------------------------|
| `systemctl mask <unit>`         | Mask the unit, making it impossible to start       |
| `systemctl unmask <unit>`       | Unmask the unit, allowing it to start again        |

A masked unit cannot be started — either manually or automatically. Masking works by creating a symlink from the unit file to `/dev/null`, effectively rendering it inoperable. This is useful as a safety mechanism when a unit is unnecessary, conflicts with other units, or poses a risk.

To make a masked unit operational again, unmask it, then enable and start it:

    $ sudo systemctl unmask <unit>
    $ sudo systemctl enable --now <unit>

`unmask` removes the symlink to `/dev/null`. `enable` configures the unit to start at boot, and the `--now` flag also starts it immediately.

## Reloading and Restarting systemd

| Command                     | Description                                                          |
|-----------------------------|----------------------------------------------------------------------|
| `systemctl daemon-reload`   | Reload systemd's manager configuration (e.g., after modifying unit files) |
| `systemctl reset-failed`    | Reset the failed state of all units                                  |

## Listing Units

| Command                               | Description                                      |
|---------------------------------------|--------------------------------------------------|
| `systemctl list-units`                | List loaded units (active, with pending jobs, or failed) |
| `systemctl list-units --all`          | List all units (active, inactive, or failed)     |
| `systemctl list-units --type <type>`  | List all active units of a given type            |
| `systemctl list-timers`               | List active timers                               |
| `systemctl list-sockets`              | List active sockets                              |
| `systemctl list-unit-files`           | List all unit files and their enablement states  |

The output of `systemctl list-units` has five columns:

- **UNIT** — the name of the unit
- **LOAD** — the unit's [load state](04_systemd.md#load-state)
- **ACTIVE** — the unit's [active state](04_systemd.md#active-state)
- **SUB** — the unit's [substate](04_systemd.md#unit-substates)
- **DESCRIPTION** — a brief description of the unit

## Inspecting Units

| Command                               | Description                              |
|---------------------------------------|------------------------------------------|
| `systemctl list-dependencies <unit>`  | Display the dependency tree of a unit    |
| `systemctl cat <unit>`                | Display the unit file content            |
| `systemctl show <unit>`               | Display all properties of a unit         |

---

[← Previous: Systemd Overview](04_systemd.md) | [Next: Unit Files →](06_systemd_unit_files.md)
