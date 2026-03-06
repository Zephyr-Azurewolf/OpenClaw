# OpenClaw
Scripts related to OpenClaw AI agent
OpenClaw Zero-Persistence RAM Enclave & State Sync
This script provides a highly secure, production-ready bootstrap environment for OpenClaw (2026 builds) and other local AI agent frameworks. It solves the critical "Security vs. State" dilemma by utilizing hybrid Linux mounts to isolate secrets in volatile memory while ensuring conversational state is safely persisted to disk.

##The Problem
Running local AI agents often requires injecting sensitive credentials (like Google Gemini, OpenAI, or Anthropic API keys) into local configuration files (e.g., auth.json or auth-profiles.json).

The Security Risk: Writing plaintext API keys to an SSD leaves a permanent forensic trace. It exposes you to local file-read exploits, rogue agent skills (e.g., in shared Moltbook environments), and supply-chain attacks.

The "File Lock" Issue: Simply symlinking or bind-mounting individual secure files often causes Device or resource busy errors because modern Node.js applications attempt atomic writes (rename/replace) during migrations or boot sequences, which the Linux kernel blocks on file-level mounts.

The Amnesia Problem: Moving the entire agent directory to a tmpfs (RAM disk) solves the security issue but destroys the agent's memory (sessions and transcripts) every time the service restarts.

##The Solution
This script creates a Zero-Persistence Enclave for credentials while maintaining a Bi-Directional State Sync for the agent's memory.

##Key Features
Azure Key Vault Integration: Fetches credentials securely at runtime rather than relying on local .env files.

Directory-Level RAM Mounts: Mounts the configuration and session directories directly to tmpfs. This bypasses strict symlink security checks and allows the application to perform atomic file operations freely without throwing kernel locks.

Dead-Man's Switch (Trap): Uses a Linux trap to intercept exit signals (like Ctrl+C or service stops). Before the RAM disk evaporates, it safely synchronizes the active .jsonl session transcripts back to the SSD.

Sterile Platter: If the machine loses power or is compromised post-shutdown, the SSD contains absolutely no API keys.

##How It Works (Step-by-Step)
Cleanup & Teardown: The script safely unmounts any hanging tmpfs volumes from previous crashes and kills zombie processes.

RAM Allocation: It carves out two temporary file systems (tmpfs) in your system's RAM: one for the agent's credentials (5MB) and one for the active session logs (30MB).

State Restoration: It pulls the latest saved agent state (the "Soul") from the secure SSD backup folder and loads it into the RAM-based session directory.

Credential Injection: It fetches the live API key from Azure Key Vault and writes the auth.json payload directly into the volatile RAM directory.

Execution: The environment variables are set, the vault variable is unset from script memory, and the OpenClaw gateway is launched.

Graceful Exit: Upon receiving an exit signal, the trap function triggers, backing up the updated session files from RAM to the SSD before the system reclaims the volatile memory.

##Prerequisites
Linux environment (Ubuntu/Debian recommended).

az cli installed and authenticated to your Azure account.

OpenClaw framework installed.

##Setup Instructions
Clone the repository and place the script in your home directory or a secure bin folder.

Update the configuration variables at the top of the script:

<AGENT_NAME>: Your specific OpenClaw agent directory name.

<SECRET_NAME>: The name of your secret in Azure Key Vault.

<VAULT_NAME>: The name of your Azure Key Vault instance.

Make the script executable: chmod +x start_agent.sh

Run the script: ./start_agent.sh

##Security Note regarding --inplace Syncing
The sync function currently utilizes cp -ru to update the SSD with the latest files from RAM. For enterprise deployments where absolute atomic writes are required to prevent data corruption during unexpected power loss, consider modifying the save_state function to sync to a staging directory first, followed by an atomic mv or hard-link swap to the final persistence folder.
