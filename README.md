# Setting up of JupyterHub with KeyCloak Authentication on EKS

### Pre Req

- [AWS Command Line Interface (AWS CLI) version 2](https://aws.amazon.com/cloudwatch/) is a utility for controlling AWS resources
- [eksctl](https://eksctl.io/) is a utility for managing Amazon EKS clusters
- [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html) is a utility for managing Kubernetes
- [helm](https://www.eksworkshop.com/beginner/060_helm/helm_intro/install/index.html) is a tool for automating Kubernetes deployments


### Installation steps


#### EKS Cluster with EBS
```bash
eksctl create cluster -f ./eksctl.yaml
```

#### Install AWS ALB LB Controller

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

eksctl create iamserviceaccount \
  --cluster=jupyterhub-demo \
  --region=us-east-1 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

helm repo add eks https://aws.github.io/eks-charts

helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=jupyterhub-demo \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller 

kubectl get deployment -n kube-system aws-load-balancer-controller
```

### Install KeyCloak

```
kubectl apply -f ./keycloak.yaml
```

#### Install JupyterHub

```bash
helm repo add jupyterhub https://hub.jupyter.org/helm-chart/
helm upgrade --install jupyterhub jupyterhub/jupyterhub --namespace default --version 3.2.1 --values hub.yaml --debug
```

### Setup Ingress

```bash
# Make sure to create a Route53 Domain, Certificate for your Domain
# Make sure to update the HOST_DOMAIN and Certificate ARN of your domain in the `ingress-resource-alb.yaml` file
kubectl apply -f ./ingress-resource-alb.yaml
```
