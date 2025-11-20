
# AWS EKS Infrastructure with Terraform

A production-ready Terraform project for deploying and managing Amazon EKS (Elastic Kubernetes Service) clusters with comprehensive IAM management, EFS storage integration, and RBAC configuration.

## 🏗️ Architecture Overview

This project provisions a complete EKS infrastructure including:

- **VPC & Networking**: Custom VPC with public/private subnets across multiple AZs
- **EKS Cluster**: Managed Kubernetes control plane with worker node groups
- **IAM**: Role-based access control with multiple user personas (Admin, Developer, ReadOnly)
- **Storage**: EFS CSI driver with static provisioning for persistent storage
- **Security**: Bastion host for secure cluster access, security groups, and OIDC provider
- **RBAC**: Kubernetes role bindings for namespace and cluster-level permissions

## 📋 Prerequisites

- **Terraform**: >= 1.6.0
- **AWS CLI**: Configured with appropriate credentials
- **kubectl**: For Kubernetes cluster management
- **AWS Account**: With permissions to create EKS, VPC, IAM, and EC2 resources
- **SSH Key Pair**: For bastion host access

## 🚀 Quick Start

### 1. Clone the Repository

```bash
git clone <repository-url>
cd <project-directory>
```

### 2. Configure Variables

Create a `terraform.tfvars` file in the `environment/development/` directory:

```hcl
# AWS Configuration
region                      = "us-east-1"
aws_region                  = "us-east-1"
environment                 = "dev"
business_division           = "hr"

# VPC Configuration
vpc_cidr_block              = "10.0.0.0/16"
vpc_public_subnets          = ["10.0.101.0/24", "10.0.102.0/24"]
vpc_private_subnets         = ["10.0.1.0/24", "10.0.2.0/24"]
vpc_enable_nat_gateway      = true

# EKS Configuration
cluster_version             = "1.28"
eks_oidc_root_ca_thumbprint = "9e99a48a9960b14926bb7f3b02e22da2b0ab7280"

# EC2 Bastion Configuration
instance_type               = "t3.micro"
instance_keypair            = "your-key-pair-name"

# Kubernetes Configuration
namespace                   = "dev"

# EFS CSI Driver
efs_image_repo              = "602401143452.dkr.ecr.us-east-1.amazonaws.com/eks/aws-efs-csi-driver"
```

### 3. Initialize Terraform

```bash
cd environment/development
terraform init
```

### 4. Review the Plan

```bash
terraform plan
```

### 5. Deploy Infrastructure

```bash
terraform apply
```

## 📁 Project Structure

```
.
├── environment/
│   ├── development/          # Development environment configuration
│   │   ├── backend.tf       # S3 backend configuration
│   │   ├── main.tf          # Main orchestration file
│   │   ├── provider.tf      # Provider configurations
│   │   ├── variable.tf      # Variable definitions
│   │   └── output.tf        # Output values
│   ├── staging/             # Staging environment (placeholder)
│   └── production/          # Production environment (placeholder)
│
├── module/
│   ├── vpc/                 # VPC module - networking infrastructure
│   ├── eks/                 # EKS cluster and node group configuration
│   ├── iam/                 # IAM roles for EKS (master, node, user roles)
│   ├── iam_users/           # IAM users and groups management
│   ├── kubernetes/          # Kubernetes RBAC and namespace configuration
│   ├── ec2/                 # Bastion host configuration
│   └── efs-static/          # EFS storage configuration
│       ├── efs_csi/         # EFS CSI driver installation
│       └── efs_app/         # EFS application and PV/PVC setup
│
└── README.md
```

## 🔧 Module Details

### VPC Module (`module/vpc`)
Creates a complete networking infrastructure:
- VPC with DNS support
- Public and private subnets across multiple AZs
- Internet Gateway for public subnets
- NAT Gateway for private subnet internet access (optional)
- Route tables and associations
- Kubernetes-specific subnet tags for load balancers

### EKS Module (`module/eks`)
Provisions the Kubernetes cluster:
- EKS control plane with specified version
- Managed node groups with auto-scaling
- OIDC provider for IAM roles for service accounts (IRSA)
- CloudWatch logging enabled for all control plane components
- Public endpoint access configuration

### IAM Module (`module/iam`)
Manages all IAM resources:
- EKS master role (control plane)
- EKS node group role (worker nodes)
- OIDC provider for service account authentication
- Three user roles: Admin, Developer, ReadOnly
- Appropriate policy attachments for each role

### IAM Users Module (`module/iam_users`)
Creates IAM users and groups:
- Admin users with full access
- User groups with assume-role policies
- Group memberships for role assumption
- Three groups: eksadmins, eksreadonly, eksdeveloper

### Kubernetes Module (`module/kubernetes`)
Configures Kubernetes-level resources:
- AWS auth ConfigMap for IAM role mapping
- Namespace creation
- Cluster roles and bindings
- Role-based access control (RBAC) for different user types
- Developer, ReadOnly, and Admin permissions

### EC2 Module (`module/ec2`)
Provisions bastion host:
- Amazon Linux 2 AMI
- Public subnet placement
- Elastic IP association
- Security group allowing SSH access
- Monitoring enabled

### EFS Module (`module/efs-static`)
Implements persistent storage:
- **efs_csi**: Installs EFS CSI driver via Helm with IRSA
- **efs_app**: Creates EFS file system, mount targets, storage classes, PV/PVC
- Sample application deployment using EFS
- Load balancer service for application access

## 👥 IAM User Types and Access

### 1. Admin Users
- **Role**: `eks-admin-role`
- **Permissions**: Full EKS cluster access, system:masters group
- **Users**: `eks-eksadmin1`, `eks-eksadmin2`, `eks-eksadmin3`

