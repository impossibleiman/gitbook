---
description: My 4TB HDD Arrived!!
---

# First Major Upgrade & Migrations

## Verifying the HDD's health

18th November 2026

This was a planned upgrade from the start but as of writing this, secondary storage has skyrocketed in price alongside primary storage. That being said, I caved and bought myself a second hand 4TB hard drive (smaller than the 8TBs that I wanted to get however it's a good starting ground) for around £60 from eBay. The drive arrived on the 17th November 2026.

After an easy installation of the drive, I spent the remainder of the day verifying it had no issues:

To verify the drive's name I used

{% code title="" expandable="true" %}
```
lsblk -o NAME,MODEL,SERIAL,SIZE,FSTYPE,MOUNTPOINTS
```
{% endcode %}

To find it's under sda, verifying it with

{% code title="" expandable="true" %}
```
smartctl -i /dev/sda
```
{% endcode %}

| Property               | Verified value                          |
| ---------------------- | --------------------------------------- |
| Model                  | HGST Ultrastar 7K6000 (HUS726040ALE614) |
| Serial number          | ...                                     |
| Device                 | `/dev/sda`                              |
| Capacity               | 4,000,787,030,016 bytes                 |
| Usable binary capacity | 3.64 TiB                                |
| Rotation speed         | 7200 RPM                                |
| Interface              | SATA 6 Gb/s                             |

Then I ran an initial SMART test

{% code title="" expandable="true" %}
```
smartctl -a /dev/sda
```
{% endcode %}

| SMART attribute               | Initial result |
| ----------------------------- | -------------- |
| Overall health                | PASSED         |
| Power-on hours                | 1,932          |
| Reallocated sectors           | 0              |
| Current pending sectors       | 0              |
| Offline uncorrectable sectors | 0              |
| Raw read errors               | 0              |
| Seek errors                   | 0              |
| Spin retry count              | 0              |
| UDMA CRC errors               | 1              |
| Temperature                   | 23°C           |

Only one error is pretty good since it could only be a connection issue.&#x20;

I then checked the self test history and capabilities&#x20;

{% code title="" expandable="true" %}
```
smartctl -l selftest /dev/sda
smartctl -c /dev/sda
```
{% endcode %}

Then started the extended test&#x20;

{% code title="" expandable="true" %}
```
smartctl -t long /dev/sda
```
{% endcode %}

The drive estimated **571 minutes (9 hours 31 minutes)** to complete.

This is a non-destructive, drive-internal diagnostic test that examines the disk surface and checks for read failures.

getting a final result of `1 Extended offline Completed without error 00% 1941 -`

Then I ran an independent scan via badblocks

{% code title="" expandable="true" %}
```
badblocks -b 4096 -sv /dev/sda
```
{% endcode %}

`Checking blocks 0 to 976754645`\
`Checking for bad blocks (read-only test): done`\
`Pass completed, 0 bad blocks found. (0/0/0 errors)`

And all that proves we're safe to use the hard drive for serious backups just in case our main drive fails alongside some media like films and images.&#x20;

The only issue now is allocating the 4TB, migrating the media over and setting up some good retention policies for backups.



### Partitioning and formatting the HDD

Partioning the drive into an ext4 format is ideal for this hard drive

{% code title="" expandable="true" %}
```
parted /dev/sda --script mklabel gpt mkpart primary ext4 1MiB 100%
mkfs.ext4 -L hdd /dev/sda1
```
{% endcode %}

and giving it the name hdd is a good long term solution till I get another hard drive - for now I'll just have an nvme and a hdd.

Then creating a mount point tells proxmox which filesystem to attach to at boot and let it be used in our containers

{% code title="" expandable="true" %}
```
mkdir -p /mnt/hdd

echo 'UUID=6330de28-687e-4b83-a5f5-03fef3a36f7c /mnt/hdd ext4 defaults,nofail 0 2' >> /etc/fstab

systemctl daemon-reload
mount /mnt/hdd
```
{% endcode %}

verified with&#x20;

{% code title="" expandable="true" %}
```
lsblk -f /dev/sda
df -h /mnt/hdd
```
{% endcode %}

### Creating the HDD directory structure

{% code title="" expandable="true" %}
```
mkdir -p /mnt/hdd/{dump,shared,nextcloud-data}
mkdir -p /mnt/hdd/media/{movies,shows,tv}
mkdir -p /mnt/hdd/backups/{proxmox-host,docker,games,nextcloud}
```
{% endcode %}

simple organisation of files.

## Register the HDD as Proxmox Storage

{% code title="" expandable="true" %}
```
pvesm add dir hdd \
  --path /mnt/hdd \
  --content backup \
  --is_mountpoint 1
```
{% endcode %}

We can use this for proxmox backup archies

{% code title="" expandable="true" %}
```
pvesm status
```
{% endcode %}

The storage hdd should appear as active

### Share the HDD using NFS

{% code title="" expandable="true" %}
```
apt install -y nfs-kernel-server

echo '/mnt/hdd 192.168.0.50(rw,sync,no_subtree_check,root_squash)' \
  > /etc/exports.d/hdd.exports

exportfs -ra
systemctl enable --now nfs-server
```
{% endcode %}

Using NFS, we can make the HDD accessible to the Docker VM, verified with&#x20;

{% code title="" expandable="true" %}
```
exportfs -v
```
{% endcode %}

### Mounting the HDD inside the Docker VM

Installing NFS client will help us configure a persistent mount

{% code title="" expandable="true" %}
```
apt install -y nfs-common
mkdir -p /mnt/hdd
```
{% endcode %}

{% code title="/etc/fstab" expandable="true" %}
```
192.168.0.20:/mnt/hdd /mnt/hdd nfs4 rw,vers=4.2,hard,_netdev,nofail,x-systemd.automount,x-systemd.mount-timeout=30s 0 0
```
{% endcode %}

Then we can activate the mount:

{% code title="" expandable="true" %}
```
systemctl daemon-reload
systemctl start mnt-hdd.automount
```
{% endcode %}

Verifying with

{% code title="" expandable="true" %}
```
ls /mnt/hdd
findmnt -T /mnt/hdd/media
```
{% endcode %}

## Migrate Jellyfin media to the HDD

{% code title="" expandable="true" %}
```
chown -R 65534:65534 /mnt/hdd/media
chmod 755 /mnt/hdd/media /mnt/hdd/media/movies /mnt/hdd/media/shows
```
{% endcode %}

Prepared permissions on the pve



{% code title="" expandable="true" %}
```
rsync -a --info=progress2 \
  /opt/docker/jellyfin/media/ \
  /mnt/hdd/media/
```
{% endcode %}

Back on Docker VM



### Update Docker Compose

{% code title="/opt/docker/jellyfin/docker-compose.yml" expandable="true" %}
```
volumes:
  - /opt/docker/jellyfin/config:/config
  - /opt/docker/jellyfin/cache:/cache
  - type: bind
    source: /mnt/hdd/media
    target: /media
    bind:
      create_host_path: false
```
{% endcode %}

Now pointing to the hdd's media rather than the ssd

{% code title="" expandable="true" %}
```
cd /opt/docker/jellyfin
docker compose up -d --force-recreate
```
{% endcode %}

Verify with&#x20;

{% code title="" expandable="true" %}
```
rsync -rnc --stats /opt/docker/jellyfin/media/ /mnt/hdd/media/

docker inspect jellyfin --format \
  '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```
{% endcode %}



## Migrate Nextcloud data to HDD

Proxmox permissions

{% code title="" expandable="true" %}
```
chown 33:33 /mnt/hdd/nextcloud-data
chmod 770 /mnt/hdd/nextcloud-data
```
{% endcode %}

Docker VM:

{% code title="" expandable="true" %}
```
setpriv --reuid=33 --regid=33 --clear-groups \
  rsync -a --no-owner --no-group \
  --info=progress2 \
  /opt/docker/nextcloud/data/data/ \
  /mnt/hdd/nextcloud-data/
```
{% endcode %}

### Final Synchronisation

{% code title="" expandable="true" %}
```
docker exec -u www-data nextcloud php occ maintenance:mode --on
docker stop nextcloud
```
{% endcode %}

After entering maintenance mode, synchronise remaining changes

{% code title="" expandable="true" %}
```
setpriv --reuid=33 --regid=33 --clear-groups \
  rsync -a --no-owner --no-group --delete \
  /opt/docker/nextcloud/data/data/ \
  /mnt/hdd/nextcloud-data/
```
{% endcode %}

### Update Docker Compose

{% code title="" expandable="true" %}
```
volumes:
  - ./data:/var/www/html
  - type: bind
    source: /mnt/hdd/nextcloud-data
    target: /var/www/html/data
    bind:
      create_host_path: false
```
{% endcode %}

Retain empty underlying directory on ssd:

{% code title="" expandable="true" %}
```
mkdir -p /opt/docker/nextcloud/data/data
chown 33:33 /opt/docker/nextcloud/data/data
chmod 770 /opt/docker/nextcloud/data/data
```
{% endcode %}

Recreate container

{% code title="" expandable="true" %}
```
cd /opt/docker/nextcloud
docker compose up -d --no-deps --force-recreate nextcloud
docker exec -u www-data nextcloud php occ maintenance:mode --off
```
{% endcode %}

Verify

{% code title="" expandable="true" %}
```
docker exec -u www-data nextcloud php occ status

docker exec -u www-data nextcloud \
  ls -la /var/www/html/data/.ncdata

docker exec nextcloud findmnt -T /var/www/html/data
```
{% endcode %}
