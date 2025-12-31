# Complete Setup Guide: Antigravity Claude Proxy on AWS VPS

A comprehensive step-by-step guide to deploy and use the [Antigravity Claude Proxy](https://github.com/badri-s2001/antigravity-claude-proxy) on an AWS VPS (or any Linux server).

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Server Setup](#server-setup)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running the Service](#running-the-service)
- [Testing](#testing)
- [Web Interface](#web-interface)
- [Claude Code CLI Integration](#claude-code-cli-integration)
- [Troubleshooting](#troubleshooting)
- [Maintenance](#maintenance)

---

## 🎯 Overview

This guide will help you:
- Set up the Antigravity Claude Proxy on an AWS EC2 instance
- Configure Google OAuth authentication
- Access Claude and Gemini models through a proxy API
- Use the proxy with Claude Code CLI
- Create a web interface for easy interaction

**What this proxy does:**
- Exposes Claude and Gemini models via Anthropic-compatible API
- Uses Google OAuth tokens from your accounts
- Supports multi-account load balancing
- Enables prompt caching and streaming

---

## ✅ Prerequisites

### Server Requirements
- **OS**: Ubuntu, Debian, or Kali Linux
- **RAM**: Minimum 1GB (2GB recommended)
- **Storage**: 10GB free space
- **Instance Type**: t2.micro or t3.small (AWS)

### Software Requirements
- Node.js 18 or later
- npm (comes with Node.js)
- Git

### AWS Configuration
- EC2 instance running
- Security group configured (see below)

---

## 🚀 Server Setup

### 1. Launch AWS EC2 Instance

1. **Launch Instance:**
   - AMI: Ubuntu 22.04 LTS or Kali Linux
   - Instance Type: t3.small (or t2.micro for testing)
   - Storage: 20GB gp3

2. **Configure Security Group:**

   Add these **Inbound Rules**:
   
   | Type       | Protocol | Port Range | Source    | Description           |
   |------------|----------|------------|-----------|-----------------------|
   | SSH        | TCP      | 22         | 0.0.0.0/0 | SSH access            |
   | Custom TCP | TCP      | 8080       | 0.0.0.0/0 | Proxy server          |
   | UDP        | UDP      | 0-65535    | 0.0.0.0/0 | Optional              |

   ⚠️ **Security Note**: For production, restrict port 8080 to your IP only instead of 0.0.0.0/0

3. **Note Your Public IP:**
   - Find it in EC2 console (e.g., `10.10.10.10`)
   - This will be your `SERVER_IP`

### 2. Connect to Your Server

```bash
# SSH into your server
ssh username@YOUR_SERVER_IP

# For Kali Linux
ssh kali@YOUR_SERVER_IP

# For Ubuntu
ssh ubuntu@YOUR_SERVER_IP
```

### 3. Install Node.js

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 18+
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Verify installation
node --version  # Should show v18.x.x or higher
npm --version
```

### 4. Install Git (if not installed)

```bash
sudo apt install git -y
```

---

## 📦 Installation

### Clone the Repository

```bash
# Create directory
sudo mkdir -p /opt/antigravity
cd /opt/antigravity

# Clone repository
sudo git clone https://github.com/badri-s2001/antigravity-claude-proxy.git
cd antigravity-claude-proxy

# Set permissions
sudo chown -R $USER:$USER /opt/antigravity

# Install dependencies
npm install
```

---

## ⚙️ Configuration

### Add Google Account(s)

You need to add at least one Google account for authentication:

```bash
# Interactive account management
npx antigravity-claude-proxy accounts

# Or add account directly
npx antigravity-claude-proxy accounts add
```

**This will:**
1. Open your browser for Google OAuth
2. Ask you to sign in with your Google account
3. Save the OAuth token locally

**For multiple accounts (load balancing):**
```bash
# Add additional accounts
npx antigravity-claude-proxy accounts add
npx antigravity-claude-proxy accounts add
```

**Verify accounts:**
```bash
# List all accounts
npx antigravity-claude-proxy accounts list

# Check account status
npx antigravity-claude-proxy accounts verify
```

---

## 🏃 Running the Service

### Option 1: Direct Run (Testing)

```bash
cd /opt/antigravity/antigravity-claude-proxy

# Start the server
node bin/cli.js start

# Server will start on http://localhost:8080
# Keep this terminal open
```

**Test it works:**
```bash
# In another SSH session
curl http://localhost:8080/health
```

### Option 2: Screen (Recommended for VPS)

Screen keeps the server running even after you disconnect SSH.

```bash
# Install screen
sudo apt install screen -y

# Start a screen session
screen -S antigravity

# Run the server
cd /opt/antigravity/antigravity-claude-proxy
node bin/cli.js start

# Detach from screen: Press Ctrl+A then D
```

**Screen commands:**
```bash
# List all screen sessions
screen -ls

# Reattach to session
screen -r antigravity

# Kill a session
screen -X -S antigravity quit
```

### Option 3: PM2 (Production)

PM2 provides auto-restart and monitoring.

```bash
# Install PM2
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
```

**PM2 commands:**
```bash
pm2 status              # Check status
pm2 logs antigravity-proxy  # View logs
pm2 restart antigravity-proxy  # Restart
pm2 stop antigravity-proxy     # Stop
pm2 delete antigravity-proxy   # Remove
```

---

## 🧪 Testing

### Local Testing (on server)

```bash
# Health check
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
```

### Remote Testing (from local machine)

Replace `YOUR_SERVER_IP` with your actual server IP:

**Linux/Mac:**
```bash
# Health check
curl http://YOUR_SERVER_IP:8080/health

# Account status
curl "http://YOUR_SERVER_IP:8080/account-limits?format=table"
```

**Windows PowerShell:**
```powershell
# Health check
Invoke-RestMethod -Uri "http://YOUR_SERVER_IP:8080/health"

# Account limits
Invoke-RestMethod -Uri "http://YOUR_SERVER_IP:8080/account-limits"
```

---

## 🌐 Web Interface

A simple web interface for chatting with the proxy.

### Setup

1. **Create HTML file** on your local machine (`claude-interface.html`):

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Claude Proxy Interface</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }
        
        .container {
            max-width: 1200px;
            margin: 0 auto;
        }
        
        .header {
            background: white;
            border-radius: 12px;
            padding: 24px;
            margin-bottom: 20px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
        }
        
        .header h1 {
            color: #333;
            margin-bottom: 12px;
        }
        
        .server-config {
            display: flex;
            gap: 12px;
            align-items: center;
            flex-wrap: wrap;
        }
        
        .server-config input {
            flex: 1;
            min-width: 250px;
            padding: 10px 14px;
            border: 2px solid #e2e8f0;
            border-radius: 6px;
            font-size: 14px;
        }
        
        .server-config button {
            padding: 10px 20px;
            background: #667eea;
            color: white;
            border: none;
            border-radius: 6px;
            cursor: pointer;
            font-weight: 600;
            transition: background 0.2s;
        }
        
        .server-config button:hover {
            background: #5568d3;
        }
        
        .status-indicator {
            display: inline-block;
            width: 12px;
            height: 12px;
            border-radius: 50%;
            margin-left: 8px;
            background: #94a3b8;
        }
        
        .status-indicator.online {
            background: #10b981;
            box-shadow: 0 0 8px #10b981;
        }
        
        .main-grid {
            display: grid;
            grid-template-columns: 300px 1fr;
            gap: 20px;
        }
        
        .sidebar {
            background: white;
            border-radius: 12px;
            padding: 20px;
            height: fit-content;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
        }
        
        .sidebar h3 {
            color: #333;
            margin-bottom: 16px;
        }
        
        .info-item {
            padding: 12px;
            background: #f8fafc;
            border-radius: 6px;
            margin-bottom: 10px;
        }
        
        .info-label {
            font-size: 12px;
            color: #64748b;
            text-transform: uppercase;
            font-weight: 600;
            margin-bottom: 4px;
        }
        
        .info-value {
            font-size: 14px;
            color: #1e293b;
            font-weight: 500;
        }
        
        .chat-container {
            background: white;
            border-radius: 12px;
            padding: 24px;
            display: flex;
            flex-direction: column;
            height: 600px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
        }
        
        .messages {
            flex: 1;
            overflow-y: auto;
            margin-bottom: 20px;
            padding: 10px;
        }
        
        .message {
            margin-bottom: 16px;
            animation: fadeIn 0.3s;
        }
        
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(10px); }
            to { opacity: 1; transform: translateY(0); }
        }
        
        .message-user {
            text-align: right;
        }
        
        .message-content {
            display: inline-block;
            padding: 12px 16px;
            border-radius: 12px;
            max-width: 80%;
            word-wrap: break-word;
        }
        
        .message-user .message-content {
            background: #667eea;
            color: white;
        }
        
        .message-assistant .message-content {
            background: #f1f5f9;
            color: #1e293b;
        }
        
        .message-system {
            text-align: center;
            color: #64748b;
            font-size: 13px;
            font-style: italic;
        }
        
        .input-area {
            display: flex;
            gap: 12px;
        }
        
        .input-area textarea {
            flex: 1;
            padding: 12px;
            border: 2px solid #e2e8f0;
            border-radius: 8px;
            resize: none;
            font-family: inherit;
            font-size: 14px;
        }
        
        .input-area textarea:focus {
            outline: none;
            border-color: #667eea;
        }
        
        .input-area button {
            padding: 12px 24px;
            background: #667eea;
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-weight: 600;
            transition: background 0.2s;
        }
        
        .input-area button:hover:not(:disabled) {
            background: #5568d3;
        }
        
        .input-area button:disabled {
            background: #94a3b8;
            cursor: not-allowed;
        }
        
        .model-selector {
            margin-bottom: 16px;
        }
        
        .model-selector select {
            width: 100%;
            padding: 10px;
            border: 2px solid #e2e8f0;
            border-radius: 6px;
            font-size: 14px;
            background: white;
        }
        
        .action-buttons {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 8px;
            margin-top: 16px;
        }
        
        .action-buttons button {
            padding: 8px 12px;
            background: #f1f5f9;
            color: #475569;
            border: none;
            border-radius: 6px;
            cursor: pointer;
            font-size: 13px;
            transition: background 0.2s;
        }
        
        .action-buttons button:hover {
            background: #e2e8f0;
        }

        @media (max-width: 768px) {
            .main-grid {
                grid-template-columns: 1fr;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>Claude Proxy Interface <span class="status-indicator" id="statusIndicator"></span></h1>
            <div class="server-config">
                <input type="text" id="serverUrl" placeholder="Server URL (e.g., http://YOUR_SERVER_IP:8080)" value="http://YOUR_SERVER_IP:8080">
                <button onclick="connectServer()">Connect</button>
                <button onclick="checkHealth()">Check Health</button>
            </div>
        </div>
        
        <div class="main-grid">
            <div class="sidebar">
                <h3>Server Info</h3>
                <div class="info-item">
                    <div class="info-label">Status</div>
                    <div class="info-value" id="serverStatus">Disconnected</div>
                </div>
                <div class="info-item">
                    <div class="info-label">Accounts</div>
                    <div class="info-value" id="accountCount">-</div>
                </div>
                <div class="info-item">
                    <div class="info-label">Available</div>
                    <div class="info-value" id="availableCount">-</div>
                </div>
                <div class="info-item">
                    <div class="info-label">Rate Limited</div>
                    <div class="info-value" id="rateLimitedCount">-</div>
                </div>
                
                <div class="action-buttons">
                    <button onclick="getAccountLimits()">Account Limits</button>
                    <button onclick="refreshToken()">Refresh Token</button>
                    <button onclick="clearChat()">Clear Chat</button>
                    <button onclick="listModels()">List Models</button>
                </div>
            </div>
            
            <div class="chat-container">
                <div class="model-selector">
                    <select id="modelSelect">
                        <option value="claude-sonnet-4-5">Claude Sonnet 4.5</option>
                        <option value="claude-sonnet-4-5-thinking">Claude Sonnet 4.5 (Thinking)</option>
                        <option value="claude-opus-4-5-thinking">Claude Opus 4.5 (Thinking)</option>
                        <option value="gemini-3-flash">Gemini 3 Flash</option>
                        <option value="gemini-3-pro-high">Gemini 3 Pro High</option>
                    </select>
                </div>
                
                <div class="messages" id="messages"></div>
                
                <div class="input-area">
                    <textarea id="messageInput" placeholder="Type your message here..." rows="3"></textarea>
                    <button onclick="sendMessage()" id="sendButton">Send</button>
                </div>
            </div>
        </div>
    </div>

    <script>
        let serverUrl = 'http://YOUR_SERVER_IP:8080';
        let conversationHistory = [];
        
        async function connectServer() {
            const urlInput = document.getElementById('serverUrl');
            serverUrl = urlInput.value.trim();
            
            if (!serverUrl.startsWith('http')) {
                serverUrl = 'http://' + serverUrl;
            }
            
            await checkHealth();
        }
        
        async function checkHealth() {
            try {
                const response = await fetch(`${serverUrl}/health`);
                const data = await response.json();
                
                if (data.status === 'ok') {
                    document.getElementById('statusIndicator').classList.add('online');
                    document.getElementById('serverStatus').textContent = 'Connected';
                    addSystemMessage('Connected to server successfully!');
                    await getAccountLimits();
                }
            } catch (err) {
                document.getElementById('statusIndicator').classList.remove('online');
                document.getElementById('serverStatus').textContent = 'Disconnected';
                addSystemMessage('Failed to connect: ' + err.message);
            }
        }
        
        async function getAccountLimits() {
            try {
                const response = await fetch(`${serverUrl}/account-limits`);
                const data = await response.json();
                
                document.getElementById('accountCount').textContent = data.total || 0;
                document.getElementById('availableCount').textContent = data.available || 0;
                document.getElementById('rateLimitedCount').textContent = data.rateLimited || 0;
                
                addSystemMessage(`Account Status: ${data.available} available, ${data.rateLimited} rate-limited`);
            } catch (err) {
                addSystemMessage('Failed to get account limits: ' + err.message);
            }
        }
        
        async function refreshToken() {
            try {
                const response = await fetch(`${serverUrl}/refresh-token`, { method: 'POST' });
                const data = await response.json();
                addSystemMessage('Token refreshed successfully');
                await getAccountLimits();
            } catch (err) {
                addSystemMessage('Failed to refresh token: ' + err.message);
            }
        }
        
        async function listModels() {
            try {
                const response = await fetch(`${serverUrl}/v1/models`);
                const data = await response.json();
                const models = data.data.map(m => m.id).join(', ');
                addSystemMessage('Available models: ' + models);
            } catch (err) {
                addSystemMessage('Failed to list models: ' + err.message);
            }
        }
        
        function clearChat() {
            conversationHistory = [];
            document.getElementById('messages').innerHTML = '';
            addSystemMessage('Chat cleared');
        }
        
        async function sendMessage() {
            const input = document.getElementById('messageInput');
            const message = input.value.trim();
            
            if (!message) return;
            
            const model = document.getElementById('modelSelect').value;
            const sendButton = document.getElementById('sendButton');
            
            addUserMessage(message);
            input.value = '';
            sendButton.disabled = true;
            sendButton.textContent = 'Sending...';
            
            conversationHistory.push({
                role: 'user',
                content: message
            });
            
            try {
                const response = await fetch(`${serverUrl}/v1/messages`, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                        'x-api-key': 'test',
                        'anthropic-version': '2023-06-01'
                    },
                    body: JSON.stringify({
                        model: model,
                        max_tokens: 4096,
                        messages: conversationHistory
                    })
                });
                
                const data = await response.json();
                
                if (data.content && data.content[0]) {
                    const assistantMessage = data.content[0].text;
                    addAssistantMessage(assistantMessage);
                    
                    conversationHistory.push({
                        role: 'assistant',
                        content: assistantMessage
                    });
                } else {
                    addSystemMessage('Error: Invalid response from server');
                }
            } catch (err) {
                addSystemMessage('Error: ' + err.message);
            } finally {
                sendButton.disabled = false;
                sendButton.textContent = 'Send';
            }
        }
        
        function addUserMessage(text) {
            const messagesDiv = document.getElementById('messages');
            const messageDiv = document.createElement('div');
            messageDiv.className = 'message message-user';
            messageDiv.innerHTML = `<div class="message-content">${escapeHtml(text)}</div>`;
            messagesDiv.appendChild(messageDiv);
            messagesDiv.scrollTop = messagesDiv.scrollHeight;
        }
        
        function addAssistantMessage(text) {
            const messagesDiv = document.getElementById('messages');
            const messageDiv = document.createElement('div');
            messageDiv.className = 'message message-assistant';
            messageDiv.innerHTML = `<div class="message-content">${escapeHtml(text)}</div>`;
            messagesDiv.appendChild(messageDiv);
            messagesDiv.scrollTop = messagesDiv.scrollHeight;
        }
        
        function addSystemMessage(text) {
            const messagesDiv = document.getElementById('messages');
            const messageDiv = document.createElement('div');
            messageDiv.className = 'message message-system';
            messageDiv.textContent = text;
            messagesDiv.appendChild(messageDiv);
            messagesDiv.scrollTop = messagesDiv.scrollHeight;
        }
        
        function escapeHtml(text) {
            const div = document.createElement('div');
            div.textContent = text;
            return div.innerHTML.replace(/\n/g, '<br>');
        }
        
        document.getElementById('messageInput').addEventListener('keydown', function(e) {
            if (e.key === 'Enter' && !e.shiftKey) {
                e.preventDefault();
                sendMessage();
            }
        });
        
        window.addEventListener('load', () => {
            addSystemMessage('Welcome! Enter your server URL and click Connect to begin.');
        });
    </script>
</body>
</html>
```

2. **Replace `YOUR_SERVER_IP`** with your actual server IP in two places:
   - Line with `value="http://YOUR_SERVER_IP:8080"`
   - Line with `let serverUrl = 'http://YOUR_SERVER_IP:8080';`

3. **Open the HTML file** in your browser

4. **Click "Connect"** and start chatting!

---

## 🔧 Claude Code CLI Integration

Use the proxy with Claude Code CLI for AI-powered coding assistance.

### Prerequisites

Install Claude Code CLI from [Anthropic's website](https://claude.ai/download).

### Configuration

**1. Create Claude Code settings file:**

```bash
# Linux/Mac
mkdir -p ~/.claude
nano ~/.claude/settings.json

# Windows
mkdir %USERPROFILE%\.claude
notepad %USERPROFILE%\.claude\settings.json
```

**2. Add configuration:**

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "test",
    "ANTHROPIC_BASE_URL": "http://YOUR_SERVER_IP:8080",
    "ANTHROPIC_MODEL": "claude-opus-4-5-thinking",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-5-thinking",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-5-thinking",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-sonnet-4-5",
    "CLAUDE_CODE_SUBAGENT_MODEL": "claude-sonnet-4-5-thinking"
  }
}
```

Replace `YOUR_SERVER_IP` with your actual server IP.

**3. Set environment variables:**

**Linux/Mac (Bash):**
```bash
echo 'export ANTHROPIC_BASE_URL="http://YOUR_SERVER_IP:8080"' >> ~/.bashrc
echo 'export ANTHROPIC_API_KEY="test"' >> ~/.bashrc
source ~/.bashrc
```

**Linux/Mac (Zsh):**
```bash
echo 'export ANTHROPIC_BASE_URL="http://YOUR_SERVER_IP:8080"' >> ~/.zshrc
echo 'export ANTHROPIC_API_KEY="test"' >> ~/.zshrc
source ~/.zshrc
```

**Windows (PowerShell):**
```powershell
[Environment]::SetEnvironmentVariable('ANTHROPIC_BASE_URL', 'http://YOUR_SERVER_IP:8080', 'User')
[Environment]::SetEnvironmentVariable('ANTHROPIC_API_KEY', 'test', 'User')
```

**4. Handle onboarding (if needed):**

If Claude Code asks for login, create/edit `~/.claude.json`:

```json
{
  "hasCompletedOnboarding": true
}
```

**5. Run Claude Code:**

```bash
claude
```

---

## 🔍 Troubleshooting

### Port 8080 Already in Use

```bash
# Find process using port 8080
sudo netstat -tulpn | grep 8080

# Kill the process
sudo kill -9 <PID>

# Or kill all node processes
pkill -9 node
```

### Cannot Connect Remotely

1. **Check server is running:**
   ```bash
   curl http://localhost:8080/health
   ```

2. **Check AWS Security Group:**
   - Ensure port 8080 is open in inbound rules
   - Source should be 0.0.0.0/0 or your IP

3. **Check firewall on server:**
   ```bash
   sudo ufw status
   sudo ufw allow 8080/tcp
   ```

### Token Expired / 401 Errors

```bash
# Refresh token via CLI
curl -X POST http://localhost:8080/refresh-token

# Or re-authenticate
npx antigravity-claude-proxy accounts
# Select "Re-authenticate" for the account
```

### Account Shows as Invalid

```bash
# List accounts
npx antigravity-claude-proxy accounts list

# Re-authenticate
npx antigravity-claude-proxy accounts
# Choose the invalid account and re-authenticate
```

### Server Crashes or Stops

```bash
# If using screen, reattach
screen -r antigravity

# If using PM2, check logs
pm2 logs antigravity-proxy --lines 50

# Restart
pm2 restart antigravity-proxy
```

### Rate Limiting (429 Errors)

With multiple accounts, the proxy automatically switches. With one account:

1. **Wait for rate limit to reset** (check account limits endpoint)
2. **Add more accounts** for load balancing:
   ```bash
   npx antigravity-claude-proxy accounts add
   ```

---

## 🔧 Maintenance

### Update the Proxy

```bash
cd /opt/antigravity/antigravity-claude-proxy

# Stop the service
screen -X -S antigravity quit  # If using screen
pm2 stop antigravity-proxy     # If using PM2

# Pull latest changes
git pull

# Update dependencies
npm install

# Restart service
screen -S antigravity          # If using screen
node bin/cli.js start
# Then Ctrl+A, D to detach

pm2 restart antigravity-proxy  # If using PM2
```

### View Logs

```bash
# If using screen
screen -r antigravity

# If using PM2
pm2 logs antigravity-proxy --lines 100

# Or check log files directly
tail -f ~/.pm2/logs/antigravity-proxy-out.log
tail -f ~/.pm2/logs/antigravity-proxy-error.log
```

### Monitor Server Health

```bash
# Account status
curl "http://localhost:8080/account-limits?format=table"

# System resources (if using PM2)
pm2 monit
```

### Backup Configuration

```bash
# Backup account credentials
cp -r ~/.config/antigravity-claude-proxy ~/backup-antigravity

# Or tar it
tar -czf antigravity-backup.tar.gz ~/.config/antigravity-claude-proxy
```

---

## 📊 Available Models

### Claude Models

| Model ID | Description |
|----------|-------------|
| `claude-sonnet-4-5` | Claude Sonnet 4.5 (fast, efficient) |
| `claude-sonnet-4-5-thinking` | Claude Sonnet 4.5 with extended thinking |
| `claude-opus-4-5-thinking` | Claude Opus 4.5 with extended thinking |

### Gemini Models

| Model ID | Description |
|----------|-------------|
| `gemini-3-flash` | Gemini 3 Flash with thinking |
| `gemini-3-pro-low` | Gemini 3 Pro Low with thinking |
| `gemini-3-pro-high` | Gemini 3 Pro High with thinking |

---

## 🔐 Security Best Practices

1. **Restrict port 8080 to your IP only** in AWS Security Group
2. **Don't share your OAuth tokens** or account credentials
3. **Use SSH key authentication** instead of passwords
4. **Keep the system updated:**
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```
5. **Monitor access logs** regularly
6. **Use strong passwords** for your Google accounts

---

## 📝 API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Health check |
| `/account-limits` | GET | Account status and quota limits |
| `/v1/messages` | POST | Anthropic Messages API |
| `/v1/models` | GET | List available models |
| `/refresh-token` | POST | Force token refresh |

---

## 🎓 Common Use Cases

### 1. Personal AI Assistant
Use the web interface for daily tasks, questions, and brainstorming.

### 2. Development with Claude Code
Integrate with Claude Code CLI for AI-powered coding assistance.

### 3. API Integration
Use the Anthropic-compatible API in your own applications.

### 4. Multi-Account Load Balancing
Add multiple Google accounts to distribute requests and avoid rate limits.

---

## 📚 Additional Resources

- **Original Repository**: [antigravity-claude-proxy](https://github.com/badri-s2001/antigravity-claude-proxy)
- **Claude Code CLI**: [Anthropic Claude](https://claude.ai/download)
- **Anthropic API Docs**: [docs.anthropic.com](https://docs.anthropic.com)

---

## ⚠️ Disclaimer

This proxy:
- Is for personal/educational use only
- May violate Terms of Service of AI providers
- Could result in account suspension or bans
- Provides no guarantees or warranties

**Use at your own risk.** The authors are not responsible for any consequences.

---

## 🤝 Contributing

Found an issue? Have a suggestion? Please:
1. Check existing issues on GitHub
2. Create a new issue with details
3. Submit a pull request if you have a fix

---

## 📄 License

MIT License - see the original repository for details.

---

## 🙏 Credits

- Original project by [badri-s2001](https://github.com/badri-s2001)
- Based on insights from [opencode-antigravity-auth](https://github.com/OpenCodeDev/opencode-antigravity-auth)
- Inspired by [claude-code-proxy](https://github.com/yourusername/claude-code-proxy)

---

## ✅ Quick Start Checklist

- [ ] AWS EC2 instance launched
- [ ] Security group configured (port 8080 open)
- [ ] SSH access working
- [ ] Node.js 18+ installed
- [ ] Repository cloned
- [ ] Google account(s) added
- [ ] Server running (screen or PM2)
- [ ] Health check passes locally
- [ ] Health check passes remotely
- [ ] Web interface working
- [ ] Claude Code CLI configured (optional)

---

**Need help?** Check the [Troubleshooting](#troubleshooting) section or open an issue on GitHub.

**Happy coding! 🚀**
