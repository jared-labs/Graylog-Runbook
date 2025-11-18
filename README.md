# Deploy VMGRAY01 — Graylog 7 + OpenSearch 2.x + MongoDB (Ubuntu 24.04)

Single-VM deployment of Graylog 7 with self-managed OpenSearch and MongoDB on Ubuntu Server 24.04, running as `vmgray01` on Proxmox. Copy/paste friendly.

> Guardrail: `vmgray01` is log stack only (MongoDB + OpenSearch + Graylog). No Prometheus/Grafana/etc. here.
> Guardrail: MongoDB 7/8 **requires AVX**. `vmgray01` must use CPU type `host` in Proxmox so AVX is exposed.
> Guardrail: All components are single-node and unsecured (no TLS, no auth on OpenSearch).

* * *

## 0) VM definition in Proxmox (before OS install)

Create a new VM on your Proxmox cluster:

- Name: `VMGRAY01`
- Guest OS: Linux → Ubuntu → 24.04 (or “Other 5.x+ Linux”)
- CPU: 2 vCPUs minimum (4+ recommended)
- CPU Type: `host` (required for AVX → MongoDB 8)
- RAM: 4 GB minimum (8+ recommended)
- Disk: 40–80 GB on `local-lvm` (or your thin pool)
- BIOS / Machine: `OVMF (UEFI)` or `SeaBIOS` (cluster standard), `q35` if that’s your default
- Network: VirtIO on `vmbr0` (flat LAN)
- QEMU Guest Agent: Enabled (check “Qemu Agent”)

Install Ubuntu Server 24.04 (minimal) inside this VM.

> Note: Using `cpu: host` trades cross-node live migration for AVX compatibility. Document this in your inventory.

* * *

## 1) Base OS prep & hostname (Ubuntu 24.04)

Log in as your normal sudo user on `vmgray01`.

### 1.1 Update packages & base tools

    sudo apt update
    sudo apt -y full-upgrade

    # Helpful base packages
    sudo apt install -y qemu-guest-agent vim curl ca-certificates gnupg lsb-release jq pwgen

    # Enable guest agent for Proxmox
    sudo systemctl enable --now qemu-guest-agent

Reboot if the kernel was updated:

    sudo reboot

### 1.2 Ensure hostname and /etc/hosts are correct

After reboot:

    hostnamectl set-hostname vmgray01

Fix the `127.0.1.1` line in `/etc/hosts` so it matches the hostname:

    sudo sed -i 's/^127\.0\.1\.1.*/127.0.1.1\tvmgray01/' /etc/hosts
    grep '^127\.0\.1\.1' /etc/hosts
    hostnamectl

> If you changed the hostname after install, log out and back in so your shell prompt matches.

### 1.3 Set server timezone (example: America/Detroit)

    sudo timedatectl set-timezone America/Detroit
    timedatectl

* * *

## 2) AVX guardrail for MongoDB (Proxmox host + vmgray01)

MongoDB 7/8 will crash with `Illegal instruction (core dumped)` if AVX isn’t exposed to the VM. Ensure AVX is present on the host and visible inside `vmgray01`.

### 2.1 Check AVX support on the Proxmox host

Run on the Proxmox node (not inside the VM):

    grep -o -w 'avx' /proc/cpuinfo | sort -u || echo "NO AVX ON HOST"

If nothing prints, your CPU doesn’t support AVX → stop here and do **not** install MongoDB 7/8.

### 2.2 Ensure VM CPU type is `host` in Proxmox

GUI path (preferred):

- Proxmox Web UI → select `VMGRAY01`
- **Hardware** → double-click **Processor**
- Set **Type** = `host`
- OK, then reboot the VM if it’s running

CLI alternative (on Proxmox host):

    qm stop <VMID>
    qm set <VMID> -cpu host
    qm start <VMID>

### 2.3 Re-check AVX inside `vmgray01`

Back in the Ubuntu VM:

    lscpu | grep -i flags
    grep -m1 -o -w 'avx' /proc/cpuinfo || echo "NO AVX IN GUEST"

You must see `avx` listed. If not, MongoDB 8 will fail with `signal=ILL`.

* * *

## 3) Install MongoDB 8.0 on Ubuntu 24.04

Follow Graylog’s recommended MongoDB 8.0 repo (using the `jammy` list file).

### 3.1 Add MongoDB 8.0 repository

    sudo apt-get install -y gnupg curl

    curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
      sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg \
      --dearmor

    echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/8.0 multiverse" | \
      sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

    sudo apt-get update

### 3.2 Install MongoDB

    sudo apt-get install -y mongodb-org

### 3.3 Enable & verify MongoDB

    sudo systemctl daemon-reload
    sudo systemctl enable mongod.service
    sudo systemctl restart mongod.service
    sudo systemctl status mongod.service --no-pager

### 3.4 Hold MongoDB version (avoid surprise major upgrades)

    sudo apt-mark hold mongodb-org

* * *

## 4) Install OpenSearch 2.x and handle demo config postinst failure

