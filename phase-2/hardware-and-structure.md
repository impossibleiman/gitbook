---
description: This page outlines my upcoming hardware upgrades and new server structure
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

# Hardware & Structure

#### 18th July 2026 @ \~6AM

## Hardware Upgrade

Given my current hardware, 16GB is just not enough for a server that wants to host game servers whether it be Modded Minecraft or a planned Ark Survival Ascended server. That being said, I continued to buy some more memory.&#x20;

Now instead of having <mark style="background-color:red;">16GB (2x8) DDR4 2666MHz</mark>, I now have <mark style="background-color:green;">64GB (2x32) 3200MHz</mark>.

Note that the speed upgrade was not necessary, however on the basis of having to upgrade memory anyway, I thought I'd be best to future proof now than later. The obvious upgrade here is the capacity increase. With 4x the memory, I could easily host both a minecraft and ark server with the new limiting factors being core speed and storage.

The next upgrade to look at will be adding more NVMe SSDs for game servers and HDDs for backups and media.

## Structure

Using Draw.io, I created a new scheme for how I want my server to run. The diagram shows all the virtual machines and containers I'd like within this phase.

<figure><picture><source srcset="../.gitbook/assets/18-07-2026-Dark_Mode (1).png" media="(prefers-color-scheme: dark)"><img src="../.gitbook/assets/18-07-2026-Light_Mode (1).png" alt="New Proposed Structure Plan"></picture><figcaption><p>Click to enlarge</p></figcaption></figure>

Whilst this is a plan may look structured and finalised, this is of course subject to change in the future. Remember, at this point I am still new to proxmox and I will definitely find better ways in the future to structure my containers and machines.

The [left side of the diagram](#user-content-fn-1)[^1] will run under <mark style="background-color:$info;">vmbr0 and Tailscale</mark>. Note that in the previous phase, I setup tailscale on my linux container for my minecraft server. I realised later that this wasn't the best choice as it is not scalable.

The [right side of the diagram](#user-content-fn-2)[^2] will run through <mark style="background-color:$info;">vmbr1</mark>. This is a virtualised network used primarily as a way to experiment without having to worry about ruining my own network. As far as the virtual to real world translation goes, this is a fairly good trade-off in my opinion.



#### 18th July 2026 @ \~7:30PM

I finally received my 32x2 GB RAM sticks and the unbelievable has occured. One of the RAM sticks sent to me is not compatible with my motherboard; one stick is ECC Registered while the other is not.

I tried running both individually to be met with a steady amber light on the motherboard with both sticks.

[^1]: LXC 100 | VM 101 | LXC 102

[^2]: VM 200 | VM 201 | VM 202
