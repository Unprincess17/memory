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
# Install Nginx from official repository
apt update -y
apt install -y curl gnupg2 ca-certificates lsb-release
echo "deb http://nginx.org/packages/mainline/$(lsb_release -sc) nginx" | tee /etc/apt/sources.list.d/nginx.list
curl -fsSL https://nginx.org/keys/nginx_signing.key | apt-key add -
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
