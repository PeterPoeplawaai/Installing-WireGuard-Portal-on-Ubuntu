# Installing WireGuard Portal on Ubuntu 24.04

This guide walks you through installing WireGuard Portal (`wg-portal`) on top of an existing WireGuard installation on Ubuntu 24.04. WireGuard Portal interacts directly with your local system's WireGuard network interfaces using the local backend configuration.
---
## What is WireGuard Portal?

**WireGuard Portal (`wg-portal`)** is a lightweight, self-hosted web management interface and REST API for WireGuard VPN servers. Built as a single native Go binary, it sits on top of an existing WireGuard installation and communicates directly with the Linux kernel via `wgctrl`—allowing you to create, modify, or revoke peers in real time without restarting interfaces or dropping active VPN connections.

### Key Capabilities

* **Zero-Downtime Peer & Interface Management:** Manage multiple local WireGuard interfaces (`wg0`, `wg1`, etc.) and apply peer changes live in kernel space without service interruptions.
* **User Self-Service Portal:** Users can log in to view their assigned devices, download `.conf` files, or scan QR codes to set up mobile and desktop clients independently.
* **Enterprise Authentication:** Supports local user accounts alongside external identity providers via **LDAP / Active Directory**, **OAuth 2.0 / OIDC**, and **Passkeys (WebAuthn)**.
* **Automated Lifecycle & Access Control:** Enforce automatic peer expiration dates, toggle peer access on the fly, and distribute configuration files directly via email.
* **API & Observability:** Exposes a full REST API, webhook triggers, and a Prometheus metrics endpoint for monitoring active connections and bandwidth usage.


### Prerequisites
- An Ubuntu 24.04 server with `sudo` or `root` privileges.
- An existing, functional WireGuard installation (e.g., `wg0`).
- Basic utilities such as `wget` and `nano` installed.
---
### Step 1: Download the WireGuard Portal Binary
First, check your system architecture to ensure you download the matching binary:
```bash
uname -m
```
For standard 64-bit Intel/AMD systems (`x86_64`), download the latest release binary:
```bash
export WG_PORTAL_VERSION="v2.3.1"
wget -O wg-portal https://github.com/h44z/wg-portal/releases/download/${WG_PORTAL_VERSION}/wg-portal_linux_arm64
```
Verify that the downloaded binary is valid:
```bash
file wg-portal
```
---
### Step 2: Set Up Installation Directories and Binary
Create the required directories under `/opt/wg-portal` and install the binary into place:
```bash
sudo mkdir -p /opt/wg-portal/config
sudo mkdir -p /opt/wg-portal/data
sudo install wg-portal /opt/wg-portal/
sudo chmod +x /opt/wg-portal/wg-portal
```
---
### Step 3: Create the Configuration File
Create and open the YAML configuration file at `/opt/wg-portal/config/config.yml`:
```bash
sudo nano /opt/wg-portal/config/config.yml
```
Paste the following configuration template into the file. Be sure to replace `YOUR_SERVER_IP` and generate strong, random strings for `session_secret` and `csrf_secret`.

> **Note:** You can generate random secrets in your terminal using `openssl rand -hex 32`.
```yaml
logLevel: info

backend:
  default: local
  local_resolvconf_prefix: ""

database:
  type: sqlite
  sqlite:
    file: /opt/wg-portal/data/wg-portal.db

web:
  listening_address: 0.0.0.0:8888  # use private ip address of your instance to keep it only reachable for connected VPN clients
  external_url: http://YOUR_SERVER_IP:8888   # use private ip address of your instance to keep it only reachable for connected VPN clients
  session_secret: change-this-to-a-random-secure-string
  csrf_secret: change-this-to-another-random-secure-string
```
---
### Step 4: Create a Systemd Service
To manage WireGuard Portal as a background service that starts automatically on boot, create a systemd unit file:
```bash
sudo nano /etc/systemd/system/wg-portal.service
```
Add the following service configuration:
```ini
[Unit]
Description=WireGuard Portal
Documentation=https://wgportal.org
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/wg-portal
ExecStart=/opt/wg-portal/wg-portal
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```
---
### Step 5: Enable and Start the Service
Reload systemd to register the new service, then enable and start it:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wg-portal
```
Verify that the service is active and running without errors:
```bash
sudo systemctl status wg-portal
```
---
### Step 6: Access the Web Interface
Ensure port `8888/tcp` is allowed through your firewall (or configure a reverse proxy with HTTPS).

```bash
sudo iptables -I INPUT 3 -i wg0 -p tcp --dport 8888 -j ACCEPT
sudo netfilter-persistent save
```

Open your web browser and navigate to `http://YOUR_SERVER_IP:8888`.
Log in using the initial administrator credentials:

```initial credentials:
Username: admin@wgportal.local
Password: wgportal-default
```

From the dashboard, import or manage your existing WireGuard interfaces and configure peers. Don't forget to change the default password!
