# Dell-Wyse-5070-Homelab
A comprehensive guide to building a self-hosted homelab on a Dell Wyse 5070 using Debian, OpenMediaVault, Docker, Pi-hole, Jellyfin, Tailscale and BorgBackup.

<img width="1920" height="968" alt="OMV-panel" src="https://github.com/user-attachments/assets/d310b406-c730-4962-951e-a4229b50b88d" />

<br></br>

> [!CAUTION]
> Throughout this guide, I use specific hostnames (e.g., `server`), local IP addresses (e.g., `192.168.18.4`, `192.168.18.1`), and a specific username (`jakub`) as examples. Please treat these as placeholders and replace them with the actual values corresponding to your own network and hardware environment.

<br></br>

## 1. Creating a bootable USB

1. Download Debian netinst CD image (amd64):
```
https://www.debian.org/CD/netinst/index.en.html
```
2. Burn ISO image to USB, make sure it's not currently mounted:
```shell
sudo dd if=FILENAME.iso of=DISK_PATH bs=4M status=progress && sync
```
For example:
```shell
sudo dd if=debian-13.3.0-amd64-netinst.iso of=/dev/sda bs=4M status=progress && sync
```
Use `lsblk` to identify the correct disk (USB) name, for example:
<img width="991" height="186" alt="Pasted image 20260212194411" src="https://github.com/user-attachments/assets/de9ea6bc-5098-4fb5-959b-ebfffe75c9af" />

**WARNING** - do not use the partition name ( `sda1`), use the **disk** name (`sda`):
<img width="991" height="86" alt="Pasted image 20260212223845" src="https://github.com/user-attachments/assets/325569a3-c5a8-4a62-83d2-1373d88d734b" />

## 2. Configuring BIOS
Since my Dell Wyse 5070 didn't come with WiFi antenna, for steps 2. and 3. I needed an external monitor, a keyboard and an Ethernet cable connected to my router.

1. Boot up the device and keep pressing F2 to for BIOS.
2. Change following settings, save and exit BIOS:
   - **General** → **Boot Sequence** → set `UEFI` first 
   - **System Configuration** → **SATA Operation** → change to `AHCI`
   - **System Configuration** → **USB Configuration** → check `Enable USB Boot Support`
   - **Secure Boot** → **Secure Boot Enable** → uncheck `Secure Boot Enable`
   - **Power Management** → **AC Recovery** → change to `Power On`
   - **Virtualization Support** → enable both `Virtualization` and `VT for Direct I/O`

## 3. Installing Debian

1. Boot up the device and keep pressing F12 for boot options and choose the USB drive.
2. Select `Install`.
3. Select language (e.g. `English`), location (e.g. `Poland`), locales (e.g. `en_US.UTF-8`), keyboard (e.g. `Polish`).
4. Enter the hostname (e.g. `server`) and domain name (can be blank).
5. Enter a root password.
6. Enter a full name (e.g. `jakub`), a name for the new user (e.g. `jakub`), but something else than `admin` and password.
7. Select `Guided - use entire disk` partitioning method, select SATA disk and `All files in one partition (recommended for new users)`.
8. After installing the base system, select Debian archive mirror country (e.g. `Poland`), Debian archive mirror (e.g. `deb.debian.org`), HTTP proxy information (can be blank).
9. Choose software to install - choose only `SSH server` and `standard system utilities`.
10. Remove USB drive and reboot.

## 4. Setting up SSH

1. Back on your desktop, find the device IP address with:
```shell
sudo arp-scan --localnet
```
<img width="991" height="238" alt="Pasted image 20260213003131" src="https://github.com/user-attachments/assets/15925eee-2567-483a-b10f-ed762afe168f" />

2. Connect with the device:
```shell
ssh jakub@192.168.18.4
```
<img width="991" height="295" alt="Pasted image 20260213004542" src="https://github.com/user-attachments/assets/80c5393b-4c6d-47a2-aa44-f373155a2851" />


3. Switch to root, install `sudo` and add yourself to it:
```shell
su -
```
```shell
apt update && apt install sudo -y
```
```shell
usermod -aG sudo jakub
```
```shell
exit   #exit root
```
```shell
exit   #exit ssh
```
4. Connect again, update system and install `curl` and `gnupg`:
```shell
sudo apt update && sudo apt upgrade -y
sudo apt install curl gnupg -y
```

## 5. Installing OpenMediaVault

1. Use installation script from official GitHub page:
   https://github.com/OpenMediaVault-Plugin-Developers/installScript
