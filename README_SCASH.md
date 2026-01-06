# ElectrumX-SCASH Server - python electrum-scash server

```
Licence: MIT
ElectrumX-SCASH Maintainers: GoodMan365
Language: Python (>= 3.10)
```


This project is a fork of [spesmilo/electrumx](https://github.com/spesmilo/electrumx).
The original author haven't provided support for SCASH, which we intend to provide and keep.

ElectrumX-SCASH allows users to run their own Electrum-Scash server. It connects to your
full node and indexes the blockchain, allowing efficient querying of the history of
arbitrary addresses. The server can be exposed publicly, and joined to the public network
of servers.

This guide provides complete installation and configuration instructions. This guide assumes ubuntu OS but can be adopted to other systems as well.

## Prerequisites

- Ubuntu/Debian-based system (tested on Ubuntu 22.04 LTS)
- Python 3.10 or higher (tested on python 3.12)
- Git
- SCASH daemon (scashd) binary
- OpenSSL for SSL certificate generation

## Installation Guide

### 1. System Preparation

```bash
# Update system
sudo apt update
sudo apt upgrade -y

# Install Python and dependencies
sudo apt install -y python3.12 python3.12-dev python3.12-venv \
    build-essential libssl-dev libffi-dev \
    git net-tools telnet openssl

# Clone repository
sudo mkdir -p /opt
cd /opt
sudo git clone https://github.com/GoodMan365/electrumx-scash.git
cd electrumx-scash
sudo chown -R $USER:$USER .
```

### 2. Python Environment Setup
```bash
cd /opt/electrumx-scash

# Create virtual environment
python3.12 -m venv venv

# Activate virtual environment
source venv/bin/activate
sudo apt install -y python3.12-dev python3.12-venv build-essential

# Upgrade pip and install dependencies
pip install --upgrade pip setuptools wheel

# Install ElectrumX-SCASH
pip install -e .  # Editable mode
pip install .
```

### 3. SSL Certificate Setup

```bash
# Create SSL certificate directory
sudo mkdir -p /etc/self_signed
sudo chmod 755 /etc/self_signed

# Create SSL configuration file
sudo tee /etc/self_signed/input_for_ssl.conf << 'EOF'
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn

[ dn ]
C = US
ST = State
L = City
O = MyOrg
CN = YOUR_SERVER_IP_OR_DOMAIN

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
IP.1 = YOUR_SERVER_IP      # Replace with your public IP
IP.2 = 127.0.0.1          # Localhost
EOF

# Generate SSL certificates
cd /etc/self_signed
sudo openssl req -x509 -nodes -days 1825 -newkey rsa:2048 \
    -keyout ip.key \
    -out ip.crt \
    -config input_for_ssl.conf

# Set proper permissions
sudo chmod 644 /etc/self_signed/ip.crt /etc/self_signed/ip.key
```

**Important:** Replace `YOUR_SERVER_IP_OR_DOMAIN` and `YOUR_SERVER_IP` with your actual server IP address or domain name.

### 4. SCASH Daemon Service Setup

#### Option A: If you already have scashd.service
Check your existing service name:
```bash
# List all services containing "scash"
systemctl list-unit-files | grep -i scash

# Check service status
sudo systemctl status scashd.service  # or your service name
```
If your service has a different name (e.g., `scash-daemon.service`), you'll need to update the ElectrumX service file accordingly in the next step.

#### Option B: Create scashd.service (if not exists)
```bash
# First, ensure you have SCASH daemon installed
# Example path: /home/$(whoami)/scash-2.0.0-narnia-core-27.0.0-x84_64-pc-linux-gnu/scashd
# Adjust the path to match your actual scashd location

# Create scashd systemd service
sudo tee /etc/systemd/system/scashd.service << EOF
[Unit]
Description=SCASH Node Daemon
After=network.target

[Service]
User=$(whoami)
Group=$(whoami)
Type=simple
ExecStart=/home/$(whoami)/scash-2.0.0-narnia-core-27.0.0-x84_64-pc-linux-gnu/scashd \
    -conf=/home/$(whoami)/.scash/scash.conf
Restart=always
RestartSec=30
TimeoutStopSec=120
LimitNOFILE=8192

# Hardening
PrivateTmp=true
ProtectSystem=full
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd
sudo systemctl daemon-reload

# Enable scashd to start on boot
sudo systemctl enable scashd.service

# Start scashd (make sure your scash.conf is properly configured first)
sudo systemctl start scashd.service

# Check status
sudo systemctl status scashd.service
```

**Important:** Adjust the `ExecStart` path to match your actual scashd binary location and configuration file path.

### 5. ElectrumX-SCASH Configuration

```bash
# Create configuration file
sudo tee /etc/electrumx-scash.conf << EOF
COIN=Scash
NET=mainnet
DAEMON_URL=http://rpcuser:rpcpassword@127.0.0.1:8342
DB_DIRECTORY=/var/lib/electrumx-scash
SERVICES=tcp://localhost:50001,ssl://:50002,rpc://localhost
MAX_SESSIONS=2000
SSL_CERTFILE=/etc/self_signed/ip.crt
SSL_KEYFILE=/etc/self_signed/ip.key
StandardOutput=journal
StandardError=journal
MAX_SEND=16000000
MAX_RECV=3000000

COST_SOFT_LIMIT=8000
COST_HARD_LIMIT=80000
BANDWIDTH_UNIT_COST=40000

REQUEST_TIMEOUT=60
INITIAL_CONCURRENT=20
SESSION_TIMEOUT=3600
EOF

# Create database directory
sudo mkdir -p /var/lib/electrumx-scash
sudo chown -R $USER:$USER /var/lib/electrumx-scash

# Set proper permissions
sudo chmod 644 /etc/electrumx-scash.conf
```

**Important Configuration Notes:**
1. **DAEMON_URL**: Replace `rpcuser:rpcpassword` with your actual SCASH RPC credentials from `scash.conf`
2. **PORT**: Ensure port `8342` matches your SCASH daemon RPC port in `scash.conf` 8342 is default PORT


### 6. ElectrumX-SCASH Service Setup

```bash
# Create systemd service file
sudo tee /etc/systemd/system/electrumx-scash.service << EOF
[Unit]
Description=ElectrumX-SCASH Server
After=network.target scashd.service
Requires=scashd.service
BindsTo=scashd.service
PartOf=scashd.service

[Service]
EnvironmentFile=/etc/electrumx-scash.conf
ExecStart=/opt/electrumx-scash/venv/bin/python /opt/electrumx-scash/venv/bin/electrumx_server
User=$(whoami)
Group=$(whoami)
Type=simple
KillMode=process
TimeoutSec=60
Restart=on-failure
RestartSec=30
WorkingDirectory=/opt/electrumx-scash

# Wait for scashd RPC to be ready
ExecStartPre=/bin/sleep 15

# Hardening measures
PrivateTmp=true
ProtectSystem=full
NoNewPrivileges=true
PrivateDevices=true
MemoryDenyWriteExecute=true

[Install]
WantedBy=multi-user.target
EOF

```
**Important Configuration Notes:**
1. If your scashd service has a different name, update it here:
2. Replace "scashd.service" with your actual service name in all three places:
3. After=network.target YOUR_SCASH_SERVICE_NAME.service
4. Requires=YOUR_SCASH_SERVICE_NAME.service
5. BindsTo=YOUR_SCASH_SERVICE_NAME.service
6. PartOf=YOUR_SCASH_SERVICE_NAME.service

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable ElectrumX-SCASH to start on boot
sudo systemctl enable electrumx-scash.service
```

### 7. Starting the Services

```bash
# Start SCASH daemon first (if not already running)
sudo systemctl start scashd.service

# Wait for scashd to initialize RPC (30-60 seconds for mainnet)
echo "Waiting for scashd to initialize..."
sleep 45

# Start ElectrumX-SCASH
sudo systemctl start electrumx-scash.service

# Check status of both services
sudo systemctl status scashd.service electrumx-scash.service
```

## Verification

### Basic Service Checks

```bash
# Check service status
sudo systemctl status electrumx-scash.service

# Check listening ports
sudo ss -tulpn | grep :5000

# Check process
ps aux | grep electrumx_server
```

### Connection Tests

```bash
# Test TCP connection (port 50001)
telnet 127.0.0.1 50001
{"id":1,"method":"server.version","params":["mynet","1.4"]}
{"id":2,"method":"blockchain.block.header","params":[1]}
```
if everything goes fine you will get proper response. close telnet by `ctrl-]` and then `quit`


```bash
# Follow ElectrumX logs
sudo journalctl -u electrumx-scash.service -f
```

## Firewall Configuration

If using a firewall, open necessary ports:

```bash
# For UFW
sudo ufw allow 50001/tcp comment 'ElectrumX-SCASH TCP'
sudo ufw allow 50002/tcp comment 'ElectrumX-SCASH SSL'

# For firewalld
sudo firewall-cmd --permanent --add-port=50001/tcp
sudo firewall-cmd --permanent --add-port=50002/tcp
sudo firewall-cmd --reload
```
## SCASH Daemon Configuration

Ensure your `~/.scash/scash.conf` has proper RPC settings:

```ini
# Basic configuration
server=1		# Accepts JSON-RPC commands
daemon=0 		# Run in the background
rpcuser=your_rpc_username	# Username which you will use in ElectrumX-SCASH
rpcpassword=your_strong_password	#Password for Username
rpcallowip=127.0.0.1		#Allowed incoming connection from; only local machine
rpcport=8342			#RPC port for listening

listen=1 		# Turn listening mode on
txindex=1		# Make sure Core keeps an index of all txs
addressindex=1
timestampindex=1
spentindex=1

# Performance settings
maxconnections=40
maxuploadtarget=5000
```

Hopefully it will work by now.




