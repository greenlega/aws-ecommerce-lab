# aws-ecommerce-lab
Automated deployment of a High Availability e-commerce architecture on AWS. Integrates EC2, RDS, EFS, and custom UI within WooCommerce.
# AWS High Availability E-Commerce Architecture 

This repository contains the source files and documentation for deploying a fault-tolerant, scalable, and highly available e-commerce infrastructure on Amazon Web Services (AWS). The project integrates a custom frontend UI with a WordPress + WooCommerce backend.

Architecture Overview
The infrastructure is fully provisioned using **AWS CloudFormation** and includes the following key components:
- **VPC (Virtual Private Cloud)** with public and private subnets across multiple Availability Zones (Multi-AZ) for redundancy.
- **Application Load Balancer (ALB)** to securely distribute incoming HTTP traffic across multiple web servers.
- **Auto Scaling Group (ASG)** managing EC2 instances to dynamically scale based on load and automatically replace unhealthy instances.
- **Amazon EFS (Elastic File System)** serving as a shared network drive for the `wp-content` directory, ensuring all EC2 instances serve identical media and plugin files.
- **Amazon RDS (Relational Database Service)** running MySQL in a private subnet, providing a secure and reliable database backend.

 Deployment Guide

### 1. Infrastructure Provisioning
1. Navigate to the AWS Console -> **CloudFormation**.
2. Create a new stack using the provided template file (`1.yaml` / `template.json`).
3. Wait for the stack status to reach `CREATE_COMPLETE`.

### 2. Database Configuration
1. Go to the **RDS** console and copy the generated Endpoint for the database instance.
2. Connect to an active EC2 instance via **AWS Systems Manager (Session Manager)**.
3. Configure the WordPress database connection by editing `wp-config.php`:
   ```bash
   sudo nano /var/www/html/wp-config.php
Note: Ensure the DB_HOST strictly contains the endpoint URL without any http:// prefix or trailing slashes.

3. Application Setup (WordPress & WooCommerce)
Access the Application Load Balancer's DNS name in a web browser.

Complete the standard WordPress installation.

Navigate to the WordPress Dashboard (/wp-admin) -> Plugins -> Add New.

Search for, install, and activate the WooCommerce plugin.

4. Custom Frontend Integration
In the WordPress Dashboard, go to Pages -> Add New Page.

Title the page (e.g., "Cloud Store").

Add a Custom HTML block and paste the code from cloud-store-template.html provided in this repository.

Go to Settings -> Reading and set "Your homepage displays" to A static page, selecting the newly created "Cloud Store" page.

To ensure the custom design uses the full screen, navigate to Appearance -> Editor (Site Editor), locate the default theme's Header and Footer in the List View, and delete them.

Troubleshooting & Architectural Challenges
During the deployment, several real-world cloud engineering challenges were encountered and successfully resolved:

Challenge 1: Database Connection Hang
Issue: WordPress failed to connect to the database, causing the setup screen to freeze.

Root Cause: The RDS Endpoint was accidentally formatted as a web URL (e.g., http://...rds.amazonaws.com/). RDS strictly requires the raw DNS hostname.

Resolution: Removed the http:// prefix and trailing slashes from the DB_HOST constant in wp-config.php, restoring instantaneous database connectivity.

Challenge 2: WooCommerce Plugin Installation Failures (PCLZIP_ERR_MISSING_FILE & 504 Gateway Time-out)
Issue: When trying to install the heavy WooCommerce plugin, the installation would fail with missing file errors or time out completely.

Root Cause: This is a classic Load Balancer issue. The ALB routed the download request to Server A (saving the .zip to its local /tmp/ directory). However, when the unzip command was triggered, the ALB routed the request to Server B. Since Server B didn't have the file in its local /tmp/, it threw a missing file error. Furthermore, extracting a heavy plugin directly to an EFS network drive takes significantly longer than a local SSD, causing the ALB's 60-second timeout limit to trigger (504 Error).

Resolution: Temporarily stopped (scaled down) one of the EC2 instances. This forced the ALB to route all traffic (both downloading and unzipping) to a single active server, allowing the heavy WooCommerce archive to be successfully extracted into the shared EFS wp-content directory.

Challenge 3: "White Screen of Death" on Custom HTML
Issue: Pasting the custom cloud-store-template.html resulted in a blank white page.

Root Cause: The original HTML file contained full document structure tags (<!DOCTYPE html>, <html>, <body>). Embedding these inside a WordPress page (which already generates these tags) caused severe browser rendering conflicts.

Resolution: Stripped the root HTML wrappers and retained only the <style>, inner DOM elements, and <script> logic. Used the WordPress Site Editor to remove the default theme's Header and Footer, creating a perfect, conflict-free fullscreen canvas for the custom store UI.

High Availability (HA) Testing
The architecture was successfully tested for fault tolerance (Step 8 of the lab requirement). By manually stopping an active EC2 instance:

The Application Load Balancer immediately detected the instance failure and routed all user traffic to the remaining healthy instance.

The site experienced zero downtime.

The Auto Scaling Group detected the missing capacity and automatically launched a new replacement EC2 instance to restore the cluster to full strength.
