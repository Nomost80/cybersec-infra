The goal is to have a reverse proxy that serve the e-commerce website on the internet. It will do load balancing between two web servers.

# Installation on CentOS7

1. Add repo
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=https://nginx.org/packages/mainline/centos/7/x86_64/
gpgcheck=0
enabled=1
EOF
```

2. Update the system
```bash
sudo yum update
```

3. Install the package
```bash
sudo yum install -y nginx
```

4. Create a www user
```bash
sudo useradd --system --home /etc/nginx --shell /bin/false www
```

5. Set Nginx folders owner
```bash
sudo chown -R www:www /etc/nginx /var/log/nginx
``` 

# Setup

1. Move default config
```bash
sudo mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.default
```

2. Setup Nginx
   
```bash
export DOMAIN=alphapar.fr

cat <<EOF | sudo tee /etc/nginx/nginx.conf
user   www www;  ## Default: nobody
worker_processes   5;  ## Default: 1

error_log   /var/log/nginx/error.log warn;

worker_rlimit_nofile   8192;

events {
  worker_connections   4096;  ## Default: 1024
}

http {
  include   /etc/nginx/mime.types;
  include   /etc/nginx/confd/*conf;   

  log_format   main  '$remote_addr - $remote_user [$time_local] "$request" '
                   '$status $body_bytes_sent "$http_referer" '
                   '"$http_user_agent" "$http_x_forwarded_for"';

  access_log   /var/log/nginx/access.log  main;

  keepalive_timeout   65;

  ssl_prefer_server_ciphers   on;
  ssl_protocols               TLSv1.2 TLSv1.3;
  ssl_session_cache           shared:SSL:10m;
  ssl_session_timeout         10m;

  ssl_certificate           /etc/nginx/certs/chain.crt;
  ssl_certificate_key       /etc/nginx/certs/server.key;

  upstream applications {
    server web1.alphapar.fr:8080;
    server web2.alphapar.fr:8080;
  }

  server {
    listen        80;  
    server_name   ${DOMAIN} www.${DOMAIN};
    return        301 https://$server_name$request_uri; #Redirection; 
  }

  server {
    listen        443 ssl http2;
    server_name   ${DOMAIN} www.${DOMAIN}; 

    access_log   /var/log/nginx/access.log main;

    add_header   'Access-Control-Allow-Origin' 'https://www.alphapar.fr';
    add_header   'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';

    location / {
      proxy_set_header host $host;
      proxy_set_header X-real-ip $remote_addr;
      proxy_set_header X-forward-for $proxy_add_x_forwarded_for;
      proxy_pass   http://applications;
    }
  }
}
EOF
```

3. Open ports with firewalld
```bash
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-service=https --permanent
sudo firewall-cmd --reload
```

4. Add nginx to permissive domain
```bash
sudo yum install -y policycoreutils-python
sudo semanage permissive -a httpd_t
```

5. Enable and start nginx
```bash
sudo systemctl enable --now nginx
```

# Note

https://www.nginx.com/blog/using-nginx-plus-with-selinux/

# ToDo

[ ] Instead of setting nginx permissive, just allow the restricted operations
[ ] Enable caching