# AWS VPC Ansible Module

A comprehensive Ansible module for creating and managing AWS VPC infrastructure with public, private, and database subnets, including security groups, NAT gateways, and VPC endpoints.

## Features

- **VPC Creation**: Creates a custom VPC with DNS resolution and hostnames enabled
- **Multi-tier Architecture**: 
  - Public subnets with Internet Gateway access
  - Private subnets with NAT Gateway access
  - Database subnets with no internet access
- **Security Groups**: Pre-configured security groups for web, application, and database tiers
- **High Availability**: Multi-AZ deployment across availability zones
- **VPC Endpoints**: Optional S3 and EC2 VPC endpoints for enhanced security
- **Tagging**: Comprehensive tagging for resource management and cost allocation

## Architecture

```
Internet Gateway
       |
   Public Subnets (10.0.1.0/24, 10.0.2.0/24)
       |
   NAT Gateways
       |
   Private Subnets (10.0.11.0/24, 10.0.12.0/24)
       |
   Database Subnets (10.0.21.0/24, 10.0.22.0/24)
```

## Prerequisites

- Ansible 2.9 or higher
- AWS CLI configured or AWS credentials available
- boto3 Python library installed
- AWS IAM permissions for VPC, EC2, and related services

## Required IAM Permissions

Your AWS credentials need the following permissions:
- `ec2:CreateVpc`
- `ec2:CreateSubnet`
- `ec2:CreateInternetGateway`
- `ec2:CreateNatGateway`
- `ec2:CreateRouteTable`
- `ec2:CreateSecurityGroup`
- `ec2:CreateVpcEndpoint`
- `ec2:AllocateAddress`
- `ec2:AssociateAddress`
- `ec2:AttachInternetGateway`
- `ec2:ModifyVpcAttribute`

## Installation

1. Clone or download this repository
2. Install required Python packages:
   ```bash
   pip install boto3 ansible
   ```

3. Configure AWS credentials:
   ```bash
   aws configure
   ```
   Or set environment variables:
   ```bash
   export AWS_ACCESS_KEY_ID=your_access_key
   export AWS_SECRET_ACCESS_KEY=your_secret_key
   export AWS_DEFAULT_REGION=us-east-1
   ```

## Configuration

### 1. Update Variables

Edit `group_vars/all.yml` to customize your VPC configuration:

```yaml
# VPC Configuration
vpc_name: "my-production-vpc"
vpc_cidr: "10.0.0.0/16"
environment: "production"
aws_region: "us-east-1"

# Availability Zones
az_list:
  - "us-east-1a"
  - "us-east-1b"
  - "us-east-1c"

# Network Configuration
public_subnets:
  - name: "{{ vpc_name }}-public-1"
    cidr: "10.0.1.0/24"
    az: "{{ az_list[0] }}"

# Security Configuration
admin_cidr: "203.0.113.0/24"  # Your admin IP range
```

### 2. Set Up Vault (Optional but Recommended)

For sensitive data like AWS credentials:

```bash
# Create vault file
ansible-vault create group_vars/vault.yml

# Edit vault file
ansible-vault edit group_vars/vault.yml
```

Add your AWS credentials:
```yaml
vault_aws_access_key: "YOUR_AWS_ACCESS_KEY"
vault_aws_secret_key: "YOUR_AWS_SECRET_KEY"
```

## Usage

### Basic VPC Creation

```bash
# Create VPC infrastructure
ansible-playbook -i inventory.ini site.yml --tags vpc

# Create with vault
ansible-playbook -i inventory.ini site.yml --tags vpc --ask-vault-pass
```

### Advanced Usage

```bash
# Create VPC with custom variables
ansible-playbook -i inventory.ini site.yml --tags vpc \
  -e "vpc_name=my-custom-vpc" \
  -e "environment=staging" \
  -e "vpc_cidr=172.16.0.0/16"

# Dry run to see what would be created
ansible-playbook -i inventory.ini site.yml --tags vpc --check

# Verbose output
ansible-playbook -i inventory.ini site.yml --tags vpc -vvv
```

### Specific Components

```bash
# Create only VPC and subnets
ansible-playbook -i inventory.ini vpc_module.yml --tags vpc

# Create with VPC endpoints
ansible-playbook -i inventory.ini vpc_module.yml \
  -e "create_vpc_endpoints=true"
```

## Customization

### Adding More Subnets

Edit `group_vars/all.yml`:

```yaml
# Add additional public subnets
public_subnets:
  - name: "{{ vpc_name }}-public-1"
    cidr: "10.0.1.0/24"
    az: "{{ az_list[0] }}"
  - name: "{{ vpc_name }}-public-2"
    cidr: "10.0.2.0/24"
    az: "{{ az_list[1] }}"
  - name: "{{ vpc_name }}-public-3"
    cidr: "10.0.3.0/24"
    az: "{{ az_list[2] }}"
```

### Custom Security Groups

Modify the security group rules in `vpc_module.yml`:

```yaml
- name: Create Security Groups
  ec2_group:
    name: "{{ item.name }}"
    description: "{{ item.description }}"
    vpc_id: "{{ vpc_result.vpc.id }}"
    rules: "{{ item.rules }}"
  loop:
    - name: "{{ vpc_name }}-custom-sg"
      description: "Custom security group"
      rules:
        - proto: tcp
          ports:
            - 8080
          cidr_ip: 10.0.0.0/16
```

### Different Regions

Update the region in `group_vars/all.yml` and adjust availability zones:

```yaml
aws_region: "eu-west-1"
az_list:
  - "eu-west-1a"
  - "eu-west-1b"
  - "eu-west-1c"
```

## Output

After successful execution, the playbook will display:

- VPC ID and CIDR block
- Internet Gateway ID
- Public subnet IDs and CIDR blocks
- Private subnet IDs and CIDR blocks
- Database subnet IDs and CIDR blocks
- Security group IDs

## Cleanup

To destroy the VPC infrastructure, create a cleanup playbook:

```yaml
---
- name: Cleanup AWS VPC Infrastructure
  hosts: localhost
  connection: local
  gather_facts: false
  
  tasks:
    - name: Delete VPC (this will delete all associated resources)
      ec2_vpc:
        state: absent
        vpc_id: "vpc-xxxxxxxxx"
        tags:
          Name: "my-production-vpc"
```

## Troubleshooting

### Common Issues

1. **Permission Denied**: Ensure your AWS credentials have the required IAM permissions
2. **Region Mismatch**: Verify the region in your configuration matches your AWS CLI configuration
3. **CIDR Conflicts**: Ensure your CIDR blocks don't overlap with existing networks
4. **Availability Zone Issues**: Some regions may not have all availability zones

### Debug Mode

Run with debug output to troubleshoot:

```bash
ansible-playbook -i inventory.ini site.yml --tags vpc -vvv
```

## Security Considerations

- Use IAM roles instead of access keys when possible
- Encrypt sensitive data using Ansible Vault
- Regularly rotate AWS credentials
- Implement least privilege access
- Monitor VPC Flow Logs for security analysis

## Cost Optimization

- NAT Gateways incur hourly charges - consider using NAT instances for development
- VPC Endpoints have usage charges - only enable if needed
- Use appropriate instance types for your workload
- Implement proper tagging for cost allocation

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Support

For issues and questions:
- Create an issue in this repository
- Check AWS documentation for service-specific questions
- Review Ansible documentation for playbook syntax

## Changelog

### Version 1.0.0
- Initial release
- VPC creation with public, private, and database subnets
- Security groups configuration
- NAT Gateway setup
- VPC endpoints support
- Comprehensive tagging