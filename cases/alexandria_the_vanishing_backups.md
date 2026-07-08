# Incident Post-Mortem: Alexandria (The Vanishing Backups)

**Scenario source:** SadServers practice environment. The timeline below reflects the scenario's simulated backstory (silent failure over 72h); actual hands-on diagnosis time was ~5 minutes.

## 1. Executive Summary

A critical backup cron job was identified as silently failing for 3 days (simulated). The backup script located at `/opt/backup/backup.sh` was failing to produce the daily compressed archives in `/var/backups/daily/`. The system showed no external error logs or active alerts, and the `cron` daemon was running normally. The issue was resolved by fixing the targeted file path in the root crontab, resolving a permission conflict, and clearing a stale application lock file.

---

## 2. Timeline & Detection

* **T-72h** - Last successful backup archive recorded in `/var/backups/daily/` (per scenario backstory — no real-time monitoring caught the gap between this point and detection).
* **00:00** - Incident identified: Verified that `/var/backups/daily/` lacked any new archives for the past 72 hours.
* **00:02** - Manual Script Testing: Attempted to run `./backup.sh` as the unprivileged `admin` user. The script failed immediately with `tar: Cannot open: Permission denied` when attempting to write to the protected `/var/backups/daily/` directory.
* **00:03** - Concurrency Lock Collision: Attempted execution using elevated privileges (`sudo ./backup.sh`). The script aborted with `Error: Backup already running (lock file exists)`.
* **00:04** - Process Investigation: Executed `ps aux | grep "[b]ackup"` to find active background jobs. No real running processes were found, indicating a stale/ghost lock file.
* **00:05** - Resolution Applied: Cleared the stale lock file, corrected the targeted script string in the `root` crontab, and successfully validated automated execution.

---

## 3. Root Cause Analysis (RCA)

The outage was caused by a multi-layered failure chain across permissions, scheduling configuration, and poor error observability:

1. **The Silent Failure Cloak (`> /dev/null 2>&1`):** The automated backup job configuration redirected both standard output (`stdout`) and standard error (`stderr`) to `/dev/null`. This completely masked execution failures from the system logs and automated alerting tools.
2. **Path Misconfiguration:** The configuration inside `sudo crontab -l` was targeting a deprecated script path (`/opt/backup/old_backup.sh`) instead of the active production script (`/opt/backup/backup.sh`).
3. **Ghost Lock Mechanism State:** Due to previous manual/unprivileged execution attempts by the `admin` user, the script crashed mid-execution due to directory write permissions. Because the script exited abruptly, it bypassed its cleanup routine, leaving a stale `backup.lock` file in the directory. This safely but permanently blocked all subsequent manual attempts until cleared.

---

## 4. Detection Gap

This incident was only found through a manual check of `/var/backups/daily/`, not through any automated system. Specifically:

* No alert existed for "backup job did not run" or "backup job exited non-zero" — the cron job's own errors were silently discarded by `> /dev/null 2>&1`, so there was nothing for a monitoring tool to catch even if one had existed.
* No freshness check existed on the backup directory itself (e.g. "alert if newest file in `/var/backups/daily/` is older than 25h").
* A basic healthcheck-style approach (script pings a monitoring endpoint like Healthchecks.io or pushes an exit-code metric on completion) would have surfaced this within one missed run (~24h) instead of 3 days.

This gap is the actual highest-value fix here — the permission and path issues are one-off bugs, but the lack of observability is what let the failure go silent for days.

---

## 5. Corrective & Preventative Actions

* **Fix immediate infrastructure:** Update the root crontab to accurately point to `/opt/backup/backup.sh` and remove the `> /dev/null 2>&1` suppression to restore observability.
* **Implement Resilient Script Error Handling (Bash Traps):** Update the backup script logic to use a `trap` on `EXIT` so cleanup always runs, regardless of how the script terminates (normal completion, error, or signal):

```bash
# Example of a resilient pattern added to the script
cleanup() {
  rm -f /opt/backup/backup.lock
}
trap cleanup EXIT
```

  Note: trapping `EXIT` alone is sufficient and safer than trapping `INT TERM EXIT` with the same inline handler — `exit` inside a handler re-triggers the `EXIT` trap, which can cause a handler bound to multiple signals to fire more than once. Using a dedicated cleanup function bound only to `EXIT` avoids that ambiguity while still covering normal exits, errors, and received signals.
* **Add basic freshness monitoring:** Alert if the newest file in `/var/backups/daily/` exceeds a defined age threshold (e.g. 25h), independent of whether the cron job itself reports success.
* **Add exit-code observability:** Have the script report success/failure explicitly (e.g. to a monitoring endpoint, log aggregator, or simple status file) instead of discarding output entirely.
