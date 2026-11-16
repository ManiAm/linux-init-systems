# Unit Files

Unit files are plain-text configuration files that tell systemd how to manage units. Each file defines a single unit — its description, dependencies, startup behavior, and other properties.

## Directories

Unit files are stored in several directories, listed here by priority (highest first):

| Directory                    | Purpose                                                  |
|------------------------------|----------------------------------------------------------|
| `/etc/systemd/system/`       | Local customizations and overrides (highest priority)    |
| `/lib/systemd/system/`       | Default units provided by the OS or installed packages   |
| `~/.config/systemd/user/`    | User-specific units (per-user services)                  |

When unit files with the same name exist in multiple directories, the higher-priority directory wins. This allows administrators to override vendor-provided units by placing a modified copy in `/etc/systemd/system/`.

## Enablement States

The following command lists all unit files and their enablement states:

    $ systemctl list-unit-files

The STATE column shows one of the following values:

| State              | Description                                                                                  |
|--------------------|----------------------------------------------------------------------------------------------|
| `enabled`          | The unit is enabled and will start automatically at boot                                     |
| `enabled-runtime`  | The unit is enabled for the current runtime only (does not persist after reboot)             |
| `linked`           | The unit file is symlinked from outside the search path, making it available but not enabled |
| `linked-runtime`   | Same as `linked`, but for the current runtime only (does not persist after reboot)           |
| `alias`            | The unit file is an alias for another unit                                                   |
| `masked`           | The unit is completely disabled and cannot be started                                        |
| `masked-runtime`   | The unit is masked for the current runtime only (does not persist after reboot)              |
| `static`           | The unit has no `[Install]` section; it cannot be enabled directly but can be started as a dependency |
| `disabled`         | The unit is disabled and will not start at boot, but can be started manually                 |
| `indirect`         | The unit is not enabled directly but has an `Also=` setting or is aliased under another name |
| `generated`        | The unit file was dynamically generated at runtime                                           |
| `transient`        | The unit was created temporarily and will not persist after the current boot                 |
| `bad`              | The unit file is invalid or could not be parsed                                              |

## Unit File Structure

Unit files follow a standard INI-style format consisting of sections with key-value pairs:

- **Sections** — defined in brackets `[]`, such as `[Unit]`, `[Service]`, and `[Install]`
- **Directives** — key-value pairs under each section, such as `Description=`, `ExecStart=`, or `WantedBy=`
- **Comments** — lines starting with `#` or `;`

Here is an example service unit file (`example.service`):

```ini
[Unit]
Description=Example Service
After=network.target

[Service]
ExecStart=/usr/bin/example
Restart=always

[Install]
WantedBy=multi-user.target
```

### `[Unit]` Section

The `[Unit]` section provides general metadata and dependency information. It is common to all unit types. Key directives include:

| Directive       | Purpose                                                              |
|-----------------|----------------------------------------------------------------------|
| `Description=`  | A human-readable description of the unit                             |
| `After=`        | Start this unit after the specified units (ordering only)            |
| `Before=`       | Start this unit before the specified units (ordering only)           |
| `Requires=`     | Units that must be active; systemd will start them if needed         |
| `Wants=`        | Units that should be active, but failure does not affect this unit   |

An important distinction: `Requires=` and `Wants=` create **requirement dependencies** — they control whether other units are pulled in. `After=` and `Before=` create **ordering dependencies** — they control the sequence in which units start, but do not cause units to be started. These are independent. For example, `After=network.target` alone does not start `network.target`; it only ensures ordering *if* `network.target` is being started. To both require and order a dependency, combine them (e.g., `Requires=network.target` and `After=network.target`). Dependencies are covered in detail in [Unit Dependencies](08_systemd_dependencies.md).

### `[Install]` Section

The `[Install]` section defines how the unit is enabled or disabled via `systemctl enable` and `systemctl disable`. It does not affect runtime behavior — it only configures how the unit integrates into the boot process. Key directives include:

| Directive       | Purpose                                                              |
|-----------------|----------------------------------------------------------------------|
| `WantedBy=`     | The target that pulls in this unit when enabled (most common)        |
| `RequiredBy=`   | The target that requires this unit when enabled                      |
| `Alias=`        | Alternative names for the unit                                       |
| `Also=`         | Additional units to enable or disable alongside this one             |

For example, `WantedBy=multi-user.target` means that running `systemctl enable` creates a symlink so that `multi-user.target` pulls in this unit during boot.

### Unit Type-specific Sections

In addition to `[Unit]` and `[Install]`, each unit type can have its own section with type-specific directives:

| Section         | Purpose                                                                       |
|-----------------|-------------------------------------------------------------------------------|
| `[Service]`     | Service startup, stop, and restart behavior (e.g., `ExecStart=`, `Type=`)     |
| `[Socket]`      | Socket configuration for socket-based activation (e.g., `ListenStream=`)      |
| `[Mount]`       | Filesystem mount points and options (e.g., `What=`, `Where=`)                 |
| `[Automount]`   | Automount configuration for on-demand mounting (e.g., `Where=`)               |
| `[Swap]`        | Swap device or file configuration (e.g., `What=`, `Priority=`)                |
| `[Timer]`       | Timer schedules for triggering other units (e.g., `OnCalendar=`)              |
| `[Path]`        | Filesystem path monitoring and triggers (e.g., `PathExists=`, `PathChanged=`) |
| `[Slice]`       | Resource management grouping via cgroups                                      |

Not all unit types have a type-specific section. Device units are auto-generated by udev, target units group other units, and scope units are created at runtime via the systemd API (e.g., `systemd-run --scope`) — all three use only `[Unit]` and `[Install]`.

---

[← Previous: Systemd Management](05_systemd_management.md) | [Next: Service Configuration →](07_systemd_services.md)
