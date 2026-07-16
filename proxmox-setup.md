---
description: >-
  This page outlines what I did when I finally made my server with parts from
  #hardware
---

# Proxmox Setup

### 16th July 2026

#### Pre-Install

On my main PC, I flashed proxmox-ve\_9.2-1.iso onto my Kingston 128gb USB with Balena Etcher

I then used the USB to boot up the Server pc into the Proxmox installation:

* I selected my 1TB nvme drive to be the boot drive
* Logged into my ISP and set a DHCP reservation for my server's fixed IP of 192.168.0.20/24
* Used my router's gateway
* Used my ISP's DNS
* Set my login username and password

Then proxmox directed me to 192.168.0.20:8006 which I connected to from my Main PC.

#### Post-Install

One of the possible options of a post-setup would be to do the following:

* pve (left-hand side) > Updates > Repositories
* Add Repository
* Repository: No-Subscription
* Disable enterprise repositories

Instead, I went ahead and ran the PVE Post Install script from Proxmox VE Scripts:

* Disabled 'pve-enterprise' repository
* Disabled 'ceph enterprise' repository
* Added 'pve-no-subscription' repository
* Selected no to Adding 'pvetest' repository
* Disabled Subscription nag
* Disabled high availability
* Selected no to Disabling Corosync
* Updated Proxmox

I then rebooted proxmox and cleared my browser cache on my main pc with `Ctrl Shift R`

I am then to later make an automatic updating system for the server to run fortnightly.

Despite current storage limitations, I have setup a nightly (5am) backup schedule to backup all VMs with a retention plan to keep the last 2. Once a few hard drives or a NAS system have been added, I will increase the retention plan from 2 to 30.&#x20;

Within my ISO images, I added Debian 13.6.0, Windows Server 2025 and Windows 11.