```shell
sudo curl -sSL https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/install | sudo bash
```
2. In your web browser, go to device's IP address and login to OMV admin panel using default credentials, for example: http://192.168.18.4
<img width="450" height="280" alt="Pasted image 20260213013240" src="https://github.com/user-attachments/assets/988bb932-9f81-4343-97d8-c0f8dd696c5a" />


3. In **System** → **Workbench** → **Administrator Password**, change admin password.
4. In **Network** → **Interfaces**, edit `enp1s0` IPv4 settings:
	   - Method: `Static`
	   - Address: `192.168.18.4`
	   - Netmask: `255.255.255.0`
	   - Gateway: `192.168.18.1`

<img width="943" height="244" alt="Pasted image 20260213014633" src="https://github.com/user-attachments/assets/39e2c8e5-8e27-4d13-9fbc-ce6acffad1a5" />


## 6. OMV plugins - file system, media library, docker

### 6.1. `sharerootfs` plugin and SMB configuration

1. Go to **System** → **Plugins**.
2. Search and install `openmediavault-sharerootfs`.
3. Go to **Storage** → **Shared Folders**.
4. Create a new folder (e.g. `media`) and save.
<img width="1580" height="250" alt="Pasted image 20260213104346" src="https://github.com/user-attachments/assets/64684aab-7ab3-49cf-865e-ca457ded40ee" />

5. Click on it, select `Permissions`, add them for yourself and save.
<img width="1578" height="350" alt="Pasted image 20260213104322" src="https://github.com/user-attachments/assets/5a19f32c-e9d4-41b9-a708-d823e7acdafb" />

6. Click on it again, select `Access control list` and add permissions.
<img width="1555" height="350" alt="Pasted image 20260213131613" src="https://github.com/user-attachments/assets/81d1c257-c902-4975-8b7b-06233c51e514" />

7. Go to **Services** → **SMB/CIFS** → **Settings** and check `Enabled` and save.
<img width="499" height="275" alt="Pasted image 20260213104427" src="https://github.com/user-attachments/assets/32063ed2-2457-4663-908f-9c4d2d515942" />

8. Go to **Services** → **SMB/CIFS** → **Shares** → create new and select `media` folder.
9. Go to **Users**→ **Users** → select `Permissions` and add them for yourself.
   You can now access the folder from your desktop via SMB / local network, using your user's credentials, not the ones of OMV admin.

   **WARNING** - Since this is a root partition, remember to carefully monitor the space. It's a good idea to leave **at least 10-15 GB of free storage** for the system files.

### 6.2. `compose` plugin and docker

1. Go to **System** → **Plugins**.
2. Search and install `openmediavault-compose`.
3. Go to **Storage** → **Shared Folders**.
4. Create a new folder for compose settings (e.g. **`compose_configs`**)
5. Go to **Services → Compose → Settings**, choose the new shared folder and save.
<img width="750" height="250" alt="Pasted image 20260213112148" src="https://github.com/user-attachments/assets/9cefe980-142f-449e-99c9-227c4edb6d52" />

6. Again in **Services → Compose → Settings**, check if Docker status is `Installed and Running`, otherwise click `Reinstall Docker`.
<img width="750" height="250" alt="Pasted image 20260213112221" src="https://github.com/user-attachments/assets/101a09ad-4f24-458e-858c-3dec09aa2b2b" />

### 6.3. Portainer

1. Go to **Services** → **Compose** → **Files**, and add a new file.
2. Name it `portainer`.
3. In a text field, paste the following and save:
```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./portainer-data:/data
    ports:
      - 9000:9000
```
4. Click on it and select `Up`.
5. Go to the Portainer admin panel http://192.168.18.4:9000 and change admin password.

### 6.4. Pi-hole

1. Go to **Services** → **Compose** → **Files**, and add a new file.
2. Name it `pihole`.
3. In a text field, paste the following and save:
```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    hostname: pihole
    restart: unless-stopped
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:80/tcp" # port 8080, since omv itself uses port 80
    environment:
      - TZ=Europe/Warsaw
      - WEBPASSWORD=YOUR_PASSWORD # set your Pi-hole password here
      - FTLCONF_LOCAL_IPV4=YOUR_IP # set your device IP, e.g. 192.168.18.4
    volumes:
      - ./etc-pihole:/etc/pihole
      - ./etc-dnsmasq.d:/etc/dnsmasq.d
    dns:
      - 1.1.1.1 # backup DNS
      - 8.8.8.8 # backup DNS
```
4. Click on it and select `Up`.
5. If you see `...failed to bind host port 0.0.0.0:53/tcp: address already in use` error, login via SSH to your device as a root and disable `systemd-resolved`. 
```shell
ssh root@192.168.18.4
```
```shell
nano /etc/systemd/resolved.conf # edit text file
```
```shell
rm /etc/resolv.conf
```
```shell
echo "nameserver 1.1.1.1" > /etc/resolv.conf
```
	Change `#DNSStubListener=yes` to `DNSStubListener=no`
