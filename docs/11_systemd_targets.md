# Targets and Boot Flow

The [systemd overview](04_systemd.md) introduced targets as units that group other units for synchronized state transitions. This chapter explains how targets work in practice — how they replace SysV runlevels, how they define the boot sequence, and how to use them for system administration.

## What Targets Do

A target is a unit that does nothing by itself — it has no `ExecStart=`, no daemon, no process. Its sole purpose is to **group other units together** and act as a synchronization point. When systemd activates a target, it activates all units that the target depends on (via `Wants=`, `Requires=`, etc.) and all units that declare themselves part of that target (via `WantedBy=` or `RequiredBy=` in their `[Install]` section).

This makes targets useful for two things:

1. **Defining system states** — a target like `multi-user.target` represents the state "the system is ready for multiple users with networking." Activating it brings up everything needed to reach that state.

2. **Ordering the boot sequence** — targets depend on other targets, forming a chain. `graphical.target` depends on `multi-user.target`, which depends on `basic.target`, which depends on `sysinit.target`. This chain defines the boot order.

## Common Targets

To list all available targets on a system:

    $ systemctl list-units --type=target --all

The most important targets and what they represent:

### Boot Sequence Targets

These targets are activated in order during a normal boot:

| Target                  | Purpose                                                                |
|-------------------------|------------------------------------------------------------------------|
| `local-fs-pre.target`   | Synchronization point before local filesystems are mounted             |
| `local-fs.target`       | All local filesystems are mounted                                      |
| `sysinit.target`        | System initialization is complete (filesystems, swap, udev, etc.)      |
| `basic.target`          | Basic system is ready (timers, sockets, and basic services are up)     |
| `network-pre.target`    | Synchronization point before network configuration begins              |
| `network.target`        | Network configuration is complete (interfaces are configured)          |
| `network-online.target` | Network is fully online (connectivity is verified, not just configured)|
| `multi-user.target`     | Full multi-user system with networking (equivalent to a console server)|
| `graphical.target`      | Graphical login screen is available (extends `multi-user.target`)      |

### Administrative Targets

| Target                 | Purpose                                                                |
|------------------------|------------------------------------------------------------------------|
| `rescue.target`        | Single-user mode with basic filesystems (for maintenance)              |
| `emergency.target`     | Minimal shell with root filesystem only (for emergency repairs)        |
| `shutdown.target`      | System is shutting down                                                |
| `reboot.target`        | System is rebooting                                                    |
| `sleep.target`         | System is entering a sleep state (suspend or hibernate)                |

> **Note:** `rescue.target` mounts filesystems and starts a few essential services. `emergency.target` is more minimal — it drops you to a root shell with almost nothing running (not even filesystems beyond the root). Use `emergency.target` when the system is too broken for `rescue.target` to work.

## Mapping to SysV Runlevels

SysV init uses numbered [runlevels](03_sysv_init.md#runlevels) (0–6) to define system states. systemd replaces these with targets. The mapping is:

| SysV Runlevel | systemd Target          | Description                              |
|---------------|-------------------------|------------------------------------------|
| 0             | `poweroff.target`       | Halt / power off                         |
| 1             | `rescue.target`         | Single-user mode                         |
| 2             | `multi-user.target`     | Multi-user (no networking on some distros)|
| 3             | `multi-user.target`     | Full multi-user with networking          |
| 4             | `multi-user.target`     | Unused / custom (mapped to multi-user)   |
| 5             | `graphical.target`      | Multi-user with graphical interface      |
| 6             | `reboot.target`         | Reboot                                   |

For backward compatibility, systemd provides `runlevelN.target` aliases (e.g., `runlevel3.target` is a symlink to `multi-user.target`). This allows legacy scripts and tools that reference runlevels to continue working.

## The Default Target

The **default target** is the target systemd activates at boot. It determines what state the system boots into. To check the current default:

    $ systemctl get-default
    graphical.target

To change it:

    $ sudo systemctl set-default multi-user.target

This changes the system to boot into a console (no graphical interface) on the next reboot.

Under the hood, `set-default` creates a symlink at `/etc/systemd/system/default.target` pointing to the chosen target file. When systemd starts, it reads this symlink to determine which target to activate:

    $ ls -l /etc/systemd/system/default.target
    /etc/systemd/system/default.target -> /lib/systemd/system/graphical.target

## Switching Targets at Runtime

The `systemctl isolate` command switches the system to a different target **immediately**, without rebooting. It starts the specified target and its dependencies, and stops everything else:

    $ sudo systemctl isolate multi-user.target

This switches from a graphical desktop to a console-only environment. The graphical login manager and all graphical services are stopped.

    $ sudo systemctl isolate graphical.target

This switches back to the graphical environment.

> **Note:** Not all targets support isolation. Only targets with `AllowIsolate=yes` in their unit file can be used with `isolate`. The major targets (`multi-user.target`, `graphical.target`, `rescue.target`, `emergency.target`) all allow it.

### Rescue and Emergency Mode

To drop into single-user maintenance mode:

    $ sudo systemctl isolate rescue.target

This stops most services, mounts filesystems, and provides a root shell. It is useful for fixing configuration errors, repairing filesystems, or resetting passwords.

For more severe problems where even `rescue.target` fails to start:

    $ sudo systemctl isolate emergency.target

This provides a root shell with only the root filesystem mounted (read-only). From here you can manually mount filesystems and diagnose the issue.

Both modes can also be selected at boot time by adding `systemd.unit=rescue.target` or `systemd.unit=emergency.target` to the kernel command line (via the GRUB boot menu).

## The Boot Sequence

When the system powers on, the following sequence occurs:

1. **Firmware / bootloader** — the BIOS or UEFI firmware loads the bootloader (typically GRUB), which loads the Linux kernel and initial RAM disk (initrd).

2. **Kernel initialization** — the kernel initializes hardware, mounts the root filesystem, and starts the init process (PID 1). On a systemd-based system, PID 1 is `systemd`.

3. **systemd activates the default target** — systemd reads the `default.target` symlink and builds a dependency tree of all units needed to reach that target. It then activates units in the order determined by their dependency relationships, starting independent units in parallel.

4. **Target chain** — for a typical server booting into `multi-user.target`, the activation order follows this chain:

    ```
    local-fs-pre.target
          ↓
    local-fs.target       (mount local filesystems)
          ↓
    sysinit.target        (swap, udev, early setup)
          ↓
    basic.target          (timers, sockets, basic services)
          ↓
    multi-user.target     (networking, SSH, application services)
    ```

    For a desktop system, `graphical.target` adds one more step on top of `multi-user.target`, starting the display manager (login screen).

5. **Login prompt** — once the final target is reached, the system presents either a console login (`getty`) or a graphical login screen, depending on the default target.

### Inspecting the Boot

To see the full dependency tree that a target pulls in:

    $ systemctl list-dependencies multi-user.target

Add `--all` to expand the tree recursively (showing dependencies of dependencies):

    $ systemctl list-dependencies multi-user.target --all

To see how long the boot took and which units were slowest:

    $ systemd-analyze

    Startup finished in 3.512s (kernel) + 8.293s (userspace) = 11.806s
    graphical.target reached after 8.150s in userspace.

To see a breakdown by unit, sorted by time:

    $ systemd-analyze blame

To generate an SVG diagram showing the boot timeline with parallel activations:

    $ systemd-analyze plot > boot.svg

This produces a visual timeline that shows exactly when each unit started and finished, making it easy to identify bottlenecks.

---

[← Previous: Journald](10_systemd_journald.md)
