+++
title = "EKS Cluster Provisioning with VPC Configuration using Terraform"
+++

Terraform's declarative approach allows to describe the infrastructure in a high-level manner where it evaluates the current state of the infrastructure and determines what actions need to be taken to reach the desired state. Also, its idempotent nature means that applying the same configuration multiple times will result in the same end state as applying it once.

In this blog post I am going over how to setup EKS cluster using Terraform including all the pre-requistites of setting up VPC, SG etc.

### Pre-Requisites
- [Install Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
- [Install awscli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- Configure access, secret key and region values for the IAM user using `aws configure`

### Provider
Terraform has a Amazon Web Services (AWS) provider that is used to interact with the resources supported by AWS. First, we need to configure the provider with the proper credentials.

Configuration for the AWS Provider can be derived from several sources, which get applied in the following order:

- Parameters in the provider configuration
- Environment variables
- Shared credentials files
- Shared configuration files
- Container credentials
- Instance profile credentials and Region

In this case, I used the `aws configure` command to store the credentials in `.aws/credentials` file locally, thus the `Shared credentials files` method will get implemented.

In a production scenario, the standard method is to use assumed roles or OIDC (OpenID Connect) for authentication with AWS, where the AWS SDKs and CLI typically handle the credential retrieval process. 

#### provider.tf
```
terraform {
    required_providers {
        aws = {
            source = "hashicorp/aws"
            version = "~> 5.0"
        }
    }
}
```
---
### Initialization
Run `terraform init` to prepare the working directory for Terraform operations by setting up the necessary dependencies and configurations specified in the Terraform files.
```
~/awsproject  terraform init

Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.43.0...
- Installed hashicorp/aws v5.43.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

The file `terraform-provider-aws_v5.43.0_x5` is the AWS provider plugin binary used by Terraform to interact with the AWS API and manage AWS resources.
```
~/awsproject  ls .terraform/providers/registry.terraform.io/hashicorp/aws/5.43.0/darwin_amd64
total 1033600
-rwxr-xr-x  1 sanchit  staff  519433808 Mar 30 09:58 terraform-provider-aws_v5.43.0_x5
```
---
### Resources
`vpc.tf` file defines the VPC (Virtual Private Cloud) module with details about VPC itself, subnets, nat gateway and any other necessary networking components. With this module structure, you can create a reusable VPC module that can be included in your Terraform configurations for different environments or projects.
```
provider "aws" {
    region = var.aws_region
}

data "aws_availability_zones" "available_zones" {}

module "vpc" {
    source = "terraform-aws-modules/vpc/aws"
    version = "5.7.0"

    name                    = "sanchit-vpc"
    cidr                    = var.vpc_cidr
    azs                     = data.aws_availability_zones.available_zones.names
    private_subnets         = var.private_subnets
    public_subnets          = var.public_subnets
    enable_nat_gateway      = true
    enable_dns_support      = true
    enable_dns_hostnames    = true
    tags = {
        Terraform           = "true"
        Environment         = "dev"
        Cluster             = var.cluster_name
    }
}
```

`security-group.tf` file specifies the security groups for the VPC that is referenced in EKS cluster configuration, controlling inbound and outbound traffic to and from the resources.
```
resource "aws_security_group" "worker_mgmt" {
  name_prefix = "worker_node_management"
  vpc_id      = module.vpc.vpc_id
}

resource "aws_security_group_rule" "worker_mgmt_ingress" {
    description       = "allow inbound traffic from eks"
    from_port         = 0
    protocol          = "-1"
    to_port           = 0
    security_group_id = aws_security_group.worker_mgmt.id
    type              = "ingress"
    cidr_blocks       = var.ingress_sg_cidr_range
}

resource "aws_security_group_rule" "worker_mgmt_egress" {
    description       = "allow outbound traffic to anywhere"
    from_port         = 0
    protocol          = "-1"
    to_port           = 0
    security_group_id = aws_security_group.worker_mgmt.id
    type              = "egress"
    cidr_blocks       = var.egress_sg_cidr_range
}
```

`eks-cluster.tf` file configures the Amazon EKS (Elastic Kubernetes Service) cluster using eks module which includes cluster details, addons, node group and any other EKS-specific settings.
```
module "eks" {
    source = "terraform-aws-modules/eks/aws"
    version = "~>20.0"
    cluster_name = var.cluster_name
    cluster_version = var.cluster_version
    enable_irsa = true
    tags = {
        Terraform   = "true"
        Environment = "dev"
        Cluster     = var.cluster_name
    }

    cluster_endpoint_public_access  = true
    cluster_addons = {
        coredns = {
            most_recent = true
        }
        kube-proxy = {
            most_recent = true
        }
        vpc-cni = {
            most_recent = true
        }
    }

    vpc_id = module.vpc.vpc_id
    subnet_ids = module.vpc.private_subnets

    eks_managed_node_group_defaults = {
        ami_type               = var.ami_type
        instance_types         = var.instance_type
        vpc_security_group_ids = [aws_security_group.worker_mgmt.id]
    }

    eks_managed_node_groups = {
        node_group = {
            min_size      = var.min_capacity
            max_size      = var.max_capacity
            desired_size  = var.desired_capacity
            capacity_type = var.capacity_type
            key_name      = var.key_name
        }
    }
}
```

`variables.tf` file defines the input variables required for the Terraform configuration in the project.
```
variable "aws_region" {
    default = "us-east-2"
    description = "AWS Region"
}

variable "vpc_cidr" {
    description = "Default VPC CIDR range"
}

variable "private_subnets" {
    type = list(string)
    description = "List of private subnet CIDR blocks"
}

variable "public_subnets" {
    type = list(string)
    description = "List of public subnet CIDR blocks"
}

variable "cluster_name" {
    type = string
    description = "Name of EKS cluster"
}

variable "cluster_version" {
    type = string
    description = "Version of EKS cluster"
}

variable "ami_type" {
    type = string
    description = "The AMI type for worker nodes"
}

variable "instance_type" {
    type = list(string)
    description = "The instance type for worker nodes"
}

variable "min_capacity" {
    type = number
    description = "The minimum capacity for the node group"
}

variable "max_capacity" {
    type = number
    description = "The maximum capacity for the node group"
}

variable "desired_capacity" {
    type = number
    description = "The desired capacity for the node group"
}

variable "capacity_type" {
    type = string
    description = "The capacity type for worker nodes"
}

variable "key_name" {
    type = string
    description = "The key pair name for SSH access"
}

variable "ingress_sg_cidr_range" {
    type = list(string)
    description = "Ingress Security Group CIDR Allowed range"
}

variable "egress_sg_cidr_range" {
    type = list(string)
    description = "Ingress Security Group CIDR Allowed range"
}
```

`terraform.tfvars` file contains the values for the input variables defined in variables.tf, allowing to customize the configuration without modifying the main Terraform files.
```
# VPC
vpc_cidr = "10.0.0.0/16"

# Subnets
private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
public_subnets  = ["10.0.6.0/24", "10.0.7.0/24"]

# Security Group
ingress_sg_cidr_range = ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"]
egress_sg_cidr_range  = ["0.0.0.0/0"]

# EKS
cluster_name     = "sanchit-eks"
cluster_version  = "1.29"
ami_type         = "AL2_x86_64"
instance_type    = ["t2.micro"]
min_capacity     = 2
max_capacity     = 3
desired_capacity = 2
capacity_type    = "SPOT"
key_name         = "sanchit-key"
```

`outputs.tf` file is used to define outputs that one wants to expose from the modules or resources.
```
output "vpc_id" {
    value = module.vpc.vpc_id
}

output "cluster_id" {
    value = module.eks.cluster_id
}

output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "oidc_provider_arn" {
  value = module.eks.oidc_provider_arn
}
```
---
Re-Initialization for the VPC and EKS Modules:

```
~/awsproject  terraform init

Initializing the backend...
Initializing modules...
Downloading registry.terraform.io/terraform-aws-modules/vpc/aws 5.7.0 for vpc...
- vpc in .terraform/modules/vpc
Downloading registry.terraform.io/terraform-aws-modules/eks/aws 20.8.4 for eks...
- eks in .terraform/modules/eks
- eks.eks_managed_node_group in .terraform/modules/eks/modules/eks-managed-node-group
- eks.eks_managed_node_group.user_data in .terraform/modules/eks/modules/_user_data
- eks.fargate_profile in .terraform/modules/eks/modules/fargate-profile
Downloading registry.terraform.io/terraform-aws-modules/kms/aws 2.1.0 for eks.kms...
- eks.kms in .terraform/modules/eks.kms
- eks.self_managed_node_group in .terraform/modules/eks/modules/self-managed-node-group
- eks.self_managed_node_group.user_data in .terraform/modules/eks/modules/_user_data

Initializing provider plugins...
- Finding hashicorp/tls versions matching ">= 3.0.0"...
- Finding hashicorp/time versions matching ">= 0.9.0"...
- Finding hashicorp/null versions matching ">= 3.0.0"...
- Finding hashicorp/cloudinit versions matching ">= 2.0.0"...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Installing hashicorp/tls v4.0.5...
- Installed hashicorp/tls v4.0.5 (signed by HashiCorp)
- Installing hashicorp/time v0.11.1...
- Installed hashicorp/time v0.11.1 (signed by HashiCorp)
- Installing hashicorp/null v3.2.2...
- Installed hashicorp/null v3.2.2 (signed by HashiCorp)
- Installing hashicorp/cloudinit v2.3.3...
- Installed hashicorp/cloudinit v2.3.3 (signed by HashiCorp)
- Using previously-installed hashicorp/aws v5.43.0

Terraform has made some changes to the provider dependency selections recorded
in the .terraform.lock.hcl file. Review those changes and commit them to your
version control system if they represent changes you intended to make.

Terraform has been successfully initialized!
```

---
Plan (Truncated Output)
```
~/awsproject  terraform plan
...
Plan: 63 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + cluster_endpoint  = (known after apply)
  + cluster_id        = (known after apply)
  + oidc_provider_arn = (known after apply)
  + vpc_id            = (known after apply)
```
---
Apply (Truncated Output)

```
~/awsproject  terraform apply
...
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
...
Apply complete! Resources: 63 added, 0 changed, 0 destroyed.

Outputs:

cluster_endpoint = "https://167EB04858CCE0855BF0D0DA91DC159C.gr7.us-east-2.eks.amazonaws.com"
oidc_provider_arn = "arn:aws:iam::211125521645:oidc-provider/oidc.eks.us-east-2.amazonaws.com/id/167EB04858CCE0855BF0D0DA91DC159C"
vpc_id = "vpc-090f2c6840df227b1"
```

---
### Validation

VPC:
```
~/awsproject  aws --region us-east-2 ec2 describe-vpcs --vpc-ids 'vpc-090f2c6840df227b1' 
{
    "Vpcs": [
        {
            "CidrBlock": "10.0.0.0/16",
            "DhcpOptionsId": "dopt-0a1b84a258b357e47",
            "State": "available",
            "VpcId": "vpc-090f2c6840df227b1",
            "OwnerId": "211125521645",
            "InstanceTenancy": "default",
            "CidrBlockAssociationSet": [
                {
                    "AssociationId": "vpc-cidr-assoc-010848292ad0321b4",
                    "CidrBlock": "10.0.0.0/16",
                    "CidrBlockState": {
                        "State": "associated"
                    }
                }
            ],
            "IsDefault": false,
            "Tags": [
                {
                    "Key": "Name",
                    "Value": "sanchit-vpc"
                },
                {
                    "Key": "Cluster",
                    "Value": "sanchit-eks"
                },
                {
                    "Key": "Environment",
                    "Value": "dev"
                },
                {
                    "Key": "Terraform",
                    "Value": "true"
                }
            ]
        }
    ]
}
```

![VPC](https://github.com/sanchitpathak7/terraform/assets/44384286/958aa794-1319-4ab8-823e-043812e26020)

EKS Cluster:
```
~/awsproject  aws --region=us-east-2 eks describe-cluster --name=sanchit-eks
{
    "cluster": {
        "name": "sanchit-eks",
        "arn": "arn:aws:eks:us-east-2:211125521645:cluster/sanchit-eks",
        "createdAt": 1711832940.716,
        "version": "1.29",
        "endpoint": "https://167EB04858CCE0855BF0D0DA91DC159C.gr7.us-east-2.eks.amazonaws.com",
        "roleArn": "arn:aws:iam::211125521645:role/sanchit-eks-cluster-20240330210835870900000001",
        "resourcesVpcConfig": {
            "subnetIds": [
                "subnet-00352605c8be7996d",
                "subnet-03d24ec39d2fa152a"
            ],
            "securityGroupIds": [
                "sg-0ab70766a8ec0f71d"
            ],
            "clusterSecurityGroupId": "sg-0a2953c5f02876864",
            "vpcId": "vpc-090f2c6840df227b1",
            "endpointPublicAccess": true,
            "endpointPrivateAccess": true,
            "publicAccessCidrs": [
                "0.0.0.0/0"
            ]
        },
        "kubernetesNetworkConfig": {
            "serviceIpv4Cidr": "172.20.0.0/16",
            "ipFamily": "ipv4"
        },
        "logging": {
            "clusterLogging": [
                {
                    "types": [
                        "api",
                        "audit",
                        "authenticator"
                    ],
                    "enabled": true
                },
                {
                    "types": [
                        "controllerManager",
                        "scheduler"
                    ],
                    "enabled": false
                }
            ]
        },
        "identity": {
            "oidc": {
                "issuer": "https://oidc.eks.us-east-2.amazonaws.com/id/167EB04858CCE0855BF0D0DA91DC159C"
            }
        },
        "status": "ACTIVE",
        "certificateAuthority": {
            "data": "<REDACTED>"
        },
        "platformVersion": "eks.5",
        "tags": {
            "Cluster": "sanchit-eks",
            "Environment": "dev",
            "terraform-aws-modules": "eks",
            "Terraform": "true"
        },
        "encryptionConfig": [
            {
                "resources": [
                    "secrets"
                ],
                "provider": {
                    "keyArn": "arn:aws:kms:us-east-2:211125521645:key/6cc636b4-13c9-47aa-9fc2-4654a1c0e5ea"
                }
            }
        ],
        "health": {
            "issues": []
        }
    }
}
```
---
Generate Kubeconfig:
```
 ~/awsproject  aws --region=us-east-2 eks update-kubeconfig --name=sanchit-eks
Added new context arn:aws:eks:us-east-2:211125521645:cluster/sanchit-eks to /Users/sanchit/.kube/config
```

When trying to access the cluster, got error
`Your current IAM principal doesn't have access to Kubernetes objects on this cluster.
This might be due to the current principal not having an IAM access entry with permissions to access the cluster.`

Fixed by creating Access Policy AmazonEKSClusterAdminPolicy for the IAM user.

---
Now works :)
```
~/awsproject  kubectl get nodes                                                                                       
NAME                                       STATUS   ROLES    AGE   VERSION
ip-10-0-1-100.us-east-2.compute.internal   Ready    <none>   24m   v1.29.0-eks-5e0fdde
ip-10-0-2-208.us-east-2.compute.internal   Ready    <none>   25m   v1.29.0-eks-5e0fdde
```

---
When `terraform destroy` is run, Terraform will compare the current state of your infrastructure with the desired state described in your Terraform configuration and plan the destruction of any resources that are no longer needed. This can all be done in a YAML-based format as well using [EKSCTL](https://eksctl.io/). Will explore that in a different blog.