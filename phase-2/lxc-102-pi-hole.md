---
description: Pi-Hole setup
---

# LXC 102 - Pi-Hole

## Setup

<details>

<summary>1 Core | 512MB RAM | 8GB NVMe SSD</summary>

VM ID: 102

Hostname: Pi-Hole

Password: \*\*\*\*\*\*\*\*

Template: debian-13-standard\_13.6-1\_amd64.tar.zst

Disk Size: 8 GiB

Cores: 1

Memory: 512 MiB

IPv4/CIDR: 192.168.0.3/24

</details>

Once setup, I had to go through the pve terminal and set the [DNS to 1.1.1.1](#user-content-fn-1)[^1] as it was using Tailscale's magic dns at 100.100.100.100.

Then I noted my current DHCP reservations and start/end IPs.

I want Pi-Hole to handle DHCP:

```
Network:        192.168.0.0/24
Gateway:        192.168.0.1
Pi-hole:        192.168.0.3
DHCP range:     192.168.0.10 – 192.168.0.250
DNS server:     192.168.0.3
```

Then turn on Pi-Hole's DHCP on then turn off your ISP's DHCP

To then test on my main pc

```shellscript
ipconfig /release
ipconfig /renew

nslookup debian.org
```

`Server: pi.hole`\
`Address: 192.168.0.3`

From here Pi-Hole now handles DHCP and DNS of my network. The admin panel shows blocked queries.

In Pi-Hole, navigate to:\
System > Settings > Local DNS Records

Here we need to add jellyfin.imanizadyar.com to 192.168.0.50

Now any device connected to the DNS server (on the local network) can access jellyfin.imanizadyar.com however we need to make sure everyone connected to the tailscale can access jellyfin.imanizadyar.com.

To do this, on Tailscale we visit the DNS settings at the top. Scroll down to add a custom nameserver.\
`Nameserver: 192.168.0.3`\
`Restrict to domain`\
`Domain: imanizadyar.com`<br>

<mark style="color:$success;">Now, using my phone, I can attempt to connect to jellyfin.imanizadyar.com via mobile data and it works!</mark>



Now let's repeat these for:

* dashboard
* prometheus
* affine
* nextcloud
* pihole

We will start by adding these to the local DNS settings on Pi-Hole: service.imanizadyar.com. They will all go to 192.168.0.50 expect Pi-Hole which goes to 192.168.0.3

Then append your Caddyfile to add your docker services. Ensure Pi-Hole goes to 192.168.0.3:80

then run&#x20;

```shellscript
docker exec caddy caddy validate --config /etc/caddy/Caddyfile
```

if it ends with 'Valid Configuration' run:

```shellscript
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
```

#### Dashboard

Using homepage, I ran into a problem setting up the reverse proxy. The website loaded into an error 500. Checking the logs with docker logs homepage:<br>

```
[2026-07-18T08:03:31.039Z] error: Host validation failed for: dashboard.imanizadyar.com. Hint: Set the HOMEPAGE_ALLOWED_HOSTS environment variable to allow requests from this host / port.
```

So I added:

```
    environment:
      HOMEPAGE_ALLOWED_HOSTS: dashboard.imanizadyar.com
```

into my docker-compose.yml <mark style="color:$success;">which now works!</mark><br>





For affine, we need to make sure we have the right depdencies: postgre & redis. Some problems I ran into were that we had to run docker compose exec affine npx prisma migrate deploy before it would crash

Redis: `WARNING Memory overcommit must be enabled! Without it, a background save or replication may fail under low memory condition.`

```
sudo sysctl vm.overcommit_memory=1
```

Finally done!

If you're reading this in this current state then just know I was really really tired and I've been up all night working on this so I will definitely refactor this page when I've gotten good sleep!





Later on I ensured through the pve host to run Pi-Hole on bootup:

```shellscript
pct set 102 -onboot 1
```

However since it is my DNS server, I want it to let my network settle first before booting it:

```shellscript
pct set 102 -startup order=1,up=30
```

So then setting up other containers and VMs to boot on start-up will require:

```shellscript
pct set </ID/> -startup order=2,up=30
```

As all containers running on vmbr0 established through tailscale will have pi-hole as a dependency.

[^1]: pct 102 --nameserver 1.1.1.1
