# 🩺 Linux Troubleshooting Walkthroughs

The commands in [README.md](README.md) are the *what*. This is the *how* — the
actual diagnostic sequence I'd walk through for common production problems,
basic to advanced, in the order I'd really check things (not a random list).

Each one follows the same shape: **symptom → diagnose → fix → prevent
recurrence.**

---

## Basic

### 1. "The server feels slow"

- `uptime` — check the load average first. Compare it to core count (`nproc`)
  — load of 8 on a 4-core box is a real problem; load of 8 on a 32-core box
  might not be.
- `top -c` / `htop` — sort by CPU, then by memory (`Shift+M` in top), see
  what's actually consuming resources.
- `vmstat 2 5` — watch `r` (runnable processes waiting for CPU) and `wa`
  (I/O wait). High `wa` means it's disk/network, not CPU.
- `iostat -xz 2` (if available) — confirm which disk is saturated if `wa`
  is high.
- **Fix/prevent:** kill or renice the offending process if it's a fluke;
  if it's recurring, that's a capacity or autoscaling conversation, not a
  one-off restart.

### 2. "Disk is full"

- `df -h` — confirm which mount is actually full.
- `du -sh /* 2>/dev/null | sort -rh | head -10` from the root of that mount
  to find the biggest directories.
- Common repeat offenders: `/var/log` (uncontrolled logging), Docker
  layers/images (`docker system df`), core dumps, old kernel versions in
  `/boot`.
- Careful with deleting a file that's still held open by a running process
  (`lsof | grep deleted`) — `rm` won't reclaim the space until the process
  that has it open closes it; you may need to restart that process, or
  truncate instead of delete: `> /path/to/file.log`.
- **Fix/prevent:** logrotate for anything writing logs without one, disk
  usage alerting *before* 100%, not after.

### 3. "Service won't start"

- `systemctl status <service> -l` first — read the actual error, not just
  "failed."
- `journalctl -u <service> -n 100 --no-pager` for the full recent log.
- Check the obvious: config file syntax (most services have a `--test` or
  `-t` config-check flag, e.g. `nginx -t`), permissions on files it needs to
  read, and whether the port it wants is already in use (`ss -tlnp | grep
  <port>`).
- **Fix/prevent:** if it's a permissions issue after a redeploy, that's a
  deployment-script gap worth fixing once, not fixing by hand every time.

### 4. "Can't SSH into a server"

- From another host: `ping`, then `nc -zv <host> 22` (or `telnet host 22`) to
  isolate network reachability from the SSH service itself.
