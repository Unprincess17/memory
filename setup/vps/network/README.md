# Xray

```bash
#!/bin/bash
set -e

# 检查是否为 Root 用户
if [[ $EUID -ne 0 ]]; then
   echo "Error: This script must be run as root." 
   exit 1
fi

echo ">>> Installing dependencies..."
apt update -y && apt install -y curl unzip jq openssl

echo ">>> Installing Xray core..."
# 使用官方脚本安装/更新
bash <(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh) install

echo ">>> Generating REALITY keys and configuration..."
# 1. 动态生成 UUID
UUID=$(xray uuid)

# 2. 动态生成 X25519 密钥对
KEYS=$(xray x25519)
PRIVATE_KEY=$(echo "$KEYS" | sed -n 's/.*PrivateKey: \([^ ]*\).*/\1/p')
PUBLIC_KEY=$(echo "$KEYS" | sed -n 's/.*Password: \([^ ]*\).*/\1/p')

# 3. 动态生成 ShortID (8字节/16位十六进制)
SHORT_ID=$(openssl rand -hex 8)

# 4. 准备路径
mkdir -p /usr/local/etc/xray

# 5. 写入配置 (使用变量替换)
cat > /usr/local/etc/xray/config.json <<EOF
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "$UUID",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none",
        "fallbacks": [
          {
            "dest": 8080,
            "xver": 0
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.microsoft.com:443",
          "xver": 0,
          "serverNames": [
            "www.microsoft.com"
          ],
          "privateKey": "$PRIVATE_KEY",
          "shortIds": [
            "$SHORT_ID"
          ]
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    }
  ]
}
EOF

echo ">>> Opening firewall..."
if command -v ufw >/dev/null 2>&1; then
    ufw allow 443/tcp
    ufw allow 8080/tcp
fi

echo ">>> Restarting Xray..."
systemctl enable xray
systemctl restart xray

echo "--------------------------------------------------"
echo "REALITY Setup Complete!"
echo "Client Configuration Info:"
echo "UUID:       $UUID"
echo "Public Key: $PUBLIC_KEY"
echo "Short ID:   $SHORT_ID"
echo "Port:       443"
echo "SNI:        www.microsoft.com"
echo "--------------------------------------------------"
```

# Nginx
```bash
#!/bin/bash

# 1. Update package index and install prerequisite packages
apt update -y
apt install -y curl gnupg2 ca-certificates lsb-release

# 2. Set up the modern keyring directory
mkdir -p /etc/apt/keyrings

# 3. Download and dearmor the official Nginx signing key
curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor -o /etc/apt/keyrings/nginx-archive-keyring.gpg

# 4. Determine OS name (e.g., ubuntu or debian) and codename, then add the repository
OS_NAME=$(lsb_release -is | tr '[:upper:]' '[:lower:]')
OS_CODENAME=$(lsb_release -sc)

echo "deb [signed-by=/etc/apt/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/mainline/${OS_NAME} ${OS_CODENAME} nginx" | tee /etc/apt/sources.list.d/nginx.list

# 5. Set up APT pinning to prioritize Nginx official packages over system-provided versions
echo -e "Package: *\nPin: origin nginx.org\nPin-Priority: 900\n" | tee /etc/apt/preferences.d/99nginx

# 6. Update the package index with the new repository and install Nginx
apt update -y
apt install -y nginx
```

# Backup
```bash
# Create the user account for backup operations
useradd -m -s /bin/bash shufan

## Create the .ssh directory
mkdir -p /home/shufan/.ssh

## echo "$backupsvr-pubkey" > /home/shufan/.ssh/authorized_keys

## Set the correct strict permissions required by SSH daemon
chmod 700 /home/shufan/.ssh
chmod 600 /home/shufan/.ssh/authorized_keys

## Assign ownership to the shufan user
chown -R shufan:shufan /home/shufan/.ssh

# Grant passwordless sudo access to rsync for the shufan user
# 1. Write the rule directly to the drop-in file
echo "shufan ALL=(root) NOPASSWD: /usr/bin/rsync" | tee /etc/sudoers.d/99-shufan-rsync > /dev/null

# 2. Set the mandatory permissions (read-only for root and its group)
chmod 0440 /etc/sudoers.d/99-shufan-rsync

# 3. Safely validate the syntax to ensure sudo doesn't break
visudo -c -f /etc/sudoers.d/99-shufan-rsync
```


