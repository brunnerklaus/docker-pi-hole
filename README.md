docker-pihole
========
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

This project runs [Pihole](https://github.com/pi-hole/pi-hole) and [cloudflared](https://github.com/crazy-max/docker-cloudflared) as docker containers.

- **The Pi-hole**[®](https://pi-hole.net/trademark-rules-and-brand-guidelines/) is a [DNS sinkhole](https://en.wikipedia.org/wiki/DNS_Sinkhole) that protects your devices from unwanted content, without installing any client-side software.

- **Cloudflared** an [Argo Tunnel client](https://github.com/cloudflare/cloudflared) proxy-dns Docker image who runs an [argo-tunnel](https://developers.cloudflare.com/argo-tunnel)

- **DNS-Over-HTTPS** is a protocol for performing DNS lookups securly via HTTPS. (If you are not aware of what DNS is, please read this [primer](https://developers.cloudflare.com/1.1.1.1/dns-over-tls/) before continuing).


## Install raspbian
Download [Raspberry Pi Imager](https://www.raspberrypi.org/downloads/https://www.raspberrypi.org/downloads/) and install it on your SDCard

### Enable SSH
Mount `boot` partition of the the SD-Card and create `/boot/ssh.txt` file to enable SSH after installing raspbian.
```bash
cd /Volumes/boot
touch ssh.txt
```

## Update system packages
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y vim screen pydf git dnsutils
```

## Diable IPv6
dd this loine in `/etc/sysctl.conf`
```bash
net.ipv6.conf.all.disable_ipv6 = 1
```

For the change to take effect without rebooting:
```bash
sudo sysctl -p
```
Verify that IPv6 address does not show up in ifconfig.

## Install zsh
```bash
sudo apt install -y zsh
wget -O .zshrc https://git.grml.org/f/grml-etc-core/etc/zsh/zshrc
chsh -s /bin/zsh
```

## Disable swapping
Take care of the SD card :rocket:
```bash
sudo dphys-swapfile swapoff
udo systemctl disable dphys-swapfile.service
```

## Set static IP
For a static IP address on an Ethernet connection edit `/etc/dhcpcd.conf`

Type in the following lines on the top of the file:

```bash
interface eth0
static ip_address=192.168.1.XX/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1 1.1.1.1
```

```bash
sudo reboot
```

## Set locale
`en_US.UTF-8` via `raspi-config`

## Log2ram
Usefull for RaspberryPi for not writing on the SD card all the time. You need it because your SD card doesn't want to suffer anymore! See [log2ram](https://github.com/azlux/log2ram) project for more information.

```bash
echo "deb http://packages.azlux.fr/debian/ buster main" | sudo tee /etc/apt/sources.list.d/azlux.list
wget -qO - https://azlux.fr/repo.gpg.key | sudo apt-key add -
sudo apt update
sudo apt install -y log2ram
```

Modify `/etc/log2ram.conf`and configure size for the RAM foldeer

```bash
sudo vim /etc/log2ram.conf
```
```
SIZE=128M
```


You can now check the mount folder in ram with (You will see lines with log2ram if working) - Reboot your RaspberryPi to active the changes..
```
# df -h
…
log2ram         128M   34M   95M  27% /var/log
…

# mount
…
log2ram on /var/log type tmpfs (rw,nosuid,nodev,noexec,relatime,size=131072k,mode=755)
…
```

## Install docker
See https://github.com/docker/docker-install 🐳 for more information
```bash
curl -sSL https://get.docker.com | sh
```
If you would like to use Docker as a non-root user, you should now consider
adding your user to the "docker" group with something like:
Remember that you will have to log out and back in for this to take effect!
```bash
sudo usermod -aG docker pi
```

### Install required packages
```bash
sudo apt update
sudo apt install -y python3-pip libffi-dev
```

### Install Docker Compose from pip (using Python3)
This might take a while

```bash
sudo pip3 install docker-compose
```

## Install Portainer
If you are running Linux, deploying Portainer is as simple as:

```bash
docker volume create portainer_data
docker run -d -p 9000:9000 -p 8000:8000 --name portainer --restart always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
```
Voilà, you can now use Portainer by accessing the port 9000 on the server where Portainer is running. See [Portainer Project](https://portainer.readthedocs.io/en/latest/deployment.html) for more information.

## Clone the repo
```bash
git clone https://github.com/brunnerklaus/docker-pihole.git ~/docker-pi-hole
cd ~/docker-pi-hole
```

## Set Crendentials and DNS
```bash
cp .env.example .env
```
```yml
PIHOLE_WEBPASSWORD=changeme
PIHOLE_DNS1=10.5.0.3#5053
PIHOLE_DNS2=no
```

## Quick Start
Run Pihole standalone
[Docker-compose](https://docs.docker.com/compose/install/) example:

```yaml
version: "3"

# More info at https://github.com/pi-hole/docker-pi-hole/ and https://docs.pi-hole.net/
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "67:67/udp"
      - "80:80/tcp"
      - "443:443/tcp"
    environment:
      TZ: Europe/Vienna
      # WEBPASSWORD: 'set a secure password here or it will be random'
    # Volumes store your data between container upgrades
    volumes:
       - etc:/etc/pihole/
       - dnsmasq:/etc/dnsmasq.d/
    dns:
      - 127.0.0.1
      - 1.1.1.1
    # Recommended but not required (DHCP needs NET_ADMIN)
    #   https://github.com/pi-hole/docker-pi-hole#note-on-capabilities
    cap_add:
      - NET_ADMIN
    restart: unless-stopped
```

Before starting edit the docker-compose file and set
```yml
TZ: Europe/Vienna
WEBPASSWORD: 'changeme'
```
to your need.

## Usage

Docker compose is the recommended way to run this image. You can use the following docker-compose file, then run all containers:

```bash
docker-compose up -d
docker-compose logs -f
```

You can access pihole on the server where it runs on port 80.

## Upgrade

To upgrade, pull the newer image and launch the container:

```bash
docker-compose pull
docker-compose up -d
```


## Check cloudflared
```bash
docker exec cloudflared cloudflared --version
```

### Check TLS traffic for DNS requests
```bash
sudo apt install tshark
```
Capture traffic on port 443
```bash
sudo tshark -f 'tcp port 443'
Capturing on 'eth0'
    1 0.000000000  127.0.0.1 → 1.1.1.1    TLSv1.2 92 Application Data
    2 0.000486927  127.0.0.1 → 1.1.1.1    TLSv1.2 121 Application Data
    3 0.010827079    1.1.1.1 → 127.0.0.1  TCP 60 443 → 44498 [ACK] Seq=1 Ack=106 Win=67 Len=0
    4 0.011915568    1.1.1.1 → 127.0.0.1  TLSv1.2 408 Application Data
    5 0.011917859    1.1.1.1 → 127.0.0.1  TLSv1.2 85 Application Data
```

## How can I help ?

All kinds of contributions are welcome :raised_hands:! The most basic way to show your support is to star :star2: the project, or to raise issues :speech_balloon:

Thanks again for your support, it is much appreciated! :pray:

## License

See [LICENSE](LICENSE) for the complete license.
[badge-license]: https://img.shields.io/badge/license-Apache2-green.svg?style=flat
