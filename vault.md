Vault is a secret manager. The goal is to centralize every secrets in a secure way and to offer secret/encryption as a service.

# Installation

1. Set Vault Version
```bash
export VAULT_VERSION=1.3.0
```

2. Download the binary release
```bash   
curl -O https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip
```

3. Extract the binary
```bash   
unzip vault_${VAULT_VERSION}_linux_amd64.zip
```

4. Make Vault globally available
```bash   
sudo mv vault /usr/local/bin/
```

5. Enable command autocompletion
```bash
vault -autocomplete-install
```

6. Create Vault folders
```bash
sudo mkdir /etc/vault
sudo mkdir -p /var/lib/vault/data
sudo mkdir /var/log/vault
```

7. Create Vault user
```bash
sudo useradd --system --home /etc/vault --shell /bin/false vault
```

8. Set Vault folders owner
```bash   
sudo chown -R vault:vault /etc/vault /var/lib/vault/ /var/log/vault
```

9. Add systemd unit
```bash   
cat <<EOF | sudo tee /etc/systemd/system/vault.service
[Unit]
Description="HashiCorp Vault - A tool for managing secrets"
Documentation=https://www.vaultproject.io/docs/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault/config.hcl

[Service]
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/local/bin/vault server -config=/etc/vault/config.hcl
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

10. Create Vault config file
```bash
cat <<EOF | sudo tee /etc/vault/config.hcl
ui                      = true
api_addr                = "http://127.0.0.1:8200"
max_lease_ttl           = "10h"
default_lease_ttl       = "10h"
cluster_name            = "vault"
disable_sealwrap        = true
disable_printable_check = true
log_level               = "trace"   

listener "tcp" {
   address            = "127.0.0.1:8200"
   tls_disable        = true
}

storage "file" {
   path  = "/var/lib/vault/data"
}
EOF
```
Note: For the moment Vault is only available in localhost.

11. Reload systemd daemon
```bash
sudo systemctl daemon-reload
```

12. Open Firewall port
```bash
sudo firewall-cmd --add-port=8200/tcp --permanent
sudo firewall-cmd --reload
``` 

13. Enable and start Vault
```bash
sudo systemctl enable --now vault
```

# Setup

One of the early fundamental problems when bootstrapping and initializing Vault was that the first user (the initializer) received a plain-text copy of all of the unseal keys. This defeats the promises of Vault's security model, and it also makes the distribution of those keys more difficult. That's why it's better to use PGP keys to encrypt the unseal keys and root token.

https://www.vaultproject.io/guides/operations/production

1. Generate a PGP key for the key shares and one for the root token. If the pgp keys already exist we need to import them.
```bash
gpg --gen-key
```
If we need to gain entropy while generating random bytes : `find / | xargs file`

2. Save the public keys as base64 files
```bash
mkdir /etc/vault/pgp
gpg --export uid | base64 > /etc/vault/pgp/xxxx.asc
```

3. Init Vault
```bash
# By default Vault try to use the https endpoint even if TLS is disabled
export VAULT_ADDR=http://127.0.0.1:8200
export VKP=/etc/vault/pgp/xxxx.asc

vault operator init \
  -key-shares=5 -key-threshold=3 \
  -pgp-keys="${VKP},${VKP},${VKP},${VKP},${VKP}" \
  -root-token-pgp-key=$VKP > /etc/vault/init.file
```

4. Unseal Vault (we need to decode at least 3 shared keys)
```bash
echo "unsealkey..." | base64 --decode | gpg --batch --passphrase ... -dq
history -c
vault operator unseal
```

3. Login to Vault
```bash
vault login
```

4. Enable logging
```bash
vault audit enable file file_path=/var/log/vault/audit.log
```

## Auto Unseal

When Vault first boots, it does not yet have the master key in memory, and therefore it cannot decrypt its own data. While Vault is at this state, we say it is “sealed.” Vault uses Shamir’s Secret Sharing, which splits the master key into multiple pieces, each of which must be provided manually by a human operator and recombined in order to regenerate the master key and unseal Vault. This unseal process is manual and tedious.

We need an encryption as a service to auto unseal vault. We could use a second Vault instance but this second instance need to be unsealed as well. It's the same problem. And we can't use external services like AWS KMS because this LAB is not connected to internet.

https://learn.hashicorp.com/vault/day-one/autounseal-transit

## PKI

The goal is to generate a certificate for Vault itself because for the moment TLS is disabled.

1. Set vars
```bash
export NAME=Alphapar
export DOMAIN=alphapar.fr
export HOST_FQDN="vault.corp.${DOMAIN}"
```

2. Enable PKI secrets engine
```bash
vault secrets enable pki
```

2. Setup max TTL
```bash
vault secrets tune -max-lease-ttl=87600h pki
```

3. Generate the root certificate
```bash
mkdir /etc/vault/certs
cd /etc/vault/certs

