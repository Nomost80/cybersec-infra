1. Create www folder
```bash
sudo mkdir /srv/www
```

2. Create www user
```bash
sudo useradd --system --home /srv/www --shell /bin/false www
```

3. Set www folder owner
```bash
sudo chown -R www:www /srv/www
```

4. Install required packages
```bash
sudo yum install -y git nodejs
```

5. Open port with firewalld
```bash
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

6. Clone the project
```bash
cd /srv/www
git clone ....
```

7. Install dependencies
```bash
cd ./eshop/app
npm install
```

8. Create a systemd unit
```bash
cat <<EOF | sudo tee /etc/systemd/system/eshop.service
[Unit]
Description="eshop web application"
Requires=network-online.target
After=network-online.target

[Service]
User=www
Group=www
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/bin/node /srv/www/eshop/app.js
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
EOF
```

9. Enable eshop service
```bash
sudo systemctl enable --now eshop
```