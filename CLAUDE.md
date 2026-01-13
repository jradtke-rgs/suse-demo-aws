# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains OpenTofu infrastructure-as-code for deploying a complete SUSE product demo environment in AWS. The project is organized as separate OpenTofu modules that must be deployed in a specific order due to state dependencies.

## Architecture

The infrastructure is modularized into four main components:

1. **shared-services/** - Foundation infrastructure (VPC, networking, security groups)
2. **rancher-manager/** - SUSE Rancher Manager for Kubernetes management
3. **observability/** - SUSE Observability monitoring stack
4. **security/** - SUSE Security and compliance tools

### State Dependencies

All product modules depend on the shared-services OpenTofu state file located at `shared-services/terraform.tfstate`. This creates a critical dependency chain:
- Product modules reference shared infrastructure via `terraform_remote_state` data sources
- The shared-services module MUST be deployed first and destroyed last
- Never modify shared-services after other modules are deployed

## Common Development Commands

### Deployment Order (Required)
```bash
# 1. Deploy shared services first
cd shared-services
tofu init
tofu plan
tofu apply

# 2. Deploy individual SUSE products
cd ../rancher-manager
tofu init && tofu plan && tofu apply

cd ../observability
tofu init && tofu plan && tofu apply

cd ../security
tofu init && tofu plan && tofu apply
```

### Destruction Order (Required)
```bash
# Destroy in reverse order to respect dependencies
cd ./security && tofu destroy && cd -
cd ./observability && tofu destroy && cd -
cd ./rancher-manager && tofu destroy && cd -
cd ./shared-services && tofu destroy && cd -
```

### Configuration Management
Each module has its own `terraform.tfvars.example` file that should be copied to `terraform.tfvars` and customized before deployment.

### Security Configuration
- Default CIDR blocks are set to `0.0.0.0/0` for demonstration purposes
- In `terraform.tfvars`, restrict `allowed_ssh_cidr_blocks` and `allowed_web_cidr_blocks` to specific IP addresses for production use
- SSH keys are managed per module via the `ssh_public_key` variable

## Key Technical Details

### Provider Requirements
- **OpenTofu >= 1.5.0** - Infrastructure provisioning tool
  - Installation: https://opentofu.org/docs/intro/install/
  - macOS: `brew install opentofu`
- AWS Provider ~> 5.0
- All modules use consistent default tags (Environment, Project, ManagedBy, Owner)

### SUSE Product Integration
- Each product module includes SUSE Customer Center (SCC) registration variables (`suse_email`, `suse_regcode`)
- User data scripts handle SUSE product installation and configuration
- AMI selection uses latest SUSE Linux Enterprise Server images

### Version Management
All product and infrastructure component versions are managed through variables in `common-vars.tf` and configured via `terraform.tfvars`:

**Available Version Variables:**
- `rancher_version` - SUSE Rancher Manager version
- `cert_manager_version` - Cert-manager version (used across all modules)
- `neuvector_version` - NeuVector security platform version
- `k3s_version` - K3s Kubernetes distribution version (applies to rancher-manager, observability, security modules)

**K3s Version Control:**
- **Default behavior**: Leave `k3s_version` empty or commented out to use latest stable K3s release
- **Pinned version**: Set to specific version (e.g., `k3s_version = "v1.30.0+k3s1"`) to pin across all modules
- Version format must match K3s release tags (e.g., `v1.30.0+k3s1`)
- Find available versions at: https://github.com/k3s-io/k3s/releases
- Implementation uses `INSTALL_K3S_VERSION` environment variable in user-data scripts
- Changing K3s version on existing instances requires instance recreation (triggers user-data change)

**Configuration Location:**
- Variable definitions: `common-vars.tf` (lines 203-225)
- Configuration: `terraform.tfvars` (unified config for all modules)
- Example configuration: `terraform.tfvars.example` (lines 117-127)

### Network Architecture
- Single VPC with public subnets only (no NAT Gateway for cost optimization)
- All instances receive public IPs and are internet-accessible
- Security groups control access to specific ports (SSH:22, HTTP:80, HTTPS:443, K8s API:6443)

### DNS and Certificate Management
- Optional Route53 integration for custom domain names
- Cert-manager integration for TLS certificate automation
- EIP allocation available for stable public IP addresses

## Important Caveats

- This is a **demo/lab environment only** - not suitable for production
- Cost optimization decisions (public subnets, no NAT Gateway) compromise security
- All infrastructure uses a subdomain: `suse-demo-aws.kubernerdes.com`
- Manual confirmation required for tofu destroy operations