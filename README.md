# OpenClaw Gateway: Secure Bootstrap & Service Architecture

This deployment strategy is optimized for local bare-metal deployments. It avoids plain-text environment files entirely by utilizing systemd-creds to bind encrypted credentials to the specific machine's ID.

# 🚀 Key Features
Volatile Secret Management: API keys are fetched from securely encrypted local credentials and assembled directly into RAM.

Zero-Disk Leakage: Sensitive JSON profiles are vaporized the moment the service stops, leaving only empty 0-byte anchors on the SSD.

Systemd Integration: Automated lifecycle management including auto-recovery on failure and graceful teardown.

Process Handover: Uses the exec command to replace the shell process with the application, reducing overhead and improving signal handling.

# 📂 Architecture Overview
Systemd creates a secure RuntimeDirectory at /run/openclaw-vault.

Systemd decrypts the credential directly into the RAM directory during service boot.

A Bind Mount overlays the RAM file onto the application's expected configuration path.

The application executes, seeing the keys as local files.

On exit, Systemd unmounts the files and deletes the RAM directory.

# 🛠️ Core Mechanism: The RAM Overlay
While the full deployment scripts are located in the repository, the core mechanism that secures the credentials relies on creating an empty file on the physical disk and overlaying it with the volatile RAM file.

If running or testing the mount manually, execute the following to create the SSD anchor and bind the mount:

Bash
touch "$APP_DIR/auth-profiles.json" 
sudo mount --bind "$RAM_DIR/auth-profiles.json" "$APP_DIR/auth-profiles.json"
# 🖥️ Deployment & Installation
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

# Deployment Files:

🔗 View "LOCAL Secure Bootstrap" for the full bootstrap script.

# Systemd Configuration:

View "Version 6.0 [Required Systemd code snippet]"

# 🛡️ Security Verification
To verify the deployment succeeded, the vault sealed properly, and secrets are not leaking to the disk:

1. Stop the active service:

Bash
sudo systemctl stop openclaw-gateway 
2. Check the persistent path:

Bash
cat /path/to/your/openclaw/agent/auth-profiles.json
Expected Result: The file should be completely empty (0 bytes) or return a "No such file or directory" error.
