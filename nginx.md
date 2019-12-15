The goal is to have a reverse proxy that serve the e-commerce website on the internet. It will do load balancing between two web servers.

# Installation on CentOS7

1. Add repo
```bash
cat <<EOF | tee /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=https://nginx.org/packages/mainline/centos/7.7.1908/$basearch/
gpgcheck=0
enabled=1
EOF
```

2. Update the system
```bash
yum update
```

3. Install the package
```bash
yum install nginx
```

# Setup

1. Set vars
```bash
export DOMAIN=alphapar.fr
```

2. Move default config
```bash
mv /etc/nginx.conf /etc/nginx.conf.old
```

2. Setup Nginx
```bash
cat <<EOF | tee /etc/nginx.conf
user       www www;  ## Default: nobody
worker_processes  5;  ## Default: 1
error_log  logs/error.log;
pid        logs/nginx.pid;
worker_rlimit_nofile 8192;

events {
    worker_connections  4096;  ## Default: 1024
}

http {
  ssl_session_cache   shared:SSL:10m;
  ssl_session_timeout 10m;

  upstream applications {
    server 10.0.50.3;
    server 10.0.50.4;
  }

  server {
    listen          80;  
    server_name     ${DOMAIN} www.${DOMAIN};
    return          301 https://$server_name$request_uri; #Redirection; 
  }

  server {
    listen 443 ssl http2;
    server_name ${DOMAIN};
    access_log      logs/big.server.access.log main;

    location /images/ {
      # dont forget to create the folder  
      root /data;
    }

    location / {
      proxy_pass      https://applications;
      health_check;
    }

    ssl on;
    ssl_protocols [TLSv1.3];
    ssl_certificate /etc/nginx/ssl/${DOMAIN}/server.crt;
    ssl_certificate_key /etc/nginx/ssl/${DOMAIN}/server.key;
    ssl_trusted_certificate /etc/nginx/ssl/${DOMAIN}/ca-certs.pem;
  }
}
EOF
```

3. Enable and start Nginx
```bash
systemctl enable --now nginx
```

# ToDo

[ ] Enable caching