# Complete Odoo 17 Setup: Production Guide + Virtual Environment (WSL)

This guide combines the robust Cybrosys production setup with virtual environment best practices for a safe development environment.

## Step 1: Update WSL System
```bash
sudo apt update
sudo apt upgrade -y
```

## Step 2: Install System Dependencies
```bash
# Security packages
sudo apt install -y openssh-server fail2ban

# Add deadsnakes PPA for newer Python versions
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update

# Install Python 3.11 (required for Odoo 17)
sudo apt install -y python3.11 python3.11-venv python3.11-dev python3.11-distutils
sudo apt install -y python3-pip

# System libraries (from Cybrosys guide)
sudo apt install -y libxml2-dev libxslt1-dev zlib1g-dev libsasl2-dev libldap2-dev 
sudo apt install -y build-essential libssl-dev libffi-dev libmysqlclient-dev 
sudo apt install -y libjpeg-dev libpq-dev libjpeg8-dev liblcms2-dev libblas-dev libatlas-base-dev

# Node.js and npm for CSS/JS processing
sudo apt install -y npm nodejs
sudo ln -s /usr/bin/nodejs /usr/bin/node
sudo npm install -g less less-plugin-clean-css
sudo apt install -y node-less

# Git for version control
sudo apt install -y git
```

## Step 3: Setup PostgreSQL Database
```bash
# Install PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Switch to postgres user and create Odoo database user
sudo su - postgres

# Create database user (you'll be prompted for password)
createuser --createdb --username postgres --no-createrole --no-superuser --pwprompt odoo17

# Grant superuser privileges
psql
ALTER USER odoo17 WITH SUPERUSER;
\q
exit
```

## Step 4: Create System User (Production Style)
```bash
# Create dedicated system user for Odoo
sudo adduser --system --home=/opt/odoo17 --group odoo17
```

## Step 5: Get Odoo 17 Source Code
```bash
# Switch to odoo17 user
sudo su - odoo17 -s /bin/bash

# Clone Odoo 17 source code
git clone https://www.github.com/odoo/odoo --depth 1 --branch 17.0 --single-branch .

# Exit back to your user
exit
```

## Step 6: Create Virtual Environment with Python 3.11
```bash
# Switch to odoo17 user
sudo su - odoo17 -s /bin/bash

# Create virtual environment with Python 3.11
python3.11 -m venv venv

# Activate virtual environment
source venv/bin/activate

# Upgrade pip in virtual environment
pip install --upgrade pip

# Install Odoo requirements in virtual environment
pip install -r requirements.txt

# Verify installation
python -c "import psycopg2, lxml, PIL, gevent; print('All packages installed successfully!')"

# Exit back to your user
exit
```

## Step 7: Install Wkhtmltopdf (PDF Reports)
```bash
# Download and install wkhtmltopdf
sudo wget https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.5/wkhtmltox_0.12.5-1.bionic_amd64.deb
sudo dpkg -i wkhtmltox_0.12.5-1.bionic_amd64.deb
sudo apt install -f
```

## Step 8: Setup Configuration File
```bash
# Copy example config file
sudo cp /opt/odoo17/debian/odoo.conf /etc/odoo17.conf

# Edit configuration
sudo nano /etc/odoo17.conf
```

**Configuration content:**
```ini
[options]
; Master password for database operations
admin_passwd = your_admin_password_here
db_host = False
db_port = False
db_user = odoo17
db_password = your_db_password_here
addons_path = /opt/odoo17/addons
logfile = /var/log/odoo/odoo17.log

; Additional recommended settings
xmlrpc_port = 8069
longpolling_port = 8072
proxy_mode = False
workers = 2
max_cron_threads = 1
```

```bash
# Set proper permissions
sudo chown odoo17: /etc/odoo17.conf
sudo chmod 640 /etc/odoo17.conf

# Create log directory
sudo mkdir /var/log/odoo
sudo chown odoo17:root /var/log/odoo
```

## Step 9: Create Systemd Service (Modified for Virtual Environment)
```bash
sudo nano /etc/systemd/system/odoo17.service
```

**Service file content (modified to use virtual environment):**
```ini
[Unit]
Description=Odoo17
Documentation=http://www.odoo.com
After=postgresql.service

[Service]
Type=simple
User=odoo17
Group=odoo17
ExecStart=/opt/odoo17/venv/bin/python /opt/odoo17/odoo-bin -c /etc/odoo17.conf
WorkingDirectory=/opt/odoo17
Environment=PATH="/opt/odoo17/venv/bin:$PATH"
Environment=VIRTUAL_ENV="/opt/odoo17/venv"
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
# Set service permissions
sudo chmod 755 /etc/systemd/system/odoo17.service
sudo chown root: /etc/systemd/system/odoo17.service

# Reload systemd
sudo systemctl daemon-reload
```

## Step 10: Start and Enable Odoo Service
```bash
# Start Odoo service
sudo systemctl start odoo17.service

# Check status
sudo systemctl status odoo17.service

# Enable auto-start on boot
sudo systemctl enable odoo17.service

# View logs
sudo tail -f /var/log/odoo/odoo17.log
```

## Step 11: VS Code Integration for Your Windows Project

Since your project files are on Windows, create this setup:

**VS Code launch.json (for debugging):**
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug Odoo 17 (WSL + VirtualEnv)",
            "type": "python",
            "request": "launch",
            "program": "${workspaceFolder}/odoo-bin",
            "python": "/opt/odoo17/venv/bin/python",
            "args": [
                "-c", "/etc/odoo17.conf",
                "--dev=xml,reload,qweb,werkzeug,all"
            ],
            "console": "integratedTerminal",
            "justMyCode": false,
            "env": {
                "VIRTUAL_ENV": "/opt/odoo17/venv",
                "PATH": "/opt/odoo17/venv/bin:$PATH"
            }
        }
    ]
}
```

## Step 12: Development Workflow Commands

**To work on your Windows project with WSL backend:**
```bash
# In VS Code terminal, switch to WSL
wsl

# Stop the service for development
sudo systemctl stop odoo17.service

# Navigate to your Windows project
cd /mnt/c/Users/YourName/Projects/Odoo17/

# Activate virtual environment
source /opt/odoo17/venv/bin/activate

# Run Odoo in development mode
python odoo-bin -c /etc/odoo17.conf --dev=all

# Or run with custom database
python odoo-bin --addons-path=addons,custom_addons --database=dev_db --dev=all
```

## Benefits of This Merged Approach:

✅ **Production-Ready**: Follows industry best practices  
✅ **Safe Dependencies**: Virtual environment prevents conflicts  
✅ **System Integration**: Proper systemd service, logging, users  
✅ **Development Friendly**: Can easily switch between service and dev mode  
✅ **VS Code Compatible**: Works with your Windows project files  
✅ **Scalable**: Easy to add custom addons, enterprise modules  
✅ **Maintainable**: Clean upgrade path for future Odoo versions  

## Quick Commands Reference:
```bash
# Start/Stop/Restart service
sudo systemctl start/stop/restart odoo17.service

# View logs
sudo tail -f /var/log/odoo/odoo17.log

# Development mode
sudo systemctl stop odoo17.service
source /opt/odoo17/venv/bin/activate
cd /mnt/c/path/to/your/project
python odoo-bin --dev=all

# Install additional Python packages
sudo su - odoo17 -s /bin/bash
source venv/bin/activate
pip install package_name
exit
```

This setup gives you the **best of both worlds**: production-grade reliability with development flexibility and dependency isolation!