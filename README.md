# Deploy VMGRAY01 — Graylog 7 + OpenSearch 2.x + MongoDB (Ubuntu 24.04)

> Single-VM deployment of Graylog 7 with self-managed OpenSearch and MongoDB on Ubuntu Server 24.04, running as `vmgray01` on Proxmox. Copy/paste friendly.

---

## At a Glance

| Field | Value |
|-------|-------|
| VM/CT Name | `VMGRAY01` |
| Host Node | Proxmox cluster (CPU type `host` required) |
| IP Address | `10.0.0.50` (example) |
| OS | Ubuntu Server 24.04 LTS (minimal) |
| vCPU / RAM / Disk | 2–4 / 4–8 GB / 40–80 GB |
| Key Ports | `9000` (Graylog Web/API), `9200` (OpenSearch), `27017` (MongoDB) |
| Credentials | Graylog admin password set during install (step 7.2) |
| Depends On | Proxmox (with AVX-capable CPU) |
| Start Order | MongoDB → OpenSearch → Graylog |

> **Warning:** `vmgray01` is log stack only (MongoDB + OpenSearch + Graylog). No Prometheus/Grafana/etc. here.

> **Warning:** MongoDB 7/8 **requires AVX**. `vmgray01` must use CPU type `host` in Proxmox so AVX is exposed.

> **Warning:** All components are single-node and unsecured (no TLS, no auth on OpenSearch).

---

## Prerequisites

- [ ] Proxmox node with available resources (AVX-capable CPU)
- [ ] Network: static IP reserved or DHCP reservation configured
- [ ] ISO: Ubuntu Server 24.04 uploaded to Proxmox storage
- [ ] CPU type set to `host` on the VM (for AVX passthrough)

---

## 0 — Create the VM in Proxmox

Create a new VM on your Proxmox cluster:

- Name: `VMGRAY01`
- Guest OS: Linux → Ubuntu → 24.04 (or "Other 5.x+ Linux")
- CPU: 2 vCPUs minimum (4+ recommended)
- CPU Type: `host` (required for AVX → MongoDB 8)
- RAM: 4 GB minimum (8+ recommended)
- Disk: 40–80 GB on `local-lvm` (or your thin pool)
- BIOS / Machine: `OVMF (UEFI)` or `SeaBIOS` (cluster standard), `q35` if that's your default
- Network: VirtIO on `vmbr0` (flat LAN)
- QEMU Guest Agent: Enabled (check "Qemu Agent")

Install Ubuntu Server 24.04 (minimal) inside this VM.

> **Note:** Using `cpu: host` trades cross-node live migration for AVX compatibility. Document this in your inventory.

---

## 1 — Base OS Prep & Hostname

Log in as your normal sudo user on `vmgray01`.

### 1.1 Update packages & base tools

```bash
sudo apt update
sudo apt -y full-upgrade

# Helpful base packages
sudo apt install -y qemu-guest-agent vim curl ca-certificates gnupg lsb-release jq pwgen

# Enable guest agent for Proxmox
sudo systemctl enable --now qemu-guest-agent
```

Reboot if the kernel was updated:

```bash
sudo reboot
```

### 1.2 Ensure hostname and /etc/hosts are correct

After reboot:

```bash
hostnamectl set-hostname vmgray01

sudo sed -i 's/^127\.0\.1\.1.*/127.0.1.1\tvmgray01/' /etc/hosts
grep '^127\.0\.1\.1' /etc/hosts
hostnamectl
```

> **Note:** If you changed the hostname after install, log out and back in so your shell prompt matches.

### 1.3 Set server timezone

```bash
sudo timedatectl set-timezone America/Detroit
timedatectl
```

---

## 2 — AVX Guardrail for MongoDB

MongoDB 7/8 will crash with `Illegal instruction (core dumped)` if AVX isn't exposed to the VM. Ensure AVX is present on the host and visible inside `vmgray01`.

### 2.1 Check AVX support on the Proxmox host

Run on the Proxmox node (not inside the VM):

```bash
grep -o -w 'avx' /proc/cpuinfo | sort -u || echo "NO AVX ON HOST"
```

