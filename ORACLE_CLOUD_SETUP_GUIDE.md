# Oracle Cloud Infrastructure (OCI) Complete Setup Guide

## Quick Reference URLs

| Resource | URL |
|:---------|:----|
| **OCI Console** | https://cloud.oracle.com |
| **Sign Up (Free Tier)** | https://www.oracle.com/cloud/free/ |
| **Documentation** | https://docs.oracle.com/en-us/iaas/Content/home.htm |

---

## Part 1: Account Setup

### 1.1 Create Free Tier Account
1. Go to https://www.oracle.com/cloud/free/
2. Click **"Start for free"**
3. Fill in your details:
   - Email (use a real one - verification required)
   - Country (determines available regions)
   - **Home Region** ⚠️ CHOOSE CAREFULLY - cannot change later!
4. Add payment method (credit card required but won't be charged)
5. Complete verification

> **IMPORTANT:** Your **Home Region** is where your Always Free resources live. Pick one with good availability:
> - Less popular: `ap-seoul-1`, `ap-osaka-1`, `sa-saopaulo-1`
> - Popular (harder to get ARM): `us-ashburn-1`, `eu-frankfurt-1`

### 1.2 Access the Console
1. Go to https://cloud.oracle.com
2. Enter your **Cloud Account Name** (tenancy name)
3. Click **Continue** → Sign in with your credentials

---

## Part 2: Generate SSH Keys

Before creating a VM, you need SSH keys for secure access.

### On Mac/Linux:
```bash
# Generate a new SSH key pair
ssh-keygen -t ed25519 -C "oci-vm" -f ~/.ssh/oci_vm_key

# View your PUBLIC key (you'll paste this into OCI)
cat ~/.ssh/oci_vm_key.pub
```

### On Windows (PowerShell):
```powershell
ssh-keygen -t ed25519 -C "oci-vm" -f $env:USERPROFILE\.ssh\oci_vm_key
type $env:USERPROFILE\.ssh\oci_vm_key.pub
```

**Save the output** - you'll need it when creating the VM.

---

## Part 3: Create a Virtual Cloud Network (VCN)

A VCN is required before creating any VM.

### 3.1 Quick VCN Creation (Recommended)
1. In OCI Console, click **☰ Menu** → **Networking** → **Virtual Cloud Networks**
2. Click **"Start VCN Wizard"**
3. Select **"Create VCN with Internet Connectivity"** → **Start VCN Wizard**
4. Fill in:
   - **VCN Name**: `my-vcn`
   - **Compartment**: (leave as root compartment)
   - Leave CIDR blocks as default
5. Click **Next** → **Create**
6. Wait for completion (green checkmarks)

### 3.2 Note Your Subnet OCID
1. Click on your new VCN
2. Click **Subnets** (left menu)
3. Click on the **Public Subnet**
4. Copy the **OCID** (you'll need this for automation scripts)

---

## Part 4: Open Firewall Ports (Security List)

By default, only SSH (port 22) is open. To allow HTTP/HTTPS:

### 4.1 Add Ingress Rules
1. In your VCN, click **Security Lists** (left menu)
2. Click **Default Security List for my-vcn**
3. Click **Add Ingress Rules**
4. Add these rules:

| Source CIDR | Protocol | Dest Port | Description |
|:------------|:---------|:----------|:------------|
| `0.0.0.0/0` | TCP | 80 | HTTP |
| `0.0.0.0/0` | TCP | 443 | HTTPS |
| `0.0.0.0/0` | TCP | 8080 | Custom app |

5. Click **Add Ingress Rules**

---

## Part 5: Create a Compute Instance (VM)

### 5.1 Always Free Options

| Shape | Specs | Notes |
|:------|:------|:------|
| `VM.Standard.E2.1.Micro` | 1 OCPU, 1GB RAM | AMD x64, easy to get |
| `VM.Standard.A1.Flex` | Up to 4 OCPU, 24GB RAM | ARM, hard to get |

### 5.2 Create the VM
1. Click **☰ Menu** → **Compute** → **Instances**
2. Click **Create Instance**
3. Fill in:

**Name and Compartment:**
- **Name**: `my-server`

**Placement:**
- **Availability Domain**: Try each one if you get "Out of capacity"

**Image and Shape:**
- Click **Change image** → **Oracle Linux** or **Ubuntu** (Canonical)
- Click **Change shape**:
  - For AMD (easy): Select `VM.Standard.E2.1.Micro`
  - For ARM (hard): Select `VM.Standard.A1.Flex`, set 4 OCPUs, 24GB RAM

**Networking:**
- **VCN**: Select `my-vcn`
- **Subnet**: Select your public subnet
- **Public IPv4 address**: ✅ **Assign a public IPv4 address**

**SSH Keys:**
- Select **Paste public keys**
- Paste your public key from Part 2

4. Click **Create**
5. Wait for status: **RUNNING** (may take 1-2 minutes)

### 5.3 Note the Public IP
Once running, copy the **Public IP address** from the instance details page.

---

## Part 6: Connect via SSH

### From Mac/Linux:
```bash
# Replace with your actual IP and key path
ssh -i ~/.ssh/oci_vm_key opc@<PUBLIC_IP>

# Example:
ssh -i ~/.ssh/oci_vm_key opc@129.213.45.123
```

### From Windows (PowerShell):
```powershell
ssh -i $env:USERPROFILE\.ssh\oci_vm_key opc@<PUBLIC_IP>
```

> **NOTE:**
> - **Oracle Linux**: Default user is `opc`
> - **Ubuntu**: Default user is `ubuntu`
> - **CentOS**: Default user is `opc`

### First Connection Tips:
```bash
# Update the system
sudo dnf update -y   # Oracle Linux / CentOS
sudo apt update && sudo apt upgrade -y   # Ubuntu

# Check disk space
df -h

# Check memory
free -h
```

---

## Part 7: Open Firewall on the VM (iptables)

OCI has **two firewalls**: Security List (cloud) AND iptables (VM). You must open both!

### Oracle Linux / CentOS:
```bash
# Open port 80 (HTTP)
sudo firewall-cmd --permanent --add-port=80/tcp

# Open port 443 (HTTPS)
sudo firewall-cmd --permanent --add-port=443/tcp

# Open port 8080 (custom)
sudo firewall-cmd --permanent --add-port=8080/tcp

# Apply changes
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-all
```

### Ubuntu:
```bash
# Ubuntu uses iptables directly on OCI images
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 8080 -j ACCEPT

# Save rules (persist after reboot)
sudo netfilter-persistent save
```

---

## Part 8: Common Setup Commands

### Install Node.js (Ubuntu):
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node --version
```

### Install Python (Ubuntu):
```bash
sudo apt install -y python3 python3-pip python3-venv
python3 --version
```

### Install Docker (Ubuntu):
```bash
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
# Log out and back in for group change to take effect
```

### Install Nginx (Ubuntu):
```bash
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
curl http://localhost  # Test
```

---

## Part 9: Attach Block Storage (Optional)

Free tier includes 200GB block storage.

### 9.1 Create Block Volume
1. **☰ Menu** → **Storage** → **Block Volumes**
2. Click **Create Block Volume**
3. Fill in:
   - **Name**: `data-volume`
   - **Size**: 50 GB (or up to 200GB total across all)
   - **Availability Domain**: Same as your VM!
4. Click **Create**

### 9.2 Attach to VM
1. Go to your instance → **Attached Block Volumes**
2. Click **Attach Block Volume**
3. Select your volume → **Attach**
4. Click on the attachment → **iSCSI Commands & Information**
5. Copy the **Attach Commands** and run on your VM

### 9.3 Format and Mount
```bash
# List disks
lsblk

# Format (ONLY do this once on a new volume!)
sudo mkfs.ext4 /dev/sdb

# Create mount point
sudo mkdir /data

# Mount
sudo mount /dev/sdb /data

# Make persistent (add to fstab)
echo '/dev/sdb /data ext4 defaults,_netdev 0 2' | sudo tee -a /etc/fstab
```

---

## Part 10: OCI CLI (Command Line)

For automation and scripting.

### Install OCI CLI:
```bash
bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"
```

### Configure:
```bash
oci setup config
# Follow prompts - you'll need:
# - User OCID (Profile → My Profile → OCID)
# - Tenancy OCID (Profile → Tenancy → OCID)
# - Region (e.g., eu-frankfurt-1)
# - Generate new API key when prompted
```

### Useful Commands:
```bash
# List instances
oci compute instance list --compartment-id <COMPARTMENT_OCID>

# Start an instance
oci compute instance action --action START --instance-id <INSTANCE_OCID>

# Stop an instance
oci compute instance action --action STOP --instance-id <INSTANCE_OCID>

# Reboot an instance
oci compute instance action --action SOFTRESET --instance-id <INSTANCE_OCID>
```

---

## Troubleshooting

### "Out of host capacity" Error
- Try a different **Availability Domain**
- Try at off-peak hours (early morning UTC)
- Use the ARM grabber script to retry automatically

### SSH Connection Refused
1. Check Security List has port 22 open
2. Check VM firewall: `sudo firewall-cmd --list-all`
3. Verify you're using the correct username (`opc` or `ubuntu`)
4. Verify your private key permissions: `chmod 600 ~/.ssh/oci_vm_key`

### Cannot Access Web App
1. Check Security List has your port open
2. Check VM firewall has your port open
3. Verify app is listening: `sudo netstat -tlnp | grep <PORT>`

---

## Quick Reference: Important OCIDs

| Resource | Where to Find |
|:---------|:--------------|
| **User OCID** | Profile (top right) → My Profile → OCID |
| **Tenancy OCID** | Profile → Tenancy → OCID |
| **Compartment OCID** | Identity → Compartments → (root) → OCID |
| **Subnet OCID** | Networking → VCN → Subnets → (your subnet) → OCID |
| **Instance OCID** | Compute → Instances → (your instance) → OCID |
| **Image OCID** | Compute → Custom Images → (image) → OCID |
