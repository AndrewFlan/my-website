---
date: "2026-05-04T14:05:55-05:00"
draft: false
title: "Website Project"
tags:
  - DevOps
  - AWS
  - IaC
  - CI/CD
  - Terraform
  - Ansible
  - Github Actions
  - Automation
---

## About my Website Project

### Introduction

I wanted to complete a project to build and host a website in AWS that would be automated and easy to maintain while also being cost aware.

The goals of this project was:

- Purchase a Domain via Route53
- Use Terraform to:
  - Setup an EC2 instance that will host the static website
  - Setup Security Groups for the EC2 Instance
  - Configure DNS for the website
  - Keep Terraform State in S3
- User Ansible to:
  - Install and setup various packages, SSH config, firewall settings, and other settings
  - Install nginx and setup a default site
  - Setup certbot to get a certificate and setup auto-renewal
- Via GitHub Actions, do:
  - For Terraform:
    - Lint the code being pushed to ensure syntax is correct
    - Terraform Init and Plan for any new Pull Requests and Merges
    - When code is merged to the Main branch, it will also run Terraform Apply to update anything.
  - For Ansible:
    - Lint the code being pushed to ensure syntax is correct
  - For the Website:
    - When code is pushed to the main branch, it will be deployed the EC2 instance to update the contents of the Website

### GitHub Repo Links

- [Terraform Code](https://github.com/AndrewFlan/site-terraform)
- [Ansible Code](https://github.com/AndrewFlan/site-ansible)
- [Website Code](https://github.com/AndrewFlan/my-website)

## Details

### Terraform

#### main.tf

The `main.tf` file does the following:

- Stores the Terraform state file in a S3 bucket called `andrewflanigan-terraform-state` and enables state locking via a DynamoDB table called `terraform-state-lock`
- Configures the AWS provider and some default tags that apply to every resource created
- Configures items for AWS Systems Manager:
  - Creates an IAM Instance profile to wrap the IAM role and attach to the EC2 instance
  - Creates an IAM role for the EC2 instance to use wth AWS Systems Manager (SSM). Lets the EC2 instance assume the role
  - Attaches AWS's managed SSM policy to the role to give the EC2 instance permission to communicate with SSM
  - Attaches an inline policy to the role giving it permission to read and write to the S3 bucket. Used to copy over the website's static files
- Uploads my SSH public key to AWS to be installed on the EC2 instance, mainly used for Ansible
- Retrieves the latest Ubuntu 24.04 AMI for use in other resource blocks
- Creates an EC2 instance with the needed properties: AMI, Instance type, SSH Public Key, Security Group, IAM Instance Profile, EBS, etc
- Allocates an Elastic IP to the Instance for use with DNS

#### network.tf

The `network.tf` file does the following:

- Creates a security group for the EC2 instance that allows:
  - INGRESS:
    - SSH (Port 22) for any CIDR range. My IP address is not static, so this is needed. The EC2 instance only allows connections from trusted SSH keys.
    - HTTP/HTTPS (Port 80/443)
  - EGRESS
    - All allowed

#### dns.tf

The `dns.tf` file does the following:

- Looks up my existing Route53 hosted zone by domain name
- Creates an A-record for my root domain pointing to the Elastic IP of the EC2 instance with a TTL of 300 seconds (5 minutes)
- Creates an A-record for the www.root domain pointing to the Elastic IP of the EC2 instance with a TTL of 300 seconds (5 minutes)

#### variables.tf

The `variables.tf` file contains the following variables:

- AWS Region
- Project Name
- Instance Type: Set to `t3.micro`

#### outputs.tf

The `outputs.tf` file outputs the Elastic IP, Instance ID, and the SSH command once Terraform Apply completes

#### Terraform Workflow

This workflow triggers on pushes and pull requests to the main branch. The workflow runs on an Ubuntu VM. The workflow with checkout the code, sets up the AWS connection via OIDC (temporary credentials valid for only an hour), installs Terraform, runs Terraform Init, Terraform Plan, and Terraform Apply if the code is pushed to the Main branch.

### Ansible

#### playbook.yml

Specifies the Roles to run: `common`, `nginx`, and `certbot`

#### roles/common/tasks/main.yml

The Common role does the following:

- Runs system updates via `apt update` and `apt upgrade`
- Installs some base packages: `curl`, `git`, `unzip`, `ufw`
- Installs fail2ban
- Creates a `deploy` user on the instance. Used to deploy static site files if not using SSM
- Creates the website root directory
- Does SSH hardening:
  - `PasswordAuthentication no`: disables password login, key only
  - `PermitRootLogin no`: prevents logging in directly as root
  - `X11Forwarding no`: disables GUI forwarding, not needed on a server
  - `MaxAuthTries 3`: cuts off after 3 failed attempts
- Sets up UFM (Uncomplicated Firewall)
  - Allows SSH
  - Allows HTTP/HTTPS
  - Default denies everything else incoming
- Sets ups fail2ban
  - Bans IPs with too many failed SSH attempts
- Downloads and installs the AWS CLI on the instance. Needed for SSM

#### roles/nginx/tasks/main.yml

The Nginx role does the following:

- Installs Nginx
- Removes the default Nginx site
- Converts the site.conf.j2 template into Nginx site config
- Enables the site
- The `roles/nginx/templates/site.conf.j2` file:
  - Redirects HTTP to HTTPS
  - Sets up HTTPS on port 443
  - Sets up the Let's Encrypt certificate

#### roles/certbot/tasks/main.yml

The Certbot role does the following:

- Installs Certbot and its Nginx plugin
- Gets the Let's Encrypt Certificate
- Enables Certbot auto-renewal

### Website

[Hugo](https://gohugo.io/) is used to generate the static site files for my website. I also use the [Hugo Profile](https://themes.gohugo.io/themes/hugo-profile/) theme for my site.

#### Deploy Workflow

The workflow for this repository is responsible for deploying out the static site files to the EC2 instance. When code is pushed to the Main branch, the workflow triggers. The workflow does the following:

- Runs on an Ubuntu VM
- Checks outs the code and makes sure the Git submodules are loaded for the theme
- Installs Hugo
- Runs Hugo to build the site
- Configures the AWS Connection for SSM
- Installs the SSM plugin
- Copies the files to S3. Via SSM, syncs the files from S3 to the EC2 instance. Waits for the command to complete
