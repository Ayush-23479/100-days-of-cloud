# Day 25: EC2 Provisioning and CloudWatch Alarm Integration (`xfusion-ec2` & `xfusion-alarm`)

This document summarizes the deployment of an Ubuntu-based EC2 instance (`xfusion-ec2`) and the configuration of an automated CloudWatch alarm (`xfusion-alarm`) linked to an Amazon Simple Notification Service (SNS) topic (`xfusion-sns-topic`).

---

## 🏗 Architecture & Overview

Monitoring compute resources is essential for maintaining infrastructure reliability. CloudWatch metrics track EC2 resource utilization and trigger automated alerts via SNS when performance thresholds are crossed.

```
┌────────────────────────────────────────────────────────────────────────┐
│                            us-east-1 Region                            │
│                                                                        │
│   ┌──────────────────────┐    Metric: CPUUtilization  ┌─────────────┐  │
│   │     EC2 Instance     ├───────────────────────────►│ CloudWatch  │  │
│   │     xfusion-ec2      │    (5-minute average)     │   Metric    │  │
│   └──────────────────────┘                            └──────┬──────┘  │
│                                                              │         │
│                                           Evaluates Metric   │         │
│                                           (CPU >= 90%)       ▼         │
│   ┌──────────────────────┐    Sends Alert             ┌─────────────┐  │
│   │      SNS Topic       │◄───────────────────────────┤ CloudWatch  │  │
│   │   xfusion-sns-topic  │    (ALARM State)           │    Alarm    │  │
│   └──────────────────────┘                            │xfusion-alarm│  │
│                                                       └─────────────┘  │
└────────────────────────────────────────────────────────────────────────┘

```

---

## 🛠 Task Specifications

| Resource / Parameter | Target Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target execution region |
| **EC2 Instance Name** | `xfusion-ec2` | Ubuntu Server 22.04 LTS instance |
| **Instance Type** | `t2.micro` / `t3.micro` | Free Tier / Standard compute |
| **CloudWatch Alarm Name** | `xfusion-alarm` | Metric alarm for CPU monitoring |
| **Monitored Metric** | `CPUUtilization` | Namespace: `AWS/EC2` |
| **Statistic** | `Average` | Average CPU load over 5 minutes |
| **Threshold Condition** | `>= 90%` | Triggers when CPU utilization reaches or exceeds 90% |
| **Evaluation Period** | `1 consecutive period (5 min)` | 1 datapoint within 5 minutes |
| **Alarm Action** | `xfusion-sns-topic` | Triggers SNS notification on ALARM state |

---

## 🚀 Implementation Steps

### Option A: Automated AWS CLI Deployment

```bash
# 1. Fetch latest Ubuntu 22.04 LTS AMI ID in us-east-1
AMI_ID=$(aws ec2 describe-images --region us-east-1 \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" --output text)

# 2. Launch EC2 instance with strict Name tag matching
INSTANCE_ID=$(aws ec2 run-instances --region us-east-1 \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=xfusion-ec2}]' \
  --query 'Instances[0].InstanceId' --output text)

# 3. Wait until instance enters 'running' state
aws ec2 wait instance-running --region us-east-1 --instance-ids $INSTANCE_ID

# 4. Fetch SNS Topic ARN
SNS_ARN=$(aws sns list-topics --region us-east-1 \
  --query "Topics[?contains(TopicArn, 'xfusion-sns-topic')].TopicArn" --output text)

# 5. Create CloudWatch Metric Alarm
aws cloudwatch put-metric-alarm --region us-east-1 \
  --alarm-name "xfusion-alarm" \
  --alarm-description "Triggers when xfusion-ec2 CPU utilization >= 90% over 5 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 90 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions "$SNS_ARN" \
  --dimensions Name=InstanceId,Value=$INSTANCE_ID

```

### Option B: AWS Management Console (GUI)

1. **Launch EC2 Instance:**
* Navigate to **EC2** > **Launch instance**.
* Set **Name** tag strictly to **`xfusion-ec2`**.
* Select **Ubuntu Server 22.04 LTS** OS image.
* Complete launch wizard and copy the generated **Instance ID**.


2. **Configure CloudWatch Alarm:**
* Navigate to **CloudWatch** > **Alarms** > **Create alarm**.
* Select metric: **EC2** > **Per-Instance Metrics** > **`CPUUtilization`** for `xfusion-ec2`.
* Statistic: `Average`, Period: `5 minutes`.
* Condition: Static, `>= 90%`.
* Action: Send notification to existing SNS topic **`xfusion-sns-topic`**.
* Alarm Name: **`xfusion-alarm`**.



---

## 🔍 Verification & Diagnostics

Verify the alarm status using the AWS CLI:

```bash
aws cloudwatch describe-alarms \
  --region us-east-1 \
  --alarm-names xfusion-alarm \
  --query "MetricAlarms[0].[AlarmName, StateValue, Threshold, EvaluationPeriods]" \
  --output table

```

### Expected Output:

```text
-----------------------
|   DescribeAlarms    |
+---------------------+
|  xfusion-alarm      |
|  OK                 |
|  90.0               |
|  1                  |
+---------------------+

```

---

## 💡 Lessons Learned & Common Pitfalls

* **Exact Tag Naming Matters:** Automated validation scripts rely on strict string matching for resource tags. A single missing character (e.g., naming an instance `xfusion-ec` instead of `xfusion-ec2`) will prevent the grading bot from finding the resource, even if the instance is running correctly.
* **Initial Metric State (`INSUFFICIENT_DATA`):** Newly created alarms stay in `INSUFFICIENT_DATA` for 3–5 minutes until CloudWatch receives the first telemetry datapoint from the EC2 instance. This is expected behavior and will transition to `OK` automatically once metrics populate.