vault write -field=certificate pki/root/generate/internal \
  common_name="${NAME} Root Authority" \
  ttl=87600h > root.crt
```

4. Configure the CA and CRL URLs
```bash
vault write pki/config/urls \
  issuing_certificates="https://${HOST_FQDN}:8200/v1/pki/ca" \
  crl_distribution_points="https://${HOST_FQDN}:8200/v1/pki/crl"
```

5. Enable the pki secrets engine at the pki_int path
```bash
vault secrets enable -path=pki_int pki
```

6. Tune pki_int TTL
```bash
vault secrets tune -max-lease-ttl=43800h pki_int
```

7. Generate a CSR for the intermediate authority
```bash
vault write -format=json pki_int/intermediate/generate/internal \
  common_name="${NAME} Intermediate Authority" \
  | jq -r '.data.csr' > intermediate.csr
```

8. Sign the intermediate certificate with the root certificate and save the generated certificate
```bash
vault write -format=json pki/root/sign-intermediate csr=@intermediate.csr \
  format=pem_bundle ttl="43800h" \
  | jq -r '.data.certificate' > intermediate.crt
```

9. Import the intermediate certificate into Vault
```bash
vault write pki_int/intermediate/set-signed certificate=@intermediate.crt
```

10. (Optional) Add the root certificate to the trusted zone. It allows to only pass the intermediate certificate instead of the full chain to the service.
```bash
# On Centos7
sudo cp root.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

# On Debian
cp root.crt /usr/local/share/ca-certificates
sudo update-ca-certificates
```

11. Create a role for the domain
```bash
vault write pki_int/roles/alphapar-dot-fr \
  allowed_domains=$DOMAIN \
  allow_subdomains=true \
  max_ttl="1440h"
```

12. Request a certificate for Vault
```bash
vault write pki_int/issue/alphapar-dot-fr common_name=$HOST_FQDN ttl="1440h"
```

13. Copy the private key in `/etc/vault/certs/vault.key` and concatenate in this order : the certificate - intermediate ca certificate in `/etc/vault/certs/vault.crt`
    
14.  In `/etc/vault/config.hcl` remove `tls_disable = true` and add/edit the following lines:
```bash
api_addr = "https://${HOST_FQDN}:8200"

listener "tcp" {
  address       = "${IP}:8200"
  tls_cert_file = "/etc/vault/certs/vault.crt"
  tls_key_file  = "/etc/vault/certs/vault.key" 
}
```

15.  Edit Vault environment variables and reload the shell
```bash
# If Vault is still listening in 127.0.0.1:8200
echo "export VAULT_ADDR=https://${HOST_FQDN}:8200" >> ~/.bashrc
# The tls_cert_file param is not working well
echo "export VAULT_CACERT=/etc/vault/certs/vault.crt" >> ~/.bashrc
```

16. Restart Vault
```bash
systemctl restart vault
```

17. Import the intermediate CA to the browser

## LDAP

1. Set vars
```bash
export DOMAIN_NAME=corp.alphapar
export DOMAIN_EXT=fr
export DOMAIN="${DOMAIN_NAME}.${DOMAIN_EXT}"
```

2. Enable LDAP authentication
```bash
vault auth enable ldap
```

3. Setup LDAP link
```bash
vault write auth/ldap/config \
  url="ldap://${DOMAIN}" \
  userdn="ou=Users,dc=${DOMAIN_NAME},dc=${DOMAIN_EXT}" \
  groupdn="ou=Groups,dc=${DOMAIN_NAME},dc=${DOMAIN_EXT}" \
  groupfilter="(&(objectClass=group)(member:1.2.840.113556.1.4.1941:={{.UserDN}}))" \
  groupattr="cn" \
  upndomain=$DOMAIN \
  certificate=@ldap.crt \
  insecure_tls=false \
  starttls=true
