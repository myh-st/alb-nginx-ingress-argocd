
# My EKS Project

This repository contains the necessary Kubernetes manifests to set up an EKS cluster with ALB and NGINX ingress controllers, deploy a sample application, and manage it using ArgoCD.

## Repository Structure

```
my-eks-project/
├── manifests/
│   ├── sample-app-deployment.yaml
│   └── sample-app-service.yaml
│   ├── alb-ingress.yaml
│   └── nginx-ingress.yaml
├── argocd/
│   └── argocd-install.yaml
└── README.md
```
### Prerequisites
1. **AWS CLI**: Installed and configured.
2. **kubectl**: Installed and configured.
3. **eksctl**: Installed.
4. **Helm**: Installed.
5. **ArgoCD CLI**: Installed.

## Steps to Deploy

### Step 1: Create an EKS Cluster

```sh
eksctl create cluster --name my-cluster --region us-west-2 --nodegroup-name linux-nodes --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 4 --managed
```

### Step 2: Install the AWS Load Balancer Controller

1. **Create IAM Policy**

```sh
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

2. **Create IAM Role and Service Account**

```sh
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

3. **Install Cert-Manager**

```sh
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml
```

4. **Install AWS Load Balancer Controller using Helm**

```sh
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set region=us-west-2 \
  --set vpcId=<VPC_ID> \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Step 3: Install NGINX Ingress Controller

```sh
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### Step 4: Deploy the Sample Application

1. **Apply Deployment and Service Manifests**

```sh
kubectl apply -f manifests/sample-app-deployment.yaml
kubectl apply -f manifests/sample-app-service.yaml
```

2. **Apply Ingress Resources**

For ALB:

```sh
kubectl apply -f manifests/alb-ingress.yaml
```

For NGINX:

```sh
kubectl apply -f manifests/nginx-ingress.yaml
```

### Step 5: Install ArgoCD

1. **Install ArgoCD**

```sh
kubectl apply -f argocd/argocd-install.yaml
```

2. **Access ArgoCD UI**

```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

3. **Login to ArgoCD**

Retrieve the initial password:

```sh
kubectl get pods -n argocd
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

4. **Open your browser and navigate to `https://localhost:8080`**, login with `admin` and the password retrieved above.

### Step 6: Connect ArgoCD to Your Repository

1. **Create a new application in ArgoCD** pointing to your repository containing the Kubernetes manifests for the sample application and ingress resources.

2. **Sync the application** to deploy the manifests.

## Conclusion

You have now set up an EKS cluster with both ALB and NGINX ingress controllers, deployed a sample application, and connected it to ArgoCD for continuous deployment. You can access your application using the default AWS-provided DNS names for the load balancers created by the ingress controllers.
