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
PUBLIC_KEY=$(echo "$KEYS" | sed -n 's/.*PublicKey: \([^ ]*\).*/\1/p')

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
