# Day 22: Configuring Secure Passwordless Root SSH Access (`datacenter-ec2`)

This document summarizes the process of configuring secure, key-based, passwordless SSH access directly to the `root` user of an Amazon EC2 instance (`datacenter-ec2`). It covers key pair generation, Security Group firewall configuration, and overriding default cloud-init restrictions on root logins.

---

## 🏗 Architecture & Core Concepts

Secure Shell (SSH) relies on cryptographic keys rather than passwords to verify client identity. In AWS, the public key is injected into the instance at launch, while the private key remains securely stored on the connecting client machine.

```
┌──────────────────────────────────────────────────────────────────┐
│                          AWS Environment                         │
│                                                                  │
│   ┌──────────────────────┐         Inbound Rule                  │
│   │    Security Group    ├─────────────────────────────┐         │
│   │   (datacenter-sg)    │   Allow TCP Port 22         │         │
│   └──────────┬───────────┘                             │         │
│              │ Applies to                              ▼         │
│              ▼                                 ┌──────────────┐  │
│   ┌──────────────────────┐   Authenticates     │ Client Host  │  │
│   │     EC2 Instance     │◄────────────────────┤ (aws-client) │  │
│   │   (datacenter-ec2)   │ Private Key Match   │  ~/.ssh/     │  │
│   │ /root/.ssh/auth_keys │                     │    id_rsa    │  │
│   └──────────────────────┘                     └──────────────┘  │
└──────────────────────────────────────────────────────────────────┘

```

### Technical Concepts:

* **Asymmetric Key Cryptography:** The client (`aws-client`) generates a paired public (`id_rsa.pub`) and private (`id_rsa`) key. The server verifies the client by challenging it with the public key.
* **Instance Metadata & Cloud-Init:** AWS uses a service called `cloud-init` to inject the public key into the default user's `~/.ssh/authorized_keys` file during the first boot.
* **Root Login Override:** By default, Amazon Linux and Ubuntu AMIs disable direct `root` login. The public key is prefixed with a `command=` parameter that forces a warning message ("Please login as the user..."). Removing this prefix and updating `sshd_config` enables direct root access.

---

## 🛠 Task Specifications

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target execution region |
| **Instance Name** | `datacenter-ec2` | Tagged compute resource |
| **Instance Type** | `t2.micro` | Low-cost general purpose instance |
| **SSH Key Name** | `id_rsa` | RSA 2048-bit local SSH key |
| **Security Group** | `datacenter-sg` | Network firewall allowing TCP Port 22 |
| **Target User** | `root` | Root administrator account on EC2 |

---

## 🚀 Deployment Execution

The provisioning process was automated using the AWS CLI and shell scripting on the `aws-client` host.

### Phase 1: Key Generation & Network Setup

1. **Generate RSA Key Pair:** Created a new 2048-bit RSA key natively on the client if it didn't exist (`ssh-keygen -t rsa -b 2048`).
2. **Create Security Group:** Provisioned `datacenter-sg` in the default VPC and authorized ingress traffic on TCP port 22 from `0.0.0.0/0`.
3. **Import Key Pair:** Uploaded `id_rsa.pub` to AWS EC2 as a recognized key pair.

### Phase 2: Compute Provisioning

Launched the `datacenter-ec2` instance using Amazon Linux 2023, attaching the newly imported key pair and `datacenter-sg`. A custom `user-data` script was passed to attempt enabling root login automatically.

### Phase 3: Root Access Override

Because AWS automatically wraps root SSH keys with a forced command warning, direct root access was initially blocked. This was resolved by logging in as the default `ec2-user` to strip the restriction:

```bash
# Executed via the default ec2-user to clean up root configurations
ssh -o StrictHostKeyChecking=no ec2-user@<PUBLIC_IP> \
  'sudo sed -i "s/^.*ssh-rsa/ssh-rsa/" /root/.ssh/authorized_keys && \
   sudo sed -i "s/^#.*PermitRootLogin.*/PermitRootLogin yes/" /etc/ssh/sshd_config && \
   sudo systemctl restart sshd'

```

* **`sed` action 1:** Removed the `command="echo 'Please login as...'"` prefix from the root `authorized_keys` file.
* **`sed` action 2:** Ensured `PermitRootLogin yes` was explicitly set in the SSH daemon configuration.

---

## 🔍 Verification Commands

Verified direct passwordless root access bypassing the default OS restrictions:

```bash
ssh -o StrictHostKeyChecking=no root@<PUBLIC_IP> "uptime; hostname"

```

**Expected Output:**

```text
 10:45:12 up 5 min,  0 users,  load average: 0.00, 0.00, 0.00
ip-172-31-45-12.ec2.internal

```

---

## 💡 Best Practices & Security Standards

* **Avoid Direct Root Login in Production:** Enabling `PermitRootLogin yes` is generally considered an anti-pattern for production environments. The industry standard is to log in as a standard user (e.g., `ec2-user` or `ubuntu`) and elevate privileges using `sudo`. This lab demonstrates how the mechanism works under the hood.
* **Strict Key Permissions:** The private key (`id_rsa`) must be secured with `chmod 400` or `chmod 600`. SSH clients will explicitly reject connection attempts if the private key is readable by other local users.
* **Restrict SSH Ingress:** While this lab opened Port 22 to `0.0.0.0/0` (the entire internet), production security groups should restrict Port 22 strictly to known corporate IP addresses, bastion hosts, or VPN CIDR blocks.