### 2. Developer Users
- **Role**: `eks-developer-role`
- **Permissions**: 
  - Namespace-level: Full access to resources in assigned namespace
  - Cluster-level: Read access to nodes, pods, services, deployments
- **Users**: `eks-eksdeveloper1`

### 3. ReadOnly Users
- **Role**: `eks-readonly-role`
- **Permissions**: Read-only access to cluster resources
- **Users**: `eks-eksreadonly1`

## 🔐 Accessing the Cluster

### Configure kubectl

```bash
aws eks update-kubeconfig --region us-east-1 --name <cluster-name>
```

### Access via Bastion Host

```bash
# Get bastion IP from Terraform output
terraform output bastion_public_ip

# SSH to bastion
ssh -i your-key.pem ec2-user@<bastion-ip>

# Configure kubectl on bastion
aws eks update-kubeconfig --region us-east-1 --name <cluster-name>
```

### Assume IAM Role (for group users)

```bash
# Configure AWS CLI to assume role
aws sts assume-role --role-arn <role-arn> --role-session-name my-session

# Export credentials
export AWS_ACCESS_KEY_ID=<access-key>
export AWS_SECRET_ACCESS_KEY=<secret-key>
export AWS_SESSION_TOKEN=<session-token>

# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name <cluster-name>
```

## 💾 EFS Storage Usage

The project includes a sample application demonstrating EFS usage:

### Storage Class
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
```

### Persistent Volume and Claim
- **PV**: `efs-pv` - 5Gi capacity, ReadWriteMany access
- **PVC**: `efs-claim` - Bound to the PV

### Sample Application
- **Deployment**: `myapp1` - 2 replicas with EFS mount
- **Service**: LoadBalancer type for external access
- **Test Pod**: `efs-write-app` - Continuously writes to EFS

### Verify EFS Setup

```bash
# Check storage class
kubectl get sc efs-sc

# Check PV and PVC
kubectl get pv,pvc

# Check application pods
kubectl get pods

# View data written to EFS
kubectl exec -it efs-write-app -- cat /data/efs-static.txt

# Get LoadBalancer URL
kubectl get svc myapp1-lb-service
```

## 📤 Outputs

After successful deployment, Terraform provides these outputs:

```hcl
vpc_id                  # VPC identifier
vpc_name                # VPC name
eks_cluster_endpoint    # EKS API server endpoint
eks_cluster_name        # EKS cluster name
bastion_public_ip       # Bastion host public IP
efs_csi_role_arn        # EFS CSI driver IAM role ARN
efs_helm_metadata       # EFS Helm release metadata
```

View outputs:
```bash
terraform output
```

## 🔄 State Management

The project uses S3 backend for remote state storage:

- **Bucket**: `terraform-on-aws-eks2025110431290`
- **Key**: `dev/eks-cluster/terraform.tfstate`
- **Region**: `us-east-1`
- **DynamoDB Table**: `dev-efs-eks202511043` (for state locking)
- **Encryption**: Enabled

## 🛠️ Maintenance Operations

### Scaling Node Groups

Edit the node group configuration in `module/eks/nodegroup.tf`:

```hcl
scaling_config {
  desired_size = 2  # Change as needed
  min_size     = 1
  max_size     = 4
}
```

Apply changes:
```bash
terraform apply
```

### Updating EKS Version

Update `cluster_version` in your `terraform.tfvars`:

```hcl
cluster_version = "1.29"  # New version
```

Apply with caution:
```bash
terraform plan
terraform apply
```

### Adding New IAM Users

Modify `module/iam_users/main.tf` to add new users and groups, then apply:

```bash
terraform apply
```

## 🧹 Cleanup

To destroy all resources:

```bash
cd environment/development
terraform destroy
```

**Warning**: This will delete all resources including the EKS cluster, VPC, and data stored in EFS. Ensure you have backups if needed.

## ⚠️ Important Notes

1. **Costs**: Running this infrastructure incurs AWS charges (EKS cluster, EC2 instances, NAT Gateway, EFS, etc.)

2. **Security**: 
   - The bastion host allows SSH from anywhere (0.0.0.0/0) - restrict this in production
   - EKS API endpoint is publicly accessible - consider private endpoint for production

3. **OIDC Thumbprint**: The default thumbprint is for us-east-1. Update for other regions:
   ```bash
   openssl s_client -connect oidc.eks.region.amazonaws.com:443 -showcerts
   ```

4. **EFS Image Repository**: Update `efs_image_repo` based on your AWS region. Refer to [AWS documentation](https://docs.aws.amazon.com/eks/latest/userguide/add-ons-images.html)

5. **State Locking**: Ensure the DynamoDB table for state locking exists before running Terraform

## 🐛 Troubleshooting

### EKS Cluster Not Accessible

```bash
# Verify AWS credentials
aws sts get-caller-identity

# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name <cluster-name>

# Check IAM permissions in aws-auth ConfigMap
kubectl get configmap aws-auth -n kube-system -o yaml
```

### EFS Mount Issues

```bash
# Check EFS CSI driver pods
kubectl get pods -n kube-system | grep efs

# Verify EFS mount targets
aws efs describe-mount-targets --file-system-id <efs-id>

# Check security group rules
aws ec2 describe-security-groups --group-ids <efs-sg-id>
```

### Node Group Not Ready

```bash
# Check node status
kubectl get nodes

# Describe node for details
kubectl describe node <node-name>

# Check node group in AWS console or CLI
aws eks describe-nodegroup --cluster-name <cluster-name> --nodegroup-name <nodegroup-name>
```

## 📚 Additional Resources

- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [EFS CSI Driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly in a development environment
5. Submit a pull request

**Note**: This is a development configuration. For production use, review and adjust security settings, enable private endpoints, implement proper backup strategies, and follow AWS best practices for EKS deployments.
