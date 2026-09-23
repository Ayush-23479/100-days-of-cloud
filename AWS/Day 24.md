# Day 24: Application Load Balancer (ALB) Setup for Nginx Application

This repository documents the architecture, implementation steps, and validation process for deploying an AWS Application Load Balancer (`xfusion-alb`) in front of an existing EC2 web server (`xfusion-ec2`) running Nginx.

---

## 🏗 Architecture & Overview

The objective of this setup is to establish an enterprise-grade high-availability entry point for the application, offloading public HTTP traffic handling to an Application Load Balancer (ALB) while enforcing security best practices at the instance level.

```
                         [ Internet Traffic ]
                                  │
                                  ▼ (HTTP : Port 80)
                  ┌───────────────────────────────┐
                  │          xfusion-sg           │
                  │     (ALB Security Group)      │
                  └───────────────┬───────────────┘
                                  │
                                  ▼
                  ┌───────────────────────────────┐
                  │          xfusion-alb          │
                  │  (Application Load Balancer)  │
                  └───────────────┬───────────────┘
                                  │
                                  ▼ Forward (Port 80)
                  ┌───────────────────────────────┐
                  │          xfusion-tg           │
                  │        (Target Group)         │
                  └───────────────┬───────────────┘
                                  │
                                  ▼ Registered Target
                  ┌───────────────────────────────┐
                  │          xfusion-ec2          │
                  │   (Nginx Server on Port 80)   │
                  └───────────────────────────────┘

```

### Technical Highlights:

* **Layer 7 Load Balancing:** The ALB handles inbound HTTP requests on Port 80 and routes them to the backend Target Group based on listener rules.
* **Least Privilege Inbound Rules:** The EC2 instance security group was updated to only allow incoming HTTP traffic originating directly from `xfusion-sg` (the ALB's security group), shielding the server from direct public exposure.
* **Multi-AZ Availability:** The ALB is attached across multiple Availability Zone subnets in `us-east-1` to ensure network resiliency.

---

## 🛠 Task Specifications

| Resource / Parameter | Name / Configuration | Details |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Primary deployment region |
| **Load Balancer Name** | `xfusion-alb` | Application Load Balancer (Internet-facing) |
| **Target Group Name** | `xfusion-tg` | Target Type: `Instances`, Protocol: `HTTP`, Port: `80` |
| **ALB Security Group** | `xfusion-sg` | Inbound: Allow `HTTP (80)` from `0.0.0.0/0` |
| **Target EC2 Instance** | `xfusion-ec2` | Registered target running Nginx on Port 80 |

---

## 🚀 Implementation Steps

### Step 1: Security Group Creation & Configuration

1. Created `xfusion-sg` assigned to the default VPC.
2. Added an inbound rule to `xfusion-sg`:
* **Protocol:** TCP
* **Port:** 80
* **Source:** `0.0.0.0/0` (Public Access)


3. Modified the default security group attached to `xfusion-ec2` to allow inbound traffic on Port 80 **only** from the source security group `xfusion-sg`.

### Step 2: Target Group Setup & Registration

1. Created Target Group `xfusion-tg` using **Instances** target type.
2. Configured the routing protocol to **HTTP** on port **80**.
3. Registered the `xfusion-ec2` instance as a target under `xfusion-tg`.

### Step 3: Application Load Balancer Provisioning

1. Provisioned `xfusion-alb` with **Internet-facing** scheme.
2. Mapped the ALB to two or more Availability Zones/Subnets within the default VPC.
3. Attached security group `xfusion-sg` to the ALB.
4. Added an HTTP Listener on Port 80 configured with a default rule to forward traffic to `xfusion-tg`.

---

## 🔍 Verification & Testing

1. **Target Group Health Status:**
Navigated to **Target Groups** > `xfusion-tg` > **Targets** tab and verified that the Health State of `xfusion-ec2` transitioned to **Healthy**.
2. **HTTP Traffic Validation:**
Fetched the DNS name of `xfusion-alb` and executed an HTTP request from the terminal:
```bash
curl -I http://<xfusion-alb-dns-name>.elb.amazonaws.com

```


**Expected Output:**
```http
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html
...

```



---

## 💡 Key Takeaways

* **Decoupling Architecture:** Introducing an ALB decouples public-facing networking from underlying application servers, enabling seamless scaling, zero-downtime rolling updates, and SSL/TLS termination.
* **Health Checks:** Target groups continuously monitor backend targets. If Nginx crashes or the instance becomes unresponsive, the ALB automatically stops forwarding traffic to the failed instance.