If nothing prints, your CPU doesn't support AVX → stop here and do **not** install MongoDB 7/8.

### 2.2 Ensure VM CPU type is `host` in Proxmox

GUI path (preferred):

- Proxmox Web UI → select `VMGRAY01`
- **Hardware** → double-click **Processor**
- Set **Type** = `host`
- OK, then reboot the VM if it's running

CLI alternative (on Proxmox host):

```bash
qm stop <VMID>
qm set <VMID> -cpu host
qm start <VMID>
```

### 2.3 Re-check AVX inside vmgray01

Back in the Ubuntu VM:

```bash
lscpu | grep -i flags
grep -m1 -o -w 'avx' /proc/cpuinfo || echo "NO AVX IN GUEST"
```

You must see `avx` listed. If not, MongoDB 8 will fail with `signal=ILL`.

---

## 3 — Install MongoDB 8.0

Follow Graylog's recommended MongoDB 8.0 repo (using the `jammy` list file).

### 3.1 Add MongoDB 8.0 repository

```bash
sudo apt-get install -y gnupg curl

curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg \
  --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/8.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

sudo apt-get update
```

### 3.2 Install MongoDB

```bash
sudo apt-get install -y mongodb-org
```

### 3.3 Enable & verify MongoDB

```bash
sudo systemctl daemon-reload
sudo systemctl enable mongod.service
sudo systemctl restart mongod.service
sudo systemctl status mongod.service --no-pager
```

### 3.4 Hold MongoDB version

```bash
sudo apt-mark hold mongodb-org
```

> **Note:** This prevents surprise major upgrades via `apt upgrade`.

---

## 4 — Install OpenSearch 2.x

OpenSearch is installed from the official 2.x APT repo. Graylog 7 supports up to 2.19.3. This homelab uses 2.x with the demo config disabled when needed.

### 4.1 Add OpenSearch 2.x repository

```bash
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | \
  sudo gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-keyring

echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring] https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/opensearch-2.x.list

sudo apt-get update
```

### 4.2 Install OpenSearch (with initial admin password)

Pick a strong initial admin password:

```bash
export OPENSEARCH_INITIAL_ADMIN_PASSWORD='StrongP@ssw0rd!'
```

Install OpenSearch (latest 2.x or pin a specific version):

```bash
sudo OPENSEARCH_INITIAL_ADMIN_PASSWORD="${OPENSEARCH_INITIAL_ADMIN_PASSWORD}" \
  apt-get -y install opensearch
```

(Optional) Hold OpenSearch:

```bash
sudo apt-mark hold opensearch
```

### 4.3 If install fails on demo configuration

If you see:

- `ERROR: Something went wrong during demo configuration installation`
- `dpkg` leaves `opensearch` in a broken state

Then:

1. Edit the postinst script:

```bash
sudo nano /var/lib/dpkg/info/opensearch.postinst
```

Find the block that calls `install_demo_configuration.sh`:

```bash
if [ "$OPENSEARCH_INSTALL_DEMO_CONFIG" != "false" ]; then
    /usr/share/opensearch/bin/install_demo_configuration.sh ...
fi
```

Comment out the entire block:

```bash
# if [ "$OPENSEARCH_INSTALL_DEMO_CONFIG" != "false" ]; then
#     /usr/share/opensearch/bin/install_demo_configuration.sh ...
# fi
```

Save and exit.

2. Re-run configuration:

```bash
sudo dpkg --configure -a
```

3. Confirm package state:

```bash
dpkg -l | grep opensearch
```

### 4.4 Minimal single-node OpenSearch config for Graylog

Overwrite `/etc/opensearch/opensearch.yml` with a minimal, unsecured single-node configuration:

```yaml
cluster.name: graylog
node.name: ${HOSTNAME}
path.data: /var/lib/opensearch
path.logs: /var/log/opensearch
discovery.type: single-node
network.host: 0.0.0.0
action.auto_create_index: false
plugins.security.disabled: true
```

```bash
sudo tee /etc/opensearch/opensearch.yml >/dev/null <<'EOF'
cluster.name: graylog
node.name: ${HOSTNAME}
path.data: /var/lib/opensearch
path.logs: /var/log/opensearch
discovery.type: single-node
network.host: 0.0.0.0
action.auto_create_index: false
plugins.security.disabled: true
EOF
```

