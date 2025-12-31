Complete Setup Guide: Antigravity Claude Proxy on AWS VPS
A comprehensive step-by-step guide to deploy and use the Antigravity Claude Proxy on an AWS VPS (or any Linux server).
📋 Table of Contents

Overview
Prerequisites
Server Setup
Installation
Configuration
Running the Service
Testing
Web Interface
Claude Code CLI Integration
Troubleshooting
Maintenance


🎯 Overview
This guide will help you:

Set up the Antigravity Claude Proxy on an AWS EC2 instance
Configure Google OAuth authentication
Access Claude and Gemini models through a proxy API
Use the proxy with Claude Code CLI
Create a web interface for easy interaction

What this proxy does:

Exposes Claude and Gemini models via Anthropic-compatible API
Uses Google OAuth tokens from your accounts
Supports multi-account load balancing
Enables prompt caching and streaming


✅ Prerequisites
Server Requirements

OS: Ubuntu, Debian, or Kali Linux
RAM: Minimum 1GB (2GB recommended)
Storage: 10GB free space
Instance Type: t2.micro or t3.small (AWS)

Software Requirements

Node.js 18 or later
npm (comes with Node.js)
Git

AWS Configuration

EC2 instance running
Security group configured (see below)


🚀 Server Setup
1. Launch AWS EC2 Instance

Launch Instance:

AMI: Ubuntu 22.04 LTS or Kali Linux
Instance Type: t3.small (or t2.micro for testing)
Storage: 20GB gp3


Configure Security Group:
Add these Inbound Rules:
TypeProtocolPort RangeSourceDescriptionSSHTCP220.0.0.0/0SSH accessCustom TCPTCP80800.0.0.0/0Proxy serverUDPUDP0-655350.0.0.0/0Optional
⚠️ Security Note: For production, restrict port 8080 to your IP only instead of 0.0.0.0/0
Note Your Public IP:

Find it in EC2 console (e.g., 15.207.112.170)
This will be your SERVER_IP



2. Connect to Your Server
bash# SSH into your server
ssh username@YOUR_SERVER_IP

# For Kali Linux
ssh kali@YOUR_SERVER_IP

# For Ubuntu
ssh ubuntu@YOUR_SERVER_IP
3. Install Node.js
bash# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 18+
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Verify installation
node --version  # Should show v18.x.x or higher
npm --version
4. Install Git (if not installed)
bashsudo apt install git -y

📦 Installation
Clone the Repository
bash# Create directory
sudo mkdir -p /opt/antigravity
cd /opt/antigravity

# Clone repository
sudo git clone https://github.com/badri-s2001/antigravity-claude-proxy.git
cd antigravity-claude-proxy

# Set permissions
sudo chown -R $USER:$USER /opt/antigravity

# Install dependencies
npm install

⚙️ Configuration
Add Google Account(s)
You need to add at least one Google account for authentication:
bash# Interactive account management
npx antigravity-claude-proxy accounts

# Or add account directly
npx antigravity-claude-proxy accounts add
This will:

Open your browser for Google OAuth
Ask you to sign in with your Google account
Save the OAuth token locally

For multiple accounts (load balancing):
bash# Add additional accounts
npx antigravity-claude-proxy accounts add
npx antigravity-claude-proxy accounts add
Verify accounts:
bash# List all accounts
npx antigravity-claude-proxy accounts list

# Check account status
npx antigravity-claude-proxy accounts verify

🏃 Running the Service
Option 1: Direct Run (Testing)
bashcd /opt/antigravity/antigravity-claude-proxy

# Start the server
node bin/cli.js start

# Server will start on http://localhost:8080
# Keep this terminal open
Test it works:
bash# In another SSH session
curl http://localhost:8080/health
Option 2: Screen (Recommended for VPS)
Screen keeps the server running even after you disconnect SSH.
bash# Install screen
sudo apt install screen -y

# Start a screen session
screen -S antigravity

# Run the server
cd /opt/antigravity/antigravity-claude-proxy
node bin/cli.js start

# Detach from screen: Press Ctrl+A then D
Screen commands:
bash# List all screen sessions
screen -ls

# Reattach to session
screen -r antigravity

# Kill a session
screen -X -S antigravity quit
Option 3: PM2 (Production)
PM2 provides auto-restart and monitoring.
bash# Install PM2
sudo npm install -g pm2

# Create ecosystem file
cd /opt/antigravity/antigravity-claude-proxy
cat > ecosystem.config.js << 'EOF'
module.exports = {
  apps: [{
    name: 'antigravity-proxy',
    script: './bin/cli.js',
    args: 'start',
    instances: 1,
    autorestart: true,
    watch: false,
    max_memory_restart: '1G',
    env: {
      NODE_ENV: 'production'
    }
  }]
}
EOF

# Start with PM2
pm2 start ecosystem.config.js

# Save PM2 configuration
pm2 save

# Enable auto-start on boot
pm2 startup
# Run the command it outputs (usually starts with 'sudo')

# View logs
pm2 logs antigravity-proxy

# Monitor
pm2 monit
PM2 commands:
bashpm2 status              # Check status
pm2 logs antigravity-proxy  # View logs
pm2 restart antigravity-proxy  # Restart
pm2 stop antigravity-proxy     # Stop
pm2 delete antigravity-proxy   # Remove

🧪 Testing
Local Testing (on server)
bash# Health check
curl http://localhost:8080/health

# Account limits
curl "http://localhost:8080/account-limits?format=table"

# List models
curl http://localhost:8080/v1/models

# Test message
curl -X POST http://localhost:8080/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: test" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-5",
    "max_tokens": 1024,
    "messages": [
      {
        "role": "user",
        "content": "Hello! Confirm the proxy is working."
      }
    ]
  }'
Remote Testing (from local machine)
Replace YOUR_SERVER_IP with your actual server IP:
Linux/Mac:
bash# Health check
curl http://YOUR_SERVER_IP:8080/health

# Account status
curl "http://YOUR_SERVER_IP:8080/account-limits?format=table"
Windows PowerShell:
powershell# Health check
Invoke-RestMethod -Uri "http://YOUR_SERVER_IP:8080/health"

# Account limits
Invoke-RestMethod -Uri "http://YOUR_SERVER_IP:
