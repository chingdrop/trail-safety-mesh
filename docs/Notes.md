When the Raspberry pi connects to the home network, Eero only sees a IPv6 address. You have to go into OpnSense DHCP leases to find the IP address.

I believe that there is a startup process that handles the initial update and upgrade because there is a process handle for an auto-update in htop. After about 20 minutes, the raspi will prompt you to reboot the device when you log in.

`sudo cp /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.bak`
`sudo nano /etc/apt/sources.list.d/ubuntu.sources`

Append these lines to the file.

```bash
## Ubuntu updates pocket -- bug fixes after release.
## Required for matching versions of -dev packages when -security has shipped fixes.
Types: deb
URIs: http://ports.ubuntu.com/ubuntu-ports/
Suites: noble-updates
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

## Ubuntu backports -- newer software backported to the LTS release.
## Optional but useful for newer tooling. Same components as main.
Types: deb
URIs: http://ports.ubuntu.com/ubuntu-ports/
Suites: noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

Install common packages
`sudo apt-get install -y htop iotop iftop iw jq rsync tmux curl ca-certificates gnupg lsb-release bash-completion net-tools bind9-dnsutils ufw unattended-upgrades needrestart pciutils usbutils wireless-tools`

Install, configure Chrony and check if chrony is tracking.

```bash
sudo apt-get install -y chrony

sudo systemctl stop systemd-timesyncd
sudo systemctl disable systemd-timesyncd

sudo systemctl enable chrony
sudo systemctl start chrony

chronyc tracking
```

Add current raspi user to the video group.
`sudo usermod -aG video raspi`

Add new VCIO rules specifying the kernel for udev.
`sudo nano /etc/udev/rules.d/10-vcio.rules`

```bash
# /dev/vcio is the Pi's GPU mailbox device. Vcgencmd uses it for telemetry.
# Default Ubuntu udev leaves it 600 root:root which forces sudo.
# This rule sets group=video, mode=0660 to match Raspberry Pi OS behaviour.
KERNEL=="vcio", GROUP="video", MODE="0660"
```

Reload udevadm with the new rules.
`sudo udevadm control --reload-rules`
`sudo udevadm trigger --name-match=vcio`

Hostname seems to already be set from the raspberry pi imager.

SSH folder seems to already be created with the correct permissions, same with the authorized_keys file.
Add your public key to the authorized_keys file.
`sudo nano ~/.ssh/authorized_keys`

Add local SSH hardening rules.
`sudo nano /etc/ssh/sshd_config.d/01-local-hardening.conf`

```bash
# Local SSH hardening for field-gateway.
#
# IMPORTANT: sshd uses FIRST-WINS precedence on duplicate keys (unlike
# most Linux configs, which use last-wins). This file is named 01- so
# it parses BEFORE 50-cloud-init.conf, which would otherwise re-enable
# password auth.
#
# Goal: key-only SSH auth, no GUI, idle disconnect on flaky links.

PasswordAuthentication no
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 3
```

Reload SSH daemon with the new rules.
`sshd -t`

Initialize the firewall by adding these rules.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed
```

Add allow rules for OpenSSH and management ports for OpenTakServer.

```bash
sudo ufw allow from "10.0.0.0/8" to any port 22 proto tcp comment "SSH from RFC1918"
sudo ufw allow from "172.16.0.0/12" to any port 22 proto tcp comment "SSH from RFC1918"
sudo ufw allow from "192.168.0.0/16" to any port 22 proto tcp comment "SSH from RFC1918"

sudo ufw allow from "fc00::/7" to any port 22 proto tcp comment "SSH from IPv6 private"
sudo ufw allow from "fe80::/10" to any port 22 proto tcp comment "SSH from IPv6 private"

sudo ufw allow from "10.0.0.0/8" to any port 8443 proto tcp comment "OTS Marti API/web (RFC1918)"
sudo ufw allow from "172.16.0.0/12" to any port 8443 proto tcp comment "OTS Marti API/web (RFC1918)"
sudo ufw allow from "192.168.0.0/16" to any port 8443 proto tcp comment "OTS Marti API/web (RFC1918)"

sudo ufw allow from "10.0.0.0/8" to any port 8089 proto tcp comment "OTS CoT TLS (RFC1918)"
sudo ufw allow from "172.16.0.0/12" to any port 8089 proto tcp comment "OTS CoT TLS (RFC1918)"
sudo ufw allow from "192.168.0.0/16" to any port 8089 proto tcp comment "OTS CoT TLS (RFC1918)"

sudo ufw allow from "10.0.0.0/8" to any port 1883 proto tcp comment "Mosquitto MQTT (RFC1918)"
sudo ufw allow from "172.16.0.0/12" to any port 1883 proto tcp comment "Mosquitto MQTT (RFC1918)"
sudo ufw allow from "192.168.0.0/16" to any port 1883 proto tcp comment "Mosquitto MQTT (RFC1918)"
```

Enable the firewall.
`ufw --force enable`

Check to make sure unattended-upgrades is configured properly.
`sudo nano /etc/apt/apt.conf.d/20auto-upgrades`

