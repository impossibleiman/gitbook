---
description: >-
  This page outlines what I did when I finally made my server with parts from
  the previous page
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

# Proxmox Setup

#### 16th July 2026

## Pre-Install

On my main PC, I flashed [proxmox-ve\_9.2-1.iso](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso) onto my Kingston 128gb USB with [Balena Etcher](https://etcher.balena.io/)

<details>

<summary>I then used the USB to boot up the Server pc into the Proxmox installation</summary>

* I selected my 1TB nvme drive to be the boot drive
* Logged into my ISP and set a DHCP reservation for my server's fixed IP of 192.168.0.20/24
* Used my router's gateway
* Used my ISP's DNS
* Set my login username and password

</details>

Then proxmox directed me to 192.168.0.20:8006 which I connected to from my Main PC.

## Post-Install

One of the possible options of a post-setup would be to do the following:

* pve (left-hand side) > Updates > Repositories
* Add Repository
* Repository: No-Subscription
* Disable enterprise repositories

<details>

<summary>Instead, I went ahead and ran the <a href="https://community-scripts.org/scripts/post-pve-install">PVE Post Install</a> script from <a href="https://community-scripts.org/">Proxmox VE Scripts</a></summary>

* Disabled 'pve-enterprise' repository
* Disabled 'ceph enterprise' repository
* Added 'pve-no-subscription' repository
* Selected no to Adding 'pvetest' repository
* Disabled Subscription nag
* Disabled high availability
* Selected no to Disabling Corosync
* Updated Proxmox

</details>

I then rebooted proxmox and cleared my browser cache on my main pc with `Ctrl Shift R`

I am then to later make an automatic updating system for the server to run fortnightly.

Despite current storage limitations, I have setup a nightly (5am) backup schedule to backup all VMs with a retention plan to keep the last 2. Once a few hard drives or a NAS system have been added, I will increase the retention plan from 2 to 30.&#x20;

Within my ISO images, I added [Debian 13.6.0](https://www.debian.org/distrib/), [Windows Server 2025](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2025) and [Windows 11](https://www.microsoft.com/en-us/software-download/windows11).
