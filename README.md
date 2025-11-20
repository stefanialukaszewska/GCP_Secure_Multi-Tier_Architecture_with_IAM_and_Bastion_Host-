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

- Traffic Control: Implementation of strict Firewall Rules and Service Accounts to filter traffic and ensure communication occurs only on specific, authorized ports.

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

<details>
<summary> Create custom network using CLI: </summary>
  
```
# Set Project 
gcloud config set project my-project-name

# Create Custom Mode VPC Network
gcloud compute networks create my-vpc-network \
    --subnet-mode=custom \
    --description="VPC Network for Secure Architecture with IAM and Bastion Host" \
    --bgp-routing-mode=regional
```
</details>