### 4.5 Set JVM heap size

Edit `/etc/opensearch/jvm.options` (adjust `1g` to half of system RAM):

```bash
sudo sed -i 's/^-Xms.*/-Xms1g/' /etc/opensearch/jvm.options
sudo sed -i 's/^-Xmx.*/-Xmx1g/' /etc/opensearch/jvm.options
```

### 4.6 Kernel map count (required)

```bash
sudo sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf
```

### 4.7 Enable & start OpenSearch

```bash
sudo systemctl daemon-reload
sudo systemctl enable opensearch.service
sudo systemctl start opensearch.service
sudo systemctl status opensearch.service --no-pager
```

---

## 5 — Install Java 17

Required by Graylog 7:

```bash
sudo apt-get install -y openjdk-17-jre-headless
java -version
```

---

## 6 — Install Graylog 7

### 6.1 Add Graylog 7 repository

```bash
wget https://packages.graylog2.org/repo/packages/graylog-7.0-repository_latest.deb
sudo dpkg -i graylog-7.0-repository_latest.deb
sudo apt-get update
```

### 6.2 Install Graylog server

```bash
sudo apt-get install -y graylog-server
```

(Optional) Hold Graylog version:

```bash
sudo apt-mark hold graylog-server
```

---

## 7 — Configure Graylog

Configuration file: `/etc/graylog/server/server.conf`

### 7.1 Generate `password_secret`

```bash
pwgen -N 1 -s 96
```

Copy the output.

### 7.2 Generate `root_password_sha2` for the admin user

```bash
echo -n "Enter Graylog admin password: " && \
  head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1
```

- Enter the password you want to use for Graylog's `admin` user.
- Copy the resulting SHA256 hash.

### 7.3 Edit server.conf

```bash
sudo nano /etc/graylog/server/server.conf
```

Set or update the following keys (uncomment if needed):

```ini
password_secret = <output-from-pwgen>
root_password_sha2 = <sha256-hash-from-step-7.2>

# Bind Graylog web + API
http_bind_address = 0.0.0.0:9000

# (Optional but recommended) What Graylog advertises to clients
http_publish_uri = http://vmgray01.lan:9000/
# or: http_publish_uri = http://10.0.0.50:9000/

# Point at local OpenSearch (note: config key still says "elasticsearch")
elasticsearch_hosts = http://127.0.0.1:9200
```

Leave everything else at defaults unless you have specific needs. Save and exit.

> **Warning:** Set `elasticsearch_hosts` **before** starting Graylog the first time, or login can break.

---

## 8 — Service Enable & Start Order

Order matters: **MongoDB → OpenSearch → Graylog**.

### 8.1 MongoDB

```bash
sudo systemctl enable mongod.service
sudo systemctl restart mongod.service
sudo systemctl status mongod.service --no-pager
```

### 8.2 OpenSearch

```bash
sudo systemctl enable opensearch.service
sudo systemctl restart opensearch.service
sudo systemctl status opensearch.service --no-pager
```

### 8.3 Graylog server

```bash
sudo systemctl daemon-reload
sudo systemctl enable graylog-server.service
sudo systemctl start graylog-server.service
sudo systemctl status graylog-server.service --no-pager
```

---

## 9 — Validation

### 9.1 Check services with systemd

```bash
systemctl status mongod.service --no-pager
systemctl status opensearch.service --no-pager
systemctl status graylog-server.service --no-pager
```

All should show `active (running)`.

### 9.2 Verify OpenSearch HTTP endpoint

```bash
curl http://localhost:9200
```

Expect JSON with:

- `cluster_name` = `graylog`
- `version.number` = `2.x.x`

### 9.3 Verify listening ports

```bash
sudo ss -tulpn | grep -E '27017|9200|9000' || echo "Expected ports not listening"
```

Expected:

- `27017` → `mongod`
- `9200` → `opensearch`
- `9000` → `graylog-server` (Java process)

### 9.4 Check Graylog logs if it fails to start

```bash
sudo journalctl -u graylog-server.service -n 100 --no-pager
```

Look for:

