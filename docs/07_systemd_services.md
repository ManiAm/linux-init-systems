# Service Configuration

The [previous chapter](06_systemd_unit_files.md) covered unit file structure and the sections common to all unit types (`[Unit]` and `[Install]`). This chapter focuses on the `[Service]` section — the directives that control how systemd starts, stops, monitors, and restarts services.

## Execution Directives

These directives define the commands systemd runs at each stage of a service's lifecycle:

| Directive          | Description                                                                         |
|--------------------|-------------------------------------------------------------------------------------|
| `ExecStart=`       | Command to start the service (required for all types except `oneshot`)              |
| `ExecStartPre=`    | Commands to run before `ExecStart=` (can be specified multiple times)               |
| `ExecStartPost=`   | Commands to run after the service has started successfully                          |
| `ExecStop=`        | Command to run when stopping the service                                           |
| `ExecStopPost=`    | Commands to run after the service has stopped (runs even on failure)                |
| `ExecReload=`      | Command to reload the service's configuration without restarting                    |
| `ExecCondition=`   | Command that must exit 0 to proceed; exit 1–254 skips the unit gracefully, 255 fails it |

## Permissions and Environment

| Directive             | Description                                                 |
|-----------------------|-------------------------------------------------------------|
| `User=`               | Run the service as this user                                |
| `Group=`              | Run the service as this group                               |
| `WorkingDirectory=`   | Set the working directory for the service process           |
| `Environment=`        | Set environment variables (e.g., `Environment="VAR=value"`) |
| `EnvironmentFile=`    | Load environment variables from a file                      |

## Service Behavior

| Directive             | Description                                                                                  |
|-----------------------|----------------------------------------------------------------------------------------------|
| `Type=`               | Startup type (see [Service Startup Types](#service-startup-types) below)                     |
| `Restart=`            | Restart policy (see [Service Restart Behavior](#service-restart-behavior) below)             |
| `RestartSec=`         | Time to wait before restarting (default: 100ms)                                              |
| `TimeoutSec=`         | General timeout for start and stop operations (sets both `TimeoutStartSec=` and `TimeoutStopSec=`) |
| `TimeoutStartSec=`    | Timeout for starting the service                                                             |
| `TimeoutStopSec=`     | Timeout for stopping the service                                                             |
| `RemainAfterExit=`    | Keep the unit active after the main process exits (commonly used with `Type=oneshot`)        |
| `SuccessExitStatus=`  | Additional exit codes or signals that should be treated as a successful exit                  |
| `WatchdogSec=`        | Watchdog timeout — the service must call `sd_notify("WATCHDOG=1")` within this interval or be restarted |
| `NotifyAccess=`       | Controls which processes can send readiness notifications via `sd_notify()` (`none`, `main`, `all`) |
| `BusName=`            | D-Bus bus name the service will acquire (required for `Type=dbus`)                           |

## Resource and Process Management

| Directive             | Description                                                                      |
|-----------------------|----------------------------------------------------------------------------------|
| `PIDFile=`            | Path to the PID file (used with `Type=forking`)                                 |
| `KillMode=`           | How to stop the service's processes (`control-group`, `process`, `mixed`, `none`) |
| `KillSignal=`         | Signal sent to stop the service (default: `SIGTERM`)                            |
| `LimitNOFILE=`        | Maximum number of open file descriptors                                         |
| `CPUWeight=`          | CPU allocation weight relative to other services (cgroup v2; range 1–10000)     |
| `MemoryMax=`          | Maximum memory the service can use (cgroup v2; e.g., `MemoryMax=512M`)          |

## Input/Output

| Directive           | Description                                                           |
|---------------------|-----------------------------------------------------------------------|
| `StandardInput=`    | Where the service reads input from (e.g., `null`, `tty`)             |
| `StandardOutput=`   | Where the service writes output (e.g., `journal`, `syslog`, `file:`) |
| `StandardError=`    | Where the service writes errors (e.g., `inherit`, `journal`)         |

## Service Startup Types

The `Type=` directive defines how systemd determines when a service is considered started. This affects when systemd proceeds to start dependent units.

| Type       | Description                                                                        | Service is considered ready                   |
|------------|------------------------------------------------------------------------------------|-----------------------------------------------|
| `simple`   | The default. The `ExecStart=` process is the service                               | Immediately after `ExecStart=` is forked      |
| `exec`     | Like `simple`, but with better error detection for missing binaries or bad `User=` | After `ExecStart=` has been successfully executed |
| `forking`  | The service forks a child process and the parent exits (traditional daemon pattern) | After the parent process exits                |
| `oneshot`  | Runs a single command and exits (often paired with `RemainAfterExit=yes`)          | After `ExecStart=` completes                  |
| `dbus`     | The service acquires a D-Bus bus name (requires `BusName=`)                        | After the D-Bus name is acquired              |
| `notify`   | The service signals readiness to systemd via `sd_notify()` (requires `NotifyAccess=`) | After the service calls `sd_notify("READY=1")` |
| `idle`     | Like `simple`, but delays execution until all active jobs are dispatched (5s timeout) | Same as `simple`, after a startup delay      |

## Service Restart Behavior

The `Restart=` directive defines the conditions under which systemd automatically restarts a service after it exits:

| Policy         | Restarts when                                                              | Typical use                          |
|----------------|----------------------------------------------------------------------------|--------------------------------------|
| `no`           | Never (default)                                                            | One-time tasks, manual management    |
| `always`       | Always, regardless of exit reason                                          | Critical services that must always run |
| `on-success`   | Clean exit (exit code 0, or signals SIGHUP/SIGINT/SIGTERM/SIGPIPE)         | Tasks that should repeat on success  |
| `on-failure`   | Unclean exit code, unclean signal, timeout, or watchdog                    | Most production services             |
| `on-abnormal`  | Unclean signal, timeout, or watchdog (but not exit codes)                  | Fault-tolerant daemons               |
| `on-abort`     | Terminated by an unclean signal (e.g., SIGSEGV, SIGABRT, SIGBUS)          | Crash-recovery scenarios             |
| `on-watchdog`  | The watchdog timeout (`WatchdogSec=`) elapses without a keep-alive ping    | Services with health-check monitoring |

The following table (from the systemd man page) shows exactly which exit causes trigger a restart for each policy:

| Exit cause        | `no` | `always` | `on-success` | `on-failure` | `on-abnormal` | `on-abort` | `on-watchdog` |
|-------------------|------|----------|--------------|--------------|---------------|------------|---------------|
| Clean exit        |      | ✓        | ✓            |              |               |            |               |
| Unclean exit code |      | ✓        |              | ✓            |               |            |               |
| Unclean signal    |      | ✓        |              | ✓            | ✓             | ✓          |               |
| Timeout           |      | ✓        |              | ✓            | ✓             |            |               |
| Watchdog          |      | ✓        |              | ✓            | ✓             |            | ✓             |

---

[← Previous: Unit Files](06_systemd_unit_files.md) | [Next: Unit Dependencies →](08_systemd_dependencies.md)
