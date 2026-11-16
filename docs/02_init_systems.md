# Init Systems

The [previous chapter](01_init.md) described how the kernel launches the init process (PID 1) and the responsibilities it must fulfill — system initialization, service management, process lifecycle, and shutdown. This chapter surveys the major init systems that have been developed to carry out those responsibilities.

An **init system** (also called a **service manager**) is the program that runs as PID 1 and manages the system's services and overall state. Different init systems take different approaches — from simple sequential scripts to parallel, dependency-driven frameworks. The table below summarizes the most notable ones:

| Init System                 | Year | Key Features                                                        | Modern Usage                                  |
|-----------------------------|------|---------------------------------------------------------------------|-----------------------------------------------|
| SysV init                   | 1983 | Sequential startup, runlevels, shell scripts                        | Rare; mostly legacy systems                   |
| Runit                       | 2004 | Lightweight, three-stage design, process supervision                | Minimalist distros (e.g., **Void Linux**)     |
| Service Management Facility | 2005 | XML manifests, dependency tracking, automatic recovery              | **Solaris** and derivatives (e.g., Illumos)   |
| launchd                     | 2005 | Unified service/scheduling framework, on-demand activation          | Default in **macOS** and **iOS**              |
| Upstart                     | 2006 | Event-driven, parallel startup                                      | Deprecated; replaced by **systemd**           |
| OpenRC                      | 2007 | Dependency-based, parallel startup, SysV script compatibility       | **Gentoo** and **Alpine Linux**               |
| systemd                     | 2010 | Parallel startup, socket activation, unit files, integrated logging | Default in most **major Linux distributions** |

## SysV init

SysV init (System V init) is one of the earliest init systems, originating in UNIX System V in 1983. It uses a series of shell scripts organized into **runlevels** (numbered 0–6) to control the startup, management, and shutdown of services. Each runlevel represents a specific system state, such as single-user mode or multi-user mode.

Services are started sequentially based on their order in directories like `/etc/rc.d/` or `/etc/init.d/`. This design is simple and predictable, but the lack of parallel execution and dynamic dependency handling makes it slow on modern hardware. These limitations led to its replacement by systemd in most Linux distributions.

## Runit

Runit is a lightweight init system introduced in 2004 as an alternative to SysV init. It emphasizes simplicity, reliability, and speed, making it popular in minimalist distributions like Void Linux.

Runit divides the boot and service lifecycle into three distinct stages:

1. **System initialization** — one-time setup tasks (mounting filesystems, setting hostname)
2. **Service supervision** — long-running services are monitored and automatically restarted if they fail
3. **System shutdown** — orderly teardown of services and filesystems

This staged design, combined with built-in process supervision and minimal resource usage, makes Runit well-suited for systems that prioritize fast boot times and simplicity.

## Service Management Facility (SMF)

The Service Management Facility (SMF) is a service management framework introduced in 2005 as part of Solaris 10, later adopted by Illumos and OpenSolaris. It replaces SysV init with a more robust approach to service initialization, supervision, and fault recovery.

SMF uses XML manifests to define services and their dependencies, enabling automated dependency resolution and parallel startup. It features built-in service monitoring and automatic recovery — failed services are restarted without manual intervention. Administrators manage services through tools like `svcadm` (control) and `svcs` (status).

## launchd

launchd is a service management framework introduced by Apple in 2005 as the default init system for macOS and iOS. It replaced several older systems — including init, cron, and inetd — by consolidating service launching, scheduling, and process monitoring into a single framework.

Services are defined using XML-based property lists (`.plist` files). launchd activates services on demand rather than at boot, reducing resource usage. Combined with parallel startup and dependency tracking, this approach improves boot performance and provides flexibility for managing both system-level and user-level services.

## Upstart

Upstart is an event-driven init system developed by Canonical in 2006 for Ubuntu. Unlike SysV init, which follows a fixed startup sequence, Upstart reacts to dynamic events — such as hardware changes or network status updates — and starts, stops, or restarts services in response.

This event-driven model enables parallel execution and dependency management, resulting in faster boot times compared to SysV init. Upstart introduced features like job monitoring and task scheduling. However, it was eventually phased out in favor of systemd, which offered broader capabilities and wider adoption.

## OpenRC

OpenRC is a lightweight, dependency-based init system introduced in 2007. It is designed as an alternative to SysV init while maintaining compatibility with existing SysV init scripts. It is widely used in Gentoo and Alpine Linux.

OpenRC uses dependency tracking to determine the correct startup order and supports parallel service startup for faster boot times. It is known for its simplicity, portability, and minimal resource usage. Unlike systemd, OpenRC focuses solely on service management without integrating additional components — an approach that appeals to users who prefer modular, UNIX-like systems.

## systemd

systemd is a modern init system introduced in 2010 to replace SysV init and Upstart. It provides faster boot times through aggressive parallel startup and fine-grained dependency management.

systemd introduces several features not found in older init systems:

- **Unit files** — declarative configuration files that define services, mount points, timers, and other system resources
- **Socket activation** — services are started on demand when their socket receives a connection
- **D-Bus integration** — inter-process communication for service coordination
- **journald** — a centralized, structured logging system that replaces traditional syslog

While praised for its efficiency and feature set, systemd has faced criticism for its complexity and monolithic design. Despite this, it has become the default init system in most major Linux distributions:

| Distribution                    | Transition Path                                                        |
|---------------------------------|------------------------------------------------------------------------|
| Debian                          | SysV init → **systemd** (Debian 8 "Jessie", 2015)                     |
| Ubuntu                          | SysV init → Upstart (6.10) → **systemd** (15.04 "Vivid Vervet", 2015) |
| Red Hat Enterprise Linux (RHEL) | SysV init → **systemd** (RHEL 7, 2014)                                |

---

[← Previous: The Init Process](01_init.md) | [Next: SysV Init →](03_sysv_init.md)
