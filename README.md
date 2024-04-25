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

1. Create a certificate:

```sh
CN="platform.mlopsbook.online"
O="ML"

openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout auth-tls.key -out auth-tls.crt -subj "/CN=${HOST}/O=${O}"

kubectl create secret -n keycloak tls auth-tls-secret --key auth-tls.key --cert auth-tls.crt

```

Deploy Postgres:
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install -n keycloak keycloak-db bitnami/postgresql-ha --set postgresql.replicaCount=1
```

Setup Keycloak:
```sh
KC_HOSTNAME=platform.mlopsbook.online
```

Create an ACM certificate and store the certificate ARN:
```sh
KC_CERT_ARN=arn:aws:acm:us-west-2:XXXXXXXXXX:certificate/0636e530-ccd8-4d63-a196-9c681566ba47
```

Deploy KeyCloak
```sh
cat <<EOF > keycloak.yaml
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  namespace: keycloak
  labels:
    app: keycloak
spec:
  ports:
    - name: https
      port: 443
      targetPort: 8443
  selector:
    app: keycloak
  type: ClusterIP
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: keycloak
  labels:
    app: keycloak
spec:
  replicas: 2
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
        - name: keycloak
          image: quay.io/keycloak/keycloak:20.0.2
          args: ["start", "--cache-stack=kubernetes"]
          volumeMounts:
          - name: certs
            mountPath: "/etc/certs"
            readOnly: true
          env:
            - name: KEYCLOAK_ADMIN
              value: "mladmin"
            - name: KEYCLOAK_ADMIN_PASSWORD
              value: "mladmin"
            - name: KC_HTTPS_CERTIFICATE_FILE
              value: "/etc/certs/tls.crt"
            - name: KC_HTTPS_CERTIFICATE_KEY_FILE
              value: "/etc/certs/tls.key"
            - name: KC_HEALTH_ENABLED
              value: "true"
            - name: KC_METRICS_ENABLED
              value: "true"
            - name: KC_HOSTNAME
              value: $KC_HOSTNAME
            - name: KC_PROXY
              value: "edge"
            - name: KC_DB
              value: postgres
            - name: KC_DB_URL
              value: "jdbc:postgresql://keycloak-db-postgresql-ha-pgpool/postgres"
            - name: KC_DB_USERNAME
              value: "postgres"
            - name: KC_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: keycloak-db-postgresql-ha-postgresql
                  key: password
            - name: jgroups.dns.query
              value: keycloak
          ports:
            - name: jgroups
              containerPort: 7600
            - name: https
              containerPort: 8443
          readinessProbe:
            httpGet:
              scheme: HTTPS
              path: /health/ready
              port: 8443
            initialDelaySeconds: 60
            periodSeconds: 1
      volumes:
      - name: certs
        secret:
          secretName: auth-tls-secret
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-ingress
  namespace: keycloak
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/group.name: ml-on-k8s
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/backend-protocol: "HTTPS"
    alb.ingress.kubernetes.io/certificate-arn: $KC_CERT_ARN
spec:
  ingressClassName: alb
  rules:
    - host: $KC_HOSTNAME
      http:
        paths:
          - pathType: Prefix
            path: "/"
            backend:
              service:
                name: keycloak
                port:
                  number: 443
EOF
kubectl apply -f keycloak.yaml
```

#### Install JupyterHub

1. JupyterHub requires a database. Postgres and MySQL are supported. 

2. JupyterHub requires EBS. Create a gp3 storage class:

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
allowVolumeExpansion: true
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
```

3. Helm Install

```
jh_version=3.3.5
helm repo add jupyterhub https://hub.jupyter.org/helm-chart/
helm repo update

helm upgrade --cleanup-on-fail \
  --install jhub jupyterhub/jupyterhub \
  --namespace jupyter \
  --create-namespace \
  --version=$jh_version \
  --values hub.yaml
```

### Admin Portal

JupyterHub's admin portal is located at `/hub/admin#/`.

---

Use cases for Jupyter are:
1.	⁠executable documentation (show how an algorithm works piece-by-piece, with markdown sections in between code sections; also works for lab reports with post-processing)
2.	⁠rapid prototyping (quickly iterate on pieces of an algorithm without having to re-execute the rest of the code - especially important if some of the steps are slow)
3.	⁠tutorials (allows the student to work through code piece-by-piece while being able to see text, image, or plot results in between sections)

It isn't the right tool for everything, but it does come in handy for some tasks.  It's a partner to an IDE, not a replacement.


## Config using Keycloak

Create a wildcard cert in ACM. 
Create an ingress for [[Keycloak]] 
Setup DNS to point to Keycloak ingress. 

## Resources 

- [Notebooks at scale at Adobe](https://blog.developer.adobe.com/reimagining-jupyter-notebooks-for-enterprise-scale-8bc6340d504a)
- [The Littlest JupyterHub — The Littlest JupyterHub  documentation](https://tljh.jupyter.org/en/latest/index.html)
- [On-demand Notebooks with JupyterHub, Jupyter Enterprise Gateway and Kubernetes](https://blog.jupyter.org/on-demand-notebooks-with-jupyterhub-jupyter-enterprise-gateway-and-kubernetes-e8e423695cbf)
- https://jupytext.readthedocs.io/en/latest/