OpenSearch is installed from the official 2.x APT repo. Graylog 7 supports up to 2.19.3, but this homelab uses 2.x with the demo config disabled when needed.

### 4.1 Add OpenSearch 2.x repository

    curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | \
      sudo gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-keyring

    echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring] https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | \
      sudo tee /etc/apt/sources.list.d/opensearch-2.x.list

    sudo apt-get update

### 4.2 Install OpenSearch (with initial admin password)

Pick a strong initial admin password:

    export OPENSEARCH_INITIAL_ADMIN_PASSWORD='StrongP@ssw0rd!'

Install OpenSearch (latest 2.x or pin a specific version):

    sudo OPENSEARCH_INITIAL_ADMIN_PASSWORD="${OPENSEARCH_INITIAL_ADMIN_PASSWORD}" \
      apt-get -y install opensearch

(Optional) Hold OpenSearch:

    sudo apt-mark hold opensearch

### 4.3 If install fails on demo configuration (real-world quirk)

If you see:

- `ERROR: Something went wrong during demo configuration installation`
- `dpkg` leaves `opensearch` in a broken state

Then:

1) Edit the postinst script:

    sudo nano /var/lib/dpkg/info/opensearch.postinst

Find the block that calls `install_demo_configuration.sh`:

    if [ "$OPENSEARCH_INSTALL_DEMO_CONFIG" != "false" ]; then
        /usr/share/opensearch/bin/install_demo_configuration.sh ...
    fi

Comment out the entire block:

    # if [ "$OPENSEARCH_INSTALL_DEMO_CONFIG" != "false" ]; then
    #     /usr/share/opensearch/bin/install_demo_configuration.sh ...
    # fi

Save and exit.

2) Re-run configuration:

    sudo dpkg --configure -a

3) Confirm package state:

    dpkg -l | grep opensearch

### 4.4 Minimal single-node OpenSearch config for Graylog

Overwrite `/etc/opensearch/opensearch.yml` with a minimal, unsecured single-node configuration:

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

### 4.5 Set JVM heap size (example: 1 GB heap)

Edit `/etc/opensearch/jvm.options`:

    sudo sed -i 's/^-Xms.*/-Xms1g/' /etc/opensearch/jvm.options
    sudo sed -i 's/^-Xmx.*/-Xmx1g/' /etc/opensearch/jvm.options

Adjust `1g` to half of system RAM.

### 4.6 Kernel map count (required)

    sudo sysctl -w vm.max_map_count=262144
    echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf

### 4.7 Enable & start OpenSearch

    sudo systemctl daemon-reload
    sudo systemctl enable opensearch.service
    sudo systemctl start opensearch.service
    sudo systemctl status opensearch.service --no-pager

* * *

## 5) Install Java 17 (required by Graylog 7)

    sudo apt-get install -y openjdk-17-jre-headless
    java -version

* * *

## 6) Install Graylog 7 (server only)

### 6.1 Add Graylog 7 repository

    wget https://packages.graylog2.org/repo/packages/graylog-7.0-repository_latest.deb
    sudo dpkg -i graylog-7.0-repository_latest.deb
    sudo apt-get update

### 6.2 Install Graylog server

    sudo apt-get install -y graylog-server

(Optional) Hold Graylog version:

    sudo apt-mark hold graylog-server

* * *

## 7) Configure Graylog (`/etc/graylog/server/server.conf`)

### 7.1 Generate `password_secret`

Use `pwgen` to generate a long random string:

    pwgen -N 1 -s 96

Copy the output.

### 7.2 Generate `root_password_sha2` for the `admin` user

    echo -n "Enter Graylog admin password: " && \
      head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1

- Enter the password you want to use for Graylog’s `admin` user.
- Copy the resulting SHA256 hash.

### 7.3 Edit `server.conf`

Open the config:

    sudo nano /etc/graylog/server/server.conf

Set or update the following keys (uncomment if needed):

    password_secret = <output-from-pwgen>
    root_password_sha2 = <sha256-hash-from-step-7.2>

    # Bind Graylog web + API
    http_bind_address = 0.0.0.0:9000

    # (Optional but recommended) What Graylog advertises to clients
    http_publish_uri = http://vmgray01.lan:9000/
    # or: http_publish_uri = http://10.0.0.50:9000/

Point Graylog at local OpenSearch (note: still named `elasticsearch_hosts` in config):

    elasticsearch_hosts = http://127.0.0.1:9200

Leave everything else at defaults unless you have specific needs. Save and exit.

> Important: Set `elasticsearch_hosts` **before** starting Graylog the first time, or login can break.

* * *

## 8) Service enable / start order (MongoDB → OpenSearch → Graylog)

Order matters: MongoDB first, then OpenSearch, then Graylog.

### 8.1 MongoDB

    sudo systemctl enable mongod.service
    sudo systemctl restart mongod.service
    sudo systemctl status mongod.service --no-pager

### 8.2 OpenSearch

    sudo systemctl enable opensearch.service
    sudo systemctl restart opensearch.service
    sudo systemctl status opensearch.service --no-pager

