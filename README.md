![image](https://github.com/user-attachments/assets/2af05332-f090-4bbf-b5b6-bb79da5c6468)

# Managing AWS EKS with Terraform and deploy nginx application

In this tutorial, we'll learn how to use Terraform to manage an AWS Elastic Kubernetes Service (EKS) cluster.

## What is Terraform?

Terraform is an Infrastructure-as-Code (IaC) tool that enables you to define and provision infrastructure resources using a declarative configuration language.

## Why Use Terraform for AWS EKS?

1. **Infrastructure-as-Code**: Automate and version control your infrastructure.
2. **Scalability**: Easily scale your EKS cluster with Terraform configurations.
3. **Efficiency**: Provision multiple AWS services with a single tool.

## Prerequisites

1. **Terraform**: Installed on your local machine. You can set it up following this [guide](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli).
2. **AWS Account**: You’ll need an AWS account to access AWS EKS and other services. If you don’t have one, sign up [here](https://signin.aws.amazon.com/signup?request_type=register).
3. **AWS CLI**: Installed and configured with your AWS credentials.
4. **kubectl**: Installed to manage your EKS cluster from your machine.

## Table of Contents

1. [Define the AWS Provider](#define-the-aws-provider)
2. [Define the VPC and Networking Resources](#define-the-vpc-and-networking-resources)
3. [Define the EKS Cluster](#define-the-eks-cluster)
4. [Create EKS Worker Nodes](#create-eks-worker-nodes)
5. [Apply the Terraform Configuration](#apply-the-terraform-configuration)
6. [Configure kubectl to Access the EKS Cluster](#configure-kubectl-to-access-the-eks-cluster)
7. [Deploy an Application Using Terraform](#deploy-an-application-using-terraform)
8. [Clean Up](#clean-up)

## Define the AWS Provider

Create main.tf file and add the following configuration:

```
provider "aws" {
  region = "us-east-1"
}

```
## Define the VPC and Networking Resources

AWS EKS requires a Virtual Private Cloud (VPC) and subnets to run.
Add the configuration to the network.tf file. This will create a VPC, subnets, an internet gateway and route tables for your EKS cluster.

```
data "aws_availability_zones" "available" {}

resource "aws_vpc" "eks_vpc" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "eks_subnet" {
  count                   = 2
  vpc_id                  = aws_vpc.eks_vpc.id
  cidr_block              = cidrsubnet(aws_vpc.eks_vpc.cidr_block, 8, count.index)
  availability_zone       = element(data.aws_availability_zones.available.names, count.index)
  map_public_ip_on_launch = true
}

resource "aws_internet_gateway" "eks_gw" {
  vpc_id = aws_vpc.eks_vpc.id
}

resource "aws_route_table" "eks_route_table" {
  vpc_id = aws_vpc.eks_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.eks_gw.id
  }
}

resource "aws_route_table_association" "eexampleks_route_table_association" {
count = 2
  subnet_id      = element(aws_subnet.eks_subnet[*].id, count.index)
  route_table_id = aws_route_table.eks_route_table.id
}

```
## Define the EKS Cluster

Let's create the EKS cluster itself, along with the required IAM role to manage the cluster in eks.tf

```
resource "aws_iam_role" "eks_role" {
  name = "eks_role"
    
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
        Principal = {
          Service = ["ec2.amazonaws.com","eks.amazonaws.com"]
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "eks_policy" {
  role       = aws_iam_role.eks_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

resource "aws_iam_role_policy_attachment" "eks_node_policy" {
  role       = aws_iam_role.eks_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "eks_cni_policy" {
  role       = aws_iam_role.eks_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
}

resource "aws_iam_role_policy_attachment" "eks_ec2_policy" {
  role       = aws_iam_role.eks_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}

resource "aws_eks_cluster" "eks_cluster" {
  name     = "eks-cluster"
  role_arn = aws_iam_role.eks_role.arn

  vpc_config {
    subnet_ids = aws_subnet.eks_subnet[*].id
  }
}

```

- The EKS cluster needs an IAM role that grants it permissions to manage AWS services. Therefore, we attach eks_role with the necessary policies to EKS.
- We also create an EKS cluster using aws_eks_cluster and specify the VPC subnets created earlier.

## Create EKS Worker Nodes
Create eks-workers.tf file. This will create a node group that will scale between 1 to 3 nodes.

```
resource "aws_eks_node_group" "node_group" {
  cluster_name    = aws_eks_cluster.eks_cluster.name
  node_group_name = "eks-node-group"
  node_role_arn   = aws_iam_role.eks_role.arn
  subnet_ids      = aws_subnet.eks_subnet[*].id

  scaling_config {
    desired_size = 2  # Initial number of nodes
    max_size     = 3  # Maximum number of nodes
    min_size     = 1  # Minimum number of nodes
  }

```
- We define an EKS node group, which is a set of EC2 instances (worker nodes) that run the Kubernetes workloads.
- The node group can scale between 1 and 3 nodes, depending on your workload's demand.

## Apply the Terraform Configuration
- Initialize terraform

```
terraform init

```
- See the changes Terraform plans to make in our infrastructure. 
```
terraform plan

```
![image](https://github.com/user-attachments/assets/e5b657be-8f1b-4d7f-96f8-8555940c8bd2)

- If the planning looks good then apply the configuration to create the EKS cluster and its resources.
```
terraform apply

```
Confirm with yes when prompted.
![image](https://github.com/user-attachments/assets/f7fc7344-0269-4553-b38d-3e1f05722426)

- Check your AWS console, your EKS cluster should be up and running!(It took 15-20 mins to complete)
  ![image](https://github.com/user-attachments/assets/c5013eaf-4018-4b1d-9138-02c0ab36b1ef)


## Configure kubectl to Access the EKS Cluster
To manage the EKS cluster, we need to configure kubectl. This can be done using the AWS CLI.

```
aws eks --region eu-west-2 update-kubeconfig --name eks-cluster

```

- Confirm that you have access to the EKS cluster by listing Kubernetes nodes and services.
![image](https://github.com/user-attachments/assets/1c9821e8-c6e7-4059-af37-481331a02fb2)

## Deploy an Application Using Terraform
Let’s deploy a simple Nginx application to the EKS cluster using the Kubernetes provider in Terraform (deploy-nginx.tf)

```
provider "kubernetes" {
  host                   = aws_eks_cluster.eks_cluster.endpoint
  cluster_ca_certificate = base64decode(aws_eks_cluster.eks_cluster.certificate_authority.0.data)
  token                  = data.aws_eks_cluster_auth.eks_auth.token
}

data "aws_eks_cluster_auth" "eks_auth" {
  name = aws_eks_cluster.eks_cluster.name
}

resource "kubernetes_pod" "nginx" {
  metadata {
    name = "nginx"
    labels = {
      app = "nginx"
    }
  }

  spec {
    container {
      name  = "nginx"
      image = "nginx:latest"

      resources {
        limits = {
          cpu    = "0.5"
          memory = "512Mi"
        }
        requests = {
          cpu    = "0.25"
          memory = "256Mi"
        }
      }
    }
  }
}

resource "kubernetes_service" "nginx_service" {
  metadata {
    name = "nginx-service"
  }

  spec {
    selector = {
      app = "nginx"
    }
    port {
      port = 80
      target_port = 80
    }
    type = "LoadBalancer"
  }
}

```

- The Kubernetes Pod defines an Nginx pod in the EKS cluster.
- The Kubernetes Service exposes the Nginx pod as a LoadBalancer service so that it can be accessed publicly.
- Now deploy it to your EKS cluster.

```
terraform init -upgrade
  
terraform plan

terraform apply

```

Once the deployment is complete, you can check that the Nginx pod and service are running in your cluster

```
kubectl get pods

kubectl get svc

```
![image](https://github.com/user-attachments/assets/4d0e8b3b-f2e4-40ca-99e3-cc0163924a1d)

![image](https://github.com/user-attachments/assets/7ee0c728-e9a3-4622-b4d9-7f9fcf61084c)

## Clean Up 
If you no longer need all these resources, you can destroy them all to avoid unnecessary charges.
```
terraform destroy

```



  
