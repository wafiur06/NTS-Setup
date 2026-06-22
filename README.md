# NTS Server Setup with Chrony

This project demonstrates a simple and secure Network Time Security (NTS) lab environment using Chrony on both the server and client systems. NTS adds cryptographic security to NTP, ensuring time packets are not tampered with.

## Environment Setup Overview

### 🖥️ Environment
| Item         | Details            |
| ------------ | ------------------ |
| Hypervisor   | VirtualBox         |
| Provisioning | Vagrant            |
| OS           | Ubuntu Jammy 22.04 |
| NTP          | chrony with NTS    |
| Network      | `192.168.56.0/24`  |

### 💻 VMs
| VM     | Hostname           | IP              | Resources        | Role       |
| ------ | ------------------ | --------------- | ---------------- | ---------- |
| Server | `nts-server.local` | `192.168.56.10` | 1 vCPU / 1048 MB | NTS Server |
| Client | `nts-client.local` | `192.168.56.20` | 1 vCPU / 1048 MB | NTS Client |

### 🔌 Ports
| Port | Protocol | Purpose |
| ---- | -------- | ------- |
| 123  | UDP      | NTP     |
| 4460 | TCP      | NTS-KE  |

---

## 🏗️ Architecture Diagram
![NTS Architecture Diagram](Gemini_Generated_Image_jw6e2hjw6e2hjw6e.png)


## ⚙️ Prerequisites
* Must have **VirtualBox** and **Vagrant** installed on your host machine.

## 🚀 Lab Environment Setup
Change directory to your `Vagrantfile` directory and provision the lab:

```bash
# Use the command below to provision the lab
vagrant up

# Open two separate terminals to SSH into both VMs
vagrant ssh nts-server
vagrant ssh nts-client

```

---

## Phase 01: Configure the NTS Server (`nts-server.local`)

**Step 1: Generate the OpenSSL configuration**
This supports Subject Alternative Name (SAN) tracking:

```bash
mkdir -p ~/certs && cd ~/certs
cat << 'EOF' > nts.conf
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = nts-server.local

[v3_req]
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = nts-server.local
IP.1 = 192.168.56.10
EOF

```

**Step 2: Issue the 365-day self-signed TLS certificate**

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nts-server.key -out nts-server.crt \
  -config nts.conf -extensions v3_req

```

**Step 3: Install Chrony and Configure Firewall**

```bash
sudo apt update && sudo apt install -y chrony

# allow firewall rules
sudo ufw allow 123/udp
sudo ufw allow 4460/tcp

```

**Step 4: Deploy certificates to Chrony's system directory**
Apply Ubuntu-specific group permissions (`_chrony`):

```bash
sudo cp nts-server.crt /etc/chrony/chrony.crt
sudo cp nts-server.key /etc/chrony/chrony.key
sudo chown root:_chrony /etc/chrony/chrony.*
sudo chmod 640 /etc/chrony/chrony.key
sudo chmod 644 /etc/chrony/chrony.crt

```

**Step 5: Overwrite the default configuration file cleanly**

```bash
cat << 'EOF' | sudo tee /etc/chrony/chrony.conf > /dev/null
# Upstream public sources for the server's own internal clock
server cloudflare.com iburst nts
server nts.netnod.se iburst nts

# Core paths
keyfile /etc/chrony/chrony.keys
driftfile /var/lib/chrony/chrony.drift
logdir /var/log/chrony

# System clock thresholds
maxupdateskew 100.0
rtcsync
makestep 1 3
leapsectz right/UTC

# Enable local NTS server daemon engine
ntsservercert /etc/chrony/chrony.crt
ntsserverkey /etc/chrony/chrony.key
ntsdumpdir /var/lib/chrony

# Authorize connections from the Vagrant host-only private network
allow 192.168.56.0/24
EOF

```

**Step 6: Fire up the daemon**

```bash
sudo systemctl restart chrony

```

---

## Phase 02: Securely Bridge the Certificate (Host Machine Actions)

Execute these commands inside your **host terminal** (not the VMs) to bridge the certificate across the isolated network layer:

```bash
# Pull the server public cert out to your physical machine
vagrant ssh nts-server -c "cat /home/vagrant/certs/nts-server.crt" > host-nts-server.crt

# Push the public cert up onto the isolated client machine
vagrant ssh nts-client -c "cat << 'EOF' > /home/vagrant/nts-server.crt
$(cat host-nts-server.crt)
EOF"

```

---

## Phase 03: Configure the NTS Client (`nts-client.local`)

**Step 1: Install Chrony**

```bash
sudo apt update && sudo apt install -y chrony

```

**Step 2: Stage the server's public certificate**
Apply the correct system permission context:

```bash
sudo cp /home/vagrant/nts-server.crt /etc/chrony/nts-server.crt
sudo chown root:_chrony /etc/chrony/nts-server.crt
sudo chmod 644 /etc/chrony/nts-server.crt

```

**Step 3: Map the `.local` domain name**
Add it directly inside the client's host database lookup file:

```bash
echo "192.168.56.10 nts-server.local" | sudo tee -a /etc/hosts

```

**Step 4: Deploy the client configuration file**

```bash
cat << 'EOF' | sudo tee /etc/chrony/chrony.conf > /dev/null
# Query your internal custom NTS engine
server nts-server.local iburst nts

# Explicitly trust the incoming self-signed certificate structure
ntstrustedcerts /etc/chrony/nts-server.crt

# Base tracking parameters
keyfile /etc/chrony/chrony.keys
driftfile /var/lib/chrony/chrony.drift
logdir /var/log/chrony
maxupdateskew 100.0
rtcsync
makestep 1 3
leapsectz right/UTC
EOF

```

**Step 5: Start the synchronization protocol and Validate**

```bash
sudo systemctl restart chrony

# Verification (wait a couple of seconds to sync)
# Look for the '^* nts-server.local' active baseline line marker
chronyc sources -v

# Run with sudo to bypass the 501 Not authorised restriction (Optional)
sudo chronyc authdata 

```

```

এই কোডটুকু কপি করে গিটহাবে পুশ করে দিলেই আপনার পেজটি খুব সুন্দর এবং অর্গানাইজড দেখাবে!

```
