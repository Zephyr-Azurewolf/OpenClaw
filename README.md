OpenClaw Gateway: Secure Bootstrap & Service Architecture
This repository contains a hardened deployment configuration for the OpenClaw Gateway. It utilizes a "Secret Injection" pattern to ensure that sensitive API credentials never touch persistent storage, residing strictly in volatile RAM (tmpfs) during runtime.

This architecture supports both cloud-native environments (via Azure Key Vault) and localized bare-metal deployments (via Systemd Credentials).

🚀 Key Features
Volatile Secret Management: API keys are fetched from a secure source (Azure Key Vault or Systemd Credentials) and assembled directly into RAM.

Zero-Disk Leakage: Sensitive JSON profiles are vaporized the moment the service stops, leaving only empty 0-byte anchors on the SSD.

Systemd Integration: Automated lifecycle management including auto-recovery on failure and graceful teardown.

Process Handover: Uses the exec command to replace the shell process with the application, reducing overhead and improving signal handling.

📂 Architecture Overview
Systemd creates a secure RuntimeDirectory at /run/openclaw-vault.

Bootstrap Script fetches the API Key (via Azure CLI or Systemd LoadCredential) and writes it to the RAM directory.

Bind Mount overlays the RAM file onto the application's expected configuration path.

Application executes, seeing the keys as local files.

On Exit, Systemd unmounts the files and deletes the RAM directory.

🛠️ Installation & Setup
Prerequisites for Cloud Implementation (Version 5.0):

Azure CLI installed and configured with a Managed Identity or Service Principal.

tmpfs support enabled in the Linux kernel.

Prerequisites for Local Implementation (Version 6.0):

Raw secret files stored securely on the host machine.

tmpfs support enabled in the Linux kernel.

🛡️ Security Verification
To verify that secrets are not leaking to the disk, stop the service and check the file content:

Bash
sudo systemctl stop openclaw-gateway 
(Or whatever you named the service, e.g. openclaw.service, I have the user daemon disabled on my build)
cat /path/to/your/openclaw/agent/auth-profiles.json
Result should be empty (0 bytes) or return a "No such file or directory" error.

User will need
Version 5.0 [Required Systemd code snippit] for cloud implementation

OR

Version 6.0 [Required Systemd code snippit] for local implementation