```bash
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

Create a local rule for unattended-upgrades.
`sudo nano /etc/apt/apt.conf.d/52unattended-upgrades-local`

```bash
// Local overrides for unattended-upgrades on field-gateway.
//
// Naming: 52- comes after 50unattended-upgrades alphabetically. APT config
// uses LAST-WINS for repeated SCALAR keys, but LISTS are ADDITIVE. To
// truly replace a list, use the #clear directive (literal "#", NOT a
// comment).
//
// Goal: security-only, no auto-reboot, suitable for vehicle field deployment.

#clear Unattended-Upgrade::Allowed-Origins;
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
    "${distro_id}ESM:${distro_codename}-infra-security";
}; 

// Field deployment: never reboot autonomously. Operator decides.
Unattended-Upgrade::Automatic-Reboot "false";
Unattended-Upgrade::Automatic-Reboot-WithUsers "false";

// If a package is held back, retry minimally rather than pulling in
// regular-pocket packages.
Unattended-Upgrade::MinimalSteps "true";

// Keep the SD card uncluttered.
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
```

Enable, start and confirm unattended-upgrades is configured.

```bash
sudo systemctl enable unattended-upgrades
sudo systemctl start unattended-upgrades
apt-config dump | grep 'Unattended-Upgrade'
```

Create the directory.
`sudo mkdir /etc/systemd/journald.conf.d`
Create the Journaling config.

```bash
# Journald size limits for field-gateway.
# Context: SAR field deployment on SD card. Cap on-disk journal to bound
# write wear and storage growth across multi-week deployments.

[Journal]
SystemMaxUse=200M
SystemMaxFileSize=50M
RuntimeMaxUse=50M
Storage=persistent
ForwardToSyslog=no
```

Restart Journaling
`sudo systemctl restart systemd-journald`
Configure fstab to commit on the new schedule.
`sudo nano /etc/fstab`

```bash
LABEL=writable  /       ext4    defaults,noatime,commit=60      0       1
LABEL=system-boot       /boot/firmware  vfat    defaults        0       1
```

Restart daemons and remount base volume.

```bash
sudo systemctl daemon-reload
sudo mount -o remount /
```

Install pre-requisite packages for OpenTakServer.
`sudo apt-get install -y python3-venv python3-pip python3-dev python3.12-venv build-essential libffi-dev libssl-dev zlib1g-dev nginx postgresql rabbitmq-server git`
Check that necessary services are active and install OpenTakServer.

```bash
sudo systemctl status nginx
sudo systemctl status postgresql
sudo systemctl status rabbitmq-server

curl -s -L https://i.opentakserver.io/ubuntu_installer | bash -
```

Select No for ZeroTier and Mumble.
Restart the UFW rules for MQTT.

```bash
sudo ufw delete 12
sudo ufw delete 11
sudo ufw delete 10
```

Configure UFW for OpenTakServer.

```bash
# OTS HTTPS web/API
sudo ufw allow from 10.0.0.0/8 to any port 443 proto tcp comment 'OTS HTTPS web/API (RFC1918)'
sudo ufw allow from 172.16.0.0/12 to any port 443 proto tcp comment 'OTS HTTPS web/API (RFC1918)'
sudo ufw allow from 192.168.0.0/16 to any port 443 proto tcp comment 'OTS HTTPS web/API (RFC1918)'

# OTS certificate enrollment
sudo ufw allow from 10.0.0.0/8 to any port 8446 proto tcp comment 'OTS certificate enrollment (RFC1918)'
sudo ufw allow from 172.16.0.0/12 to any port 8446 proto tcp comment 'OTS certificate enrollment (RFC1918)'
sudo ufw allow from 192.168.0.0/16 to any port 8446 proto tcp comment 'OTS certificate enrollment (RFC1918)'

sudo ufw allow from 10.0.0.0/8 to any port 8883 proto tcp comment 'OTS RabbitMQ MQTT TLS for Meshtastic (RFC1918)'
sudo ufw allow from 172.16.0.0/12 to any port 8883 proto tcp comment 'OTS RabbitMQ MQTT TLS for Meshtastic (RFC1918)'
sudo ufw allow from 192.168.0.0/16 to any port 8883 proto tcp comment 'OTS RabbitMQ MQTT TLS for Meshtastic (RFC1918)'

sudo ufw allow from 10.0.0.0/8 to any port 1883 proto tcp comment 'OTS RabbitMQ MQTT for Meshtastic (RFC1918)'
sudo ufw allow from 172.16.0.0/12 to any port 1883 proto tcp comment 'OTS RabbitMQ MQTT for Meshtastic (RFC1918)'
sudo ufw allow from 192.168.0.0/16 to any port 1883 proto tcp comment 'OTS RabbitMQ MQTT for Meshtastic (RFC1918)'
```

Install pre-requisites for Meshtasticd
`sudo apt-get install -y software-properties-common`
Install Meshtasticd and load one of the default configs.

```bash
sudo add-apt-repository ppa:meshtastic/beta
sudo apt-get install meshtasticd -y
sudo cp /etc/meshtasticd/available.d/lora-RAK6421-13300-slot1.yaml   /etc/meshtasticd/config.d/lora-RAK6421-13300-slot1.yaml
```
