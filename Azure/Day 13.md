

---

## 🛠️ Day 13: Establish Passwordless SSH Root Authentication for Azure VMs

### 📋 Scenario & Goal

The objective was to provision secure, passwordless SSH access for the `root` administrative account on an Azure Ubuntu Virtual Machine named **`devops-vm`**. This requires building a reliable cryptographic trust layout between the landing client host and the remote cloud workload, bypassing default cloud-init account restriction policies safely.

### 💡 Core Engineering Lessons Learned

* **Asymmetric Trust Topology:** Understanding that verification requests must originate from the client holding the private signature key (`id_rsa`), making testing loops attempted inside the target machine fail automatically.
* **Provider-Level Restriction Blocks:** Recognizing that Azure baseline Linux machine patterns explicitly prepend warning strings to `/root/.ssh/authorized_keys` to block direct root shell interaction. Resolving this requires structural file overwrites (`>`) rather than standard appends (`>>`).
* **Multi-Channel Management Planes:** Mastering both lower-level terminal bridging tunnels (`ssh` hopping) and high-level platform runtime fabrics (Azure Instance **Run Command** engine).

---

### 🔧 Method 1: The Native Terminal Bridge Solution (Shell CLI)

This approach uses a two-stage privilege cascade directly through your client's administrative shell environment.

#### 1. Extract Key and Establish Target Boundaries

Run this on your local landing host (`~ ➜`) to grab the asymmetric signature format and identify your remote network endpoint targets:

```bash
cat /root/.ssh/id_rsa.pub
az vm list-ip-addresses --resource-group kml_rg_main-563d84d4709c487a --name devops-vm --output table

```

#### 2. Cross the Cloud-Init Bridge User

Tunnel into the node utilizing the baseline system account credentials:

```bash
ssh azureuser@<VM_PUBLIC_IP>

```

#### 3. Overwrite Cloud Blocks & Hard Code POSIX Security Metrics

Escalate parameters to system administrator level, overwrite the target state file to eliminate Azure's root login restrictions, and fix configurations:

```bash
sudo su -
mkdir -p /root/.ssh

# CRITICAL: Overwrite (>) instead of Append (>>) to clean Azure's baseline blockers
echo "ssh-rsa AAAA...your_key_string... root@azure-client" > /root/.ssh/authorized_keys

# Lock down resource folder policies
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys

# Override global sshd service parameter behaviors
echo "PermitRootLogin prohibit-password" >> /etc/ssh/sshd_config
systemctl restart sshd

# Exit completely back to the local landing prompt (~ ➜)
exit
exit

```

---

### 🔧 Method 2: The Azure Portal Management Plane Solution (Run Command)

This alternative mechanism bypasses terminal intermediate hopping loops entirely, pushing script configurations straight down to the VM system kernel over internal cloud fabrics.

```
 [ Azure Portal ]                                      [ devops-vm ]
  Run Command (RunShellScript)  ──(Internal Fabric)──> Executes Script as Root

```

#### 1. Navigate to the Run Workspace

Log in via the administrative portal dashboard, locate **`devops-vm`**, and select **Run command** inside the Operations sidebar column menu layout.

#### 2. Execute the Injector Script

Select **`RunShellScript`** and inject this automated configuration matrix to clean and replace target properties directly:

```bash
#!/bin/bash
mkdir -p /root/.ssh

# Overwrite provider files to strip default terminal login lockouts
echo "ssh-rsa AAAA...your_key_string... root@azure-client" > /root/.ssh/authorized_keys

# Standardize permissions
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys

# Enforce secure operational configurations
echo "PermitRootLogin prohibit-password" >> /etc/ssh/sshd_config
systemctl restart sshd

```

Click **Run** and await the validation confirmation report.

---

### 🎯 Ultimate Verification (Executed From Local Client Host Prompt)

To verify the target security architecture has been fully unified, drop back to your local terminal workspace (**`~ ➜`**) and execute an instant authentication handshake:

```bash
ssh root@<VM_PUBLIC_IP>

```

### ✅ Outcome

The server drops into an active, passwordless encrypted shell session under the identity of the root user without displaying warnings or prompting for passwords. Both execution tracks successfully validated, securing a perfect green checkmark for Day 13!
