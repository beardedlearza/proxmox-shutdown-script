# Proxmox Staggered Shutdown

Automatically shuts down all running LXC containers and VMs on a Proxmox host at a scheduled time. Shutdowns are staggered to reduce load spikes, with graceful shutdown first and force-stop as a fallback.

## Features

- **Staggered shutdown** — configurable delay between each shutdown command to avoid simultaneous load spikes
- **Two-phase approach** — graceful shutdown first, automatic force-stop if the timeout is exceeded
- **Parallel phases** — LXC containers and VMs are each handled sequentially, but both phases complete before the script moves on
- **Log rotation** — automatically rotates the log file if it exceeds 10 MB
- **Detailed logging** — every action is timestamped and written to `/var/log/proxmox-midnight-shutdown.log`

## Files

| File | Description |
|---|---|
| `proxmox-shutdown.sh` | The main shutdown script |
| `SETUP_INSTRUCTIONS.md` | Full installation and configuration guide |

## Quick Start

### 1. Copy the script to your Proxmox host

```bash
scp proxmox-shutdown.sh root@your-proxmox-host:/usr/local/bin/
```

### 2. Make it executable

```bash
ssh root@your-proxmox-host
chmod +x /usr/local/bin/proxmox-shutdown.sh
```

### 3. Schedule it

**Option A — cron (simple):**

```bash
crontab -e
# Add this line to run at midnight:
0 0 * * * /usr/local/bin/proxmox-shutdown.sh
```

**Option B — systemd timer (recommended):**

```bash
# Create the service unit
cat > /etc/systemd/system/proxmox-shutdown.service <<EOF
[Unit]
Description=Shutdown all Proxmox VMs and LXCs

[Service]
Type=oneshot
ExecStart=/usr/local/bin/proxmox-shutdown.sh
EOF

# Create the timer unit
cat > /etc/systemd/system/proxmox-shutdown.timer <<EOF
[Unit]
Description=Run Proxmox shutdown at midnight

[Timer]
OnCalendar=00:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

systemctl daemon-reload
systemctl enable --now proxmox-shutdown.timer
```

## Configuration

All options are at the top of `proxmox-shutdown.sh`:

| Variable | Default | Description |
|---|---|---|
| `STAGGER_DELAY` | `5` | Seconds between each shutdown command |
| `SHUTDOWN_TIMEOUT` | `300` | Seconds to wait for graceful shutdown (5 min) |
| `FORCE_STOP_TIMEOUT` | `600` | Total timeout before giving up entirely (10 min) |
| `LOG_FILE` | `/var/log/proxmox-midnight-shutdown.log` | Log file path |

### Change the scheduled time

For cron, adjust the first two fields:

```
0 2    * * *   # 2:00 AM
30 23  * * *   # 11:30 PM
```

For the systemd timer, update `OnCalendar` in the timer file:

```ini
OnCalendar=02:00:00    # 2:00 AM
OnCalendar=23:30:00    # 11:30 PM
```

### Exclude specific VMs or containers

Edit the script's `for` loops to skip particular IDs:

```bash
# Skip VM 100
for vmid in $(qm list | awk 'NR>1 && $3=="running" && $1!=100 {print $1}'); do

# Skip LXC 200
for ctid in $(pct list | awk 'NR>1 && $2=="running" && $1!=200 {print $1}'); do
```

## Shutdown Sequence

```
Phase 1 — LXC containers: graceful shutdown, one by one with stagger delay
Phase 2 — VMs:            graceful shutdown, one by one with stagger delay
Phase 3 — Wait:           poll every 10s until all stopped or SHUTDOWN_TIMEOUT reached
Phase 4 — Force stop:     any remaining containers/VMs are forcibly stopped
```

## Viewing Logs

```bash
tail -f /var/log/proxmox-midnight-shutdown.log
```

## Testing

Run the script manually to verify it works before relying on the schedule. **Do not run during production hours.**

```bash
/usr/local/bin/proxmox-shutdown.sh
```

## Notes

- The script does **not** shut down the Proxmox host itself — only guests
- Requires root privileges (standard for Proxmox management scripts)
- `pct` and `qm` must be available at `/usr/sbin/` (default on Proxmox)

## License

MIT — use freely, modify as needed.
