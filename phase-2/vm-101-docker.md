---
description: >-
  This page explains the process of how I setup my docker hosting virtual
  machine. This VM will host all of the services listed in 'Hardware &
  Structure'
layout:
  width: wide
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
  actions:
    visible: true
---

# VM 101 - Docker

#### 18th July 2026

You may notice that this page is above LXC 100. As stated in the last line of the previous page, our top priority at the moment is to setup ~~caddy reverse proxy~~ TCP Forwarding to ensure that members of the tailscale network outside of the LAN can safely connect without a conflict of IP. \
\
I have backed up the Minecraft Server from the Phase 1 LXC 101 and then deleted both LXC 101 and VM 101.

## Setup

As of writing this, I still have not upgraded to 64GB RAM. Therefore we can only allocate 8GB RAM rather than the 12GB I initially planned. This will be updated in the near future.

<details>

<summary>4 vCPUs | 8GB RAM | 64GB NVMe SSD</summary>

VM ID: 101

Name: Docker

ISO Image: debian-13.6.0-amd64-netinst.iso

Disk Size: 64 GiB (will add more later if needed)

Sockets: 1

vCPUs: 4

Memory: 8192 MiB

</details>

I will also skip the boring setup process that was outlined in [First Virtual Machine from Phase 1](../phase-1/first-virtual-machine.md#debian-setup), all you need to know is I installed SSH Server and Standard system utilities.

### Installing Docker

<details>

<summary>Following Docker's manual on how to install Docker for Debian</summary>

```shellscript
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

```shellscript
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

To verify its installation:

```shellscript
sudo systemctl status docker
```

<mark style="color:$success;">It works!</mark>

</details>

We do not need to setup any services to auto start docker as it will do it for us.

### Directory Structure

While maintaing my priority on ~~Caddy~~ TCP Forwarding, I still want to make the empty directories for later

```shellscript
mkdir -p /opt/docker/{caddy,cloudflare-tunnel,homepage,prometheus,discord-bot,discord-bot-website,personal-website,affine,jellyfin,nextcloud}
```

and then we can make the **docker-compose.yml** (I have ran Docker on my main pc in the past)

{% code title="docker-compose.yml" %}
```docker
services:
  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./data:/data
      - ./config:/config
```
{% endcode %}

To verify it works, we will create an example Caddyfile

{% code title="Caddyfile" %}
```
:80 {
    respond "Caddy is working"
}
```
{% endcode %}

### Small Hiccup

If you're still reading chronologically, you may have noticed the strikethroughs on Caddy.

I was wrong to assume that Caddy would help other users get past the IP conflict issue with tailscale.

Instead we need to use something called TCP Forwarding for games.

## Taking a step back

While Structure was initially defined, I did not plan out the subdomains I wanted to use for my TCP Forwarding & Caddy reverse proxy.

| Hostname                    | Purpose                       | Access         | Route               |
| --------------------------- | ----------------------------- | -------------- | ------------------- |
| `imanizadyar.com`           | Personal website              | Public         | Cloudflare + Caddy  |
| `projects.imanizadyar.com`  | Potential future project site | Public         | Cloudflare + Caddy  |
| `jellyfin.imanizadyar.com`  | Jellyfin                      | Tailscale-only | Tailscale + Caddy   |
| `nextcloud.imanizadyar.com` | Nextcloud                     | Tailscale-only | Tailscale + Caddy   |
| `affine.imanizadyar.com`    | Affine                        | Tailscale-only | Tailscale + Caddy   |
| `minecraft.imanizadyar.com` | ATM10                         | Tailscale-only | Tailscale + TCP     |
| `ark.imanizadyar.com`       | ARK Survival Ascended         | Tailscale-only | Tailscale + TCP/UDP |

So I will first start with the docker services like jellyfin, nextcloud and affine as they're found in the same Docker VM and aren't hosted through cloudflare.

Following my Addressing Plan, I reserved 192.168.0.50 for this VM.&#x20;

<details>

<summary>Steps to configure caddy to route jellyfin to jellyfin.imanizadyar.com</summary>

```shellscript
cd /opt/docker/caddy
docker network create proxy
```

This proxy allows compose projects to communicate with each other using Docker's internal DNS

```shellscript
nano docker-compose.yml
```

{% code title="docker-compose.yml" %}
```docker
services:
  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped

    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"

    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./data:/data
      - ./config:/config

    networks:
      - proxy

networks:
  proxy:
    external: true
```
{% endcode %}

```shellscript
docker compose up -d
```

```shellscript
cd /opt/docker/jellyfin
nano docker-compose.yml
```

```docker
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped

    ports:
      - "8096:8096"

    volumes:
      - ./config:/config
      - ./cache:/cache
      - ./media:/media

    networks:
      - proxy

networks:
  proxy:
    external: true ##This is the important bit that lets jellyfin speak to caddy
```

```shellscript
mkdir config cache media
docker compose up -d
nano /opt/docker/caddy/Caddyfile
```

```
http://jellyfin.imanizadyar.com {
    reverse_proxy jellyfin:8096
}
```

```shellscript
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
```

test with

```shellscript
curl -H "Host: jellyfin.imanizadyar.com" http://192.168.0.50
```

We know it works when we get something like \
`HTTP/1.1 302 found`

</details>

From here we can now go ahead and install Pi-Hole

{% content-ref url="lxc-102-pi-hole.md" %}
[lxc-102-pi-hole.md](lxc-102-pi-hole.md)
{% endcontent-ref %}
