# systemd Overview

The [previous chapter](02_init_systems.md) introduced systemd as the default init system on most modern Linux distributions. This chapter covers its background, design principles, and core concepts — **units**, **unit types**, and **unit states** — which form the foundation for everything that follows.

Source code: [github.com/systemd/systemd](https://github.com/systemd/systemd)

## Background

In April 2010, Lennart Poettering (Red Hat) published [Rethinking PID 1](https://0pointer.de/blog/projects/systemd.html), a blog post that laid out the case for replacing SysV init. Working with Kay Sievers (Novell), he had been developing an experimental init system called **systemd**.

The motivation was straightforward: by 2010, SysV init's [limitations](03_sysv_init.md#limitations) were holding back modern Linux systems: sequential startup wasted multi-core hardware, manual script ordering made dependency management fragile, and there was no built-in service monitoring or recovery. Upstart had improved on some of these issues with its event-driven model, but Poettering argued that its core design — events triggering shell scripts — didn't go far enough.

Two existing systems influenced systemd's design:

- **Apple's launchd** (macOS) — demonstrated that socket activation and on-demand service launching could dramatically reduce boot times
- **inetd** (traditional UNIX) — showed that a supervisor could own listening sockets and spawn services only when connections arrived

systemd's first release (v1) followed in July 2010. Fedora 15 became the first major distribution to adopt it as default in May 2011. Arch Linux and openSUSE followed in 2012, RHEL and SUSE Linux Enterprise in 2014, and Debian and Ubuntu in 2015. Today, systemd is the default init system on virtually every major Linux distribution.

## Design Principles

systemd introduced several fundamental changes compared to SysV init:

### 1. Declarative Unit Files Instead of Shell Scripts

SysV init manages services through imperative shell scripts — each script contains the actual commands to start, stop, and check a service. systemd replaces these with **unit files**: short, declarative configuration files that describe *what* to run and *how*, without requiring shell logic. This makes service definitions simpler to write, read, and maintain.

### 2. Dependency Graph Instead of Numbered Scripts

SysV init orders startup by numbering scripts (`S01`, `S02`, ...). systemd models dependencies as a **directed graph** — a structure where each unit points to what it depends on — using explicit directives like `Requires=`, `Wants=`, `After=`, and `Before=`. This lets systemd determine the correct order automatically and start independent services in parallel.

### 3. Socket Activation

systemd can create listening sockets before starting the services behind them. When a connection arrives, systemd starts the service on demand and hands over the socket. This provides three benefits:

- **Faster boot** — services don't need to wait for their dependencies to fully start; they can connect to the socket immediately (messages are buffered until the service is ready)
- **On-demand startup** — services that may never be needed don't consume resources at boot
- **Graceful restarts** — since systemd owns the socket, connections are preserved even when the service is restarted

### 4. Process Tracking via Cgroups

SysV init tracks processes using PID files — a fragile mechanism that breaks when services fork, crash, or fail to clean up. systemd places every service's processes into a dedicated Linux **control group** (cgroup). This allows systemd to:

- Reliably track all processes belonging to a service, even if they fork or daemonize
- Cleanly terminate an entire service tree on shutdown
- Enforce resource limits (CPU, memory, I/O) per service

### 5. Integrated Service Supervision

SysV init starts services and walks away — if a service crashes, init does not notice. systemd actively monitors services and can automatically restart them based on configurable policies (`Restart=on-failure`, `Restart=always`, etc.).

## Beyond Init

systemd is more than an init system. Over time, it absorbed several system management responsibilities into a unified suite of components:

| Component     | Purpose                                | Relationship to Earlier Tools                                  |
|---------------|----------------------------------------|----------------------------------------------------------------|
| **systemd**   | PID 1 — service management and boot    | Replaces SysV init or Upstart as the init/service manager      |
| **journald**  | Structured, centralized logging        | Can replace local syslog collection, but also works alongside rsyslog |
| **logind**    | User session and seat management       | Supersedes ConsoleKit's session-management role                |
| **networkd**  | Network configuration                  | Alternative to ifupdown or NetworkManager                      |
| **resolved**  | DNS resolution and caching             | Manages `/etc/resolv.conf`; alternative to dnsmasq, not a full replacement |
| **timesyncd** | NTP time synchronization               | Lightweight alternative to ntpd or chrony                      |
| **udevd**     | Device event handling                  | The udev device manager, integrated into the systemd project   |

This scope is intentional — Poettering argued that the traditional boundaries between init, logging, device management, and session handling led to inconsistencies and duplicated effort. Integrating them under one project allows tighter coordination (for example, journald captures structured logs from services managed by systemd, and logind tracks user sessions via cgroups).

This breadth is also the source of much of the criticism systemd receives.

## Advantages and Criticism

systemd's adoption was accompanied by one of the most heated debates in Linux history. The arguments on both sides are worth understanding:

| Advantages                                                                                                     | Criticism                                                                                                                   |
|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| **Faster boot** — parallel startup and socket activation significantly reduce time to a usable system          | **Complexity** — a large, interconnected project that requires learning a substantial new toolset                            |
| **Consistent management** — unit files work the same way across distributions                                  | **Scope creep** — absorbing logging, DNS, and session management goes against the UNIX philosophy of small, composable tools |
| **Reliable process tracking** — cgroups eliminate PID file problems                                            | **Tight coupling** — some software now depends on systemd-specific interfaces, reducing portability                          |
| **Service supervision** — automatic restart on failure without external tools                                  | **Linux-only** — relies on Linux-specific kernel features (cgroups, namespaces, epoll) and cannot be ported to other UNIXes  |
| **Structured logging** — journald captures logs with metadata queryable via `journalctl`                       | **Binary logs** — journald uses a binary format rather than plain text, which some administrators find harder to work with   |
| **On-demand activation** — socket, D-Bus, path, and timer activation reduce idle resource consumption          |                                                                                                                             |

Despite the debate, systemd's practical advantages led to its adoption across all major Linux distributions. Distributions that prefer alternatives — such as Void Linux (runit), Alpine Linux (OpenRC), Gentoo (OpenRC), and Devuan (SysV init) — remain available for users who want a different approach.

## Units

The [design principles](#1-declarative-unit-files-instead-of-shell-scripts) above introduced unit files as systemd's replacement for SysV init scripts. Each unit represents a single system resource — such as a service, mount point, timer, or device — and its configuration file tells systemd how to manage it.

systemd defines 11 unit types:

| Unit Type    | File Extension | Purpose                                                                      |
|--------------|----------------|------------------------------------------------------------------------------|
| **Service**  | `.service`     | Manages a daemon or one-shot process (e.g., `sshd.service`, `nginx.service`) |
| **Socket**   | `.socket`      | Manages a network or IPC socket for socket-based activation                  |
| **Target**   | `.target`      | Groups units for synchronized state transitions (e.g., `multi-user.target`)  |
| **Device**   | `.device`      | Represents a device exposed in the Linux device tree (`/dev`)                |
| **Mount**    | `.mount`       | Manages a filesystem mount point                                             |
| **Automount**| `.automount`   | Configures a filesystem automount point (mounts on first access)             |
| **Swap**     | `.swap`        | Manages a swap device or swap file                                           |
| **Timer**    | `.timer`       | Schedules tasks based on time or intervals (replaces `cron` in many cases)   |
| **Path**     | `.path`        | Watches a file or directory for changes and triggers a unit                  |
| **Slice**    | `.slice`       | Groups processes hierarchically for resource management via cgroups          |
| **Scope**    | `.scope`       | Manages externally created processes (not started by systemd)                |

## Unit States

Every unit has two levels of state: a **load state** (is the configuration loaded?) and an **active state** (is the unit running?).

### Load State

The load state indicates whether the unit's configuration file has been successfully read into memory:

| Load State    | Description                                                        |
|---------------|--------------------------------------------------------------------|
| `loaded`      | The unit file is successfully loaded and ready for use             |
| `not-found`   | The unit file could not be found on the system                     |
| `bad-setting` | The unit file contains invalid settings                            |
| `error`       | An error occurred while loading the unit file                      |
| `masked`      | The unit is completely disabled and cannot be started              |
| `stub`        | The unit is a placeholder without a complete configuration         |
| `merged`      | The unit configuration has been merged from multiple sources       |

### Active State

The active state reflects the unit's current runtime status:

| Active State    | Description                                                           |
|-----------------|-----------------------------------------------------------------------|
| `active`        | The unit is running and operational                                   |
| `inactive`      | The unit is stopped                                                   |
| `activating`    | The unit is starting up but not yet fully operational                 |
| `deactivating`  | The unit is shutting down                                             |
| `reloading`     | The unit is reloading its configuration                               |
| `failed`        | The unit encountered an error and failed to start or operate          |
| `maintenance`   | The unit requires manual intervention                                 |
| `refreshing`    | The unit is refreshing its state due to configuration updates         |

## Unit Substates

Beyond the general active state, each unit type defines its own **substates** that reflect more specific lifecycle stages. For example, a service unit transitions through substates like `start-pre` → `start` → `running` → `stop` → `dead`, while a socket unit goes through `start-pre` → `listening` → `running` → `stop-pre` → `dead`.

The tables below are a reference — you do not need to memorize them. They are useful when interpreting `systemctl status` output or debugging service issues. The **service substates** are the most commonly encountered.

### Service

| Substate                        | Description                                              |
|---------------------------------|----------------------------------------------------------|
| `dead`                          | The service is inactive                                  |
| `condition`                     | The service is evaluating its start conditions           |
| `start-pre`                     | Running pre-start tasks (`ExecStartPre=`)                |
| `start`                         | The service is starting (`ExecStart=`)                   |
| `start-post`                    | Running post-start tasks (`ExecStartPost=`)              |
| `running`                       | The service is active and operational                    |
| `exited`                        | A one-shot service completed successfully                |
| `reload`                        | The service is reloading its configuration               |
| `reload-signal`                 | The service is reloading via a signal                    |
| `reload-notify`                 | The service is reloading and notifying its state         |
| `stop`                          | The service is stopping (`ExecStop=`)                    |
| `stop-watchdog`                 | The service is stopping due to a watchdog timeout        |
| `stop-sigterm`                  | The service received SIGTERM                             |
| `stop-sigkill`                  | The service received SIGKILL                             |
| `stop-post`                     | Running post-stop tasks (`ExecStopPost=`)                |
| `final-watchdog`                | Performing final watchdog cleanup                        |
| `final-sigterm`                 | Performing final cleanup after SIGTERM                   |
| `final-sigkill`                 | Performing final cleanup after SIGKILL                   |
| `failed`                        | The service encountered an error and failed              |
| `dead-before-auto-restart`      | Dead and waiting for auto-restart                        |
| `failed-before-auto-restart`    | Failed and waiting for auto-restart                      |
| `dead-resources-pinned`         | Dead but some resources are still pinned                 |
| `auto-restart`                  | The service is auto-restarting                           |
| `auto-restart-queued`           | The service is queued for auto-restart                   |
| `cleaning`                      | The service is cleaning up resources                     |

### Socket

| Substate             | Description                                           |
|----------------------|-------------------------------------------------------|
| `dead`               | The socket is inactive                                |
| `start-pre`          | Running pre-start tasks                               |
| `start-open`         | Opening the socket                                    |
| `start-chown`        | Setting socket permissions                            |
| `start-post`         | Running post-start tasks                              |
| `listening`          | The socket is listening for connections               |
| `running`            | The socket is active and handling connections         |
| `stop-pre`           | Running pre-stop tasks                                |
| `stop-pre-sigterm`   | Pre-stop interrupted by SIGTERM                       |
| `stop-pre-sigkill`   | Pre-stop interrupted by SIGKILL                       |
| `stop-post`          | Running post-stop tasks                               |
| `final-sigterm`      | Performing final cleanup after SIGTERM                |
| `final-sigkill`      | Performing final cleanup after SIGKILL                |
| `failed`             | The socket encountered an error                       |
| `cleaning`           | Cleaning up resources                                 |

### Mount

| Substate              | Description                                           |
|-----------------------|-------------------------------------------------------|
| `dead`                | The mount is inactive (not mounted)                   |
| `mounting`            | The filesystem is being mounted                       |
| `mounting-done`       | The mount operation has completed                     |
| `mounted`             | The filesystem is successfully mounted                |
| `remounting`          | The filesystem is being remounted                     |
| `unmounting`          | The filesystem is being unmounted                     |
| `remounting-sigterm`  | Remounting interrupted by SIGTERM                     |
| `remounting-sigkill`  | Remounting interrupted by SIGKILL                     |
| `unmounting-sigterm`  | Unmounting interrupted by SIGTERM                     |
| `unmounting-sigkill`  | Unmounting interrupted by SIGKILL                     |
| `failed`              | The mount operation failed                            |
| `cleaning`            | Cleaning up resources                                 |

### Device

| Substate    | Description                                               |
|-------------|-----------------------------------------------------------|
| `dead`      | The device is inactive                                    |
| `tentative` | The device is in a tentative state, awaiting confirmation |
| `plugged`   | The device is present and operational                     |

### Target

| Substate | Description              |
|----------|--------------------------|
| `dead`   | The target is inactive   |
| `active` | The target is active     |

### Timer

| Substate  | Description                                           |
|-----------|-------------------------------------------------------|
| `dead`    | The timer is inactive                                 |
| `waiting` | The timer is waiting for its next trigger             |
| `running` | The timer has triggered and the associated unit is running |
| `elapsed` | The timer has completed all scheduled actions         |
| `failed`  | The timer encountered an error                        |

### Swap

| Substate                | Description                                      |
|-------------------------|--------------------------------------------------|
| `dead`                  | The swap is inactive                             |
| `activating`            | The swap is being activated                      |
| `activating-done`       | The activation is complete                       |
| `active`                | The swap is active and operational               |
| `deactivating`          | The swap is being deactivated                    |
| `deactivating-sigterm`  | Deactivation interrupted by SIGTERM              |
| `deactivating-sigkill`  | Deactivation interrupted by SIGKILL              |
| `failed`                | The swap encountered an error                    |
| `cleaning`              | Cleaning up resources                            |

### Path

| Substate  | Description                                  |
|-----------|----------------------------------------------|
| `dead`    | The path is inactive                         |
| `waiting` | Waiting for the watched path to change       |
| `running` | The path changed and the triggered unit is running |
| `failed`  | The path unit encountered an error           |

### Automount

| Substate  | Description                                      |
|-----------|--------------------------------------------------|
| `dead`    | The automount is inactive                        |
| `waiting` | Waiting for a filesystem access to trigger mount |
| `running` | The automount is active and operational          |
| `failed`  | The automount encountered an error               |

### Scope

| Substate       | Description                                     |
|----------------|-------------------------------------------------|
| `dead`         | The scope is inactive                           |
| `start-chown`  | The scope is starting and changing ownership    |
| `running`      | The scope is active and operational             |
| `abandoned`    | The scope was abandoned                         |
| `stop-sigterm` | The scope is stopping due to SIGTERM            |
| `stop-sigkill` | The scope is stopping due to SIGKILL            |
| `failed`       | The scope encountered an error                  |

### Slice

| Substate | Description             |
|----------|-------------------------|
| `dead`   | The slice is inactive   |
| `active` | The slice is active     |

---

[← Previous: SysV Init](03_sysv_init.md) | [Next: Systemd Management →](05_systemd_management.md)