- MongoDB connection errors
- OpenSearch connection errors
- Missing `password_secret` / `root_password_sha2`

### 9.5 Log into Graylog web UI

- URL: `http://<vmgray01-ip>:9000` (example: `http://10.0.0.50:9000`)
- Username: `admin`
- Password: (the password you used in step 7.2)

- [ ] Web UI accessible at `http://10.0.0.50:9000`
- [ ] Login succeeds with `admin` credentials
- [ ] Logs are clean (`journalctl -u graylog-server -n 20 --no-pager`)

### 9.6 Quick input smoke test (optional)

Create a UDP Syslog input in Graylog UI:

- `System → Inputs`
- Launch new input → `Syslog UDP`
- Node: `vmgray01`
- Port: `1514`
- Bind address: `0.0.0.0`
- Save & start

Send a test syslog from `vmgray01`:

```bash
logger -n 127.0.0.1 -P 1514 -t graylog-test "Hello from vmgray01"
```

Then in Graylog UI → `Search` → look for `Hello from vmgray01`.

---

## 10 — Post-Install

- [ ] Store Graylog admin password in Bitwarden
- [ ] Configure log forwarding from other VMs to Graylog
- [ ] Add `vmgray01` to OpenVAS scan targets
- [ ] Add `vmgray01` to Open-AudIT discovery

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| MongoDB crashes with `signal=ILL` / Illegal instruction | AVX not exposed to VM | Set VM CPU type to `host` in Proxmox; verify with `grep -m1 -o -w 'avx' /proc/cpuinfo`; restart mongod |
| OpenSearch `demo configuration` postinst failure | `install_demo_configuration.sh` errors during install | Comment out the demo config block in `/var/lib/dpkg/info/opensearch.postinst`, run `sudo dpkg --configure -a` |
| Graylog web UI not reachable on 9000 | `http_bind_address` not set, or missing secrets | Ensure `http_bind_address = 0.0.0.0:9000` and both `password_secret` / `root_password_sha2` are set; restart graylog-server |
| Graylog cannot connect to OpenSearch | Wrong or missing `elasticsearch_hosts` | Verify `curl http://localhost:9200` works; check `elasticsearch_hosts = http://127.0.0.1:9200` in `server.conf`; restart graylog-server |
| Graylog UI says "No data nodes found" | OpenSearch not running or not reachable | Check `sudo ss -tulpn \| grep 9200`; restart opensearch.service; verify config |

---

## Quick Reference

```bash
# Start order (matters!)
sudo systemctl restart mongod.service
sudo systemctl restart opensearch.service
sudo systemctl restart graylog-server.service

# View Graylog logs
sudo journalctl -u graylog-server.service -f

# View OpenSearch logs
sudo journalctl -u opensearch.service -f

# View MongoDB logs
sudo journalctl -u mongod.service -f

# Check all service statuses
systemctl status mongod.service opensearch.service graylog-server.service --no-pager

# Test OpenSearch endpoint
curl http://localhost:9200

# Check listening ports
sudo ss -tulpn | grep -E '27017|9200|9000'
```

---

## Quirks & Gotchas

- **AVX requirement:** MongoDB 7/8 absolutely requires AVX. Using `cpu: host` in Proxmox sacrifices cross-node live migration but is mandatory for this stack.
- **OpenSearch demo config breaks install:** The `install_demo_configuration.sh` script in the postinst can fail and leave the package in a broken `dpkg` state. Commenting it out and running `dpkg --configure -a` is the fix.
- **Config key mismatch:** Graylog 7 still uses `elasticsearch_hosts` in `server.conf` even though it connects to OpenSearch. Don't look for an `opensearch_hosts` key — it doesn't exist.
- **Start order dependency:** Services must start in order: MongoDB → OpenSearch → Graylog. If Graylog starts before its backends are ready, login can break.
- **JVM heap sizing:** Set OpenSearch heap (`-Xms`/`-Xmx`) to roughly half of system RAM. Don't exceed 50% or the OS will starve.
- **kernel vm.max_map_count:** OpenSearch requires `vm.max_map_count=262144`. Without it, OpenSearch will refuse to start.

---

*Last updated: 2025-07-14*
