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
**Objective**

The goal of this stage is to establish the networking foundation for the multi-tier architecture. Instead of using the `default` network, we implement a Custom Mode VPC Network. This provides full control over the IP address ranges and prevents unauthorized access by ensuring no subnets are created automatically in unwanted regions.

> [!NOTE]
> Remember to set project when using CLI (not setting project can cause issues, if there's more than one) \
> `gcloud config set project my-project-name`

<details>
<summary> Set environment variables: </summary>
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
</details>

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

dostanie sie do vm -> log na bastion -> gcloud auth login na swojego maila (servisowe ktore jest z defaultu nie ma uprawnien) -> gcloud compute ssh vm-app --internal-ip
