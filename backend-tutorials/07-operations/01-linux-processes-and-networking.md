# Linux processes, services, and networking

[Tutorial index](../README.md) · Next: [containers](02-containers-and-local-environments.md)

## Outcome and prerequisites

Understand how a backend exists as an operating-system process and how requests reach its socket. You need basic terminal skills and a Linux VM, WSL environment, or Linux container for commands labeled Bash. The user's host can remain Windows; PowerShell alternatives are shown where useful. Do not paste Linux service-management commands into PowerShell and expect identical behavior.

A process has an ID, parent, credentials, environment, working directory, open file descriptors, memory, and executable. Threads share process memory but have their own execution state. A service manager supervises processes across crashes and machine restarts. A container adds isolation and resource controls around processes; it does not remove these operating-system concepts.

## Files, permissions, and resource limits

Linux permissions distinguish owner, group, and others, with read, write, and execute bits. Directory execute permission controls traversal; write permission on a directory affects creation and deletion of entries. Run the service as a dedicated unprivileged account, give it read access to its artifact and narrowly scoped write access where needed, and keep secrets out of broadly readable files.

Use absolute paths in service configuration. Interactive shells may have a different `PATH`, working directory, or environment from a service manager. A program that works after manually sourcing a profile may fail during unattended startup. Configuration should identify required variables and fail clearly when missing.

Each socket, file, and pipe consumes a file descriptor. Leaks may eventually produce “too many open files” even when CPU is low. Memory exhaustion can cause the kernel or container runtime to terminate a process. CPU quotas can create throttling despite an apparently moderate average host CPU. Observe limits at the layer where the service actually runs.

## Walkthrough: inspect a local process and listener

1. Start the capstone normally on port 3000. In PowerShell, inspect the listener and connectivity:

```powershell
Get-NetTCPConnection -LocalPort 3000 -State Listen
Test-NetConnection 127.0.0.1 -Port 3000
curl.exe --fail --silent --show-error http://127.0.0.1:3000/health/live/
```

Expect a listening socket, a successful TCP check, and a successful health response. A TCP connection alone does not prove HTTP or database correctness.

2. On Linux, use the corresponding read-only inspection commands. `ss` comes from the system's networking tools and may need to be installed by the environment owner.

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,args
ss -ltnp
curl --fail --silent --show-error http://127.0.0.1:3000/health/live/
```

Process details for sockets owned by other users may require additional privileges. Do not elevate merely to explore unrelated processes.

3. Compare binding to `127.0.0.1` with binding to `0.0.0.0`. Loopback accepts local IPv4 connections; the wildcard listens on available IPv4 interfaces. Binding publicly does not itself configure a firewall or load balancer. Inside a container, loopback refers to that container's network namespace, not the host or database container.

4. Diagnose in layers: DNS resolution, route/firewall, TCP connection, TLS negotiation, HTTP routing, authentication, application logic, then dependencies. Record the first failing layer. A timeout differs from “connection refused,” and a valid HTTP 503 differs from failure to connect.

## Signals and graceful shutdown

On Linux, service managers commonly request termination with `SIGTERM` before eventually forcing termination. `SIGKILL` cannot be handled by application code. Implement graceful shutdown by marking readiness false, stopping acceptance of new work, finishing bounded in-flight operations, closing database pools and consumers, and exiting before the supervisor's deadline.

Use a bounded shutdown timer. A process that waits forever for a stuck request can block deployments; a process that exits immediately can interrupt acknowledged work. Queue consumers should stop fetching before draining current jobs. A handler that starts asynchronous cleanup but exits immediately has not actually completed cleanup.

Windows signal and console behavior differ from Linux. Verify the deployment platform's shutdown behavior directly rather than relying only on Ctrl+C during local development. Python signal handlers run under specific main-thread constraints; let the chosen production server own worker supervision and shutdown. [Python signal documentation](https://docs.python.org/3.12/library/signal.html)

## Example systemd service

The following is a configuration template for a Linux host using systemd, not a command to install or activate a service. It assumes a dedicated `taskservice` account, a Django application under `/opt/task-service`, an installed Linux virtual environment containing a pinned compatible Gunicorn release, and a restricted environment file created by the operator. Gunicorn is an optional production-server dependency; it is not included in the basic capstone requirements.

Use an adapted application with real authentication, production settings, and the capstone's demo behavior removed before deployment. This host template binds to loopback for a separately configured local reverse proxy; it does not publish the unauthenticated learning API.

```ini
[Unit]
Description=Task Service API
After=network.target

[Service]
Type=simple
User=taskservice
Group=taskservice
WorkingDirectory=/opt/task-service
EnvironmentFile=/etc/task-service/task-service.env
ExecStart=/opt/task-service/.venv/bin/gunicorn config.wsgi:application --bind 127.0.0.1:3000 --workers 2 --graceful-timeout 25
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
UMask=0077

[Install]
WantedBy=multi-user.target
```

The read-only filesystem policy means this template suits a stateless application that writes logs to standard output. Add explicit writable paths only for a demonstrated need. `After=network.target` orders units; it does not guarantee that an external database is ready. The application still needs bounded connection attempts and useful failure reporting. Consult the host's systemd version before adopting directives. [systemd service configuration](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)

With an already configured unit, `systemctl status task-service` and `journalctl -u task-service --since '10 minutes ago'` are inspection commands. Logs can contain sensitive information, so export only the relevant redacted evidence.

## Networking tradeoffs and failure exercises

Connection reuse avoids repeated TCP and TLS setup but requires idle timeouts compatible with proxies and servers. DNS caches improve efficiency but can delay endpoint changes. Keep-alive sockets can become stale after deployment, so clients need bounded retries only where operations are safe to retry. Outbound connection bursts may exhaust ephemeral ports or NAT limits even when the application has spare CPU.

TLS certificate validation checks identity and trust; disabling validation to “fix” a connection removes a security boundary. Inspect hostname, certificate chain, expiry, and clock accuracy instead. Ensure health endpoints reveal minimal information and are reachable only from intended monitoring or infrastructure paths when appropriate.

Practice stopping only your local service and observing connection refusal, then starting it and sending an invalid route to observe an HTTP error. Compare those with a deliberately unavailable local database. Write the diagnosis for each layer. Confirm the application drains an in-flight request during shutdown and exits within its documented deadline. Avoid using forced termination as the normal operational path.
