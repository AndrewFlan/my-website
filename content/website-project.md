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

# About my Website Project

## Intro

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

## GitHub Repo Links

- [Terraform Code](https://github.com/AndrewFlan/site-terraform)
- [Ansible Code](https://github.com/AndrewFlan/site-ansible)
- [Website Code](https://github.com/AndrewFlan/my-website)
