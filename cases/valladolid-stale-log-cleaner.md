# Incident Post-Mortem: Valladolid (Stale Log Cleaner Failure)

> **Scenario Source:** SadServers Practice Environment
> **Impact Duration:** Persistent silent failure (simulated backstory)
> **Time to Resolution:** ~5 minutes

---

## 1. Executive Summary

The automated systemd service `log-cleaner.service` was executing with an exit code of `0` (success), but failing to delete application log files older than 7 days inside `/var/log/app/`. Recent files were safely preserved, but old files accumulated unchecked. The issue was traced to an inverted time comparison in the `find` command combined with a missing space that broke argument parsing. The service was restored by correcting the `find` time flag and fixing the argument spacing.

---

## 2. Timeline & Detection

* **00:00** – **Incident identified:** `/var/log/app/old_data.log` (older than 7 days) persisted despite manual runs of `sudo systemctl start log-cleaner`.
* **00:02** – **Script inspection:** Reviewed the script run by the service (`/opt/scripts/cleaner.sh`). Found an incorrect time flag (`-mtime -7`), which matches files *less* than 7 days old instead of older.
* **00:03** – **Syntax error found:** Ran the `find` command manually to isolate the failure. It threw: `find: Expected a positive decimal integer argument to -maxdepth, but got '1-name'`.
* **00:04** – **Resolution applied:** Corrected the flag to `-mtime +7`, added the missing space between `-maxdepth 1` and `-name`, and reset test data using `~/reset_logs.sh`.
* **00:05** – **Validation:** Restarted the service (`sudo systemctl restart log-cleaner`) and confirmed `old_data.log` was deleted while `recent_data.log` was left untouched.

---

## 3. Root Cause Analysis (RCA)

The failure came down to two bugs in the script logic:

1. **Inverted time comparison (`-mtime -7` vs `+7`):** `find`'s `mtime` filter works on the file's modification time:
   * `-mtime -7` matches files modified *within* the last 7 days.
   * `-mtime +7` matches files modified *more than* 7 days ago.

   The script used `-7`, so it was checking for recent files instead of stale ones — the opposite of its intended purpose.

2. **Missing space merged two flags into one invalid argument:** A missing space between `-maxdepth 1` and `-name` caused the shell to read them as a single string (`1-name`), which `find` rejected as invalid, crashing mid-execution.

Because the script had no strict error handling (no `set -e`), the `find` failure was silently swallowed and the wrapping script still exited with code `0` — so systemd reported the service as healthy on every run.

---

## 4. Detection Gap

systemd's own health check only tracks whether the process exited without crashing — it has no visibility into whether the script actually did its job. Since the wrapping script exited `0` regardless of the internal `find` failure, standard unit-status checks (`systemctl is-active`, `systemctl is-failed`) would have shown green the entire time this was broken.

There was no check on the actual outcome — file age in `/var/log/app/` — only on process exit status. A simple disk-usage or file-age check (e.g. alert if the oldest file in that directory exceeds ~10 days) would have caught this regardless of what the script reported, and would have surfaced the problem within one cleanup cycle instead of running silently indefinitely.

---

## 5. Corrective & Preventative Actions

* **Fix the script logic:** Corrected the directory target (`find /var/log/app`), the time filter (`-mtime +7`), and the argument spacing.
* **Least privilege:** Avoided `chmod 777` during triage; kept the script at `755` (executable only by root/owner) rather than opening it to broader write access.
* **Enforce strict shell failure handling:** Prepend `set -euo pipefail` to the script:
  * `-e` halts the script immediately on any command failure, instead of continuing and reporting a false success.
  * `-u` treats use of an undefined variable as an error, catching typos before they cause silent bad behavior.
  * `-o pipefail` makes a pipeline (`cmd1 | cmd2`) fail if *any* stage fails, not just the last one — otherwise a failure earlier in a pipe can be masked by a successful final command.

  Together these ensure a real failure bubbles up as a non-zero exit code, so systemd (and any alerting built on top of it) actually reflects reality.
* **Add outcome-based monitoring:** Alert on file age in `/var/log/app/` directly, rather than relying solely on the service's reported exit status.
