# To deploy a Sparx PCS application on AWS Elastic Kubernetes Service


<img width="847" alt="image" src="https://github.com/user-attachments/assets/7cb2ac01-a281-4075-9319-0b313fa376f9">


**Prerequisites**

1. AWS CloudShell access
2. AWS CLI
3. EKSCTL
4. KUBECTL

## Install EKSCTL in AWS CloudShell

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl info
```

Refer to [this link](https://eksctl.io/installation/)

## Install AWS CLI in AWS CloudShell

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Refer to [this link](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)


## Install KUBECTL in AWS CloudShell

```
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.31.2/2024-11-15/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
```

Refer to [this link](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html#linux_amd64_kubectl)

## To create a EKS Cluster

Clone the below URL and create a file named cluster.config.yaml in CloudShell.

```
https://github.com/6thforce/sparxpcs-eks/blob/main/cluster-config.yaml
```

To change the region, instance type, volume size in cluster-config.yaml if need.

### To deploy a cluster using eksctl

```
eksctl create cluster -f cluster-config.yaml
```

Deploying an EKS cluster can take more than 30 minutes to complete.

### To verify the EKS Cluster

The following commands are used to verify the node status, running pod status, and namespaces in a Kubernetes cluster:

1. Verify Node Status:

```
kubectl get nodes

```

2. Verify Running Pod Status:

```
kubectl get pods -A
```

3. Verify Namespace:

```
kubectl get namespaces
```

### To associate a IAM OIDC

```
cluster_name=eks-windows-mng-fg-mix
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

### To enable a IAM roles for Service Accounts

```
eksctl create iamserviceaccount \
        --name ebs-csi-controller-sa \
        --namespace kube-system \
        --cluster eks-windows-mng-fg-mix \
        --role-name AmazonEKS_EBS_CSI_DriverRole \
        --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
        --approve \
        --override-existing-serviceaccounts
```

If you did not create IAM roles for a Service Account properly, you can delete the Service Account and recreate it. Use the following command to delete the Service Account:

```
eksctl delete iamserviceaccount \
        --name ebs-csi-controller-sa \
        --namespace kube-system \
        --cluster eks-windows-mng-fg-mix
```

### To install EBS CSI Driver Addons

Steps to Install EBS CSI Driver Add-on:

1. Navigate to the AWS Console: Open the Amazon EKS Console and Select your EKS cluster from the list.

2. Add-ons: In the cluster dashboard, click on Add-ons from the left-hand menu

3. Choose EBS CSI Driver: Click Add add-on, Search for and select the EBS CSI driver from the list of available add-ons.

4. IAM Role: Click Next and select the IAM Role you created in the previous step. This role must have the necessary permissions for the EBS CSI Driver.

5. Override and Install: Proceed to the Next step. If an error occurs during installation, select the Override existing settings option and retry the installation.

6. Install: Click Install to complete the process.

7. To verify the status of the EBS CSI Driver after installation, use the following command:

```
kubectl get pods -A
```

## To create a Namespace for pcs application

To deploy the namespace, use the manifest file available at the following link:

```
https://github.com/6thforce/sparxpcs-eks/blob/main/namespace.yaml
```

To deploy and verify the namespace

```
kubectl apply -f https://github.com/6thforce/sparxpcs-eks/blob/main/namespace.yaml
kubectl get namespace
```

## To create a Storage Class for mounting EBS volume

To deploy the storageclass, use the manifest file available at the following link:

```
https://github.com/6thforce/sparxpcs-eks/blob/main/storage-class.yaml
```

To deploy and verify the storageclass

```
kubectl apply -f https://github.com/6thforce/sparxpcs-eks/blob/main/storage-class.yaml
kubectl get sc
```

## To create a Persistent Volume Claim

To deploy the persistent volume claim, use the manifest file available at the following link:

```
https://github.com/6thforce/sparxpcs-eks/blob/main/persistent-volume-claim.yaml
```

To deploy and verify the persistent volume claim status

```
kubectl apply -f https://github.com/6thforce/sparxpcs-eks/blob/main/persistent-volume-claim.yaml
kubectl get pvc -n windows
```

## To verify the OIDC

To verify the OIDC configuration, use the following commands:

```
oidc_id=$(aws eks describe-cluster --name eks-windows-mng-fg-mix --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

## To verify the Add-ons

To check if the following add-ons are active in the EKS dashboard — VPC CNI, kube-proxy, and CoreDNS.

## To create IAM Policy for Application Loadbalancer

To create an IAM policy, download the JSON file from the following link and use it to create the IAM policy:

```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

## To create a  IAM Service Account for Application Loadbalancer

To create an IAM Service Account for the LoadBalancer using the following eksctl command, run:

```
eksctl create iamserviceaccount \
  --cluster=eks-windows-mng-fg-mix \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<account-number>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --override-existing-serviceaccounts
```

## To install a Helm package

To install a Helm package using the following command, run:

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Refer to [this link](https://helm.sh/docs/intro/install/)

## To install a AWS Application Loadbalancer controller

To download and install a ALB controller using Helm repository:

```
helm repo add eks https://aws.github.io/eks-charts  
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eks-windows-mng-fg-mix \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<vpc-id>
```

## To verify the VPC Subnets

To verify that the following tags are present in the two public subnets of your VPC for subnet auto-discovery by the controller for your internet-facing ALB, check the tags in the subnets:

```
Key: kubernetes.io/role/elb
Value: 1
```

## To deploy the pcs deployment and service

To deploy a pcs workload with replicasets, deployment and service using the following command, run:

```
kubectl apply -f https://github.com/6thforce/sparxpcs-eks/blob/main/pcs-workload.yaml
kubectl get pods -o wide -n windows
kubectl get svc
```

## To copy a server.pem file to the shared persistent volume

To copy a server.pem file to container mounted shared drive using the following command, run:

```
kubectl get pods -o wide -n windows
kubectl cp ./server.pem <pod-name>:/C:/pcsservice/Service/shared/server.pem
```

## To deploy a Ingress service manifest

To deploy a Ingress service to map a loadbalancer to pcs application using the following command, run:

```
kubectl apply -f https://github.com/6thforce/sparxpcs-eks/blob/main/ingress.yaml
kubectl get ingress -n windows
```

## To access the pcs application

To get and access the application using Application Loadbalancer URL using the following command, run:

```
kubectl get svc
```