```bash
systemctl restart systemd-resolved
```
   Try again, you might need to select `Down` and then `Up` again.
6. Go to the Pi-hole admin panel http://192.168.18.4:8080/admin with the same password that you put in the `pihole` file.
7. In DNS settings, check `Permit all origins`. Save & apply.
<img width="1280" height="656" alt="Pasted image 20260213124729" src="https://github.com/user-attachments/assets/155b14b9-ccac-4193-bb0b-66ced38cfde5" />

8. For your desktop device, change your IPv4 method to `Automatic (Only addresses)` and add DNS server IP:
<img width="565" height="265" alt="Pasted image 20260213123830" src="https://github.com/user-attachments/assets/8750292e-bdae-48aa-a2c0-d5be50695262" />

### 6.5. Jellyfin

1. Go to **Services** → **Compose** → **Files**, and add a new file.
2. Name it `jellyfin`.
3. In a text field, paste the following and save:
```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    devices:
      - /dev/dri:/dev/dri
    container_name: jellyfin
    network_mode: host
    restart: unless-stopped
    volumes:
      - /media/jellyfin_config:/config
      - /media/jellyfin_cache:/cache
      - /media:/data/movies # /media works thanks to sharerootfs plugin
```
4. Click on it and select `Up`.
5. Change permissions via SSH:
```shell
ssh root@192.168.18.4
```
```shell
sudo chown -R 1000:1000 /media
```
```shell
sudo chmod -R 775 /media
```
6. Go to the Jellyfin admin panel http://192.168.18.4:8096 and set up your server.
7. To use Jellyfin on Samsung TV (Tizen), use: https://github.com/Jellyfin2Samsung/Samsung-Jellyfin-Installer

## 7. Installing Tailscale

1. Install Tailscale via SSH:
```shell
ssh root@192.168.18.4
```
```shell
curl -fsSL https://tailscale.com/install.sh | sh
```
2. Run it:
```shell
sudo tailscale up
```
3. Open the generated link in your web browser and add your device.
4. Get your new IP created by Tailscale:
```shell
tailscale ip -4
```
5. Disable key expiry in Tailscale in the [admin panel](https://login.tailscale.com/admin/machines), so you don't have to authorize again in the future:
<img width="401" height="397" alt="Pasted image 20260220165636" src="https://github.com/user-attachments/assets/cb6f6200-9035-4738-bb42-462d06d5d71b" />

6. Activate Tailscale Funnel:
```shell
ssh root@192.168.18.4
```
```shell
sudo tailscale funnel --bg 8096
```
7. Share your newly generated URL to whom you desire, so they can access your server.

## 8. Backup

Since I'm using a single partition for the system and the media, I cannot use any OMV plugin, since they will backup ALL files, including films, music etc. 
I'll use a `BorgBackup` and I'll periodically copy files from the server to my local desktop.

1. Connect with a server via SSH:
```shell
ssh root@192.168.18.4
```

2. Install packages:
```shell
sudo apt update
sudo apt install borgbackup borgmatic
```

3. Create a repo:
```shell
sudo mkdir -p /media/backups/borg_repo
sudo borg init --encryption=none /media/backups/borg_repo
```

4. Create a `Borgmatic` config file:
```shell
sudo mkdir -p /etc/borgmatic
sudo nano /etc/borgmatic/config.yaml
```

5. Paste the following code:
```yaml
source_directories:
  - /
repositories:
  - path: /media/backups/borg_repo
exclude_patterns:
  - /media/movies
  - /media/shows
  - /media/music
  - /media/backups
  - /var/lib/docker
  - /proc
  - /tmp
  - /mnt
  - /dev
  - /sys
  - /run
  - /var/tmp

keep_daily: 7
keep_weekly: 4
keep_monthly: 6
```
6. Make a test backup, so you can see if it works and how big the backup files are. 
   In my case it's ~1,5 GB.
```shell
   sudo borgmatic --verbosity 1
```
7. Enable schedule:
 ```shell
sudo systemctl enable --now borgmatic.timer
 ```
8. You can verify its status and check the logs with the following commands:
```shell
systemctl list-timers borgmatic.timer
```
```shell
sudo journalctl -u borgmatic.service -e
```
