The goal is to centralize the logs and to be able to query them and to do alerting.

# Installation

1. Download the binary release
```bash
wget https://github.com/grafana/loki/releases/download/v1.2.0/loki-linux-amd64.zip
```

2. Extract the zip
```bash
unzip loki-linux-amd64.zip
```

3. Make Logi globally available
```bash
mv loki-linux-amd64 /usr/local/bin/
```

4. Create Loki folders
```bash
mkdir /etc/loki
mkdir /var/lib/loki
```

5. Create Loki user
```bash
useradd --system --home /etc/loki --shell /bin/false vault
```

6. Set Loki folders owner
```bash
chown -R vault:vault /etc/loki /var/lib/loki
```

7. Add System Unit
```bash
cat <<EOF | tee /etc/systemd/system/loki.service
[Unit]
Description="Loki - Like Promotheus for logs"
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/loki/loki.yaml

[Service]
User=loki
Group=loki
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/bin/loki -config.file=/etc/loki/loki.yaml
ExecReload=/bin/kill --signal HUP 
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitBurst=3
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

8. Create Loki config file
```bash
cat <<EOF | tee /etc/loki/loki.yaml
server: 
    http_listen_address: ${IP}
    http_listen_port: 80

ingester_client:
    pool_config:
        health_check_ingesters: true

ingester:
    lifecycler:
        ring: 
            kvstore: "inmemory"

storage:
    filesystem:
        directory: /var/lib/loki

cache:
    enable_fifocache: false
EOF
```

9. Reload systemd daemon
```bash
systemctl daemon-reload
```

10. Open Firewall port
```bash
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
``` 

11. Enable and start Loki
```bash
systemctl enable --now loki
```