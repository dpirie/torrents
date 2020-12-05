# Plex, Torrent Server on Ubuntu

I used to run Plex and ruTorrent on FreeNAS 9. It served me well for a long, long time but managing Jails was a pain and the 9 to 11 upgrade was quite daunting. Meanwhile, I was doing a lot of Docker container development at work. I found articles about ZFS on Linux and figured if I could mount my old files in Linux, I could configure and run all the services I need using Docker Containers, all managed with one docker-compose.yml file.

## Hardware and Booting

FreeNAS had its boot and root filesystems on a USB thumb drive. All of your own data and a small amount of configuration was stored on the ZFS filesystem. All the log file writes to the USB drive would eventually cause fatigue and corruption.

To mirror, yet modernize this architecture, I moved the root filesystem to an NVMe card using this [adapter](https://www.memoryexpress.com/Products/MX75791). My old motherboard was unable to boot from the card, so I reinstalled Ubuntu (latest LTS Version) with the boot volume on a small, old USB thumbdrive. There's no writing to this boot volume so fatigue will not be a problem.

| File System | Physical |
|-------------|----------|
| /boot       | USB Thumb Drive       |
| /           | NVMe through Adapter  |
| /vault      | ZFS pool from FreeNAS |

## Upgrade and Backout Plan

If you keep the old FreeNAS boot drive handy, you can install Ubuntu on the new drives and configure everything. If you run out of your change window and the tiny humans are getting restless without Plex, you can shutdown and reboot in to FreeNAS.

You can continue in this dual-boot scenario for some time, as long as you do not upgrade the filesystem with `zfs upgrade`.

## Relevant Articles
* https://github.com/sebgl/htpc-download-box - This was the main inspiration for all these containers. Also includes configurations for Jackett and a VPN container.
* https://www.smarthomebeginner.com/docker-home-media-server-2018-basic/
* https://blog.swakes.co.uk/automated-media-box-part3/

## Dependencies Installation

To mount the FreeNAS ZFS volume, we have to install a number of packages:
```sh
sudo apt install smartmontools htop zfsutils-linux bonnie++ unzip lm-sensors ctop
sudo apt install docker-ce docker-compose
# This is so we can also mount exFAT thumb drives (FAT32 without 4GB limit)
sudo apt install exfat-fuse exfat-utils
sudo reboot -n
sudo zpool list
sudo zpool import vault
ls /vault
```

## Containers

Most of these containers are linuxserver if one is available, vendor-provided if one is available, or a user-contributed that was referenced in an article.

| Name | Purpose |
|------|---------|
| plex | Serving all the videos to TVs, tablets and friends |
| handbrake | A standalone Handbrake container. Just copy videos in to the watch folder and it will be transcoded and moved to the output directory. Installed this to transcode Plex PVR videos that I want to keep |
| torrent | This is rTorrent and ruTorrent in one container. I used this container before and still use it for interactive torrenting because I prefer the UI. I was able to bring this up and continue seeding a thousand torrents. |
| deluge | This works better with Sonarr but I hate the UI, so I use it only for automated torrenting |
| sonarr | I used to use an IRC bot to download, but a friend recommended this instead. It may be a few minutes behind the bot, but is so much easier to configure |
| organizr | A single-page dashboard for all of these new components with links to WebUIs and external sites |
| syncthing | I use SyncThing to sync some files and directories to a friend's computer |
| unifi-controller | Controller for my Ubiquiti network gear |

## Configuration

There are few variables in the `docker-compose.yaml` file. These variables are configured in a `.env` file in this directory. This is my env file:
```
TZ="America/Edmonton"
PUID=972
PGID=972
HOST_IP="192.168.0.2"
```
All of the containers set their timezone to `TZ` and run their primary process with those UID/GID values. This way, all subdirectories and files are created as and owned by the same user, so you should (generally) not have file permission issues.

Plex needs to advertise it's local IP, which is different than the container's IP address, so we have to specifically configure it here. The value is the main IP address of your Linux host.

The UID and GID values should be set to the same numeric IDs as your own FreeNAS volume (`ls -aldn /media`). You'll find it helpful to create users for these IDs, so you can refer to them symbollically.
```sh
sudo groupadd -g 972 torrents
sudo useradd -g torrents -u 972 -M -s $(which false) torrents
```

The `umask` in many of the containers is set to `002` so all files and directories are group-writable. If you add your own Ubuntu user to the torrent group, you'll be able to easily manage the files and directories.

If you run in to owner/permission issues, a command like this will be helpful:
```sh
sudo chown -R torrents:torrents /media/
```

## Directory Layout

This is how my own directories are laid out. If yours are different, you'll have to update the references in `docker-compose.yaml`.
* The main ZFS pool is mounted as `/vault`.
* Somehow back in FreeNAS I created a pool within vault that is mounted `/media` that holds all of my video files.
* Plex media files are stored in directories under `/media`, like `/media/tv-shows`, etc.
* ruTorrent moves completed files in to `/media/Completed`.
* Deluge stores its completed files in `/media/completed`. If your filesystem isn't case-sensitive or this is confusing, you should rename one of these. Regardless, this filesystem is mounted higher, so it's just a configuration setting.
* I have always used hard-links to have two references to any file - one under `/media/Completed` and one under `/media/tv-shows/Holey.Moley/Season.1/`.
  * These are created instantaneously and use only the disk space of the first instance. They are two pointers to the same data.
  * I can seed a file forever until I remove it from Completed, and keep the show in Plex until I remove it.
  * The disk space is cleaned up when both references to the file are deleted.
  * Hard-links can only be created within the same filesystem. This is why I mounted all the media in to containers as `/media`.
* Each of the containers want to store some configuration, usually in a writable mount called under `/config`. These are created under `/vault/home/{container_name}`.

## Docker Compose Commands

All Docker Compose commands are driven by `docker-compose.yaml` file in the current directory, so you must `cd` to the correct directory (where you cloned this repo) for them to work.

All containers have sensible names (`docker ps`), so once they are running, you can restart, stop, and view logs directly with `docker` and don't need the complexity of Docker Compose. Docker Compose is just a handy way of configuring all the ports, mounts, and other properties of containers.

### Pull the lastest version
```sh
# All containers
docker-compose pull
# Just specific ones
docker-compose pull plex unifi
```

### Start

Starting a container (bringing it `up`) is done with the `up` subcommand. This will bring it up in the foreground, and take over your terminal. This is fine for initial debugging, but it's generally best to start them detached. You can always see the output with Logs.
```sh
docker-compose up --detach plex
```

### Run a Shell Inside a Container

When you're debugging processes, it is often helpful to run a shell inside the container, so you can check directory mounts, view processes, etc.
```sh
docker-compose exec plex bash
docker exec -it plex bash
```
If you get an error about path not found, someone hasn't installed bash in the container. Try running `sh` instead of `bash`.

### Logs

To see the output from a container, we view the logs of it. There are often log files stored in the writable volume.
```sh
docker logs plex
# Follow-mode, like "tail -f" (Ctrl-C to exit)
docker logs -f plex
```

### Inspect

This will dump out all the configuration properties for the containers.
```sh
docker inspect plex
```

### Updates

To update all the containers to the latest versions, just run this one command. It will pull the latest versions of each container and then recreate any containers that have newer images
```sh
docker-compose pull && docker-compose up -d
```

## Containers

All containers are configured with a restart policy of unless-stopped. If your Linux host reboots, or a container exits on error, it will automatically be restarted.

### Plex
The vendor-provided Docker container is updated occasionally, but it's really only for OS-level dependencies. It is configured to download the latest version of the PMS software onto the ephemeral slice everytime it starts. If all you want to do is update Plex to the latest, then just restart the container.
```sh
docker restart plex
```

Other container configuration options are listed in the [Docker Hub](https://hub.docker.com/r/plexinc/pms-docker) page.

* The `ADVERTISE_IP` property is configured in `.env` to point to your Linux host so local connections work
* Update port forwarding on your router for port 32400 to `HOST_IP`.
* Once it is up and running, connect to port 32400 and authenticate it to your Plex Account.
* If you don't have a PlexPass account, update the image tag.
* The third volume mount is to support a PVR Post-Processing Script to transcode shows using the Handbrake container. If you don't use the PVR, you can delete that line.
* Extract the backup that you made of your Plex data under `/vault/home/plex/Library/Application Support/Plex Media Server`
* Log files are in the `/vault/home/plex/Library/Application Support/Plex Media Server/Logs` directory.

### ruTorrent

Based on a Linux-Server Project container with rTorrent and ruTorrent installed, but this image also adds the autodl client I used to use as an IRC bot. Use both pages for configuration options, or change the reference to the latter.
* [horjulf/rutorrent-autodl](https://hub.docker.com/r/horjulf/rutorrent-autodl)
* [LinuxServer/rutorrent](https://hub.docker.com/r/linuxserver/rutorrent)

Since this is the Torrent client I use for manual torrents, I use the [Remote Torrent Adder](https://chrome.google.com/webstore/detail/remote-torrent-adder/oabphaconndgibllomdcjbfdghcmenci) extention is Chrome to send torrent files to this service.

| Port | Service |
|---|---|
| 8081 | ruTorrent Web UI (container port 80) |
| 5000 | scgi |
| 51413 | Incoming torrent connections. Port forward this port on your router to your `HOST_IP` |
| 6881/udp| UDP torrent connections |

### Deluge

A simple Linux-Server Project image with Deluge installed on it. It's UI is ugly, but it plays well with Sonarr.

* Web interface runs on port 8112 (http).
* Because of the way Deluge assigns ports for torrents, this is one of the few containers that uses `network_mode: host`.
* Ensure that the port listed under Preferences > Network > Incoming Port is forwarded on your router to `HOST_IP`
* Since Sonarr and Deluge have to use the same path to files, and all systems have to see the directories as the same filesystem for hard-links to work, I've just mounted the whole `/media` filesystem.

### Sonarr

A simple Linux-Server Project image with Sonarr installed on it.

* Web interface is on port 8989 (http)
* Will require some [configuration](https://github.com/sebgl/htpc-download-box#configuration-2) after things are running, to orchestrate all the containers.
* The private torrent site I use is natively indexed by Sonarr, so I don't have to run Jackett or a VPN.

### Handbrake

The HDHomeRun device and Plex PVR just record the raw MPEG2 stream. This is fine for shows that I watch and delete, but for shows I want to archive, I transcode them so they're much smaller for long-term storage.

* It is based on this [article](https://forums.plex.tv/t/plex-dvr-linux-docker-post-processing-script-using-handbrake-to-compress-video-and-retaining-close-caption/492673)
* The above article walks you through the process. The hardest part was configuring the PRESET value (don't use quotes).
* The `plex_dvr_post_processing.sh` file in this repo is based on [this file](https://github.com/jlesage/docker-handbrake), but with enhancements for timeouts, skip transcoding based on show name, etc.

### Organizr

There are far too many port numbers to remember. This too has all sorts of plugins to build out a dashboard for all the components. You just bring the container up and hit the web interface on port 8888.

* The configuration was fairly straightforward.
* When creating the database directory, there's a warning about it not being under the web root (`/config/www`) so I used `/config/db`.
* Under "Tabs > Tab Editor", you need to make the Homepage Active and Default
* Under "Homepage Items", you configure all the widgets that you need. Each is fairly self explanatory.
  * I have 2FA enabled on Plex, so it was easiest to manually supply my Plex Authentication Token.
  * The UniFi plugin also didn't play well with 2FA, so I made another local, readonly Admin called `organizr` with a complex password.
* You can also add more Tabs for things like the full Sonarr and Plex WebUIs

### Unifi Controller

An image of the Ubiquiti Network Controller software to mange my network gear.

