The goal is to have a reverse proxy that serve the e-commerce website on the internet. It will do load balancing between two web servers.

# Installation on CentOS7

1. Add repo
```bash
cat <<EOF | tee /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=https://nginx.org/packages/mainline/centos/7/$basearch/
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

4. Create a www user
```bash
useradd --system --home /etc/nginx --shell /bin/false www
```

5. Set Nginx folders owner
```bash
chown -R www:www /etc/nginx /var/log/nginx
``` 

# Setup

1. Move default config
```bash
mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.default
```

2. After one of the following roles
```bash
firewall-cmd add-service={http,https} --permanent
firewall-cmd --reload
systemctl enable --now nginx
```

## Load Balancer and Reverse Proxy

```bash
export DOMAIN=alphapar.fr

cat <<EOF | tee /etc/nginx.conf
user   www www;  ## Default: nobody
worker_processes   5;  ## Default: 1

error_log   /var/log/nginx/error.log warn;
pid         /var/log/nginx/nginx.pid;

worker_rlimit_nofile   8192;

events {
    worker_connections   4096;  ## Default: 1024
}

http {
  include   /etc/nginx/mime.types;
  include   /etc/nginx/confd/*conf;   

  default_type   application/json;

  log_format   main  '$remote_addr - $remote_user [$time_local] "$request" '
                   '$status $body_bytes_sent "$http_referer" '
                   '"$http_user_agent" "$http_x_forwarded_for"';

  access_log   /var/log/nginx/access.log  main;

  keepalive_timeout   65;

  ssl                         on;
  ssl_prefer_server_ciphers   on;
  ssl_protocols               [TLSv1.3];
  ssl_session_cache           shared:SSL:10m;
  ssl_session_timeout         10m;

  ssl_certificate           /etc/nginx/ssl/${DOMAIN}/chain.crt;
  ssl_certificate_key       /etc/nginx/ssl/${DOMAIN}/server.key;

  upstream applications {
    server 10.0.50.3;
    server 10.0.50.4;
  }

  server {
    listen        80;  
    server_name   ${DOMAIN} www.${DOMAIN};
    return        301 https://$server_name$request_uri; #Redirection; 
  }

  server {
    listen        443 ssl http2;
    server_name   ${IP} www.${IP}; 

    access_log   /var/log/nginx/access.log main;

    location / {
      proxy_pass   https://applications;
      health_check;
    }
  }
}
EOF
```

## Web Server

```bash
export IP=xx.xx.xx.xx

cat <<EOF | tee /etc/nginx.conf
user   www www;  ## Default: nobody
worker_processes   5;  ## Default: 1

error_log   /var/log/nginx/error.log warn;
pid         /var/log/nginx/nginx.pid;

worker_rlimit_nofile   8192;

events {
    worker_connections   4096;  ## Default: 1024
}

http {
  include   /etc/nginx/mime.types;
  include   /etc/nginx/confd/*conf;   

  default_type   application/json;

  log_format   main  '$remote_addr - $remote_user [$time_local] "$request" '
                   '$status $body_bytes_sent "$http_referer" '
                   '"$http_user_agent" "$http_x_forwarded_for"';

  access_log   /var/log/nginx/access.log  main;

  keepalive_timeout   65;

  ssl                         on;
  ssl_prefer_server_ciphers   on;
  ssl_protocols               [TLSv1.3];
  ssl_session_cache           shared:SSL:10m;
  ssl_session_timeout         10m;

  ssl_certificate           /etc/nginx/ssl/chain.crt;
  ssl_certificate_key       /etc/nginx/ssl/server.key;

  server {
    listen        80;  
    server_name   ${IP} www.${IP};
    return        301 https://$server_name$request_uri; #Redirection; 
  }

  server {
    listen        443 ssl http2;
    server_name   ${IP} www.${IP}; 

    access_log   /var/log/nginx/access.log main;

    location / {
      proxy_pass   http://localhost;
      health_check;
    }
  }
}
EOF
```

# ToDo

[ ] Enable caching