# Day 2: Provisioning an AWS Security Group for Web Application Servers

This guide covers the deployment and configuration of an Amazon EC2 Security Group acting as a virtual firewall for application instances in `us-east-1`.

---

**Architecture & Core Concepts**

An AWS Security Group controls inbound and outbound traffic at the network interface level (ENI) for EC2 instances.

* **Stateful Filtering:** Automatic tracking of connection state. Return traffic for allowed inbound requests is permitted automatically regardless of outbound rules.
* **Default Security Stance:** Blocks all incoming traffic by default until explicit ingress rules are added.
* **Scope:** Operates at the instance/ENI layer, distinct from Network Access Control Lists (NACLs) which operate statelessly at the subnet level.

```
                   [ Internet Traffic (0.0.0.0/0) ]
                                  │
                  ┌───────────────┴───────────────┐
                  │ Inbound Traffic (Port 80/22)  │
                  ▼                               ▼
     [ HTTP: Port 80 ]                    [ SSH: Port 22 ]
     └────────────────────────┬────────────────────────┘
                              │
                  [ Security Group: xfusion-sg ]
                              │
                    [ EC2 Instance / ENI ]

```

---

**Configuration Parameters**

| Parameter | Value | Description |
| --- | --- | --- |
| **Region** | `us-east-1` | Target deployment region |
| **VPC ID** | Default VPC | Network isolation boundary |
| **Security Group Name** | `xfusion-sg` | Resource identifier |
| **Description** | `Security group for Nautilus App Servers` | Resource description |
| **Inbound Rule 1** | TCP / 80 (`0.0.0.0/0`) | Allows public HTTP web traffic |
| **Inbound Rule 2** | TCP / 22 (`0.0.0.0/0`) | Allows public SSH administrative access |

---

**Deployment Execution**

**AWS CLI Deployment**

```bash
# 1. Fetch Default VPC ID
DEFAULT_VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --region us-east-1 \
  --query "Vpcs[0].VpcId" \
  --output text)

# 2. Create Security Group
SG_ID=$(aws ec2 create-security-group \
  --group-name "xfusion-sg" \
  --description "Security group for Nautilus App Servers" \
  --vpc-id $DEFAULT_VPC_ID \
  --region us-east-1 \
  --query "GroupId" \
  --output text)

# 3. Add Inbound HTTP (Port 80)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --region us-east-1

# 4. Add Inbound SSH (Port 22)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0 \
  --region us-east-1

```

**AWS Management Console Steps**

1. Open **EC2 Console** > **Network & Security** > **Security Groups**.
2. Click **Create security group**.
3. Set **Name** to `xfusion-sg`, **Description** to `Security group for Nautilus App Servers`, and select the **Default VPC**.
4. In the **Inbound rules** section, add two rules:
* **HTTP** | TCP | Port 80 | Source: `Anywhere-IPv4` (`0.0.0.0/0`)
* **SSH** | TCP | Port 22 | Source: `Anywhere-IPv4` (`0.0.0.0/0`)


5. Leave **Outbound rules** default and click **Create security group**.

---

**Verification**

Verify the security group rule configurations:

```bash
aws ec2 describe-security-groups \
  --group-names xfusion-sg \
  --region us-east-1 \
  --query "SecurityGroups[0].IpPermissions"

```

---

**Troubleshooting & Key Learnings**

* **Inbound vs. Outbound Placement:** Ensure ingress rules are strictly populated in the **Inbound rules** section within the Console UI. Adding rules to the Outbound panel will leave ports 80 and 22 closed to external traffic.
* **Production Hardening:** Opening SSH (Port 22) to `0.0.0.0/0` introduces security risks. Production setups should restrict port 22 access to specific administrator IP ranges or utilize AWS Systems Manager (SSM) Session Manager.
