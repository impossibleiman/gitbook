---
description: Here's are the specs for my first Virtual Machine running on Debian
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

# First Virtual Machine

#### 16th July 2026

## Specifications

<details>

<summary>Debian | 2 Cores | 4GB RAM | 40GB NVMe SSD</summary>

### OS

Debian-13.6.0-amd64

Linux\
7.x - 2.6 Kernel

### Disks

40GiB

### CPU

1 Socket

2 Cores

Type: Default (x86-64-v2-AES)

### Memory

4096MiB

</details>

## Debian Setup

Following the conventional Debian setup, I followed through the console and setup what was asked.\
This was nothing more than setting up hostnames, root password, user and so on.

I chose to install the standard system utilities and SSH server, leaving out the debian desktop environment and GNOME.

I also enabled QEMU guest agent in the options for the VM.&#x20;

