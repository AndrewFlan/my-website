---
date: "2026-05-04T14:05:55-05:00"
draft: false
title: "Website Project"
tags:
  - DevOps
  - AWS
  - GCP
  - IaC
  - CI/CD
  - Terraform
  - Ansible
  - Github Actions
  - Automation
---

## About my Website Project

Currently my site is running in Google Cloud Platform. Its a bit cheaper there.

### Introduction

I wanted to complete a project to build and host a website in AWS or GCP that would be automated and easy to maintain while also being cost aware.

The goals of this project were:

- Purchase a Domain via Route53
- Use Terraform to manage infrastructure across two cloud providers:
  - **AWS:**
    - Setup an EC2 instance to host the static website
    - Setup Security Groups for the EC2 instance
    - Configure DNS via Route53
    - Keep Terraform state in S3 with DynamoDB state locking
  - **GCP:**
    - Setup a Compute Engine instance to host the static website
    - Configure firewall rules, restricting SSH access to Google's IAP range only
    - Provision a static external IP and GCS bucket for deployment staging
    - Keep Terraform state in GCS
- Use Ansible to:
  - Install and configure packages, SSH hardening, firewall settings, and other server settings
  - Install Nginx and configure site serving
  - Setup Certbot to obtain certificates for all domains and configure auto-renewal
  - Manage multi-domain TLS covering `andrewflanigan.com`, `www.andrewflanigan.com`, and `gcp.andrewflanigan.com`
- Via GitHub Actions, do:
  - For Terraform:
    - Lint code on every push to ensure syntax is correct
    - Run Terraform Init and Plan on Pull Requests and Merges
    - Run Terraform Apply automatically when code is merged to main
  - For Ansible:
    - Lint code on every push to ensure syntax is correct
  - For the Website:
    - **AWS:** Deploy to EC2 via SSM with OIDC authentication, eliminating long-lived credentials
    - **GCP:** Build with Hugo, stage files to GCS, then sync to the instance via IAP-tunneled SSH using OS Login and Workload Identity Federation

### GitHub Repo Links

