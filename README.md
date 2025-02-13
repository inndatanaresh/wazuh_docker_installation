# WAZUH Docker Installation

## Step-by-Step Configuration of Single-Node Wazuh via Docker

### Steps:

1. Clone the Wazuh repository
2. Generate self-signed certificates (two methods)
3. Start the deployment
4. Default username and credentials: admin / SecretPassword

### Step 1: Clone the Wazuh Repository

```bash
# Clone the repository
git clone https://github.com/wazuh/wazuh-docker.git -b v4.9.2

# Navigate to the single-node directory
cd wazuh-docker/single-node
```

### Step 2: Generate SSL Certificates

```bash
# Create directory structure
mkdir -p config/wazuh_indexer_ssl_certs
cd config/wazuh_indexer_ssl_certs

# Generate private key for root CA
openssl genrsa -out root-ca-key.pem 2048

# Generate root CA certificate
openssl req -x509 -new -nodes -key root-ca-key.pem -sha256 -days 1024 -out root-ca.pem -subj "/C=US/ST=State/L=City/O=Organization/CN=WazuhRootCA"

# Copy root-CA for manager
cp root-ca.pem root-ca-manager.pem

# Generate Wazuh index CA certificates and private key
openssl genrsa -out wazuh.indexer-key.pem 2048

# Generate CSR
openssl req -new -key wazuh.indexer-key.pem -out wazuh.indexer.csr -subj "/C=US/ST=State/L=City/O=Organization/CN=wazuh.indexer"

# Sign the certificate
openssl x509 -req -in admin.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -out admin.pem -sha256 -days 365

# Generate Wazuh manager certificate
openssl genrsa -out wazuh.manager-key.pem 2048

# Generate CSR for Wazuh manager
openssl req -new -key wazuh.manager-key.pem -out wazuh.manager.csr -subj "/C=US/ST=State/L=City/O=Organization/CN=wazuh.manager"

# Generate Wazuh dashboard certificate
openssl genrsa -out wazuh.dashboard-key.pem 2048

# Generate CSR for Wazuh dashboard
openssl req -new -key wazuh.dashboard-key.pem -out wazuh.dashboard.csr -subj "/C=US/ST=State/L=City/O=Organization/CN=wazuh.dashboard"

# Sign the Wazuh dashboard certificate
openssl x509 -req -in wazuh.dashboard.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -out wazuh.dashboard.pem -sha256 -days 365

# Clean up CSR files
rm *.csr

# Set appropriate permissions
chmod 444 *.pem
```

### Step 3: Start the Deployment

```bash
# Foreground deployment
docker-compose up

# Background deployment
docker-compose up -d

# Check Docker containers
docker ps -a

# Wazuh default username and password
# Username: admin
# Password: SecretPassword
```

---

# WAZUH Agent Installation

### Step 1: Add the Wazuh Repository

```bash
sudo rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH

sudo cat > /etc/yum.repos.d/wazuh.repo << EOF
[wazuh]
gpgcheck=1
gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
enabled=1
name=EL-\$releasever - Wazuh
baseurl=https://packages.wazuh.com/4.x/yum/
protect=1
EOF
```

### Step 2: Install the Wazuh Agent

```bash
sudo yum install wazuh-agent
```

### Step 3: Get the Wazuh Manager Indexer Key

```bash
docker exec single-node-wazuh.manager-1 cat /var/ossec/etc/sslmanager.key
```

### Step 4: Configure the Agent

Edit the agent configuration:

```xml
<client>
    <server>
        <address>10.10.5.70</address>
        <port>1514</port>
        <protocol>tcp</protocol>
    </server>
</client>
```

### Step 5: Register the Agent

```bash
sudo /var/ossec/bin/agent-auth -m 10.10.5.70
```

### Step 6: Start the Agent

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### Troubleshooting: Agent Won't Start

```bash
# Set correct permissions
sudo chown -R root:root /var/ossec/
sudo chmod -R 750 /var/ossec/  # Or 777 if necessary

# Disable SELinux temporarily
sudo setenforce 0

# Start the agent
sudo systemctl start wazuh-agent

# Open firewall port
sudo firewall-cmd --permanent --add-port=1514/tcp
sudo firewall-cmd --reload

# Add iptables rule
sudo iptables -A INPUT -p tcp --dport 1514 -j ACCEPT
```

