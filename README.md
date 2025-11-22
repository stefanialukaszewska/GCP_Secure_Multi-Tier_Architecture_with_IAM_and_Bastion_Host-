# GCP: Secure Multi-Tier Architecture with IAM and Bastion Host

## Stage 1: Introduction

### Project Overview
This project demonstrates the implementation of a secure, segmented network infrastructure on Google Cloud Platform (GCP). The primary objective is to design a hardened environment that adheres to the Principle of Least Privilege, ensuring that critical resources are isolated from the public internet while maintaining controlled administrative access.

The architecture features a custom Virtual Private Cloud (VPC) divided into three distinct zones (Public, Private, and Management), utilizing a Bastion Host for secure SSH entry via OS Login.

### Key Features & Scope
- Multi-Tier Network Topology: A custom VPC segmented into three subnets:

  - **Public Subnet:** Entry point for external traffic.
  - **Private Subnet:** Hosted workloads isolated from the internet.
  - **Management Subnet:** Dedicated for administrative tools and monitoring.

- Secure Access Management: Deployment of a hardened Bastion Host serving as the single point of entry (Jump Server), utilizing IAM-based OS Login for granular user authentication (eliminating management of static SSH keys).

- Outbound Connectivity: Configuration of Cloud NAT to allow private instances to pull updates and patches without exposing public IP addresses.

- Traffic Control: Implementation of strict Firewall Rules to filter traffic and ensure communication occurs only on specific, authorized ports.

- Observability: Integration with Cloud Monitoring for infrastructure oversight.

### Technologies Used
- **Networking:** Google VPC, Cloud NAT, Firewall Rules
- **Compute:** Compute Engine (GCE)
- **Security & Identity:** Cloud IAM, OS Login
- **Operations:** Cloud Monitoring

## Stage 2: Implementation
### Step 1: Network Architecture & VPC Setup

The goal of this stage is to establish the networking foundation for the multi-tier architecture. Instead of using the `default` network, we implement a Custom Mode VPC Network. This provides full control over the IP address ranges and prevents unauthorized access by ensuring no subnets are created automatically in unwanted regions.

