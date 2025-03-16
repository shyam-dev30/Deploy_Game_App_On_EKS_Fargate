# 🎮 Deploying a Game Application on Amazon EKS Fargate

This project demonstrates deploying a game application on AWS EKS Fargate. It covers the complete setup process, including EKS cluster creation, namespace, deployment configuration, and integrating AWS Load Balancer Controller for external access.

---

## 🛠️ Prerequisites

- **AWS Account** with appropriate permissions
- **EC2 Instance** (recommended: `t2.medium` or higher)
- **AWS CLI**, **eksctl**, and **kubectl** installed
- **Helm** (for AWS Load Balancer Controller)

---

## 🚀 Setup & Deployment

### **1️⃣ Set Up EC2 Instance**
1. Launch an **EC2 Instance** with the following specifications:
   - Instance Type: `t2.medium`
   - OS: Amazon Linux 2 (or any compatible Linux distribution)

2. SSH into your EC2 instance and install required tools:

   ```bash
   sudo yum update -y
   sudo yum install -y curl wget unzip

   # Install AWS CLI
   curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   unzip awscliv2.zip
   sudo ./aws/install

   # Install eksctl
   curl -sLO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz"
   tar -xzf eksctl_Linux_amd64.tar.gz
   sudo mv eksctl /usr/local/bin

   # Install kubectl
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   chmod +x kubectl
   sudo mv kubectl /usr/local/bin/
   ```

### **2️⃣ Create an EKS Fargate Cluster**

1. Run the following command to create a Fargate cluster:

   ```bash
   eksctl create cluster --name game-cluster --region us-east-1 --fargate
   ```

2. Update the `kubeconfig` file:

   ```bash
   aws eks update-kubeconfig --name game-cluster --region us-east-1
   ```

### **3️⃣ Create Namespace & Fargate Profile**

1. Create a namespace for your game application:

   ```bash
   kubectl create namespace game2048
   ```

2. Create a Fargate profile linked to the namespace:

   ```bash
   eksctl create fargateprofile \ 
      --cluster game-cluster \ 
      --region us-east-1 \ 
      --name alb-sample-app \ 
      --namespace game2048
   ```

### **4️⃣ Deploy the Game Application**

1. Create the following Kubernetes manifests:

   - **deployment.yaml** (for application deployment)
   - **service.yaml** (to expose the application internally)
   - **ingress.yaml** (for external access via AWS Load Balancer Controller)

2. Apply the manifests:

   ```bash
   kubectl apply -f deployment.yaml -f service.yaml -f ingress.yaml
   ```

### **5️⃣ Set Up AWS Load Balancer Controller**

1. Download the required IAM policy:

   ```bash
   curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
   ```

2. Create an IAM policy for the controller:

   ```bash
   aws iam create-policy \ 
      --policy-name AWSLoadBalancerControllerIAMPolicy \ 
      --policy-document file://iam_policy.json
   ```

3. Create an IAM role and associate it with the EKS cluster:

   ```bash
   eksctl create iamserviceaccount \ 
      --cluster=game-cluster \ 
      --namespace=kube-system \ 
      --name=aws-load-balancer-controller \ 
      --role-name AmazonEKSLoadBalancerControllerRole \ 
      --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \ 
      --approve
   ```

4. Install Helm (if not already installed):

   ```bash
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

5. Add and update the AWS Load Balancer Controller Helm chart repository:

   ```bash
   helm repo add eks https://aws.github.io/eks-charts
   helm repo update
   ```

6. Install the AWS Load Balancer Controller using Helm:

   ```bash
   helm install aws-load-balancer-controller eks/aws-load-balancer-controller \ 
      -n kube-system \ 
      --set clusterName=game-cluster \ 
      --set serviceAccount.create=false \ 
      --set serviceAccount.name=aws-load-balancer-controller \ 
      --set region=us-east-1 \ 
      --set vpcId=<your-vpc-id>
   ```

7. Verify the controller is running:

   ```bash
   kubectl get deployment -n kube-system aws-load-balancer-controller
   ```

### **6️⃣ Access the Game Application**

1. Navigate to the **EC2 Dashboard** in the AWS console.
2. Locate the newly created **Application Load Balancer**.
3. Copy the **DNS name** of the ALB.
4. Open your web browser and paste the URL to access the game.

### **7️⃣ Troubleshooting**

1. **If the game is not accessible:**
   - Check the security group of the ALB to allow **HTTP (port 80)** traffic.

   ```bash
   kubectl describe ingress -n game2048
   ```

2. **Verify pod status:**

   ```bash
   kubectl get pods -n game2048
   ```

---

## 📊 Summary

This project deploys a game application on **AWS EKS Fargate** with automatic scaling and external access through **AWS Load Balancer Controller**. This setup provides a fully managed, cost-effective, and scalable Kubernetes environment for your workloads.

Enjoy your game! 🎮