# Tailscale
```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Write tailscale nodes to /etc/hosts for easy access
TAILSCALE_NODES=$(tailscale status --json | jq -r '.Peer[] | "\(.HostName) \(.TailscaleIPs[])"')
echo "$TAILSCALE_NODES" | while read -r line; do
    HOSTNAME=$(echo "$line" | awk '{print $1}')
    IP=$(echo "$line" | awk '{print $2}')
    if ! grep -q "$HOSTNAME" /etc/hosts; then
        echo "$IP $HOSTNAME" >> /etc/hosts
    fi
done
```

# Observability
```bash
#!/bin/bash

# ==========================================
# Configuration Variables
# Change these before running the script
# ==========================================
TOTAL_GB=1000
API_KEY="your_secret_token"
PORT=8089

# Exit immediately if a command exits with a non-zero status
set -e

echo "Starting Traffic Monitor Setup..."

# 1. Auto-detect the primary network interface
INTERFACE=$(ip route get 1.1.1.1 | grep -oP 'dev \K\S+')
if [ -z "$INTERFACE" ]; then
    echo "Could not detect primary interface. Defaulting to eth0."
    INTERFACE="eth0"
else
    echo "Detected primary interface: $INTERFACE"
fi

# 2. Install dependencies
echo "Installing vnstat, nginx, and python3..."
apt-get update -y
apt-get install -y vnstat nginx python3

# Ensure vnstat is running and tracking the interface
systemctl enable --now vnstat
# Give vnstat a moment to initialize its database for the interface
sleep 2

# 3. Set up directories
echo "Configuring directories..."
mkdir -p /var/www/traffic
chown root:www-data /var/www/traffic
chmod 755 /var/www/traffic

# 4. Create the Python calculation script
echo "Writing Python script to /usr/local/bin/traffic_calc.py..."
cat << 'EOF' > /usr/local/bin/traffic_calc.py
#!/usr/bin/env python3

import json
import subprocess
import os
import sys

# These are injected by the bash script
TOTAL_GB = REPLACEME_GB
INTERFACE = "REPLACEME_IFACE"
OUTPUT_FILE = "/var/www/traffic/traffic.txt"

def update_traffic_file():
    try:
        raw = subprocess.check_output(
            ["vnstat", "-i", INTERFACE, "-m", "--json"],
            text=True
        )
        obj = json.loads(raw)

        try:
            month_list = obj["interfaces"][0]["traffic"]["month"]
        except (KeyError, IndexError):
            month_list = []

        if not month_list:
            output_data = f"0/{TOTAL_GB}"
        else:
            this_month = month_list[-1]
            rx = this_month["rx"]
            tx = this_month["tx"]

            used_gb = round((rx + tx) / 1024**3, 2)
            left_gb = round(TOTAL_GB - used_gb, 2)

            output_data = f"{used_gb}/{left_gb}"

    except Exception as e:
        output_data = f"Error/{str(e)}"
    
    tmp_file = OUTPUT_FILE + ".tmp"
    with open(tmp_file, "w", encoding="utf-8") as f:
        f.write(output_data)
    
    os.chmod(tmp_file, 0o644)
    os.rename(tmp_file, OUTPUT_FILE)

if __name__ == "__main__":
    update_traffic_file()
EOF

# Inject variables into the Python script
sed -i "s/REPLACEME_GB/$TOTAL_GB/g" /usr/local/bin/traffic_calc.py
sed -i "s/REPLACEME_IFACE/$INTERFACE/g" /usr/local/bin/traffic_calc.py
chmod +x /usr/local/bin/traffic_calc.py

# Run it once to generate the initial file
/usr/local/bin/traffic_calc.py

# 5. Configure Cron Job
echo "Setting up cron job..."
# Remove existing entry if it exists to avoid duplicates, then add the new one
(crontab -l 2>/dev/null | grep -v "/usr/local/bin/traffic_calc.py"; echo "*/5 * * * * /usr/bin/python3 /usr/local/bin/traffic_calc.py") | crontab -

# 6. Configure Nginx
echo "Configuring Nginx..."
cat << EOF > /etc/nginx/sites-available/traffic_monitor
server {
    listen $PORT;
    server_name _;

    location /traffic {
        if (\$http_x_api_key != "$API_KEY") {
            return 403;
        }
        default_type text/plain;
        alias /var/www/traffic/traffic.txt;
    }

    location / {
        return 404;
    }
}
EOF

# Enable the site and clean up default nginx site if desired
ln -sf /etc/nginx/sites-available/traffic_monitor /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default

# Test Nginx config and restart
nginx -t
systemctl restart nginx
systemctl enable nginx

# Final Output
echo "=========================================="
echo "Setup Complete!"
echo "Interface monitored: $INTERFACE"
echo "Port: $PORT"
echo "Test command: curl -H \"X-API-KEY: $API_KEY\" http://localhost:$PORT/traffic"
echo "=========================================="
```

