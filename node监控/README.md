# 被探测主机
## 安装Prometheus
```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin node_exporter
cd /opt
sudo wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
sudo tar -xzf node_exporter-1.9.1.linux-amd64.tar.gz
sudo mv node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/
sudo rm -rf node_exporter-1.9.1.linux-amd64*
```
## 注册服务
```bash
sudo tee /etc/systemd/system/node_exporter.service <<'EOF'
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter \
  --web.listen-address=":9100" \
  --collector.textfile.directory="/var/lib/node_exporter/textfile_collector"

[Install]
WantedBy=default.target
EOF

sudo mkdir -p /var/lib/node_exporter/textfile_collector
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

## 验证：
```bash
curl localhost:9100/metrics
```

# 接收端主机
```bash
cd /opt
sudo wget https://github.com/prometheus/prometheus/releases/download/v3.6.0/prometheus-3.6.0.linux-amd64.tar.gz
sudo tar -xzf prometheus-*-linux-amd64.tar.gz
cd prometheus-*-linux-amd64
sudo mv prometheus promtool /usr/local/bin/
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo mv consoles console_libraries /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```
