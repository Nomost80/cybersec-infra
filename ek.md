# ElasticSearch

1. Add repo
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/elasticsearch.repo
[elasticsearch]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=0
autorefresh=1
type=rpm-md
EOF
```

2. Install ElasticSearch
```bash
sudo yum install -y --enablerepo=elasticsearch elasticsearch
```

3. In `/etc/elasticsearch/elasticsearch.yml` :
```bash
network.host: ek.corp.alphapar.fr
http.port: 9200
discovery.seed_hosts: ["127.0.0.1"]
```

4. Open ports with firewalld
```bash
sudo firewall-cmd --add-port=9200/tcp --permanent
sudo firewall-cmd --add-port=9300/tcp --permanent
sudo firewall-cmd --reload
```

5. Enable and start ES
```bash
sudo systemctl enable --now elasticsearch
```

## Basic Authentication
```bash
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```


## Setup TLS
```bash
xpack.security.enabled: true
xpack.security.audit.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.client_authentication: "none"
xpack.security.http.ssl.verification_mode: "full"
xpack.security.http.ssl.key: "/etc/elasticsearch/certs/server.key"
xpack.security.http.ssl.certificate: "/etc/elasticsearch/certs/server.crt"
xpack.security.transport.ssl.certificate_authorities: "/etc/elasticsearch/certs/root.crt"
```

## Enable Beat Monitoring
```bash
xpack.monitoring.enabled: true
xpack.monitoring.collection.enabled: true
```

## Role Management
filebeat : fgh7p2QQ
auditbeat : 7oi5YSbv
metricbeat : c87F40az
winlogbeat : sJ21qnfR
packetbeat: nN77q6ZR

Vault : file, audit, metric, packetbeat

# Kibana

1. Add repo
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kibana.repo
[kibana]
name=Kibana repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```

2. Install Kibana
```bash
sudo yum install -y --enablerepo=kibana kibana
```

3. Open ports with firewalld
```bash
sudo firewall-cmd --add-port=5601/tcp --permanent
sudo firewall-cmd --reload
```

4. Create log folder
```bash
sudo mkdir /var/log/kibana
```

5. Set folder owner
```bash
sudo chown -R kibana:kibana /var/log/kibana
```

6. Setup Kibana
```bash
cat <<EOF | sudo tee /etc/kibana/kibana.yml 
server.host: "ek.corp.alphapar.fr"
server.port: 5601

elasticsearch.hosts: ["https://localhost:9200"]
elasticsearch.preserveHost: true

elasticsearch.username: "kibana"
elasticsearch.password: "pass"

server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/certs/server.crt
server.ssl.key: /etc/kibana/certs/server.key

elasticsearch.ssl.certificate: /etc/kibana/certs/server.crt
elasticsearch.ssl.key: /etc/kibana/certs/server.key
elasticsearch.ssl.certificateAuthorities: [ "/etc/kibana/certs/root.crt" ]

elasticsearch.logQueries: true

logging.dest: /var/log/kibana/kibana.log
logging.verbose: true
EOF
```

7. Enable and start ES
```bash
sudo systemctl enable --now kibana
```

# Beats

There are explanations on Kibana. It's very simple ! They come with default dashboards.

An example for nginx is available here : http://10.0.40.5:5601/app/kibana#/home/tutorial/nginxLogs?_g=()

The process is the same for each beat.

1. Download and install the beat
```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/xxxbeat/xxxbeat-7.5.0-x86_64.rpm
sudo rpm -vi auditbeat-7.5.0-x86_64.rpm
```

2. Edit the conf in `/etc/xxxbeat/xxxbeat.yml`
```
name: "FQDN"

- type: log
  enabled: true

setup.template.name: "xxxxbeat"
setup.template.pattern: "xxxbeat-*"

output.elasticsearch:
  hosts: ["https://ek.corp.alphapar.fr:9200"]
  ssl:
    enabled: true
    certificate_authorities: ["/etc/pki/tls/certs/chain.crt"]
  username: "xxxbeat"
  password: "<password>"
  index: "xxxbeat-customname-%{[agent.version]}-%{+yyyy.MM.dd}"
  indices: 
    - index: "xxxbeat-customname-error%{[agent.version]}-%{+yyyy.MM.dd}"
      when.contains:
        message: "ERR"
    - index: "xxxbeat-customname-warning-%{[agent.version]}-%{+yyyy.MM.dd}"
      when.contains:
        message: "WARN"
    - index: "xxxbeat-customname-debug-%{[agent.version]}-%{+yyyy.MM.dd}"
      when.contains:
        message: "DEBUG"

setup.dashboards.enabled: true 

setup.kibana:
  host: "https://ek.corp.alphapar.fr"
  ssl:
    enabled: true
    certificate_authorities: ["/etc/pki/tls/certs/chain.crt"]

logging.level: debug
logging.selectors: ["*"]

# It enables the beat healthcheck monitoring
monitoring.enabled: true
monitoring.elasticsearch.index: "xxxbeat-customname-monitoring-%{[agent.version]}-%{+yyyy.MM.dd}"
```

3. Start the beat
```bash
sudo xxx setup
sudo service xxx start
```

4. Enable the beat
```bash
sudo systemctl enable xxx
```

# ToDo

Monitor ElasticSearch itself https://www.elastic.co/guide/en/elasticsearch/reference/current/monitoring-settings.html

# Note
https://www.elastic.co/guide/en/beats/filebeat/master/filebeat-template.html

default log indice filter : `filebeat-*,kibana_sample_data_logs*`

Get all es index : curl -XGET -u elastic:Qjhu78wX "https://ek.corp.alphapar.fr:9200/_aliases?pretty=true"

Delete an index : curl -XDELETE -u elastic:Qjhu78wX "https://ek.corp.alphapar.fr:9200/index_name"

Get user role : curl -XGET -u elastic:Qjhu78wX "https://ek.corp.alphapar.fr:9200/_security/user/beats_system"