# fail2ban
## fail2ban for frps
```bash
apt-get update
apt-get install fail2ban nftables python3-systemd
fail2ban-client --version
```

fail2ban setup script
```bash
#!/usr/bin/env bash
# Generate a Fail2ban jail for one TCP SSH proxy served by host-based frps.
# Usage: sudo bash setup-frp-jail.sh JAIL PROXY PUBLIC_PORT [FRPS_UNIT]
set -euo pipefail

if (( $# < 3 || $# > 4 )); then
  echo "Usage: $0 JAIL PROXY PUBLIC_PORT [FRPS_UNIT]" >&2
  exit 2
fi

frp_jail=$1
frp_proxy=$2
frp_port=$3
frp_unit=${4:-frps.service}
frp_config_dir=${FRP_F2B_CONFIG_DIR:-/etc/fail2ban}

if [[ ! $frp_jail =~ ^[A-Za-z0-9][A-Za-z0-9_-]*$ ]]; then
  echo "Jail name must contain only letters, digits, underscores or hyphens." >&2
  exit 2
fi
if [[ ! $frp_proxy =~ ^[A-Za-z0-9][A-Za-z0-9_.-]*$ ]]; then
  echo "Proxy name must contain only letters, digits, dots, _ or -." >&2
  exit 2
fi
if [[ ! $frp_port =~ ^[0-9]{1,5}$ ]]; then
  echo "PUBLIC_PORT must be an integer between 1 and 65535." >&2
  exit 2
fi
frp_port=$((10#$frp_port))
if (( frp_port < 1 || frp_port > 65535 )); then
  echo "PUBLIC_PORT must be between 1 and 65535." >&2
  exit 2
fi
if [[ ! $frp_unit =~ ^[A-Za-z0-9_.@-]+\.service$ ]]; then
  echo "FRPS_UNIT must be a systemd service name, such as frps.service." >&2
  exit 2
fi

for frp_required in filter.d/common.conf action.d/nftables.conf; do
  if [[ ! -f "$frp_config_dir/$frp_required" ]]; then
    echo "Missing $frp_config_dir/$frp_required; install Fail2ban first." >&2
    exit 1
  fi
done

frp_filter="$frp_config_dir/filter.d/$frp_jail.conf"
frp_jail_file="$frp_config_dir/jail.d/$frp_jail.local"
if [[ -e "$frp_config_dir/filter.d/$frp_jail.local" ]]; then
  echo "A filter override already exists: filter.d/$frp_jail.local" >&2
  echo "Review that override before using this generator for this jail." >&2
  exit 1
fi

frp_stage_dir=$(mktemp -d "${TMPDIR:-/tmp}/frp-f2b.XXXXXX")
trap 'rm -rf -- "$frp_stage_dir"' EXIT
frp_proxy_regex=${frp_proxy//./\\.}

cat > "$frp_stage_dir/filter.conf" <<EOF
[INCLUDES]
before = common.conf

[DEFAULT]
_daemon = frps
_proxy = $frp_proxy_regex
_esc = (?:\x1b\[[0-9;]*m)*
_ts = (?:\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}(?:\.\d+)?\s+)?
_meta = \[I\]\s+\[proxy/proxy\.go:\d+\]\s+\[[0-9a-f]+\]\s+

[Definition]
prefregex = ^%(__prefix_line)s%(_esc)s%(_ts)s%(_meta)s<F-CONTENT>.+</F-CONTENT>$
failregex = ^\[%(_proxy)s\]\s+get a user connection \[<ADDR>:\d+\]\s*%(_esc)s\s*$
ignoreregex =
journalmatch = _SYSTEMD_UNIT=$frp_unit
EOF

cat > "$frp_stage_dir/jail.local" <<EOF
[$frp_jail]
enabled = true
backend = systemd
filter = $frp_jail
port = $frp_port
protocol = tcp
banaction = nftables[type=multiport]
usedns = no
findtime = 2m
maxretry = 12
bantime = 10m
ignoreip = 127.0.0.1/8 ::1
EOF

mkdir -p "$frp_config_dir/filter.d" "$frp_config_dir/jail.d"
if [[ -e "$frp_filter" || -e "$frp_jail_file" ]]; then
  mkdir -p "$frp_config_dir/frp-backups"
  frp_backup_dir=$(mktemp -d "$frp_config_dir/frp-backups/$frp_jail.XXXXXX")
  if [[ -e "$frp_filter" ]]; then
    cp -a -- "$frp_filter" "$frp_backup_dir/filter.conf"
  fi
  if [[ -e "$frp_jail_file" ]]; then
    cp -a -- "$frp_jail_file" "$frp_backup_dir/jail.local"
  fi
  printf 'Previous files saved to %s\n' "$frp_backup_dir"
fi

install -m 0644 "$frp_stage_dir/filter.conf" "$frp_filter"
install -m 0644 "$frp_stage_dir/jail.local" "$frp_jail_file"
printf 'Wrote %s and %s\n' "$frp_filter" "$frp_jail_file"
printf 'Proxy %s: TCP %s. Validate a recent log sample, then run fail2ban-client reload.\n' \
  "$frp_proxy" "$frp_port"
```