```

4. Create a Vault policy
```bash
cat <<EOF | vault policy write regular -
path "secret/*" {
  capabilities = ["read", "list"]
}
EOF
```

5. Map a LDAP group to a Vault policy
```bash
vault write auth/ldap/groups/xxxx policies=regular
```

It should be noted that user -> policy mapping happens at token creation time. And changes in group membership on the LDAP server will not affect tokens that have already been provisioned. To see these changes, old tokens should be revoked and the user should be asked to reauthenticate.

## MySQL Secrets

1. Enable database secret engine
```bash
vault secrets enable database
```

2. Setup database link
```bash
vault write database/config/eshop \
  plugin_name=mysql-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(db.alphapar.fr:3306)/" \
  allowed_roles="mysql-credentials" \
  username="vault" \
  password="xxxxxxxxx"
```

3. Create the Vault role which creates crendentials
```bash
vault write database/roles/mysql-credentials \
  db_name=eshop \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
  default_ttl="1h" \
  max_ttl="24h"
```

## App Role

The goal is to allow apps to authenticate with Vault defined roles.

1. Enable app role auth method
```bash
vault auth enable approle
```

2. Create a policy
```bash
cat <<EOF | vault policy write mysql -
path "database/creds/mysql-credentials" {
  capabilities = ["read"]
}
EOF
```

3. Create a named role linked to the above policy
```bash
vault write auth/approle/role/mysql-consumer \
  secret_id_ttl=10m \
  token_num_uses=10 \
  token_ttl=20m \
  token_max_ttl=30m \
  secret_id_num_uses=40 \
  token_policies="mysql"
```

# Usage

The GUI is available at https://vault.corp.alphapar.fr:8200

## PKI

Request a certificate
```bash
vault write pki_int/issue/alphapar-dot-fr common_name="xxxx.${DOMAIN}" ttl="1440h"
```

Then we juste have to indicate the server certificate, the server private key and the chain certificates (if the root certificate is located in the trusted zones only the intermediate is needed).
The private key is not saved by Vault therefore we need to save it.

Remove expired certificates and clean CRL
```bash
vault write pki_int/tidy tidy_cert_store=true tidy_revoked_certs=true
```

## Database Credentials

Get database credentials
```bash
vault read database/creds/mysql-credentials

Key                Value
---                -----
lease_id           database/creds/my-role/2f6a614c-4aa2-7b19-24b9-ad944a8d4de6
lease_duration     1h
lease_renewable    true
password           8cab931c-d62e-a73d-60d3-5ee85139cd66
username           v-root-e2978cd0-
```


Full example workflow:

1. Get role id
```bash
vault read auth/approle/role/mysql-consumer/role-id

role_id     db02de05-fa39-4855-059b-67221c5c2f63
```

2. Get secret id
```bash
vault write -f auth/approle/role/my-role/secret-id

secret_id               6a174c20-f6de-a53c-74d2-6018fcceff64
secret_id_accessor      c454f7e5-996e-7230-6074-6ef26b7bcf86
```

3. Get mysql credentials
```js
import axios from 'axios';

const vaultWS = axios.create({
  baseURL: 'https://vault.corp.alphapar.fr:8200/v1',
  responseType: 'json'
});

vaultWS.interceptors.response.use(response => response.data);

// Ideally these credentiaks should come from process.env that would have been setted by a CI/CD system because these values have a short lifetime for security concerns.
const body = {
  role_id: 'db02de05-fa39-4855-059b-67221c5c2f63',
  secret_id: '6a174c20-f6de-a53c-74d2-6018fcceff64'
};

// get token that allows to fetch mysql credentials
const { auth } = vaultWS.post('/auth/approle/login', body);

const headers = {
  'X-Vault-Token': auth.client_token
};

// get mysql dynamic credentials
const { username, password } = axios.get('/database/creds/mysql-credentials', { headers });
```

# Note

Use `| tee` instand of `>`.

The bindpass param for the ldap auth is never taken into consideration even in the UI when there are special chars. Therefore the authenticated searching is not working...

https://www.hashicorp.com/blog/authenticating-applications-with-vault-approle/