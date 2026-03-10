---
title: "Raspberry Pi + Twingate + WireGuard"
date: 2026-03-11
draft: false
tags: ["VPN", "Automation", "Security", "Remote Access", "Twingate", "rpi","Raspberry Pi", "WireGuard"]
---

## 🚀 Raspberry Pi Remote Access via Twingate + WireGuard

#### *Use your Raspberry Pi as a secure home VPN exit node with zero port forwarding*

---

### **0. Prerequisites**

- Raspberry Pi (OS: Bullseye or Bookworm, 64-bit)
- SSH or physical access to the Pi
- A Twingate tenant (free plan is fine)
- Your Pi has a static IP or DHCP reservation
    - Get Pi's IP:

```bash
hostname -I
```

Update the Pi:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## **1. Create Remote Network & Connector in Twingate**

1. Log into the Twingate Admin Console.
2. Create a **Remote Network** (e.g., `Home-Network`).
3. Under **Connectors**, click an unused Connector → **Deploy Connector**.
4. Select **Linux (systemd)**.

Keep this window open — you’ll copy the script into the Pi.

---

## **2. Install Twingate Connector on Raspberry Pi**

Run the script provided by the Twingate console on your Raspberry Pi.

It looks like:

```bash
curl -fsSL https://b.twingate.com/download/linux/setup.sh \
  | sudo TWINGATE_URL="YOUR_TENANT.twingate.com" \
    TWINGATE_ACCESS_TOKEN="YOUR_ACCESS_TOKEN" \
    TWINGATE_REFRESH_TOKEN="YOUR_REFRESH_TOKEN" \
    bash
```

Once installed, the Connector should show **Connected** in the Twingate console.

---

## **3. Install WireGuard on Raspberry Pi**

### **3.1 Install packages**

```bash
sudo apt update
sudo apt install -y wireguard wireguard-tools
```

### **3.2 Generate server keys**

```bash
sudo -i

umask 077
wg genkey | tee /etc/wireguard/server_privatekey | wg pubkey > /etc/wireguard/server_publickey

SERVER_PRIV_KEY=$(cat /etc/wireguard/server_privatekey)
SERVER_PUB_KEY=$(cat /etc/wireguard/server_publickey)
```

### **3.3 Create WireGuard server configuration**

Create `/etc/wireguard/wg0.conf`:

```bash
cat > /etc/wireguard/wg0.conf <<EOF
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = $SERVER_PRIV_KEY
SaveConfig = true

## NAT for VPN clients
PostUp   = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; \
           iptables -A FORWARD -i eth0 -o wg0 -j ACCEPT; \
           iptables -A FORWARD -i wg0 -o eth0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; \
           iptables -D FORWARD -i eth0 -o wg0 -j ACCEPT; \
           iptables -D FORWARD -i wg0 -o eth0 -j ACCEPT
EOF
```

> If using Wi-Fi, replace eth0 with wlan0.
> 

### **3.4 Enable IP forwarding**

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### **3.5 Start WireGuard**

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo systemctl status wg-quick@wg0
```

---

## **4. Expose ONLY WireGuard (UDP 51820) via Twingate**

In Twingate Admin Console:

1. Go to **Resources → Add Resource**
2. **Name:** `wireguard-home`
3. **Address:** Pi LAN IP, e.g., `192.168.1.50`
4. Enable **Port Restrictions**
5. Add:
    - **UDP 51820**
6. Assign to a Group (e.g., `Admins`)
7. Ensure your user is in that Group

This prevents any other Pi ports from being remotely reachable.

---

## **5. Create a WireGuard Client (Peer)**

### **5.1 Generate client keys**

```bash
umask 077
wg genkey | tee ~/laptop_privatekey | wg pubkey > ~/laptop_publickey

LAPTOP_PRIV_KEY=$(cat ~/laptop_privatekey)
LAPTOP_PUB_KEY=$(cat ~/laptop_publickey)
```

### **5.2 Add the peer to the server config**

Edit `/etc/wireguard/wg0.conf`:

```
[Peer]
PublicKey = LAPTOP_PUBLIC_KEY_HERE
AllowedIPs = 10.8.0.2/32
```

Restart WireGuard:

```bash
sudo systemctl restart wg-quick@wg0
```

---

## **6. Create Client WireGuard Config**

Create a file called `laptop.conf`:

```
[Interface]
PrivateKey = LAPTOP_PRIVATE_KEY_HERE
Address = 10.8.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY_HERE
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = wireguard-home:51820
PersistentKeepalive = 25
```

Important points:

- The **Endpoint** uses the **Twingate Resource name**, not a public address.
- All traffic (`0.0.0.0/0`) will exit through your home internet.

---

## **7. Connect From Remote Device**

On your laptop/phone:

1. Install **Twingate Client**
2. Log in to your tenant
3. Turn **Twingate ON**
4. Install **WireGuard app**
5. Import `laptop.conf`
6. Turn **WireGuard tunnel ON**

Your traffic now flows:

```
Device → Twingate → Raspberry Pi (WireGuard) → Home Internet
```

No port forwarding.

No public exposure.

Fully encrypted end-to-end.

---

## **8. Verification**

Check public IP (should match your home network):

```bash
curl ifconfig.me
```

Or open [https://ifconfig.me](https://ifconfig.me/) in a browser.