Generate the two example jails:

```bash
bash setup-frp-jail.sh frp-um139 um139-ssh 9022
bash setup-frp-jail.sh frp-pve pve-ssh 8822
```

The optional fourth argument is the systemd unit. For a different instance:

```bash
bash setup-frp-jail.sh frp-lab lab-ssh 9922 frps@public.service
```

The generated files for UM139 are:

```text
/etc/fail2ban/filter.d/frp-um139.conf
/etc/fail2ban/jail.d/frp-um139.local
```

Test **the actual journal** before activation. This checks matching without
issuing bans; it can scan the retained history for the selected service:

```bash
frp_sample=$(mktemp /tmp/frps-sample.XXXXXX.log)
journalctl -u frps.service --since "10 minutes ago" -n 1000 \
  --no-pager --all -o short-iso > "$frp_sample"

fail2ban-regex "$frp_sample" \
  '/etc/fail2ban/filter.d/frp-um139.conf[logtype=file]'

rm -f -- "$frp_sample"

fail2ban-client -t
```

Expect each filter to match its own proxy's connection events. Other proxy
entries appearing as missed is normal. If a new VPS has no connection events,
make one normal SSH login through its public proxy port, then rerun the test.
An SSH login can succeed and still count as one event for this rate rule.

`fail2ban-client -t` alone does not establish that a filter matches real logs.
Resolve compilation errors or unexpected zero matches before activation.

Once the checks pass, start Fail2ban and explicitly reload these jails:

```bash
systemctl enable --now fail2ban
fail2ban-client reload
```

These commands reload the Fail2ban jails; they do not require restarting the
FRP or SSH services. Check that the running jail loaded the replacement:

```bash
fail2ban-client get frp-um139 prefregex
fail2ban-client get frp-um139 failregex
fail2ban-client get frp-um139 ignoreip
fail2ban-client status frp-um139
fail2ban-client status frp-pve
```

For an accidental ban, replace `IP_ADDRESS` with the actual address:

```bash
fail2ban-client set frp-um139 unbanip IP_ADDRESS
fail2ban-client set frp-pve unbanip IP_ADDRESS
```

Check the returned log target for filter exceptions or firewall errors. A
common file target is `/var/log/fail2ban.log`:

```bash
tail -n 100 /var/log/fail2ban.log
```

If the configured target is the systemd journal, use:

```bash
journalctl -u fail2ban.service --since "10 minutes ago" --no-pager
```
