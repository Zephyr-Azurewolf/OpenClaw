# OpenClaw Gateway: Secure Bootstrap & Service Architecture

This repository contains a hardened deployment configuration for the OpenClaw Gateway. It utilizes a "Secret Injection" pattern to ensure that sensitive API credentials never touch persistent storage, residing strictly in volatile RAM (tmpfs) during runtime.

This architecture supports both cloud-native environments (via Azure Key Vault) and localized bare-metal deployments (via Systemd Credentials).

# 🚀 Key Features
Volatile Secret Management: API keys are fetched from a secure source (Azure Key Vault or Systemd Credentials) and assembled directly into RAM.

Zero-Disk Leakage: Sensitive JSON profiles are vaporized the moment the service stops, leaving only empty 0-byte anchors on the SSD.

Systemd Integration: Automated lifecycle management including auto-recovery on failure and graceful teardown.

Process Handover: Uses the exec command to replace the shell process with the application, reducing overhead and improving signal handling.

# 📂 Architecture Overview
Systemd creates a secure RuntimeDirectory at /run/openclaw-vault.

The Bootstrap Script fetches the API Key and writes it to the RAM directory.

A Bind Mount overlays the RAM file onto the application's expected configuration path.

The application executes, seeing the keys as local files.

On exit, Systemd unmounts the files and deletes the RAM directory.

# 🛠️ Core Mechanism: The RAM Overlay
While the full deployment scripts are located in their respective repository branches, the core mechanism that secures the credentials relies on creating an empty file on the physical disk and overlaying it with the volatile RAM file.

If running or testing the mount manually, execute the following to create the SSD anchor and bind the mount:

Bash
touch "$APP_DIR/auth-profiles.json" 
sudo mount --bind "$RAM_DIR/auth-profiles.json" "$APP_DIR/auth-profiles.json"

# ☁️ Deployment: Cloud Implementation (Version 5.0)
Prerequisites:

Azure CLI installed and configured with a Managed Identity or Service Principal.

tmpfs support enabled in the Linux kernel.

Deployment Files:

🔗 [View the 'V.5' branch] for the full start_gateway.sh automation script.

Systemd Configuration:
[Required Systemd code snippet for cloud implementation]

# 🖥️ Deployment: Local Implementation (Version 6.0)
This deployment strategy is optimized for sovereign hardware nodes. It avoids plain-text environment files entirely by utilizing systemd-creds to bind encrypted credentials to the specific machine's ID.

Prerequisites:

systemd-creds available on the host machine.

tmpfs support enabled in the Linux kernel.

1. Encrypting the Keys
Instead of leaving API keys exposed in an environment file, use systemd-creds to encrypt the token. Pipe the raw keys directly into the credential path targeted by your service unit file:

Bash
echo "sk-or-v1-your-new-openrouter-key..." | sudo systemd-creds encrypt --name=or_secret - /etc/openclaw/vault/openrouter.cred

echo "your-telegram-token..." | sudo systemd-creds encrypt --name=tg_secret - /etc/openclaw/vault/telegram.cred
2. The Volatile Runtime Mirror
The Injection: During service boot, Systemd decrypts the credential directly into RAM at /run/openclaw-vault/.

The Ephemeral Loop: OpenClaw references the keys from this memory-mapped directory. If the daemon drops or the hardware loses power, the volatile directory is instantly vaporized, leaving nothing behind but the encrypted blob on the physical disk.

Deployment Files:

🔗 [View the 'V.6' branch] for the full bootstrap scripts.

# Systemd Configuration:
See:
[Required Systemd code snippets]
Version 5.0 for cloud deployment
Version 6.0 for local deployment

# 🛡️ Security Verification
To verify the deployment succeeded, the vault sealed properly, and secrets are not leaking to the disk:

Stop the active service:

Bash
sudo systemctl stop openclaw-gateway 
Check the persistent path:

Bash
cat /path/to/your/openclaw/agent/auth-profiles.json
Expected Result: The file should be completely empty (0 bytes) or return a "No such file or directory" error.