### 8.3 Graylog server

    sudo systemctl daemon-reload
    sudo systemctl enable graylog-server.service
    sudo systemctl start graylog-server.service
    sudo systemctl status graylog-server.service --no-pager

* * *

## 9) Verification & smoke tests

### 9.1 Check services with systemd

    systemctl status mongod.service --no-pager
    systemctl status opensearch.service --no-pager
    systemctl status graylog-server.service --no-pager

All should show `active (running)`.

### 9.2 Verify OpenSearch HTTP endpoint

    curl http://localhost:9200

Expect JSON similar to:

- `cluster_name` = `graylog`
- `version.number` = `2.x.x`

### 9.3 Verify listening ports

    sudo ss -tulpn | grep -E '27017|9200|9000' || echo "Expected ports not listening"

You should see:

- `27017` → `mongod`
- `9200` → `opensearch`
- `9000` → `graylog-server` (Java process)

### 9.4 Check Graylog logs if it fails to start

    sudo journalctl -u graylog-server.service -n 100 --no-pager

Look for:

- MongoDB connection errors
- OpenSearch connection errors
- Missing `password_secret` / `root_password_sha2`

### 9.5 Log into Graylog web UI

From your workstation browser:

- URL: `http://<vmgray01-ip>:9000`  
  Example: `http://10.0.0.50:9000`
- Username: `admin`
- Password: (the password you used in step 7.2)

You should land on the Graylog web interface.

### 9.6 Quick input smoke test (optional)

Create a UDP Syslog input and send a test message.

In Graylog UI:

- `System → Inputs`
- Launch new input → `Syslog UDP`
- Node: `vmgray01`
- Port: `1514`
- Bind address: `0.0.0.0`
- Save & start

Send a test syslog from `vmgray01`:

    logger -n 127.0.0.1 -P 1514 -t graylog-test "Hello from vmgray01"

Then in Graylog UI:

- `Search` → look for `Hello from vmgray01`.

* * *

## 10) Troubleshooting (real-world quirks)

### 10.1 MongoDB crashes with `signal=ILL` / Illegal instruction

Symptom:

- `systemctl status mongod` shows `code=dumped, signal=ILL`.

Fix:

1. On Proxmox host, confirm AVX:

        grep -o -w 'avx' /proc/cpuinfo | sort -u

2. Ensure VM CPU type is `host`:

        qm stop <VMID>
        qm set <VMID> -cpu host
        qm start <VMID>

3. Inside `vmgray01`, re-check AVX:

        grep -m1 -o -w 'avx' /proc/cpuinfo || echo "NO AVX IN GUEST"

4. Restart MongoDB:

        sudo systemctl restart mongod.service

### 10.2 OpenSearch `demo configuration` postinst failure

Symptom during `apt-get install opensearch`:

- Error: `Something went wrong during demo configuration installation`
- `dpkg` leaves `opensearch` unconfigured.

Fix:

1. Edit `/var/lib/dpkg/info/opensearch.postinst` and comment out the `install_demo_configuration.sh` block:

        sudo nano /var/lib/dpkg/info/opensearch.postinst

2. Re-run:

        sudo dpkg --configure -a

3. Overwrite `/etc/opensearch/opensearch.yml` with a minimal single-node config (see section 4.4).

4. Ensure `vm.max_map_count` is set and OpenSearch is started (sections 4.6–4.7).

### 10.3 Graylog web UI not reachable on 9000

Checks:

    sudo ss -tulpn | grep 9000 || echo "No listener on 9000"
    sudo journalctl -u graylog-server.service -n 100 --no-pager

Common fixes:

- `http_bind_address` not set → ensure `http_bind_address = 0.0.0.0:9000`.
- `password_secret` or `root_password_sha2` missing → set both and restart:

        sudo systemctl restart graylog-server.service

### 10.4 Graylog cannot connect to OpenSearch

Symptoms:

- Graylog logs show errors about OpenSearch / data node.
- UI setup screen says “No data nodes found”.

Checks:

    curl http://localhost:9200
    sudo ss -tulpn | grep 9200 || echo "No OpenSearch listener"

Verify `elasticsearch_hosts` in `server.conf`:

    grep '^elasticsearch_hosts' /etc/graylog/server/server.conf

It must include the correct URI:

    elasticsearch_hosts = http://127.0.0.1:9200

Restart Graylog:

    sudo systemctl restart graylog-server.service

* * *

## 11) Summary

- `vmgray01` runs MongoDB 8, OpenSearch 2.x, and Graylog 7 on Ubuntu Server 24.04.
- Proxmox CPU type `host` is required so AVX is visible and MongoDB doesn’t crash.
- OpenSearch demo configuration is disabled at the `opensearch.postinst` level when it breaks installation.
- OpenSearch is configured as a minimal, unsecured single-node backend, and Graylog uses it via `elasticsearch_hosts`.
- Service start order is **MongoDB → OpenSearch → Graylog**, all enabled via `systemctl enable`.
- Verification includes `systemctl status`, `curl http://localhost:9200`, checking port `9000`, and logging into the Graylog UI.
