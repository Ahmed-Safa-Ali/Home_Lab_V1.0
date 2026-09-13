# Ahmed_Project — Home NAS Server

Turning an old HP Pavilion g7 laptop into a home NAS server using Ubuntu Server, CasaOS, and Samba.

## Hardware

- HP Pavilion g7 Notebook
- Intel Core i5-2410M @ 2.30GHz (2 core / 4 thread)
- 4GB DDR3 RAM
- 238GB internal SSD
- 2TB external HDD (USB)

## Software

- Ubuntu Server 26.04 LTS
- CasaOS (web management dashboard)
- Samba (file sharing)

---

## Step 1 — Create the Ubuntu Server installer USB

Download Ubuntu Server LTS from the official website, then use Rufus to create a bootable USB.

**Issue:** None at this step.

---

## Step 2 — Enter BIOS / Boot Menu

Pressing F9 during normal boot did not open the Boot Menu in time.

**Fix — enter firmware settings directly from Windows:**
```
shutdown /r /fw /t 0
```

**Issue:** The USB drive did not appear under the default boot options (only "Notebook Hard Drive" and "Internal CD/DVD ROM Drive" were listed). It was found inside **System Configuration → Boot Options → Boot Order**, listed as `USB Diskette on Key/USB Hard Disk`. Moved it to the top of the list and saved with F10.

---

## Step 3 — Install Ubuntu Server

- Storage: **Use an entire disk** (full wipe, no dual-boot)
- LVM: enabled
- LUKS encryption: disabled (avoids needing a passphrase on every headless boot)

**Issue:** During installation, network autoconfiguration failed on the first attempt. Retrying the network step succeeded and obtained an IP via DHCP.

---

## Step 4 — Check network status after install

```bash
ip a
```

**Issue:** Ethernet interface (`eno1`) showed `UP` but had no IPv4 address, only a link-local IPv6 address.

**Diagnosis:**
```bash
sudo cat /etc/netplan/*.yaml
sudo networkctl status eno1
sudo journalctl -u systemd-networkd -f
```

The netplan config was correct (`dhcp4: true`). The log showed repeated:
```
eno1: Lost carrier
eno1: Gained carrier
```

**Fix:** The Ethernet cable connection was physically loose. Firmly reseating the cable in the port resolved it, and the system obtained an IP:
```
eno1: DHCPv4 address 192.168.0.112/24, gateway 192.168.0.1 acquired
```

---

## Step 5 — Install CasaOS

```bash
curl -fsSL https://get.casaos.io | sudo bash
```

Access the dashboard at `http://<server-ip>`.

**Issue:** None at this step.

---

## Step 6 — Format and mount the external HDD

```bash
lsblk
```

**Issue:** CasaOS Storage Manager showed the 2TB drive with `NaN` for used/available space, and the Format button in the web UI did not respond.

**Fix — format and mount manually:**
```bash
sudo umount /media/devmon/2-TB-HDD
sudo mkfs.ext4 /dev/sdb2
sudo mkdir -p /mnt/storage
sudo mount /dev/sdb2 /mnt/storage
df -h /mnt/storage
```

---

## Step 7 — Install and configure Samba

```bash
sudo apt update
sudo apt install samba -y
sudo smbpasswd -a <username>
```

**Issue:** `smbpasswd` first failed with "Mismatch — password unchanged" (typed the password differently on the two prompts). Retried and it succeeded.

**Issue:** A later `sudo smbpasswd -a <username>` attempt failed 3 times with "Authentication failed." This was the Ubuntu system/sudo password being entered incorrectly, not the new Samba password. Retried with the correct system password and it worked.

Edit the Samba config:
```bash
sudo nano /etc/samba/smb.conf
```

Add at the end of the file:
```ini
[storage]
   path = /mnt/storage
   browseable = yes
   writable = yes
   guest ok = no
   valid users = <username>
```

Restart the service:
```bash
sudo systemctl restart smbd
sudo systemctl status smbd
```

**Issue:** Connecting from Windows via `\\<server-ip>` failed with: *"You can't access this shared folder because your organization's security policies block unauthenticated guest access."* Root cause: Samba was not installed yet at that point. After installing Samba and adding the `[storage]` share (steps above), the connection succeeded using username `<username>` and the Samba password.

---

## Step 8 — Set a static IP (DHCP Reservation)

Done from the router admin page (TP-Link Archer C50) instead of editing netplan, to avoid losing remote access if misconfigured:

```
Advanced → Network → DHCP → Address Reservation → Add New
MAC Address: xx:xx:xx:xx:xx:xx
IP Address: 192.168.0.112
Status: Enabled
```

**Issue:** None at this step.

---

## Useful diagnostic commands

```bash
ip a
hostname -I
lsblk
df -h
sudo networkctl status eno1
sudo journalctl -u systemd-networkd -f
sudo systemctl status smbd
```

---

## Planned next steps

- WireGuard or Tailscale VPN for remote access outside the home network
- Pi-hole or AdGuard Home for network-wide ad blocking
- UFW firewall with basic rules
- Jellyfin for media streaming
- RAM upgrade (DDR3) for running more services
