# Raspberry Pi 4 Homelab Setup

This documentation provides a step-by-step guide for setting up a fresh installation of Raspberry Pi 4 for a homelab environment. It includes instructions for overclocking, increasing SWAP memory, building a kernel with AppArmor, creating a Wi-Fi hotspot, installing SAMBA, Home Assistant Supervised, Jellyfin, and Portainer. Follow the steps below to set up your Raspberry Pi 4 for a productive homelab environment.

## 1. Overclock:

```
sudo apt update && sudo apt dist-upgrade -y && sudo apt upgrade -y
sudo nano /boot/config.txt
```

In that file, add these lines:
```
arm_freq=2000
gpu_freq=750
over_voltage=6
force_turbo=1

sudo reboot
```
### 1.1. Increase SWAP memory to 1000M
```
sudo nano /etc/dphys-swapfile
```
find the line that starts with CONF_SWAPSIZE and change its value from 100 to 1000
```
sudo /etc/init.d/dphys-swapfile restart
```
### 1.2. Build a kernel with apparmor:

Documentation: https://www.raspberrypi.com/documentation/computers/linux_kernel.html

```
sudo apt install git bc bison flex libssl-dev make
git clone --depth=1 https://github.com/raspberrypi/linux
cd linux
KERNEL=kernel7l
make bcm2711_defconfig
```

Add these lines at the end of '~/linux/.config'
```
CONFIG_SECURITY_APPARMOR=y
CONFIG_SECURITY_APPARMOR_BOOTPARAM_VALUE=1
CONFIG_DEFAULT_SECURITY_APPARMOR=y
CONFIG_AUDIT=y
```
Build the kernel
```
make -j4 Image.gz modules dtbs
sudo make modules_install
sudo cp arch/arm64/boot/dts/broadcom/*.dtb /boot/
sudo cp arch/arm64/boot/dts/overlays/*.dtb* /boot/overlays/
sudo cp arch/arm64/boot/dts/overlays/README /boot/overlays/
sudo cp arch/arm64/boot/Image.gz /boot/$KERNEL.img
```

### 1.3. If you want to create a hotspot (for IOT devices)

```
sudo systemctl stop systemd-resolved
sudo apt-get install hostapd
sudo apt-get install dnsmasq
sudo systemctl stop hostapd && sudo systemctl stop dnsmasq
```

Add the following lines at the end of the file '/etc/dhcpcd.conf':
```
interface wlan0
static ip_address=192.168.0.10/24
denyinterfaces eth0
denyinterfaces wlan0
```
Edit the ssid and password (and the wifi frequency band) '/etc/hostapd/hostapd.conf':
```
interface=wlan0
driver=nl80211
ssid=raspberry
hw_mode=g
channel=7
wpa=2
wpa_passphrase=supersecurepassword
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```
Save changes and restart services:

```
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
sudo systemctl unmask hostapd
sudo systemctl stop systemd-resolved
sudo systemctl restart hostapd
sudo systemctl restart dnsmasq
```

## 2. SAMBA

```
sudo apt install samba -y
sudo nano /etc/samba/smb.conf
```

Add These lines:
```
[External disk]
comment = Raspberry Pi external disk
path = /media/ubuntu
writeable = yes
read-only = no
browsable = yes
```
Add a user and password
```
sudo smbpasswd -a ubuntu
sudo systemctl restart smbd.service
```

## 3. Home Assistant supervised

Documentation here: https://github.com/home-assistant/supervised-installer

```
sudo su \
apt install \
apparmor \
jq \
wget \
curl \
udisks2 \
libglib2.0-bin \
network-manager \
dbus \
lsb-release \
systemd-journal-remote -y
```
Install docker and Home Assistant supervised
```
curl -fsSL get.docker.com | sh
wget https://github.com/home-assistant/os-agent/releases/download/1.5.1/os-agent_1.5.1_linux_aarch64.deb
dpkg -i os-agent_1.5.1_linux_aarch64.deb
wget https://github.com/home-assistant/supervised-installer/releases/latest/download/homeassistant-supervised.deb
apt install ./homeassistant-supervised.deb
```

4. Jellyfin

Documentation here: https://jellyfin.org/docs/general/installation/linux

```
curl https://repo.jellyfin.org/install-debuntu.sh | sudo bash
sudo setfacl -m u:jellyfin:rx /media/ubuntu
```

## 5. NextCloud

Option 1: Using snap:

```
sudo apt install snapd -y
sudo snap install core
sudo ln -s /var/lib/snapd/snapd/snap/ /snap
sudo snap install nextcloud
```

If you need to change the ports:

```
sudo snap set nextcloud ports.http=82 ports.https=444
```

Additional settings for Nextcloud, run the container console or in the directory '/var/snap/nextcloud/current/nextcloud/config':

```
apt update && apt install nano -y
nano config.php
```

Add this line:
	'overwriteprotocol' => 'https',

## 6. Qbittorrent

```
docker run -d \
  --name=qbittorrent \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=Etc/UTC \
  -e WEBUI_PORT=8080 \
  -p 8080:8080 \
  -p 6881:6881 \
  -p 6881:6881/udp \
  -v /home/ubuntu/QbitTorrent:/config \
  -v /media/ubuntu/Big/Torrents:/downloads \
  --restart unless-stopped \
  lscr.io/linuxserver/qbittorrent:latest
```

the user/password is: admin/adminadmin

## 7. Add Home Assistant tunnel

Documentation: https://github.com/brenner-tobias/ha-addons

* Add this addon in the Add-on store in Homeassistant: https://github.com/brenner-tobias/ha-addons

* Install the addon and click configuration, "External Homeassistant Hostname: homeassistant.domain.com", "Cloudflare Tunnel Name: homeassistant", save changes.

* Edit the file 'configuration.yaml' and add these lines:
```
http:
  ip_ban_enabled: true
  login_attempts_threshold: 5
  use_x_forwarded_for: true
  trusted_proxies:
    - 172.30.33.0/24
```
Restart Home assistant > Start Cloudflared > See cloudflared Log > Open the required URL to log into your cloudflared account

8. Cloudflared

Documentation here: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide/local/#set-up-a-tunnel-locally-cli-setup
* Run the commands using  'sudo su'

```
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb
dpkg -i cloudflared-linux-arm64.deb
cloudflared tunnel login
cloudflared tunnel create raspberry
```
Add DNS
```
cloudflared tunnel route dns raspberry jellyfin.domain.com
cloudflared tunnel route dns raspberry nextcloud.domain.com
cloudflared tunnel route dns raspberry qbittorrent.domain.com
```
```
nano /root/.cloudflared/config.yml
```

Include this information into the file:
```
tunnel: 8272d213-a0a5-4ec7-b9fc-337df81f47fe
credentials-file: /root/.cloudflared/8272d213-a0a5-4ec7-b9fc-337df81f47fe.json
ingress:
  - hostname: jellyfin.domain.com
    service: http://localhost:8096
    disableChunkedEncoding: true
    noHappyEyeballs: true
  - hostname: nextcloud.domain.com
    service: http://localhost:82
    disableChunkedEncoding: true
    noHappyEyeballs: true
  - hostname: qbittorrent.domain.com
    service: http://localhost:8080
    disableChunkedEncoding: true
    noHappyEyeballs: true
  - service: http_status:404
```
To fix th max mem error limit
```
sysctl -w net.core.rmem_max=2500000
cloudflared tunnel run raspberry
```

If you want to install the service, run:

```
cloudflared service install
```

If you want to uninstall the service:

```
systemctl stop cloudflared
rm -r /etc/systemd/system/cloudflared*
rm -r /etc/cloudflared/
```
