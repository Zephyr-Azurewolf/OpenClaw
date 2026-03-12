OpenClaw Gateway: Secure Bootstrap & Service Architecture
This repository contains a hardened deployment configuration for the OpenClaw Gateway. It utilizes a "Secret Injection" pattern to ensure that sensitive API credentials never touch persistent storage, residing strictly in volatile RAM (tmpfs) during runtime.

🚀 Key Features
Volatile Secret Management: API keys are fetched from a secure vault (Azure Key Vault) and mounted via kernel-level bind mounts to RAM.

Zero-Disk Leakage: Sensitive JSON profiles are vaporized the moment the service stops, leaving only empty 0-byte anchors on the SSD.

Systemd Integration: Automated lifecycle management including auto-recovery on failure and graceful teardown.

Process Handover: Uses the exec command to replace the shell process with the application, reducing overhead and improving signal handling.

📂 Architecture Overview
Systemd creates a secure RuntimeDirectory at /run/openclaw-vault.

Bootstrap Script fetches the API Key and writes it to the RAM directory.

Bind Mount overlays the RAM file onto the application's expected configuration path.

Application executes, seeing the keys as local files.

On Exit, Systemd unmounts the files and deletes the RAM directory.

🛠️ Installation & Setup
1. Prerequisites
Azure CLI installed and configured with a Managed Identity or Service Principal.

tmpfs support enabled in the Linux kernel.

2. The Bootstrap Script
Save the following as start_gateway.sh and give it execute permissions (chmod +x):

Bash
#!/bin/bash
# Configuration
APP_DIR="/path/to/your/openclaw/agent"
RAM_DIR="/run/openclaw-vault" 
VAULT_NAME="your-keyvault-name"
SECRET_NAME="your-secret-name"

# 1. Fetch & Write to RAM
SECRET_VAL=$(az keyvault secret show --name "$SECRET_NAME" --vault-name "$VAULT_NAME" --query value -o tsv)
AUTH_PAYLOAD="{\"openrouter\": {\"type\": \"api_key\", \"key\": \"$SECRET_VAL\", \"apiKey\": \"$SECRET_VAL\"}}"
echo "$AUTH_PAYLOAD" > "$RAM_DIR/auth-profiles.json"

# 2. Create SSD Anchor & Bind Mount
touch "$APP_DIR/auth-profiles.json"
sudo mount --bind "$RAM_DIR/auth-profiles.json" "$APP_DIR/auth-profiles.json"

# 3. Handover to Application
export OPENROUTER_API_KEY="$SECRET_VAL"
unset SECRET_VAL
exec /usr/bin/node /path/to/openclaw gateway
3. Systemd Service Configuration
Create /etc/systemd/system/openclaw-gateway.service:

Ini, TOML
[Unit]
Description=OpenClaw Gateway Service
After=network-online.target

[Service]
Type=simple
User=youruser
Group=youruser
WorkingDirectory=/home/youruser
RuntimeDirectory=openclaw-vault

ExecStart=/bin/bash /home/youruser/start_gateway.sh

# Cleanup: Unmount the RAM overlay after the process exits
ExecStopPost=/usr/bin/sudo /usr/bin/umount -l /home/youruser/.openclaw/agents/main/agent/auth-profiles.json

Restart=on-failure
SuccessExitStatus=0 143

[Install]
WantedBy=multi-user.target
🛡️ Security Verification
To verify that secrets are not leaking to the disk, stop the service and check the file content:

Bash
sudo systemctl stop openclaw-gateway
cat /path/to/your/openclaw/agent/auth-profiles.json
# Result should be empty (0 bytes)
Would you like me to add a "Troubleshooting" section to this README covering the
