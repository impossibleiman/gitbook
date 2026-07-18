---
description: >-
  This linux container will host my All The Mods 10 Minecraft Server through
  Tailscale
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

# First Linux Container

#### 16th July 2026

## Setup

<details>

<summary>Debian | 6 Cores | 10GB RAM | 100GB NVMe SSD</summary>

### Template

Debian-13-standard\_13.6-1\_amd64

### Disks

100GiB

### CPU

6 cores

### Memory

~~8192 GiB~~ 10240 GiB

Swap 2048GiB

### Network

ipv4 & ipv6 set to static on 192.168.0.41



I also edited the console mode from tty to /dev/console. This is a common practice that allows me to see the live console from the second the OS boots up in conjunction to a virtual terminal that starts at the login screen.

</details>

## Post Setup

I installed openssh-server[^1] to be able to ssh through my main pc.&#x20;

I had to change PermitRootLogin to yes instead of strict mode to be able to login as root.\
`ssh root@192.168.0.41`

Then I installed [curl ](#user-content-fn-2)[^2]then [Tailscale](https://tailscale.com/download/linux).

### My First Problem

`CreateTUN("tailscale0") failed; /dev/net/tun does not exist`&#x20;

To solve this issue, I had to visit my pve console and run `nano /etc/lxc/101.conf`&#x20;

In here I added two lines:

* <kbd><mark style="color:$info;">features: nesting=1,<mark style="color:$info;"></kbd><kbd>keyctl=1</kbd>
* <kbd>lxc.mount.entry: /dev/net/tun dev/net/tun none bind, create=file</kbd>

Rebooted the lxc and running <kbd>tailscale up</kbd> greeted me with a URL to login with my Google account

<mark style="color:$success;">Tailscale is now running!</mark>

## Minecraft Server

Using [Java 21](#user-content-fn-3)[^3] we can utilise it's newly added garbage collector called ZGC instead of G1GC. This will come in handy later when we upgrade the hardware of the server as it is best used when you have extra CPU core & RAM headroom. This is because ZGC uses concurrent threads which prevents the server from freezing.

`adduser minecraft` add user called minecraft

`su - minecraft` switch user to minecraft

`mkdir ~/server` make a new folder called server

`cd ~/server` enter the folder

Now I need to transfer my server files from my main pc to my debian lxc

`scp -r "C:\Users\imani\Desktop\ATMServer\*" minecraft@192.168.0.41:/home/minecraft/server/`&#x20;

This is the convential way of copying files over, however for convenience I will use [FileZilla ](https://filezilla-project.org/download.php)which I have used in the past with external minecraft server providers.

```
Protocol: SFTP (SSH File Transfer Protocol)
Host: 192.168.0.41
Port: 
User: minecraft
Password: ********
```

Once all the files were transferred, I needed to make my own start.sh as you cannot run .bat on debian.

`#!/bin/bash`\
`java @user_jvm_args.txt @libraries/net/neoforged/neoforge/21.1.234/unix_args.txt nogui`&#x20;

The only difference inside the file (apart from the shebang) is using unix\_args instead of win\_args&#x20;

`chmod +x start.sh` Add permission to execute the file with `./start.sh`&#x20;

This start.sh on its own is fine to run manually however I want to make this automated to start on every reboot. Therefore I made some changes:

```
#!/bin/bash
cd /home/minecraft/server
exec java @user_jvm_args.txt @libraries/net/neoforged/neoforge/21.1.234/unix_args.txt nogui 
```

This is a more modular approach as if any updates to both the server or the directory were to occur, the script is the only file that needs to be changed.

back on the root user, I need to add a minecraft service so systemctl recognises the server

`root@lxc-debian:~# nano /etc/systemd/system/minecraft.service`&#x20;

Inside this service file:

```
[Unit]
Description=ATM10 Minecraft Server
After=network.target tailscaled.service 
Wants=network-online.target

[Service]
User=minecraft
WorkingDirectory=/home/minecraft/server
ExecStart=/home/minecraft/server/start.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Now that the minecraft service exists we'll tell debian to read new services

`systemctl daemon-reload`

Then enable the new services on start-up

`systemctl enable tailscaled`\
`systemctl enable minecraft`&#x20;

### My Second Problem

I then tried to run the server manually with `systemctl start minecraft` which looked fine until I ran `systemctl status minecraft` which showed me <mark style="color:red;">it exited with status 1</mark> (meaning the OS killed it).

Then trying to run manually with `./start.sh` made it clear we were running into a memory issue. I had set the LXC to run with 8GB of RAM (See [Memory](first-linux-container.md#id-6-cores-or-10240-gib-ram-or-100-gib-nvme-ssd)) alongside the minecraft server running at 8GB leaving no overhead for both the LXC or Java. I added an extra 2GB to the LXC and reduced the Xms and XmG to 5 and 7 respectively. So the final user\_jvm\_args.txt becomes:

```
-Xms5G
-Xmx7G
-XX:+UseZGC
-XX:+ZGenerational
-XX:+DisableExplicitGC
-XX:+PerfDisableSharedMem
```

Upon `systemctl restart minecraft` then `systemctl status minecraft`, debian tells us that <mark style="color:$success;">the service is running</mark> but I now have a small issue. I cannot see the minecraft console.&#x20;

Minecraft already has a solution for this called RCON which we can append in `server.properties`

```
enable-rcon=true
rcon.port=25575
rcon.password=******** 
```

After restarting and installing mcron on my main pc, I can access the server terminal without having to launch minecraft.

### Next Time

I will add 'Crafty Controller' on a separate docker lxc to control and monitor the server. This will remove the need to have a systemctl service for minecraft&#x20;

I also want to add automatic restarts and backups!

After that I will move to AMP - a well known game server management tool.





[^1]: apt install openssh-server

[^2]: apt install curl

[^3]: apt install openjdk-21-jre-headless -y
