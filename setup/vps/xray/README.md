#!/bin/bash
set -e

echo ">>> Installing dependencies..."
apt update -y && apt install -y curl unzip jq

echo ">>> Installing Xray core..."
bash <(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh) install

echo ">>> Creating Xray config..."
mkdir -p /usr/local/etc/xray

cat >/usr/local/etc/xray/config.json <<'EOF'
{
  "log": { "loglevel": "warning" },
  "inbounds": [
    {
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "66b4c251-1aa0-4f20-9a3b-1f5c32a4c8a9",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none",
        "fallbacks": [
          {
            "dest": "127.0.0.1:8080",
            "xver": 1
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.lovelive-anime.jp:443",
          "xver": 0,
          "serverNames": [ "www.lovelive-anime.jp" ],
          "privateKey": "2O7fw_JC8zFb3xi7mj86Zg1BgAA-FKun5syrl1C6q0w",
          "publicKey": "3u-wBdCZz8lILwhGLj19bkr1uOporgXIz7PoQNqu4nY",
          "maxTimeDiff": 70000,
          "shortIds": [ "0123456789abcdef" ]
        }
      }
    }
  ],
  "outbounds": [
    { "protocol": "freedom", "tag": "direct" }
  ]
}
EOF

echo ">>> Opening firewall..."
ufw allow 443/tcp || true
ufw allow 8080/tcp || true

echo ">>> Enabling and starting Xray..."
systemctl enable xray
systemctl restart xray
systemctl status xray --no-pager
