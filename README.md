# AWS 3-Tier Multi-VPC Peering Architecture with Banking Microservices

### End-to-End Enterprise Network Isolation, Application Load Balancer, Route 53 DNS & HTTPS Setup

---

## 📑 Table of Contents

- [1. Executive Summary](#1-executive-summary)
- [2. Architectural Diagram](#2-architectural-diagram)
- [3. Architecture Rationale & Design Decisions](#3-architecture-rationale--design-decisions)
- [4. Network Specifications & IP Plan](#4-network-specifications--ip-plan)
- [5. Step-by-Step Implementation Guide](#5-step-by-step-implementation-guide)
  - [Step 1: VPCs & Subnets Foundation](#step-1-vpcs--subnets-foundation)
  - [Step 2: Internet Gateways (IGW)](#step-2-internet-gateways-igw)
  - [Step 3: Intra-Region VPC Peering Connections](#step-3-intra-region-vpc-peering-connections)
  - [Step 4: Route Tables & Bidirectional Peering Routing](#step-4-route-tables--bidirectional-peering-routing)
  - [Step 5: Security Groups & Private Network Lockdown](#step-5-security-groups--private-network-lockdown)
  - [Step 6: EC2 Provisioning & Systemd Microservices Deployment](#step-6-ec2-provisioning--systemd-microservices-deployment)
  - [Step 7: Application Load Balancer (ALB) & Target Group Setup](#step-7-application-load-balancer-alb--target-group-setup)
  - [Step 8: ACM SSL/TLS Certificate & Route 53 Custom Domain](#step-8-acm-ssltls-certificate--route-53-custom-domain)
- [6. Verification & Testing](#6-verification--testing)
- [7. Troubleshooting & Engineering Lessons Learned](#7-troubleshooting--engineering-lessons-learned)
- [8. Teardown & Clean Up Order](#8-teardown--clean-up-order)

---

## 1. Executive Summary

This project implements an enterprise-grade, secure **3-Tier Retail Banking Microservices Architecture** on AWS in the **Singapore (`ap-southeast-1`)** region.

### Key Architectural Highlights:

- **Microservices Chaining:** `customer-profile-svc` (Profile) $\rightarrow$ `account-svc` (Account) $\rightarrow$ `statement-svc` (Statement).
- **Multi-VPC Isolation:** Each service resides in its own dedicated Amazon Virtual Private Cloud (VPC) with unique non-overlapping IPv4 CIDR blocks.
- **Non-Transitive VPC Peering:** Backend services have zero public ingress. Communications between tiers are strictly routed across private VPC Peering connections (`Profile <-> Account` and `Account <-> Statement`).
- **High Availability & Security Ingress:** External user traffic enters via an Internet-facing **Application Load Balancer (ALB)** spanning multiple Availability Zones, secured with **AWS Certificate Manager (ACM)** SSL/TLS encryption on **Port 443** with automatic HTTP-to-HTTPS redirect, routed via **Amazon Route 53** custom domain (`banking.achn-chan.space`).

---

## 2. Architectural Diagram

```
                                  [ User / Public Internet ]
                                              │
                                              ▼
                             https://banking.achn-chan.space
                                              │
                                              ▼
                                 [ Route 53 Hosted Zone ]
                               (A Record Alias to ALB)
                                              │
                                              ▼
                       ┌──────────────────────────────────────────────┐
                       │  Application Load Balancer (hc-profile-alb)  │
                       │  - Port 443 (HTTPS with ACM SSL Certificate) │
                       │  - Port 80 (HTTP 301 Redirect to HTTPS 443)  │
                       │  - Subnets: ap-southeast-1a & 1b             │
                       └──────────────────────┬───────────────────────┘
                                              │ Forward to Target Group (Port 9091)
                                              ▼
 ╔═══════════════════════════════════════════════════════════════════════════════════╗
 ║ PROFILE VPC: hc-profile-vpc (CIDR: 10.0.0.0/16)                                 ║
 ║ Subnet: hc-profile-subnet-01 (10.0.1.0/24)                                       ║
 ║                                                                                   ║
 ║   ┌───────────────────────────────────────────────────────────┐                   ║
 ║   │ EC2 Instance: profile-instance                            │                   ║
 ║   │ Private IP: 10.0.1.31                                     │                   ║
 ║   │ Service: customer-profile-svc (:9091)                     │                   ║
 ║   └─────────────────────────────┬─────────────────────────────┘                   ║
 ╚═════════════════════════════════╪═════════════════════════════════════════════════╝
                                   │
                                   │ VPC Peering 1 (pcx-0749cb1ffc0e808d6)
                                   │ UPSTREAM_URIS: http://192.168.1.141:9092
                                   ▼
 ╔═══════════════════════════════════════════════════════════════════════════════════╗
 ║ ACCOUNT VPC: hc-account-vpc (CIDR: 192.168.0.0/16)                                ║
 ║ Subnet: hc-account-subnet-01 (192.168.1.0/24)                                     ║
 ║                                                                                   ║
 ║   ┌───────────────────────────────────────────────────────────┐                   ║
 ║   │ EC2 Instance: account-instance                            │                   ║
 ║   │ Private IP: 192.168.1.141 🔒 (Strictly Private Access)    │                   ║
 ║   │ Service: account-svc (:9092)                              │                   ║
 ║   └─────────────────────────────┬─────────────────────────────┘                   ║
 ╚═════════════════════════════════╪═════════════════════════════════════════════════╝
                                   │
                                   │ VPC Peering 2 (pcx-0424f7388e881033c)
                                   │ UPSTREAM_URIS: http://172.16.1.201:9093
                                   ▼
 ╔═══════════════════════════════════════════════════════════════════════════════════╗
 ║ STATEMENT VPC: hc-statement-vpc (CIDR: 172.16.0.0/16)                             ║
 ║ Subnet: hc-statement-subnet-01 (172.16.1.0/24)                                    ║
 ║                                                                                   ║
 ║   ┌───────────────────────────────────────────────────────────┐                   ║
 ║   │ EC2 Instance: statement-instance                          │                   ║
 ║   │ Private IP: 172.16.1.201 🔒 (Strictly Private Access)     │                   ║
 ║   │ Service: statement-svc (:9093)                            │                   ║
 ║   └───────────────────────────────────────────────────────────┘                   ║
 ╚═══════════════════════════════════════════════════════════════════════════════════╝
```

---

## 3. Architecture Rationale & Design Decisions

### 1. Multi-VPC Isolation over Single-VPC Architecture
- **Blast Radius Containment:** In a single VPC with multiple subnets, all subnets share the default local route (`local`) and can communicate across the VPC by default unless complex Network ACLs or security rules are continuously enforced. Isolating microservice tiers into separate VPCs establishes a hard network perimeter. If the public-facing Profile tier is compromised, attackers cannot traverse to the Core Account or Statement tiers at the network layer.
- **Regulatory & Compliance Alignment (PCI-DSS & Basel):** Banking and financial standards require strict segregation of customer identification/profile data, transactional core ledger databases, and sensitive statement repositories across distinct administrative and network domains.

### 2. Non-Transitive Peering as a Structural Security Boundary
- By AWS design, VPC Peering is **strictly non-transitive**:
  ```
  [Profile VPC] <─── Peering 1 ───> [Account VPC] <─── Peering 2 ───> [Statement VPC]
        │                                                                   ▲
        └──────────────────────── X (Direct Access Prohibited) ─────────────┘
  ```
- **Enforced Microservices Chaining:** Profile VPC has no route to Statement VPC. Traffic cannot skip the Account tier. Any request seeking bank statement records must pass through the `Account Service` for intermediate business validation, identity checks, and authentication.

### 3. VPC Peering vs. Transit Gateway (TGW) & NAT Gateways
- **Zero Base Cost:** AWS Transit Gateway incurs an hourly attachment fee (~$36/month per VPC, exceeding $100/month for 3 VPCs) plus data processing charges. VPC Peering has **zero hourly base fee** and only charges standard intra-region data transfer rates ($0.01/GB), making it the optimal, cost-efficient choice for multi-tier microservices architectures with low-to-medium VPC counts.
- **Sub-Millisecond Latency:** VPC Peering routes directly across AWS’s global fiber backbone without passing through virtual appliance middle-boxes, eliminating throughput bottlenecks and delivering sub-millisecond point-to-point latency.

### 4. Application Load Balancer (ALB) over Direct EC2 Exposure
- **Topology Hiding & Reverse Proxy:** The EC2 instance private IP (`10.0.1.31`) and service port (`9091`) remain completely hidden from the public internet. External clients only communicate with the ALB frontend.
- **Multi-AZ High Availability:** The ALB is provisioned across multiple Availability Zones (`ap-southeast-1a` and `ap-southeast-1b`), ensuring automatic failover if a single data center experiences downtime.
- **SSL/TLS Hardware Offloading:** Computing-heavy TLS handshakes and packet decryption occur at the ALB layer, allowing downstream microservices to operate with minimal CPU overhead.

### 5. Private Security Group Lockdown (Principle of Least Privilege)
Even with peering routes configured, network access is strictly governed by stateful Security Group firewalls:

| Security Group | Inbound Port | Allowed Source | Architectural Rationale |
| :--- | :--- | :--- | :--- |
| **`hc-alb-sg`** | `80`, `443` | `0.0.0.0/0` (Internet) | Enables public client access over encrypted HTTPS (and HTTP redirect). |
| **`hc-profile-sg`** | `9091` | `hc-alb-sg` (ALB SG ID) | Completely blocks direct public access; only accepts traffic forwarded by the ALB. |
| **`hc-backend-sg`** | `9092` | `10.0.0.0/16` (Profile VPC CIDR) | Restricts Account service access exclusively to the Profile tier. |
| **`hc-statement-sg`** | `9093` | `192.168.0.0/16` (Account VPC CIDR) | Restricts Statement service access exclusively to the Account tier (Profile tier blocked). |

### 6. ACM SSL/TLS & Route 53 Custom Domain (`banking.achn-chan.space`)
- **Data-in-Transit Encryption:** Financial applications require end-to-end encrypted transit. ACM provides a free, auto-renewing SSL/TLS certificate to secure traffic against eavesdropping and Man-in-the-Middle (MITM) exploits.
- **Enforced HTTPS (301 Redirect):** Port 80 HTTP requests are automatically redirected to Port 443 HTTPS.
- **Route 53 DNS Alias:** Provides seamless, branded DNS routing to the dynamic ALB DNS endpoint without recurring DNS query fees.

---

## 4. Network Specifications & IP Plan

| Resource Name          | Layer / Tier           | VPC CIDR         | Subnet CIDR & AZ                         | Private IP       | Listening Port                                          | Upstream Target             |
| :--------------------- | :--------------------- | :--------------- | :--------------------------------------- | :--------------- | :------------------------------------------------------ | :-------------------------- |
| **`hc-profile-vpc`**   | Profile Tier           | `10.0.0.0/16`    | `10.0.1.0/24` (1a)<br>`10.0.2.0/24` (1b) | `10.0.1.31`      | `9091` (`customer-profile-svc`)                         | `http://192.168.1.141:9092` |
| **`hc-account-vpc`**   | Backend Account Tier   | `192.168.0.0/16` | `192.168.1.0/24` (1a)                    | `192.168.1.141`  | `9092` (`account-svc`)                                  | `http://172.16.1.201:9093`  |
| **`hc-statement-vpc`** | Backend Statement Tier | `172.16.0.0/16`  | `172.16.1.0/24` (1a)                     | `172.16.1.201`   | `9093` (`statement-svc`)                                | None (Leaf Node)            |
| **`hc-profile-alb`**   | Ingress Load Balancer  | `10.0.0.0/16`    | Multi-AZ (1a & 1b)                       | Dynamic AWS ENIs | `80` (HTTP) $\rightarrow$ 301 Redirect<br>`443` (HTTPS) | `hc-profile-tg` (:9091)     |

---

## 5. Step-by-Step Implementation Guide

### Step 1: VPCs & Subnets Foundation

1. Navigate to **VPC Console** in region **Singapore (`ap-southeast-1`)**.
2. **Create VPCs:**
   - **Profile:** Name = `hc-profile-vpc`, IPv4 CIDR = `10.0.0.0/16`.
   - **Account:** Name = `hc-account-vpc`, IPv4 CIDR = `192.168.0.0/16`.
   - **Statement:** Name = `hc-statement-vpc`, IPv4 CIDR = `172.16.0.0/16`.
3. **Enable DNS Hostnames:**
   - On all 3 VPCs: Select VPC $\rightarrow$ **Actions** $\rightarrow$ **Edit VPC settings** $\rightarrow$ Check **Enable DNS hostnames** $\rightarrow$ **Save changes**.
4. **Create Subnets:**
   - `hc-profile-subnet-01`: CIDR `10.0.1.0/24`, AZ `ap-southeast-1a`.
   - `hc-profile-subnet-02`: CIDR `10.0.2.0/24`, AZ `ap-southeast-1b` _(Required for ALB multi-AZ requirement)_.
   - `hc-account-subnet-01`: CIDR `192.168.1.0/24`, AZ `ap-southeast-1a`.
   - `hc-statement-subnet-01`: CIDR `172.16.1.0/24`, AZ `ap-southeast-1a`.

---

### Step 2: Internet Gateways (IGW)

1. Navigate to **Internet gateways** $\rightarrow$ **Create internet gateway**:
   - Create `hc-profile-igw` $\rightarrow$ Attach to `hc-profile-vpc`.
   - Create `hc-account-igw` $\rightarrow$ Attach to `hc-account-vpc`.
   - Create `hc-statement-igw` $\rightarrow$ Attach to `hc-statement-vpc`.

---

### Step 3: Intra-Region VPC Peering Connections

1. **Peering 1 (`profile-to-account-pcx`):**
   - Go to **Peering connections** $\rightarrow$ **Create peering connection**.
   - Name: `profile-to-account-pcx`
   - Requester VPC: `hc-profile-vpc` (`10.0.0.0/16`)
   - Accepter VPC: `hc-account-vpc` (`192.168.0.0/16`)
   - Click **Create peering connection**, then select it $\rightarrow$ **Actions** $\rightarrow$ **Accept request**.
2. **Peering 2 (`account-to-statement-pcx`):**
   - Create peering connection: Name: `account-to-statement-pcx`.
   - Requester VPC: `hc-account-vpc` (`192.168.0.0/16`).
   - Accepter VPC: `hc-statement-vpc` (`172.16.0.0/16`).
   - Click **Create peering connection**, then select it $\rightarrow$ **Actions** $\rightarrow$ **Accept request**.

---

### Step 4: Route Tables & Bidirectional Peering Routing

#### 1. Profile Route Table (`hc-profile-public-rtb`)

- **VPC:** `hc-profile-vpc`
- **Subnet Associations:** Explicitly associate `hc-profile-subnet-01` and `hc-profile-subnet-02`.
- **Routes:**
  - `0.0.0.0/0` $\rightarrow$ Target: `Internet Gateway` (`hc-profile-igw`)
  - `192.168.0.0/16` $\rightarrow$ Target: `Peering Connection` (`profile-to-account-pcx`)

#### 2. Account Route Table (`hc-account-public-rtb`)

- **VPC:** `hc-account-vpc`
- **Subnet Associations:** Explicitly associate `hc-account-subnet-01`.
- **Routes:**
  - `0.0.0.0/0` $\rightarrow$ Target: `Internet Gateway` (`hc-account-igw`)
  - `10.0.0.0/16` $\rightarrow$ Target: `Peering Connection` (`profile-to-account-pcx`) _(Return path for Profile)_
  - `172.16.0.0/16` $\rightarrow$ Target: `Peering Connection` (`account-to-statement-pcx`) _(Forward path to Statement)_

#### 3. Statement Route Table (`hc-statement-public-rtb`)

- **VPC:** `hc-statement-vpc`
- **Subnet Associations:** Explicitly associate `hc-statement-subnet-01`.
- **Routes:**
  - `0.0.0.0/0` $\rightarrow$ Target: `Internet Gateway` (`hc-statement-igw`)
  - `192.168.0.0/16` $\rightarrow$ Target: `Peering Connection` (`account-to-statement-pcx`) _(Return path for Account)_

---

### Step 5: Security Groups & Private Network Lockdown

#### 1. `hc-alb-sg` (ALB Public Ingress)

- **VPC:** `hc-profile-vpc`
- **Inbound Rules:**
  - `HTTP` (Port 80) $\rightarrow$ Source: `0.0.0.0/0`
  - `HTTPS` (Port 443) $\rightarrow$ Source: `0.0.0.0/0`
- **Outbound Rules:** All traffic (`0.0.0.0/0`)

#### 2. `hc-profile-sg` (Profile EC2 Ingress)

- **VPC:** `hc-profile-vpc`
- **Inbound Rules:**
  - `Custom TCP` (Port 9091) $\rightarrow$ Source: `hc-alb-sg` (or `0.0.0.0/0`)
  - `SSH` (Port 22) $\rightarrow$ Source: Admin IP
  - `All ICMP - IPv4` $\rightarrow$ Source: `0.0.0.0/0`
- **Outbound Rules:** All traffic (`0.0.0.0/0`)

#### 3. `hc-backend-sg` (Account EC2 Ingress - Private Lockdown)

- **VPC:** `hc-account-vpc`
- **Inbound Rules:**
  - `Custom TCP` (Port 9092) $\rightarrow$ Source: **`10.0.0.0/16`** (Profile VPC CIDR only)
  - `All ICMP - IPv4` $\rightarrow$ Source: **`10.0.0.0/16`** (Profile VPC CIDR only)
- **Outbound Rules:** All traffic (`0.0.0.0/0`)

#### 4. `hc-statement-sg` (Statement EC2 Ingress - Private Lockdown)

- **VPC:** `hc-statement-vpc`
- **Inbound Rules:**
  - `Custom TCP` (Port 9093) $\rightarrow$ Source: **`192.168.0.0/16`** (Account VPC CIDR only)
  - `All ICMP - IPv4` $\rightarrow$ Source: **`192.168.0.0/16`** (Account VPC CIDR only)
- **Outbound Rules:** All traffic (`0.0.0.0/0`)

---

### Step 6: EC2 Provisioning & Systemd Microservices Deployment

Launch instances in reverse dependency order: **Statement $\rightarrow$ Account $\rightarrow$ Profile**.

#### Instance 1: Statement Service (`statement-instance`)

- **VPC / Subnet:** `hc-statement-vpc` / `hc-statement-subnet-01`
- **Private IP:** `172.16.1.201` | **Security Group:** `hc-statement-sg`
- **Run in terminal:**

```bash
sudo dnf install -y curl unzip
curl -LO https://github.com/nicholasjackson/fake-service/releases/download/v0.26.2/fake_service_linux_amd64.zip
unzip -o fake_service_linux_amd64.zip
sudo cp fake-service /usr/local/bin/fake-service 2>/dev/null || sudo cp fake_service /usr/local/bin/fake-service
sudo chmod +x /usr/local/bin/fake-service

sudo tee /etc/systemd/system/statement-svc.service > /dev/null << 'EOF'
[Unit]
Description=Statement Service
After=network.target

[Service]
ExecStart=/usr/local/bin/fake-service
Restart=always
Environment="LISTEN_ADDR=0.0.0.0:9093"
Environment="NAME=statement-svc"
Environment="MESSAGE=HelloCloudBank | Retail Banking | statement-svc"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now statement-svc
sudo systemctl status statement-svc
```

#### Instance 2: Account Service (`account-instance`)

- **VPC / Subnet:** `hc-account-vpc` / `hc-account-subnet-01`
- **Private IP:** `192.168.1.141` | **Security Group:** `hc-backend-sg`
- **Run in terminal:**

```bash
sudo dnf install -y curl unzip
curl -LO https://github.com/nicholasjackson/fake-service/releases/download/v0.26.2/fake_service_linux_amd64.zip
unzip -o fake_service_linux_amd64.zip
sudo cp fake-service /usr/local/bin/fake-service 2>/dev/null || sudo cp fake_service /usr/local/bin/fake-service
sudo chmod +x /usr/local/bin/fake-service

sudo tee /etc/systemd/system/account-svc.service > /dev/null << 'EOF'
[Unit]
Description=Account Service
After=network.target

[Service]
ExecStart=/usr/local/bin/fake-service
Restart=always
Environment="LISTEN_ADDR=0.0.0.0:9092"
Environment="NAME=account-svc"
Environment="MESSAGE=HelloCloudBank | Retail Banking | account-svc"
Environment="UPSTREAM_URIS=http://172.16.1.201:9093"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now account-svc
sudo systemctl status account-svc
```

#### Instance 3: Customer Profile Service (`profile-instance`)

- **VPC / Subnet:** `hc-profile-vpc` / `hc-profile-subnet-01`
- **Private IP:** `10.0.1.31` | **Security Group:** `hc-profile-sg`
- **Run in terminal:**

```bash
sudo dnf install -y curl unzip
curl -LO https://github.com/nicholasjackson/fake-service/releases/download/v0.26.2/fake_service_linux_amd64.zip
unzip -o fake_service_linux_amd64.zip
sudo cp fake-service /usr/local/bin/fake-service 2>/dev/null || sudo cp fake_service /usr/local/bin/fake-service
sudo chmod +x /usr/local/bin/fake-service

sudo tee /etc/systemd/system/customer-profile-svc.service > /dev/null << 'EOF'
[Unit]
Description=Customer Profile Service
After=network.target

[Service]
ExecStart=/usr/local/bin/fake-service
Restart=always
Environment="LISTEN_ADDR=0.0.0.0:9091"
Environment="NAME=customer-profile-svc"
Environment="MESSAGE=HelloCloudBank | Retail Banking | customer-profile-svc"
Environment="UPSTREAM_URIS=http://192.168.1.141:9092"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now customer-profile-svc
sudo systemctl status customer-profile-svc
```

---

### Step 7: Application Load Balancer (ALB) & Target Group Setup

1. **Target Group (`hc-profile-tg`):**
   - Target type: `Instances` | Name: `hc-profile-tg` | Protocol: `HTTP` | Port: `9091`
   - VPC: `hc-profile-vpc` | Health check path: `/`
   - Register targets: Select `profile-instance` (`10.0.1.31`), port `9091` $\rightarrow$ Click **Include as pending below** $\rightarrow$ **Create target group**.
2. **Application Load Balancer (`hc-profile-alb`):**
   - Name: `hc-profile-alb` | Scheme: `Internet-facing` | IP type: `IPv4`
   - VPC: `hc-profile-vpc`
   - Mappings: `ap-southeast-1a` (`hc-profile-subnet-01`) and `ap-southeast-1b` (`hc-profile-subnet-02`)
   - Security group: `hc-alb-sg`
   - Listener: `HTTP:80` $\rightarrow$ Forward to `hc-profile-tg` $\rightarrow$ Click **Create load balancer**.

---

### Step 8: ACM SSL/TLS Certificate & Route 53 Custom Domain

1. **AWS Certificate Manager (ACM):**
   - In `ap-southeast-1`, click **Request certificate** $\rightarrow$ **Request a public certificate**.
   - Domain name: `*.achn-chan.space`.
   - Validation method: `DNS validation`.
   - Click **Create records in Route 53** $\rightarrow$ Certificate status transitions to **`Issued`**.
2. **ALB HTTPS Listener & Redirect:**
   - In `hc-profile-alb` Listeners:
     - Click **Add listener** $\rightarrow$ Protocol: `HTTPS:443` $\rightarrow$ Action: Forward to `hc-profile-tg` $\rightarrow$ Default SSL certificate: `*.achn-chan.space`.
     - Edit the `HTTP:80` listener $\rightarrow$ Change default action to **Redirect to URL** $\rightarrow$ Protocol: `HTTPS:443`, Status code: `301 - Permanently moved`.
3. **Route 53 DNS Record:**
   - Go to **Route 53** $\rightarrow$ Hosted zones $\rightarrow$ `achn-chan.space` $\rightarrow$ **Create record**.
   - Record name: `banking` (Target: `banking.achn-chan.space`)
   - Record type: `A` | **Alias:** Toggle ON
   - Route traffic to:
     - Endpoint: `Alias to Application and Classic Load Balancer`
     - Region: `Asia Pacific (Singapore) [ap-southeast-1]`
     - Load Balancer: Select `hc-profile-alb`
   - Routing policy: `Simple routing` $\rightarrow$ Click **Create records**.

---

## 6. Verification & Testing

### Test 1: Intermediate Chain Verification (Account $\rightarrow$ Statement)

SSH or connect to `account-instance` (`192.168.1.141`):

```bash
curl http://localhost:9092
```

Expected Output:

```json
{
  "name": "account-svc",
  "uri": "/",
  "type": "HTTP",
  "ip_addresses": ["192.168.1.141"],
  "body": "HelloCloudBank | Retail Banking | account-svc",
  "upstream_calls": {
    "http://172.16.1.201:9093": {
      "name": "statement-svc",
      "uri": "http://172.16.1.201:9093",
      "type": "HTTP",
      "ip_addresses": ["172.16.1.201"],
      "body": "HelloCloudBank | Retail Banking | statement-svc",
      "code": 200
    }
  },
  "code": 200
}
```

### Test 2: Full End-to-End HTTPS Microservices Verification

Run from your local workstation terminal:

```bash
curl -i https://banking.achn-chan.space
```

Expected Output (`HTTP/2 200 OK` with full chained JSON):

```json
HTTP/2 200
date: Sun, 11 Oct 2026 11:20:00 GMT
content-type: application/json
content-length: 1248
server: awselb/2.0

{
  "name": "customer-profile-svc",
  "uri": "/",
  "type": "HTTP",
  "ip_addresses": ["10.0.1.31"],
  "start_time": "2026-10-11T11:20:00.012391",
  "end_time": "2026-10-11T11:20:00.022450",
  "duration": "10.059ms",
  "body": "HelloCloudBank | Retail Banking | customer-profile-svc",
  "upstream_calls": {
    "http://192.168.1.141:9092": {
      "name": "account-svc",
      "uri": "http://192.168.1.141:9092",
      "type": "HTTP",
      "ip_addresses": ["192.168.1.141"],
      "body": "HelloCloudBank | Retail Banking | account-svc",
      "upstream_calls": {
        "http://172.16.1.201:9093": {
          "name": "statement-svc",
          "uri": "http://172.16.1.201:9093",
          "type": "HTTP",
          "ip_addresses": ["172.16.1.201"],
          "body": "HelloCloudBank | Retail Banking | statement-svc",
          "code": 200
        }
      },
      "code": 200
    }
  },
  "code": 200
}
```

---

## 7. Troubleshooting & Engineering Lessons Learned

### Lesson 1: Package Manager Discrepancy (`apt-get` vs `dnf`)

- **Symptom:** In User Data / startup scripts, `apt-get: command not found` error occurred in `/var/log/cloud-init-output.log`.
- **Root Cause:** Instances were launched with **Amazon Linux 2023** (RPM-based) instead of Ubuntu (Debian-based).
- **Fix:** Use `sudo dnf install -y curl unzip` on Amazon Linux.

### Lesson 2: Bash Wildcard Shell Expansion Pitfall

- **Symptom:** Systemd reported `status=203/EXEC` and service repeatedly crashed because `/usr/local/bin/fake-service` was missing.
- **Root Cause:** `mv fake*service* /usr/local/bin/fake-service` matched both `fake-service` (binary) and `fake_service_linux_amd64.zip` (archive). When `mv` is given multiple source files, the destination must be an existing directory, so the command silently aborted.
- **Fix:** Use exact filename copies: `sudo cp fake-service /usr/local/bin/fake-service 2>/dev/null || sudo cp fake_service /usr/local/bin/fake-service`.

### Lesson 3: Asymmetric Routing in VPC Peering

- **Symptom:** `ping 192.168.1.141` from Profile yielded `100% packet loss` and HTTP calls timed out after 30 seconds (`context deadline exceeded`).
- **Root Cause:** Forward route existed in Profile Route Table, but the reverse route (`10.0.0.0/16` $\rightarrow$ `pcx`) was missing in the Account Route Table. TCP SYN packets reached Account, but TCP SYN-ACK replies were dropped.
- **Fix:** Always verify bidirectional routes exist in both peer route tables.

### Lesson 4: Subnet Route Table Associations

- **Symptom:** EC2 Instance Connect displayed `Instance is not in public subnet` warning.
- **Root Cause:** When subnets are created, they implicitly associate with the VPC Main Route Table until explicitly attached to custom route tables.
- **Fix:** Always edit **Subnet associations** on the custom Route Table to explicitly associate target subnets.

---

## 8. Teardown & Clean Up Order

To avoid AWS dependency errors and avoid ongoing charges when tearing down the lab, delete resources in this specific order:

1. **Route 53:** Delete the `banking.achn-chan.space` Alias `A` Record.
2. **Application Load Balancer:** Delete `hc-profile-alb` and Target Group `hc-profile-tg`.
3. **EC2 Instances:** Terminate `profile-instance`, `account-instance`, and `statement-instance`.
4. **VPC Peering Connections:** Delete `profile-to-account-pcx` and `account-to-statement-pcx` (this automatically cleans up peering routes).
5. **Security Groups:** Delete custom security groups (`hc-alb-sg`, `hc-profile-sg`, `hc-backend-sg`, `hc-statement-sg`).
6. **Internet Gateways & Subnets:** Detach and delete IGWs, delete Subnets and Route Tables.
7. **VPCs:** Delete `hc-profile-vpc`, `hc-account-vpc`, and `hc-statement-vpc`.
