---
description: Subject to change
---

# LXC 100 - Game Servers (AMP)

Hi, if you're reading this or have just come from the last page then you'll know I'm really tired as of writing this as I've been up all night. You should not be able to see this unless you've been stalking my progress or are just looking back at the history of the gitbook. Either way thank you for going through in depth.



## Setup

<details>

<summary>8 Cores | 2GB RAM (TEMPORARY) | 200 GB NVMe SSD</summary>

ip: 192.168.0.51

</details>

I bought an AMP advanced license for £30

Ensure you add AMP IP to pi hole local dns as 192.168.0.50 and not 51 and add to caddy

instead of writing

```
reverse_proxy amp:8080
```

ensure you write

```
reverse_proxy 192.168.0.51:8080
```

as it is not a docker service

\-