> [!NOTE]
> Remember to set project when using CLI (not setting project can cause issues, if there's more than one) \
> `gcloud config set project my-project-name`

<details>
<summary> Set environment variables </summary>
Setting variables is not mandatory. but it makes work easier and can prevent potential typos 
  
```
REGION="us-central1" # pick desired region
NETWORK="secure-vpc" # enter your VPC network name
```
</details>

<details>
<summary> Create Custom Mode VPC Network </summary>
  
```
gcloud compute networks create $NETWORK \
    --subnet-mode=custom \
    --description="VPC Network for Secure Architecture with IAM and Bastion Host" \
    --bgp-routing-mode=regional
```
</details>

<details>
<summary> Create Subnets </summary>
  
```
# Public subnet
gcloud compute networks subnets create subnet-public \
    --network=$NETWORK \
    --region=$REGION \
    --range=10.1.0.0/24 \
    --description="Public subnet for Secure VPC"

# Private subnet
gcloud compute networks subnets create subnet-private \
    --network=$NETWORK \
    --region=$REGION \
    --range=10.2.0.0/24 \
    --enable-private-ip-google-access \
    --description="Private subnet for Secure VPC"

# Management subnet
gcloud compute networks subnets create subnet-mgmt \
    --network=$NETWORK \
    --region=$REGION \
    --range=10.3.0.0/24 \
    --enable-private-ip-google-access \
    --description="Management subnet for Secure VPC"
```

> `--enable-private-ip-google-access` \
> This is not mandatory, but a good practice to avoid additional costs.
> Without it, traffic from private subnets to Google services will go through Cloud NAT — and each such transfer incurs additional costs.

</details>

### Step 2: Firewall Rules implementation

In this step we define firewall rules that enforce controlled traffic flows between tiers. The goal is to expose only the Bastion Host to the internet over SSH, while keeping application and management workloads reachable exclusively through internal communication. Separate ingress and internal rules help implement the Principle of Least Privilege at the network level.

<details>
<summary> Create Firewall Rules </summary>
  
```
# SSH for Bastion Host
gcloud compute firewall-rules create ${NETWORK}-allow-ssh-bastion \
    --network=$NETWORK \
    --allow=tcp:22 \
    --source-ranges=0.0.0.0/0 \
    --direction=INGRESS \
    --target-tags=bastion \
    --description="Allow SSH to Bastion Host from anywhere"

# Allow internal communication
gcloud compute firewall-rules create ${NETWORK}-allow-internal-limited \
    --network=$NETWORK \
    --allow=tcp:22,tcp:80,tcp:443,tcp:8080,udp:0-65535,icmp  \
    --direction=INGRESS \
    --source-ranges=10.0.0.0/8 \
    --description="Allow limited internal traffic between all subnets"

```
> The `--source-ranges=0.0.0.0/0` configuration for the Bastion Host is used for demonstration purposes only.
> In a production environment, this rule should be restricted to trusted IP ranges (e.g., corporate office ranges, VPN egress IPs, or dedicated jump network).
</details>

### Step 3: OS Login, Compute Instances & Cloud NAT

This step focuses on creating the compute layer and ensuring secure connectivity. OS Login is enabled to centralize SSH access control through IAM identities. The Bastion Host is deployed in the public subnet with an external IP, while application and management VMs are created without public addresses in their respective private subnets. Cloud NAT is then configured to provide these private instances with outbound internet access (for updates and patches) without exposing them directly to the public internet.

<details>
<summary> Enable OS Login </summary>
  
```
gcloud compute project-info add-metadata \
    --metadata enable-oslogin=TRUE
```
</details>

<details>
<summary> Create VMs </summary>

Create static IP for Bastion Host (not mandatory)
```
gcloud compute addresses create bastion-ip \
    --region=$REGION \
    --description="Static external IP for Bastion Host"
```

Create VMs:
```
# Bastion Host
gcloud compute instances create vm-bastion \
    --zone=${REGION}-a \
    --subnet=subnet-public \
    --machine-type=e2-micro \
    --address=bastion-ip \
    --tags=bastion

# App / regular VM in created VPC network
gcloud compute instances create vm-app \
    --zone=${REGION}-a \
    --subnet=subnet-private \
    --no-address \
    --machine-type=e2-micro \
    --tags=private

# Mgmt
gcloud compute instances create vm-mgmt \
    --zone=${REGION}-a \
    --subnet=subnet-mgmt \
    --no-address \
    --machine-type=e2-micro \
    --tags=mgmt
```
</details>

<details>
<summary> Cloud NAT </summary>

Create router
```
gcloud compute routers create router-nat --network=$NETWORK --region=$REGION
```
Create NAT
```
gcloud compute routers nats create nat-config \
    --region=$REGION \
    --router=router-nat \
    --auto-allocate-nat-external-ips \
    --nat-all-subnet-ip-ranges
```

</details>

### Step 4: IAM Roles & Least Privilege Access Control

With OS Login and the compute layer in place, this step assigns IAM roles required for users to access virtual machines. The goal is to ensure that only authorized identities can initiate SSH sessions and act as service account users, while preserving least privilege. Permissions are granted at the project level, but could be further restricted in a production environment.

<details>
<summary> Grant IAM roles </summary>

```
PROJECT_ID=$(gcloud config get-value project)
```

### Every user requires `Compute OS Login`/`Compute OS Admin Login` role to utilize OS Login. 

```
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="user:example@example.com" \
    --role="roles/compute.osLogin"
```
### Every user requires `Service Account User` role to access VMs.

```
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="user:example@example.com" \
    --role="roles/iam.serviceAccountUser"
```

</details>

</details>

### Step 5: Monitoring & Observability

The final implementation step adds operational visibility to the environment. By enabling Cloud Monitoring and Cloud Logging APIs and installing the Ops Agent on each VM, we collect metrics (CPU, memory, disk, network) and system logs (including SSH/auth events). This allows administrators to build dashboards, define alerting policies, and detect anomalies or potential security incidents across all tiers of the architecture.


<details>
<summary> Enable Cloud Monitoring & Logging APIs </summary>

```
gcloud services enable monitoring.googleapis.com logging.googleapis.com
```

</details>

<details>
<summary> Ops Agents </summary>
  
Install Ops Agent (each VM)
```
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install
```
</details>

### Step 6: Add-ons - management VM

The Management VM (vm-mgmt) serves as a strictly internal control node. In a full-scale production environment, this node would typically host:
 - Automation Tools: Ansible or Terraform for Infrastructure as Code (IaC) management.
 - Centralized Logging: Lightweight log collectors.
 - Databases Administration: Clients for secure DB access.

For the purpose of this demonstration, we configure this node as a Network Security Audit Station to validate firewall rules and internal connectivity.

<details> 
<summary> Install Audit Tools </summary>

Access the Management VM via Bastion and install network diagnostic tools:

Login to Management VM (from Bastion Host):
```
gcloud compute ssh vm-mgmt --internal-ip
```
Install Tools:
```
sudo apt update
sudo apt install -y nmap netcat-openbsd curl iperf3 tcpdump
```
nmap: For port scanning and network discovery.
netcat (nc): For testing TCP/UDP connectivity.
curl: For testing HTTP responses from internal apps.
tcpdump: For packet analysis.

</details>

## Stage 3: Summary

Dataflow Diagram
<img width="2549" height="1460" alt="DFD" src="https://github.com/user-attachments/assets/eb8c97b5-9ed5-46bc-b4c1-95d264768086" />

<details>
<summary> Access </summary>
  
To access VMs in private subnets (private and mgmt) user (with proper permissions):
1. Accesses Bastion Host via OS Login, use `gcloud compute ssh vm-bastion --zone=${REGION}-a`
2. Needs to `gcloud auth login` in CLI to switch from the default logged service account (which do not have needed permissions) to their account
3. Use `gcloud compute ssh vm-app --internal-ip` or `gcloud compute ssh vm-mgmt --internal-ip` -> flag `--internal-ip` is ised, as those VMs do not possess external IP

</details>

<details>
<summary> Monitoring </summary>
  
Once the environment is deployed and the Ops Agent is installed on each VM, monitoring data becomes available in the Google Cloud Console. From the Cloud Monitoring dashboard, you can:
  - View real-time metrics for all VMs (CPU, memory, disk, network).
  - Inspect system and security logs in Cloud Logging (e.g., SSH attempts, system failures).
  - Create dashboards for Bastion, application, and management VMs.
  - Configure alerting policies, such as high CPU utilization or Bastion Host downtime.
  - Use log-based metrics to detect suspicious activity (e.g., repeated failed SSH logins).
  - This observability layer enables administrators to continuously verify correct operation of the multi-tier environment, detect misconfigurations, and quickly investigate potential security incidents.

</details>

<details> 
<summary> Perform Security Tests </summary>

Use the installed tools (vm-mgmt).

1. Test Web Connectivity (Should SUCCEED) Verify that the management VM can reach the application VM on permitted ports (e.g., 80).

```
# Replace with actual internal IP of vm-app (e.g., 10.2.0.2)
curl -I http://10.2.0.x
```
Expected Result: `HTTP/1.1 200 OK` (or similar response from web server).

2. Test Port Scanning (Should show FILTERED/CLOSED). Scan the vm-app to ensure non-essential ports are blocked by the Firewall.

```
# Replace with actual internal IP of vm-app (e.g., 10.2.0.2)
nmap -p 80,22,3306 10.2.0.x
```
Expected Result:

- `Port 80: OPEN`
- `Port 22: OPEN`
- `Port 3306: FILTERED` (Blocked by Firewall - proving security works)

3. Test Connectivity via Netcat Manually verify TCP handshake.

```
# Replace with actual internal IP of vm-app (e.g., 10.2.0.2)
nc -vz 10.2.0.x 80
```
Expected Result: `Connection to 10.2.0.x 80 port [tcp/http] succeeded!`

</details>
