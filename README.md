# Linux Init Systems

A comprehensive reference covering Linux initialization — from kernel boot to systemd service management.

## Introduction

| #  | Topic                                    | Description |
|----|------------------------------------------|-------------|
| 01 | [The Init Process](docs/01_init.md)      | PID 1, kernel-to-userspace handoff, init responsibilities, and process management |
| 02 | [Init Systems](docs/02_init_systems.md)  | Overview of SysV, Runit, SMF, Launchd, Upstart, OpenRC, Systemd, and container init |
| 03 | [SysV Init](docs/03_sysv_init.md)        | Runlevels, rc directories, init scripts, execution flow, and limitations |

## Systemd

| #  | Topic                                                | Description |
|----|------------------------------------------------------|-------------|
| 04 | [Systemd Overview](docs/04_systemd.md)               | Units, unit types, unit states, and substates |
| 05 | [Systemd Management](docs/05_systemd_management.md)  | `systemctl` commands — status, start/stop, enable/disable, mask, listing |
| 06 | [Unit Files](docs/06_systemd_unit_files.md)          | Unit file locations, structure, `[Unit]`, `[Install]`, and type-specific sections |
| 07 | [Service Configuration](docs/07_systemd_services.md) | Service directives, startup types, and restart behaviour |
| 08 | [Unit Dependencies](docs/08_systemd_dependencies.md) | Ordering, dependency management, conflicts, triggers, and propagation |
| 09 | [Syslog](docs/09_syslog.md)                          | Syslog fundamentals, rsyslog routing, rate limiting, log rotation |
| 10 | [Journald](docs/10_systemd_journald.md)              | Structured logging, `journalctl`, disk management, journald–rsyslog coexistence |
| 11 | [Targets and Boot Flow](docs/11_systemd_targets.md)  | Targets, runlevel mapping, default target, isolate, rescue/emergency mode, and the boot sequence |
