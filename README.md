# 🐺 OpenClaw Secure Gateway: Zero-Disk API Vault

#### Architecture Overview
This deployment architecture provides a **secure, zero-disk-leakage environment** for running the **OpenClaw Gateway (v2026.3.11+)**. It is specifically designed to prevent **sensitive API credentials** from ever being written to persistent SSD storage.

Instead of storing plain-text keys on a physical drive where they could be recovered, this **daemonized setup** utilizes **systemd** to construct a **volatile memory environment**:

* **Retrieves** the secure token directly from **Azure Key Vault**.
* **Generates** the strict **JSON schema** required by modern OpenClaw routing.
* **Writes** the payload strictly to a **volatile RAM disk (tmpfs)**.
* **Bind-mounts** the RAM-resident file over the application's expected configuration directory.
* **Vaporizes** the credentials from memory the moment the service is **stopped** or the **VM is restarted**.

---

#### Prerequisites
* **Linux Environment** utilizing **systemd** (Ubuntu/Debian preferred).
* **Azure CLI** installed and authenticated (**Managed Identity** recommended).
* **OpenClaw Gateway** version **2026.3.11** or higher.
* **Sudo Privileges** required for executing **mount --bind**.

---

#### **File Structure**
* **start_gateway.sh**: The core **bootstrap script** handling key retrieval, schema generation, and the **exec process handover**.
* **openclaw-gateway.service**: The **systemd unit file** managing the process lifecycle and the **RuntimeDirectory RAM disk**.

---

#### Installation & Deployment**

### 1. Configure the Bootstrap Script
Edit the configuration variables at the top of **start_gateway.sh** to match your **User Home**, **Provider**, and **Azure Vault** details. Then, set the execution permissions:

`chmod +x /home/yourusername/start_gateway.sh`

### 2. Deploy the Systemd Service
Copy the service file into the system manager directory and set the appropriate **strict read permissions (644)**:

`sudo cp openclaw-gateway.service /etc/systemd/system/`
`sudo chmod 644 /etc/systemd/system/openclaw-gateway.service`

### 3. Initialize the Daemon
Reload the **system manager** to register the new architecture, **enable** it for automatic boot-up, and **wake the gateway**:

`sudo systemctl daemon-reload`
`sudo systemctl enable openclaw-gateway.service`
`sudo systemctl start openclaw-gateway.service`

### 4. Verify the Architecture
Confirm that the keys are **safely mounted in RAM** and the gateway is active by **streaming the logs**:

`journalctl -u openclaw-gateway.service -f`

---

#### Security Notes
* **Process Replacement**: The bootstrap script utilizes **exec** to hand over control entirely to the **Node process**. This ensures that **systemd signals** (like **SIGTERM**) are passed directly to the OpenClaw application for a **graceful teardown**.
* **The Cleanup Crew**: The **ExecStopPost** directive ensures that even in the event of an unexpected crash, the **umount -l** command fires, dropping the **RAM overlay** and leaving no trace of the API profile on the disk.
  