- [Terraform Code AWS](https://github.com/AndrewFlan/site-terraform)
- [Terraform Code GCP](https://github.com/AndrewFlan/site-terraform-gcp)
- [Ansible Code](https://github.com/AndrewFlan/site-ansible)
- [Website Code](https://github.com/AndrewFlan/my-website)

## Details

### Terraform

The infrastructure is split across two Terraform repositories. One for AWS and one for GCP.

#### AWS

[Terraform Code AWS](https://github.com/AndrewFlan/site-terraform)

##### main.tf

The `main.tf` file does the following:

- Stores the Terraform state file in an S3 bucket called `andrewflanigan-terraform-state` with state locking via a DynamoDB table called `terraform-state-lock`
- Configures the AWS provider with default tags that apply to every resource created
- Configures items for AWS Systems Manager:
  - Creates an IAM instance profile to wrap the IAM role and attach to the EC2 instance
  - Creates an IAM role for the EC2 instance to use with SSM, allowing the instance to assume the role
  - Attaches AWS's managed SSM policy to give the EC2 instance permission to communicate with SSM
  - Attaches an inline policy giving the instance permission to read and write to S3, used to copy the website's static files
- Uploads an SSH public key to AWS for use with Ansible
- Retrieves the latest Ubuntu 24.04 AMI
- Creates an EC2 instance with the required properties: AMI, instance type, SSH key, security group, IAM instance profile, EBS, etc
- Allocates an Elastic IP and attaches it to the instance for use with DNS

##### network.tf

The `network.tf` file does the following:

- Creates a security group for the EC2 instance that allows:
  - **Ingress:** SSH (port 22) from any CIDR range (required since my IP is not static; access is restricted to trusted SSH keys), HTTP/HTTPS (ports 80/443)
  - **Egress:** All traffic allowed

##### dns.tf

The `dns.tf` file does the following:

- Looks up the existing Route53 hosted zone by domain name
- Creates an A record for the root domain pointing to the Elastic IP with a TTL of 300 seconds
- Creates an A record for the `www` subdomain pointing to the Elastic IP with a TTL of 300 seconds

##### variables.tf

The `variables.tf` file contains the following variables:

- AWS region
- Project name
- Instance type: set to `t3.micro`

##### outputs.tf

The `outputs.tf` file outputs the Elastic IP, instance ID, and SSH command once Terraform Apply completes.

##### Terraform Workflow

Triggers on pushes and pull requests to main. Authenticates to AWS via OIDC (temporary credentials valid for one hour), installs Terraform, then runs Init, Plan, and Apply (on merge to main only).

---

#### GCP

[Terraform Code GCP](https://github.com/AndrewFlan/site-terraform-gcp)

##### main.tf

The `main.tf` file does the following:

- Stores the Terraform state file in a GCS bucket called `andrewflanigan-terraform-state-gcp`
- Configures the Google provider with the project ID and region
- Creates a service account for the Compute Engine instance (`my-website-sa`) used for scoped access to GCP services
- Creates a GCS bucket for deployment staging, with a lifecycle rule to delete objects older than 1 day to prevent stale file accumulation
- Grants the instance service account `storage.objectAdmin` on the deploy bucket
- Creates a Compute Engine instance with OS Login enabled, the static IP attached, and the instance service account attached with `cloud-platform` scope

##### network.tf

The `network.tf` file does the following:

- Creates a firewall rule allowing SSH (port 22) only from Google's IAP range (`35.235.240.0/20`), ensuring the instance is never directly SSH-accessible from the internet
- Creates firewall rules allowing HTTP and HTTPS (ports 80/443) from any source
- Disables the default GCP `allow-ssh` and `allow-rdp` rules which would otherwise permit broad access

##### variables.tf

The `variables.tf` file contains the following variables:

- GCP project ID
- GCP region
- Project name
- Machine type: set to `e2-micro`
- Deploy bucket name
- Ansible SSH public key: registered in instance metadata for Ansible access

##### outputs.tf

The `outputs.tf` file outputs the static external IP address and instance name once Terraform Apply completes. The instance name is also automatically written back to the `my-website` repository as a GitHub Actions secret via the `gh` CLI.

##### Terraform Workflow

Triggers on pushes and pull requests to main. Authenticates to GCP via Workload Identity Federation (temporary credentials valid for one hour), installs Terraform, then runs Init, Plan, and Apply (on merge to main only). On apply, also runs tflint for linting and posts the plan output as a comment on pull requests.

### Ansible

#### playbook.yml

Specifies the roles to run: `common`, `nginx`, and `certbot`

#### roles/common/tasks/main.yml

The common role does the following:

- Runs system updates via `apt update` and `apt upgrade`
- Installs base packages: `curl`, `git`, `unzip`, `ufw`
- Installs fail2ban
- Creates a `deploy` group and user that owns the website root directory
- Creates the website root directory at `/var/www/andrewflanigan.com`
- Optionally adds an authorized SSH key for the `deploy` user when `deploy_public_key` is defined (used for AWS; skipped on GCP where OS Login handles access)
- Hardens SSH configuration:
  - `PasswordAuthentication no`: disables password login, key-based auth only
  - `PermitRootLogin no`: prevents logging in directly as root
  - `X11Forwarding no`: disables GUI forwarding, not needed on a server
  - `MaxAuthTries 3`: cuts off connections after 3 failed attempts
- Configures UFW (Uncomplicated Firewall):
  - Allows SSH, HTTP, and HTTPS
  - Default denies all other incoming traffic
- Ensures fail2ban is running to ban IPs with too many failed SSH attempts
- Installs the AWS CLI when `cloud_provider == "aws"`, required for SSM-based deployment
- Configures passwordless sudo for the GCP OS Login deploy service account when `cloud_provider == "gcp"`, allowing it to run `gcloud storage rsync` without a TTY

#### roles/nginx/tasks/main.yml

The Nginx role does the following:

- Installs Nginx
- Removes the default Nginx site
- Renders the `site.conf.j2` template into an Nginx site config, using `site_domain` to populate `server_name` — supports multiple comma-separated domains
- Enables the site via symlink to `sites-enabled`
- The `roles/nginx/templates/site.conf.j2` template configures an HTTP-only server block as the initial config before Certbot runs

#### roles/certbot/tasks/main.yml

The Certbot role does the following:

- Installs Certbot and its Nginx plugin
- Obtains a Let's Encrypt certificate covering all domains specified in `site_domain` (e.g. `gcp.andrewflanigan.com`, `andrewflanigan.com`, `www.andrewflanigan.com`)
- Writes the full HTTPS Nginx config from `site-https.conf.j2` after the certificate has been issued, replacing the initial HTTP-only config
- Enables Certbot auto-renewal via the `certbot.timer` systemd timer

#### Inventory

Two inventory files are maintained:

- `inventory` (AWS): connects directly via SSH using the `ubuntu` user and a private key
- `inventory-gcp` (GCP): connects via Google's IAP tunnel using a ProxyCommand, with OS Login handling authentication. GCP-specific variables (`site_domain`, `cloud_provider`, etc.) are set here, while sensitive values like `ansible_user` and the deploy SA UID are stored in a gitignored `group_vars/all.yml`

### Website

[Hugo](https://gohugo.io/) is used to generate the static site files for my website, using the [Hugo Profile](https://themes.gohugo.io/themes/hugo-profile/) theme.

#### Deploy Workflow

The website repository maintains two deploy workflows: one for AWS and one for GCP. Both triggered on pushes to the main branch. However, the AWS workflow is disabled for now while this site is being hosted on GCP.

##### AWS Deploy

- Checks out the code including Git submodules for the theme
- Installs Hugo and builds the static site
- Authenticates to AWS via OIDC (no long-lived credentials)
- Installs the SSM plugin
- Copies the built files to S3, then via SSM syncs them from S3 to the EC2 instance and waits for the command to complete

##### GCP Deploy

- Checks out the code including Git submodules for the theme
- Installs Hugo and builds the static site
- Authenticates to GCP via Workload Identity Federation (no long-lived credentials)
- Installs the `gcloud` CLI
- Syncs the built files to the GCS staging bucket using `gcloud storage rsync`
- Connects to the Compute Engine instance via an IAP tunnel using OS Login, then runs `gcloud storage rsync` on the instance to pull the files from GCS into the web root
- Error detection covers both local `gcloud` failures (before the SSH connection lands) and remote command failures (non-zero exit from the sync), ensuring silent failures are caught
