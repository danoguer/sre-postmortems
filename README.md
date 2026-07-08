# Linux Troubleshooting & Incident Postmortems

A practice log documenting systems triage and root cause analysis (RCA), built while training toward a junior DevOps/SRE role. Each entry works through a broken Linux environment — diagnosing the failure, identifying the root cause, and writing it up in a professional postmortem format.

Scenarios are sourced from **SadServers** practice labs. These are training exercises, not real production incidents — timelines in each writeup reflect the scenario's simulated backstory, not actual hands-on time (which is noted explicitly in each document).

---

## 🛠️ Environment

Scenarios are debugged inside Linux VMs (Debian/Ubuntu) using standard system diagnostic tools:
* `systemd` (Service orchestration)
* `procfs` (Process attributes and file descriptors)
* `syslog` / `journalctl` (Log aggregation)
* `strace` (System call tracing)
* Common network utilities (`ss`, `netstat`, `curl`, `nc`)

---

## 📁 Logged Incidents

### Systems & Automation Primitives
* **[Alexandria: The Vanishing Backups](./cases/alexandria-vanishing-backups.md) (Easy)**
  * *Concepts:* Vixie `cron` execution context, unprivileged execution limits, silent stdout/stderr suppression (`> /dev/null 2>&1`), stale lock-file handling.
* **[Valladolid: Cleaner Not Cleaning](./cases/valladolid-stale-log-cleaner.md) (Easy)**
  * *Concepts:* `systemd` service management, `find` time-based filters (`-mtime`), shell string-concatenation pitfalls, defensive scripting (`set -e`).

---

## 🔬 Postmortem Format

Each writeup follows a consistent, production-grade structure:

1. **Detection & Triage** — Reproducing the failure and narrowing down the cause without making destructive changes.
2. **Root Cause Analysis (RCA)** — Mapping the exact chain of failures (permissions, configuration typos, service dependency states, etc.).
3. **Detection Gap** — Analyzing what monitoring or alerting layer *should* have caught this error, but failed to do so.
4. **Corrective Actions** — Implementing concrete architectural fixes, including migrating fragile legacy patterns (e.g., `cron` $\rightarrow$ `systemd` timers) or embedding safety mechanisms (e.g., bash `trap` routines for automated cleanup).

> **The guiding principle:** The goal isn't just to "fix the box" — it's practicing the exact reasoning, debugging, and documentation habits required in production SRE environments.
