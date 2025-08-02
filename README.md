
# Oracle APEX Installation Guide - 2025 Battle-Tested Edition

**Updated: July 29, 2025 | Tested On: Oracle Linux 9, Oracle Database 23ai Free, Proxmox VM**

A comprehensive, **real-world tested** guide based on actual deployment experience and live troubleshooting.

## 📋 Password Strategy - TESTED & VALIDATED

| Component | Password | Reason |
|-----------|----------|---------|
| SYS/SYSTEM | `Password123` | Connection string compatibility |
| Database Users | `Password123` | Database connection users |
| ORDS Admin | `Password1` | ORDS security requirements |
| APEX Admin | `Password1!` | APEX requires special characters |

**CRITICAL:** APEX admin password MUST contain special characters (!"`'#$%&()[]{},:*+-/|:?_~)

**Note:** These are example passwords for testing. Use strong, unique passwords in production environments.

## 🎯 Deployment Options Overview

| Method | Complexity | Time | Best For |
|--------|------------|------|----------|
| Oracle Cloud | ⭐ | 5 min | Quick start, testing |
| Manual Install | ⭐⭐⭐⭐ | 2-3 hours | Production, custom setup |
| VM Appliance | ⭐⭐ | 30 min | Learning, isolated testing |

## 🔧 Manual Installation (Production) - BATTLE-TESTED

### Prerequisites
- Oracle Linux 9 (or RHEL/CentOS 8+)
- 8GB RAM minimum (16GB recommended)
- 50GB disk space
- sudo access

### Step 1: Update System and Install Oracle Database 23ai Free

```bash
# Update system packages
sudo dnf update -y

# Install Oracle Database directly from RPM (ONLY for 23ai Free)
sudo dnf install -y https://download.oracle.com/otn-pub/otn_software/db-free/oracle-database-free-23ai-1.0-1.el9.x86_64.rpm
```

### Step 2: Configure Database

```bash
# Configure the database (always run this after install)
sudo /etc/init.d/oracle-free-23ai configure
```

**When prompted for passwords:** Enter `Password123` for both SYS and SYSTEM

### Step 3: Start and Verify Database

```bash
# Start the database service
sudo /etc/init.d/oracle-free-23ai start

# Verify database status
sudo /etc/init.d/oracle-free-23ai status
```

### Step 4: Set Up Oracle Environment

```bash
# Create oracle user home directory (may not exist)
sudo mkdir -p /home/oracle
sudo chown oracle:oinstall /home/oracle

# Switch to oracle user
sudo su - oracle

# Set environment variables permanently
cat >> ~/.bash_profile << 'EOF'
export ORACLE_HOME=/opt/oracle/product/23ai/dbhomeFree
export ORACLE_SID=FREE
export ORACLE_BASE=/opt/oracle
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:$LD_LIBRARY_PATH
EOF

# Load environment
source ~/.bash_profile

# Test database connection
sqlplus / as sysdba
quit
```

### Step 5: Install Java

```bash
# Exit oracle user
exit

# Install Java as root
sudo dnf install -y java-17-openjdk java-17-openjdk-devel

# Verify
java -version
```

### Step 6: Download and Install APEX

```bash
sudo su - oracle
cd /home/oracle

# Download latest APEX
wget https://download.oracle.com/otn_software/apex/apex-latest.zip
unzip apex-latest.zip

# Navigate to apex directory (CRITICAL!)
cd apex

# IMPORTANT: You MUST be in the /home/oracle/apex directory for the installation to work
# Verify you're in the correct directory:
pwd
# Should show: /home/oracle/apex

# Install APEX in pluggable database (FREEPDB1, not CDB)
sqlplus 'sys/Password123@localhost:1521/FREEPDB1' as sysdba

# Run installation script (must be in apex directory)
@apexins.sql SYSAUX SYSAUX TEMP /i/

# Configure REST services (CRITICAL - DO NOT SKIP)
@apex_rest_config.sql
# Use: Password123 for all database connections

quit
```

### Step 7: Install ORDS

```bash
# Download ORDS
sudo mkdir -p /opt/oracle/ords
cd /opt/oracle/ords
sudo wget https://download.oracle.com/otn_software/java/ords/ords-latest.zip
sudo unzip ords-latest.zip
sudo chown -R oracle:oinstall /opt/oracle/ords

# Create config directory
sudo mkdir -p /opt/ords-config
sudo chown -R oracle:oinstall /opt/ords-config

# Add ORDS to PATH
sudo su - oracle
echo 'export PATH=$PATH:/opt/oracle/ords/bin' >> ~/.bash_profile
source ~/.bash_profile
```

### Step 8: Configure ORDS

```bash
# Install ORDS interactively (often works without prompts)
ords --config /opt/ords-config install
```

**Interactive Prompts:**
- **Connection:** Choose `[S] Specify the database connection`
- **Connection type:** `[1] Basic`
- **Host:** Use your system hostname (NOT localhost)
- **Port:** `1521`
- **Service:** `FREEPDB1`
- **Accept settings and continue:** `[A]`

### Step 9: Critical Post-Installation Steps

#### 9.1: Unlock Database Accounts (MANDATORY)

```bash
# Connect to database and unlock accounts that WILL be locked
sqlplus 'sys/Password123@localhost:1521/FREEPDB1' as sysdba

# Check and unlock accounts
SELECT username, account_status FROM dba_users 
WHERE username IN ('ORDS_PUBLIC_USER', 'APEX_PUBLIC_USER', 'APEX_240200');

ALTER USER ORDS_PUBLIC_USER ACCOUNT UNLOCK;
ALTER USER APEX_PUBLIC_USER ACCOUNT UNLOCK;
ALTER USER APEX_240200 ACCOUNT UNLOCK;

quit
```

#### 9.2: Configure Static Files (ESSENTIAL for UI)

```bash
# Create static files directory
sudo mkdir -p /opt/ords-config/global/doc_root/i
sudo chown -R oracle:oinstall /opt/ords-config/global/doc_root/i

# Copy APEX static files
sudo su - oracle
cp -r /home/oracle/apex/images/* /opt/ords-config/global/doc_root/i/
exit
```

#### 9.3: Start ORDS in Server Mode

```bash
# ORDS install command finishes but doesn't serve automatically
# You must start ORDS in serve mode:
sudo su - oracle
ords --config /opt/ords-config serve

# You should see:
# "HTTP and HTTP/2 cleartext listening on host: 0.0.0.0, port: 8080"
# "Oracle REST Data Services initialized"

# IMPORTANT: ORDS will run in foreground and appear to "hang"
# This is normal! Press Ctrl+C to stop ORDS and return to shell
# We'll set up the systemd service later for permanent running

# Exit oracle user to return to root for firewall configuration
exit
```

### Step 10: Configure Firewall

```bash
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --add-port=1521/tcp --permanent
sudo firewall-cmd --reload

# Verify ports are open
sudo firewall-cmd --list-ports
```

### Step 11: Create APEX Admin User (MANDATORY)

**The admin user often doesn't get created during installation - this step is REQUIRED:**

```bash
# Check if admin user exists
sqlplus 'sys/Password123@localhost:1521/FREEPDB1' as sysdba
SELECT user_name, account_locked 
FROM apex_240200.wwv_flow_fnd_user 
WHERE security_group_id = 10;

# If no users found (common), create admin:
quit

cd /home/oracle/apex
sqlplus 'sys/Password123@localhost:1521/FREEPDB1' as sysdba
@apxchpwd.sql
# Username: ADMIN (press Enter)
# Email: ADMIN (press Enter)
# Password: Password1! (MUST include special characters)
```

### Step 12: Set Up Systemd Service (Production)

```bash
sudo tee /etc/systemd/system/ords.service > /dev/null << 'EOF'
[Unit]
Description=Oracle REST Data Services
After=network.target oracle-free-23ai.service
Requires=oracle-free-23ai.service

[Service]
Type=simple
User=oracle
Group=oinstall
WorkingDirectory=/opt/oracle/ords
ExecStart=/opt/oracle/ords/bin/ords --config /opt/ords-config serve
Restart=always
RestartSec=10
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk"
Environment="ORACLE_HOME=/opt/oracle/product/23ai/dbhomeFree"
Environment="ORACLE_SID=FREE"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable ords
sudo systemctl start ords

# Verify service is running
sudo systemctl status ords
# Should show: active (running)

# Verify ORDS is listening on port 8080
sudo netstat -tlnp | grep 8080
# Should show: java process listening on port 8080

# OPTIONAL: Configure ORDS to serve APEX at root path (/)
# This allows clean URLs like https://yourdomain.com instead of https://yourdomain.com/ords/apex
sudo -u oracle /opt/oracle/ords/bin/ords --config /opt/ords-config config set standalone.context.path /
sudo systemctl restart ords
```

## 🌐 Access APEX

**URL:** `http://your-server-ip:8080/ords/apex`

**Login Credentials:**
- **Workspace:** `INTERNAL`
- **Username:** `ADMIN`
- **Password:** `Password1!`

## 🔧 Common Issues & Solutions - REAL-WORLD TESTED

### Connection Refused (curl fails)
**Symptoms:** `curl: (7) Failed to connect to server`

**Solutions:**
```bash
# 1. Check firewall (most common)
sudo firewall-cmd --list-ports
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload

# 2. Check if ORDS is running
ps aux | grep ords
netstat -tlnp | grep 8080

# 3. Start ORDS if stopped
sudo su - oracle
ords --config /opt/ords-config serve
```

### APEX Login Page Loads But Buttons Don't Work
**Symptoms:** Page loads but clicking buttons does nothing
**Solution:** Static files not configured

```bash
sudo mkdir -p /opt/ords-config/global/doc_root/i
sudo su - oracle
cp -r /home/oracle/apex/images/* /opt/ords-config/global/doc_root/i/
# Restart ORDS
```

### Account Locked Errors
**Symptoms:** `"AccountIsLocked"` JSON error
**Solution:** Unlock database accounts (happens every install)

```bash
sqlplus 'sys/Password123@localhost:1521/FREEPDB1' as sysdba
ALTER USER APEX_PUBLIC_USER ACCOUNT UNLOCK;
ALTER USER ORDS_PUBLIC_USER ACCOUNT UNLOCK;
ALTER USER APEX_240200 ACCOUNT UNLOCK;
```

### APEX Admin Login Fails
**Symptoms:** Invalid credentials on APEX login
**Solution:** Admin user doesn't exist (very common)

```bash
# Check if admin user exists
sqlplus 'sys/Password123@localhost:1521/FREEPDB1' as sysdba
SELECT user_name FROM apex_240200.wwv_flow_fnd_user;

# If no admin user, create one:
cd /home/oracle/apex
@apxchpwd.sql
# Password: Password1! (MUST contain special characters)
```

## ✅ BATTLE-TESTED Results

**Successfully Completed Installations:**
- ✅ Oracle Linux 9 on Proxmox VM
- ✅ Fresh OS installs without GUI
- ✅ Real network environments (192.168.x.x)
- ✅ Multiple failure scenarios tested and resolved

**Critical Success Factors:**
1. **Account unlocking is mandatory** - they always get locked
2. **APEX admin creation is separate step** - often fails during install
3. **Static files are essential** - UI won't work without them
4. **Firewall rules get lost** - must be re-added
5. **ORDS serve mode must be started manually** after install

**Total Real-World Installation Time:** 2-3 hours including troubleshooting

---

**Contributors:** Live deployment testing and troubleshooting session
**Validation Status:** ✅ BATTLE-TESTED - Multiple successful deployments
**Last Updated:** July 29, 2025
**Environment:** Oracle Linux 9, Proxmox VM, Real network testing

---

## 🌐 Reverse Proxy Setup (Optional - For Clean URLs)

If you want to access APEX through a clean domain name (like `apex.yourdomain.com` instead of `http://server-ip:8080/ords/apex`), you can set up a reverse proxy.

### Using Nginx Proxy Manager

**Step 1: Configure ORDS for Root Path (Recommended)**
```bash
# Configure ORDS to serve APEX at root path instead of /ords/apex
sudo -u oracle /opt/oracle/ords/bin/ords --config /opt/ords-config config set standalone.context.path /
sudo systemctl restart ords
```

**Step 2: Create Proxy Host in NPM**
- **Domain Names:** `apex.yourdomain.com`
- **Scheme:** `http`
- **Forward Hostname/IP:** `192.168.x.x` (your APEX server IP)
- **Forward Port:** `8080`
- **Websockets Support:** `ON`
- **Block Common Exploits:** `ON`

**Step 3: Add DNS Record**
Add an A record for `apex.yourdomain.com` pointing to your Nginx Proxy Manager server.

**Result:** Users can access APEX directly at `https://apex.yourdomain.com`

### Alternative: Traditional Path Setup
If you prefer the traditional `/ords/apex` path:
- Skip the context path configuration above
- Users access APEX at `https://apex.yourdomain.com/ords/apex`
- **Important:** ORDS is picky about trailing slashes - use `/ords/apex` (no trailing slash)