- If ping works but port 22 doesn't respond: check the security
  group/firewall (`iptables -L` / cloud console) and whether `sshd` is
  actually running (via console/serial access if you're locked out).
- If it worked a minute ago and now doesn't: check disk space and memory
  first (`df -h`, `free -m`) — a full disk or OOM state can silently take
  down sshd along with everything else.

---

## Intermediate

### 5. "App keeps crashing / restarting"

- `dmesg -T | grep -i "killed process"` and `journalctl -k | grep -i oom` —
  confirm or rule out the OOM killer first; it's the most common silent
  cause and people often skip straight to app logs without checking.
- If it's OOM: `free -mh` for current headroom, then look at the process's
  actual memory growth over time (`ps -o pid,vsz,rss,comm -p <pid>` sampled
  a few times, or a monitoring graph if you have one) to tell a leak from a
  legitimate spike.
- If not OOM: application logs + exit code (`systemctl status` shows the
  last exit code) — a segfault (139), vs. an unhandled exception, vs. a
  liveness-probe kill in Kubernetes, all point in different directions.
- **Fix/prevent:** if it's a slow leak, that's a restart-as-mitigation +
  root-cause-later situation — don't let "restart fixed it" become the
  final answer without a follow-up ticket.

### 6. "High CPU from a process I don't recognize"

- `top -c` to get the PID, then `ps -fp <pid>` for the full command line —
  what actually launched it and with what arguments.
- `ls -l /proc/<pid>/exe` to see what binary it actually is (names can be
  spoofed; the resolved path can't as easily).
- `lsof -p <pid>` to see open files/sockets — is it doing disk I/O, network
  I/O, or pure compute?
- If it's unexpected and can't be explained by any deployed application,
  treat it as a potential compromise, not just a performance issue, until
  ruled out.

### 7. "Two hosts can't talk to each other"

- `ping` to rule out basic reachability, but don't stop there — ICMP being
  blocked doesn't mean the actual service port is blocked, and vice versa.
- `nc -zv <host> <port>` (or `curl -v` for HTTP-based services) to test the
  actual port in question.
- `traceroute`/`mtr` to see where in the path it's actually dropping, if
  ICMP is allowed.
- Check both directions and both ends: security groups/firewall rules on
  each host, and whether the listening service is bound to the right
  interface (`0.0.0.0` vs `127.0.0.1` vs a specific private IP —
  `ss -tlnp` shows this).

### 8. "A log file grew until it filled the disk"

- Immediate: `> /path/to/logfile` to truncate it in place (safer than `rm`
  if the writing process still has the file handle open — truncating
  reclaims the space instantly without needing to restart anything).
- Root cause: is this normal volume that logrotate should have handled but
  didn't (check `logrotate -d /etc/logrotate.d/<config>` for a dry run of
  what it would do), or is the application unexpectedly log-spamming
  (usually an error loop logging the same failure thousands of times)?
- **Fix/prevent:** if logrotate config exists but didn't fire, check its
  cron/timer actually ran (`journalctl -u logrotate`); if it doesn't exist
  for this log yet, that's the actual fix.

---

## Advanced

### 9. Zombie / defunct processes piling up

- `ps aux | awk '$8=="Z"'` (or grep for `<defunct>`) to find them and their
  PPID.
- Zombies themselves are basically harmless (they're just an exit-status
  entry waiting to be reaped) — the real problem is the **parent process**
  not calling `wait()` on its children. Fixing the zombie means fixing or
  restarting the parent, not the zombie itself (you can't kill a zombie —
  it's already dead).
- If the parent is PID 1 (init/systemd) and zombies are still accumulating,
  that points at something unusual in how processes are being spawned and
  is worth escalating rather than treating as routine.

### 10. Resource limits silently capping an application

- "Too many open files" errors: `ulimit -n` for the current shell's limit,
  but the process's actual limit is `cat /proc/<pid>/limits` — these can
  differ if the service was started under a different context (systemd unit
  `LimitNOFILE=`, not just `/etc/security/limits.conf`, is what actually
  applies to systemd-managed services).
- Check current usage against the limit: `lsof -p <pid> | wc -l`.
- **Fix/prevent:** raise the limit at the layer that actually enforces it
  for that process (systemd override, not just the shell config that
  doesn't apply to it), and understand *why* it needed more — a genuine
  scaling need vs. a connection/file-descriptor leak are different problems
  wearing the same symptom.

### 11. Package/dependency conflicts during patching

- Before patching: `yum history` / `apt list --installed` to have a
  baseline, and check what a patch would actually pull in
  (`yum check-update`, `apt-get upgrade -s` for a dry run) before applying
  it blind.
- If a patch broke something: `yum history undo <id>` (RHEL/CentOS) or
  restoring from a pre-patch snapshot is almost always faster and safer
  than trying to hand-fix a half-updated dependency tree.
- **Prevent:** patch a staging/canary host first, and always have a
  rollback plan *before* patching production, not improvised after.

### 12. Systematic performance degradation under load (no obvious single cause)

This is the one that's actually a methodology, not a single command:

1. **CPU** — `top`/`vmstat`: is it compute-bound?
2. **Memory** — `free -mh`, and specifically watch for swapping
   (`vmstat` `si`/`so` columns) — a system that's swapping will look slow
   everywhere, not just in memory metrics.
3. **Disk I/O** — `iostat -xz` — high `%util` or `await` on a disk the app
   depends on.
4. **Network** — `ss -s` for connection counts, `sar -n DEV` for
   throughput, or just packet loss/retransmits if it's cross-host.
5. Only after ruling out infrastructure do I look at the **application
   layer itself** — thread pool exhaustion, database connection pool
   limits, a slow downstream dependency — because those produce the exact
   same symptom ("everything is slow") as an infra bottleneck, and
   guessing at the app layer first wastes time if the real cause is one
   layer down.

The discipline that matters here isn't knowing every command — it's
checking the layers in a fixed order every time, so "it's probably the
database" doesn't skip past a memory-swapping host that would explain
everything at once.
