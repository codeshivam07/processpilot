# ProcessPilot Pro — Dependency-Aware Linux Service Supervisor (C++17)

A small init-style process supervisor for Linux. It reads a config file listing services and
their dependencies, starts them in dependency order, detects crashes, restarts them with
exponential backoff, restarts dependents of a crashed service, and can be controlled at runtime
with the `ppctl` command over a Unix domain socket.

## Author
- **Name:** <YOUR FULL NAME>
- **Roll no. / Course:** <ROLL NUMBER>, <BRANCH>, ITER, Siksha 'O' Anusandhan University
- **Project type:** Group project topic, individually submitted. Teammates: <TEAMMATE NAMES>
- **My contribution:** <WRITE WHAT YOU PERSONALLY DID, e.g. "supervisor state machine and restart/backoff logic", "config parser and unit tests">

## Features
- INI config with validation (unknown/cyclic dependencies are rejected with clear errors)
- Dependency-ordered start (topological sort) and reverse-order stop
- Crash detection via `SIGCHLD` / `waitpid`; restart policies: `never`, `on-failure`, `always`
- Exponential backoff, `max_restarts` limit, then the service is marked `Failed`
- Restart cascade: dependents are bounced when a dependency goes down
- Graceful stop: `SIGTERM` to the process group, `SIGKILL` after a timeout
- Runtime control CLI: `status`, `start`, `stop`, `restart`, `logs`, `shutdown`
- Per-service log files

## Requirements
- Linux (Ubuntu 20.04+ recommended). Windows users: WSL2 or a Linux VM. macOS is not supported.
- `g++` 9 or newer (C++17), `make`, `bash`, `git`. No root and no other libraries needed.

```
sudo apt update && sudo apt install -y build-essential git
```

## Build
```
git clone https://github.com/codeshivam07/processpilot.git
cd processpilot
make
```
This produces `build/processpilot` (the daemon) and `build/ppctl` (the control client).

## Run the tests
```
make test
```
Expected: `19/19 unit checks passed` and `integration: 24 passed, 0 failed`.

## Run the demo
Terminal 1:
```
./build/processpilot -c examples/demo.conf -l /tmp/pp-logs
```
Terminal 2 (same folder):
```
./build/ppctl status          # db, api, web Running; flaky-job restarting, then Failed
kill -9 <pid of db>           # use the PID shown by status
./build/ppctl status          # db restarted, api and web were bounced
./build/ppctl stop db         # dependents stop first
./build/ppctl start web       # pulls in db and api
./build/ppctl logs 20         # recent events
./build/ppctl shutdown        # clean exit
```
Validate a config without running it: `./build/processpilot --check -c examples/demo.conf`

## Config format
Each service is a section. Example:
```ini
[db]
command=sleep 100000
restart=always

[api]
command=sleep 100001
depends=db
restart=on-failure
max_restarts=5
backoff_ms=500
```

| Key | Meaning | Default |
|---|---|---|
| `command` | Program and arguments (quotes supported, no shell). Required | — |
| `depends` | Comma-separated services that must be running first | none |
| `restart` | `never`, `on-failure` or `always` | `on-failure` |
| `max_restarts` | Consecutive restarts before the service is marked Failed | 5 |
| `backoff_ms` | Base restart delay, doubled each time (capped at 30 s) | 500 |
| `ready_delay_ms` | How long a service must run before dependents start | 0 |
| `stop_timeout_ms` | Wait after `SIGTERM` before `SIGKILL` | 3000 |

A section may also be written as `[service:name]`.

## Commands
`status` · `start <svc|all>` · `stop <svc|all>` · `restart <svc|all>` · `logs [n]` · `shutdown`

`start X` also starts X's dependencies; `stop X` also stops X's dependents.

## Project structure
```
src/        config parser, dependency graph, supervisor core, daemon main, ppctl client
tests/      unit_tests.cpp (parser, graph) and integration.sh (end-to-end)
examples/   demo.conf
Makefile
```

## Architecture
```
config file -> parser -> dependency graph (topological order) -> supervisor event loop
                                                                 |-- fork/exec services
                                                                 |-- SIGCHLD -> waitpid -> restart/backoff/cascade
                                                                 '-- Unix socket <- ppctl
```
Single-threaded `poll()` loop over a `signalfd` and the control socket, with a 100 ms tick.
Each service has a state: Stopped, Running, Backoff, Stopping, Failed.

## Device drivers
ProcessPilot is a user-space supervisor, so no kernel driver is required for it to work.
A kernel-module event sink (char device) is listed as future work.

## Limitations and future work
No cgroup/namespace isolation; readiness is time-based, not health-probe based; no config reload;
single host. Possible additions: health checks, `reload`, cgroup limits, a kernel-module event sink.
