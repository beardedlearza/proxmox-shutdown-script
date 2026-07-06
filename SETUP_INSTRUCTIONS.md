# Proxmox Midnight Shutdown Setup Instructions

## Installation Steps

### 1. Copy the script to your Proxmox host
```bash
# Upload proxmox-shutdown.sh to your Proxmox host
# Place it in /usr/local/bin/
scp proxmox-shutdown.sh root@your-proxmox-host:/usr/local/bin/
```

### 2. Make the script executable
```bash
ssh root@your-proxmox-host
chmod +x /usr/local/bin/proxmox-shutdown.sh
```

### 3. Set up the cron job for midnight execution
```bash
# Edit root's crontab
crontab -e

# Add this line to run at midnight (00:00) every day:
0 0 * * * /usr/local/bin/proxmox-shutdown.sh
```

### 4. Verify the cron job
```bash
# List current cron jobs
crontab -l
```

## Alternative: Using systemd timer (more modern approach)

### Create service file
```bash
cat > /etc/systemd/system/proxmox-shutdown.service <<EOF
[Unit]
Description=Shutdown all Proxmox VMs and LXCs

[Service]
Type=oneshot
ExecStart=/usr/local/bin/proxmox-shutdown.sh
EOF
```

### Create timer file
```bash
cat > /etc/systemd/system/proxmox-shutdown.timer <<EOF
[Unit]
Description=Run Proxmox shutdown at midnight

[Timer]
OnCalendar=00:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF
```

### Enable and start the timer
```bash
systemctl daemon-reload
systemctl enable proxmox-shutdown.timer
systemctl start proxmox-shutdown.timer

# Check timer status
systemctl status proxmox-shutdown.timer
systemctl list-timers
```

## Checking Logs

View the shutdown log:
```bash
tail -f /var/log/proxmox-midnight-shutdown.log
```

## Testing the Script

Run manually to test (don't do this during business hours!):
```bash
/usr/local/bin/proxmox-shutdown.sh
```

## Customization Options

### Change shutdown time
For cron, modify the first two numbers:
- `0 0` = midnight (00:00)
- `0 2` = 2 AM
- `30 23` = 11:30 PM

For systemd timer, modify OnCalendar in the timer file:
- `OnCalendar=02:00:00` = 2 AM
- `OnCalendar=23:30:00` = 11:30 PM

### Exclude specific VMs/LXCs
Edit the script to add exclusions:
```bash
# Example: Skip VM 100 and LXC 200
for vmid in $(qm list | awk 'NR>1 && $3=="running" && $1!=100 {print $1}'); do
    qm shutdown $vmid &
done

for ctid in $(pct list | awk 'NR>1 && $2=="running" && $1!=200 {print $1}'); do
    pct shutdown $ctid &
done
```

### Adjust timeout
Change `TIMEOUT=300` to a different value (in seconds)
- 300 = 5 minutes
- 600 = 10 minutes

## Notes

- The script uses graceful shutdown (`pct shutdown` and `qm shutdown`)
- All shutdowns run in parallel for faster execution
- A 5-minute timeout prevents infinite waiting
- Logs are saved to `/var/log/proxmox-midnight-shutdown.log`
- The script does NOT shut down the Proxmox host itself
