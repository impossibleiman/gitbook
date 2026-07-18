---
description: >-
  Following the 'Hardware & Structure' page, this is how I upgraded my host to
  include Tailscale and update automatically
---

# Proxmox Virtual Environment

## Automatic Backup & Updates

Before I add automatic updates, I want to assume the server is hypothetically used from 6am onwards. I also want to make sure the server backs up before any automatic updates on the day. With the current 5am daily backup, this is not feasible therefore I wil move back the automated backup by an hour. Therefore the backup occurs at 4am every day (and still only holds the last 2 backups), then it will [check for updates](#user-content-fn-1)[^1] at 5am every Thursday. If this was a production level server, both automations would be pushed back 3 hours but the game servers are regularly used till early mornings.

<details>

<summary>Steps in order</summary>

* Changed automated backup plan from 5am to 4am
* Ran a backup job
* SSH into pve shell at root@192.168.0.20

```shellscript
nano /usr/local/sbin/weekly-pve-update.sh
```

```shellscript
#!/bin/bash

set -e

export DEBIAN_FRONTEND=noninteractive ## Do not open interactive prompts

LOG="/var/log/weekly-pve-update.log"

{
    echo "========================================"
    echo "Weekly Proxmox update"
    date
    echo "========================================"

    apt-get update
    apt-get -y dist-upgrade
    apt-get -y autoremove
    apt-get clean

    if [ -f /var/run/reboot-required ]; then
        echo "REBOOT REQUIRED"
    else
        echo "No reboot required"
    fi

    date
} >> "$LOG" 2>&1
```

```shellscript
chmod +x /usr/local/sbin/weekly-pve-update.sh
```

```shellscript
nano /etc/systemd/system/weekly-pve-update.service
```

```shellscript
[Unit]
Description=Weekly Proxmox update

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/weekly-pve-update.sh
TimeoutStartSec=1h
```

```shellscript
nano /etc/systemd/system/weekly-pve-update.timer
```

```shellscript
[Unit]
Description=Run weekly Proxmox update every Thursday at 05:00

[Timer]
OnCalendar=Thu *-*-* 05:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```shellscript
systemctl daemon-reload
systemctl enable --now weekly-pve-update.timer
```

```shellscript
systemctl list-timers weekly-pve-update.timer
```

<mark style="color:$success;">Which now shows the active timer!</mark>

This should also be tested now so we don't run into any unexpected issues

```shellscript
systemctl start weekly-pve-update.service
```

```shellscript
systemctl status weekly-pve-update.service
```

```shellscript
cat /var/log/weekly-pve-update.log
```

The pve does not have any updates at the minute so it responded with:

`0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.`&#x20;

And one final check to ensure the timer is still to run for Thursday at 5am

```shellscript
systemctl status weekly-pve-update.timer
```

`Trigger: Thu 2026-07-23 05:00:00 BST; 4 days left`

<mark style="color:$success;">This shows that despite running it manually at a different time, it's still on track to update every Thursday at 5am!</mark>

</details>

This is a very simple auto update service which can be further improved later or I can use [this helper script](https://community-scripts.org/scripts/update-lxcs?from=home\&fromQ=update).&#x20;

Note that it does not automatically reboot as when we add our game servers and VMs then we need to implement code to gracefully shut down those servers before restarting proxmox. This will be improved in phase 3.



## Tailscale on main node

As mentioned in [Structure on the previous page](hardware-and-structure.md#structure), tailscale at the moment only runs in my Minecraft linux container. If each container and virtual machine ran as its own tailscale device, I would very quickly meet the 6 device limit on the free tailscale plan. Therefore this is not scalable or manageable. Instead I can install Tailscale on the host node and configure the host as a subnet router. This means each container and virtual machine routes through the host, acting as one device.

<details>

<summary>Steps in order</summary>

```shellscript
curl -fsSL https://tailscale.com/install.sh | sh
```

```shellscript
tailscale up
```

Then once authenticated and on my Tailscale admin panel, we need to enable IP forwarding:

```shellscript
nano /etc/sysctl.d/99-tailscale.conf
```

Add these lines:

```
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

Then apply the changes:

```shellscript
sysctl --system
```

Then route traffic from proxmox to my home LAN at 192.168.0.0/24

```shellscript
tailscale set --advertise-routes=192.168.0.0/24
```

Now that the route is advertised, we need to approve it on the tailscale admin dashboard.

</details>

Now we can access the pve from my pve's tailscale IP at port 8006.

At the minute, the only downside of this is that when connected to another network that also uses 192.168.0.0/24 (also known as most home networks), conflicting IPs will route to the network's rather than Tailscale's. This should later be fixed when implementing a reverse proxy like [Caddy](https://caddyserver.com/docs/quick-starts/reverse-proxy) to use a domain instead of a 192.168.0.x IP. With this being said, my next priority should now be setting up VM 101.



[^1]: sudo apt update && apt upgrade
