# Amazon EKS Deployment Walkthrough

Provisioning and running Kubernetes workloads on **Amazon Elastic Kubernetes Service (EKS)**. This walkthrough covers the complete deployment lifecycle—from installing required CLI tools and preparing Kubernetes manifests to provisioning an EKS cluster, configuring IAM permissions, integrating persistent storage, deploying applications through an Application Load Balancer (ALB), managing secrets, validating workloads, and fronting the application with Amazon CloudFront.

---

## Access the walkthrough

[![Amazon EKS walkthrough](https://img.youtube.com/vi/yKO90s1PIb4/0.jpg)](https://www.youtube.com/embed/yKO90s1PIb4?si=BSRCthI2bnWHXxW_)

[Watch the video on YouTube](https://www.youtube.com/embed/yKO90s1PIb4?si=BSRCthI2bnWHXxW_)

---

## 🛠 Deployment Strategy

<div align="center">
<img src="images/eks/eks.png" width="1000"/>
</div>

---

# Step 1: Base Setup

Before provisioning an Amazon EKS cluster, the local development environment was prepared by installing and validating all required command-line utilities.

Required tools:

- AWS CLI
- kubectl
- eksctl
- Kustomize

Version verification:

```bash
eksctl version
aws --version
kubectl version --client
```

Initially, `eksctl` was unavailable because it was not included in the Windows PATH.

The executable was downloaded from the official GitHub releases page and copied to:

```text
C:\Tools\eksctl
```

The PATH environment variable was updated:

```powershell
setx PATH "%PATH%;C:\Tools\eksctl"
```

Verification confirmed the installation:

```text
0.229.0
```

At this stage the workstation was fully configured to provision and manage Amazon EKS clusters.

<div align="center">

<img src="images/eks/S1 EKS/eks1.1.png" width="250"/>

<img src="images/eks/S1 EKS/eks1.2.png" width="250"/>

<img src="images/eks/S1 EKS/eks1.3.png" width="250"/>

<img src="images/eks/S1 EKS/eks1.4.png" width="250"/>

</div>

---

# Step 2: Preparing Kubernetes Manifests

Prior to cluster creation, every Kubernetes manifest required by the application was organized inside the **EKS/** directory.

The deployment package included:

- `configmap.yaml`
- `ingress.yaml`
- `postgres.yaml`
- `resume-matcher-ghcr.yaml`
- `service.yaml`
- `storageclass.yaml`

Each manifest defined one layer of the application stack including configuration, networking, storage, application deployment, and ingress routing.

Deployment of all resources can be performed using a single command:

```bash
kubectl apply -f ./EKS
```

Verification commands:

```bash
kubectl get pods

kubectl get svc

kubectl get ingress
```

This ensured every Kubernetes resource was correctly defined before provisioning the EKS infrastructure.

<div align="center">

<img src="images/eks/S2 EKS/eks2.1.png" width="250"/>

<img src="images/eks/S2 EKS/eks2.2.png" width="250"/>

<img src="images/eks/S2 EKS/eks2.3.png" width="250"/>

<img src="images/eks/S2 EKS/eks2.4.png" width="250"/>

<img src="images/eks/S2 EKS/eks2.5.png" width="250"/>

<img src="images/eks/S2 EKS/eks2.6.png" width="250"/>

</div>

---

# Step 3: Cluster Creation & Kubeconfig

After preparing the deployment manifests, an Amazon EKS cluster was provisioned using **eksctl**.

Identity verification:

```bash
aws sts get-caller-identity
```

Cluster creation:

```bash
eksctl create cluster \
  --name resume-cluster \
  --region ap-south-1 \
  --nodegroup-name resume-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 4 \
  --managed
```

Once CloudFormation completed successfully, kubeconfig was updated:

```bash
aws eks \
  --region ap-south-1 \
  update-kubeconfig \
  --name resume-cluster
```

Verification confirmed:

- Active EKS cluster
- Kubernetes v1.34
- kubectl connectivity
- Worker nodes joined successfully

<div align="center">

<img src="images/eks/S3 EKS/eks3.1.png" width="250"/>

<img src="images/eks/S3 EKS/eks3.2.png" width="250"/>

<img src="images/eks/S3 EKS/eks3.3.png" width="250"/>


# Step 4: Storage Integration & IAM Permissions

To enable persistent storage for stateful workloads, the required IAM permissions were configured before deploying the application. The **Amazon EBS CSI Driver** IAM policy was created and attached to the EKS worker node role, allowing Kubernetes to dynamically provision Amazon EBS volumes for PersistentVolumeClaims (PVCs).

First, the IAM policy definition was downloaded from the official AWS EBS CSI Driver repository and registered in the AWS account.

```bash
curl -o ebs-csi-policy.json \
https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json

aws iam create-policy \
  --policy-name AmazonEBSCSIDriverPolicy \
  --policy-document file://ebs-csi-policy.json
```

The required IAM policies were then attached to the EKS managed node group role:

```bash
aws iam attach-role-policy \
  --role-name eksctl-resume-cluster-nodegroup-NodeInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMReadOnlyAccess

aws iam attach-role-policy \
  --role-name eksctl-resume-cluster-nodegroup-NodeInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/ElasticLoadBalancingFullAccess

aws iam attach-role-policy \
  --role-name eksctl-resume-cluster-nodegroup-NodeInstanceRole \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AmazonEBSCSIDriverPolicy
```

Verification confirmed that all required policies were successfully attached to the worker node IAM role.

**Attached Policies**

- AmazonEBSCSIDriverPolicy
- ElasticLoadBalancingFullAccess
- AmazonSSMReadOnlyAccess

This completed the storage permissions required for dynamic Amazon EBS volume provisioning and enabled worker nodes to interact securely with AWS services during deployment.

<div align="center">

<img src="images/eks/S4 EKS/eks4.1.png" width="250"/>

<img src="images/eks/S4 EKS/eks4.2.png" width="250"/>

<img src="images/eks/S4 EKS/eks4.3.png" width="250"/>

</div>

---

# Step 5: AWS Load Balancer Controller

After configuring IAM permissions, the cluster was prepared for application exposure by installing the required Amazon EKS add-ons.

The **Amazon EBS CSI Driver** was installed to provide dynamic persistent storage, followed by deployment of the **AWS Load Balancer Controller**, which automatically provisions an **Application Load Balancer (ALB)** from Kubernetes Ingress resources.

Install the Amazon EBS CSI Driver:

```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster resume-cluster \
  --region ap-south-1
```

Add the AWS Helm repository and install the AWS Load Balancer Controller:

```bash
helm repo add eks https://aws.github.io/eks-charts

helm repo update

helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=resume-cluster \
  --set serviceAccount.create=false \
  --set region=ap-south-1 \
  --set vpcId=vpc-07907491331a31045
```

Verify the installation:

```bash
kubectl get pods -n kube-system
```

Verification confirmed:

- Amazon EBS CSI Driver running
- AWS Load Balancer Controller deployed successfully
- Controller Manager healthy
- Cluster ready to provision Application Load Balancers automatically from Kubernetes Ingress manifests

<div align="center">

<img src="images/eks/S5 EKS/eks5.1.png" width="250"/>

<img src="images/eks/S5 EKS/eks5.2.png" width="250"/>

</div>

---

# Step 6: Private Registry Authentication & Cluster Verification

Before deploying workloads, Kubernetes was configured to authenticate with the private **GitHub Container Registry (GHCR)**. A Docker registry secret was created to allow worker nodes to securely pull private container images during deployment.

Authenticate with GitHub Container Registry:

```bash
docker login ghcr.io \
-u <github-username> \
-p <personal-access-token>
```

Create the Kubernetes registry secret:

```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<github-username> \
  --docker-password=<personal-access-token>
```

Verify the secret:

```bash
kubectl get secrets
```

Perform cluster health verification:

```bash
kubectl get nodes

kubectl get pods -A
```

Validation confirmed:

- `ghcr-secret` created successfully
- Registry credentials stored as `kubernetes.io/dockerconfigjson`
- Worker nodes in the **Ready** state
- Core Kubernetes components healthy
- `aws-node`
- `coredns`
- `kube-proxy`
- `metrics-server`

With registry authentication complete and all cluster components verified, the environment was fully prepared for application deployment.

<div align="center">

<img src="images/eks/S6 EKS/eks6.1.png" width="250"/>

<img src="images/eks/S6 EKS/eks6.2.png" width="250"/>

<img src="images/eks/S6 EKS/eks6.3.png" width="250"/>

</div>

---

# Step 7: Storage Verification

With the required IAM permissions and the Amazon EBS CSI Driver configured, persistent storage was validated to ensure stateful workloads could dynamically provision and attach Amazon EBS volumes.

The custom **gp3 StorageClass** was applied to the cluster:

```bash
kubectl apply -f storageclass.yaml
```

Verify available StorageClasses:

```bash
kubectl get storageclass
```

Verify PersistentVolumeClaims:

```bash
kubectl get pvc
```

The output confirmed:

- Default StorageClass available
- Custom **gp3** StorageClass provisioned through `ebs.csi.aws.com`
- Volume expansion enabled
- PostgreSQL PersistentVolumeClaim successfully bound
- 20Gi Amazon EBS volume dynamically provisioned

This verification confirmed that Kubernetes could automatically create and manage persistent storage for stateful applications running on Amazon EKS.

<div align="center">

<img src="images/eks/S7 EKS/eks7.png" width="600"/>

</div>

---

# Step 8: Secrets & Network Security

Application credentials and sensitive configuration were securely stored inside Kubernetes Secrets instead of being hardcoded into deployment manifests.

The following secrets were created:

- PostgreSQL credentials
- Google API Key
- Google Custom Search Engine ID

Create the PostgreSQL secret:

```bash
kubectl create secret generic db-secret \
  --from-literal=POSTGRES_USER=***** \
  --from-literal=POSTGRES_PASSWORD=***** \
  --from-literal=POSTGRES_DB=*****
```

Create the Google API Key secret:

```bash
kubectl create secret generic google-api-key \
  --from-literal=API_KEY=*****
```

Create the Google Search Engine ID secret:

```bash
kubectl create secret generic google-cx-id \
  --from-literal=CX_ID=*****
```

Deploy all Kubernetes resources:

```bash
kubectl apply -f .
```

Verification:

```bash
kubectl get pods

kubectl get svc

kubectl get ingress
```

The deployment confirmed:

- Application Pods running successfully
- PostgreSQL StatefulSet healthy
- Services created
- Application Load Balancer Ingress provisioned
- Secrets successfully mounted inside workloads

To allow inbound traffic, the associated AWS Security Group was updated with the required inbound rules for:

- HTTP / HTTPS
- ALB Target Group communication
- Metrics endpoints
- Internal cluster communication

This ensured secure connectivity while exposing only the required services.

<div align="center">

<img src="images/eks/S8 EKS/eks8.1.png" width="250"/>

<img src="images/eks/S8 EKS/eks8.2.png" width="250"/>

</div>

---

# Step 9: CloudFront & Live Validation

With the infrastructure fully deployed, the application was fronted by an **Amazon CloudFront** distribution to provide HTTPS, edge caching, and a globally accessible endpoint.

The CloudFront distribution was configured to forward requests to the Application Load Balancer while providing improved latency and secure TLS termination.

Example CloudFront endpoint:

```text
https://dhj1d6dq1wiwz.cloudfront.net
```

End-to-end application validation:

- Opened the CloudFront URL
- Uploaded a resume PDF
- Triggered Resume Matcher analysis
- Retrieved matching job listings
- Verified keyword highlighting
- Confirmed PostgreSQL connectivity
- Verified successful API responses

Application logs:

```bash
kubectl logs -f resume-matcher-ghcr-xxxxx

kubectl logs -f postgres-0
```

Cluster verification:

```bash
kubectl get pods

kubectl get svc

kubectl get ingress

kubectl get pvc
```

Validation confirmed:

- CloudFront distribution serving application traffic
- ALB routing requests correctly
- Kubernetes Ingress functioning
- Persistent PostgreSQL storage operational
- Secrets injected successfully
- Resume Matcher application responding correctly
- End-to-end deployment completed successfully

<div align="center">

<img src="images/eks/S9 EKS/eks9.1.png" width="250"/>

<img src="images/eks/S9 EKS/eks9.2.png" width="250"/>

<img src="images/eks/S9 EKS/eks9.3.png" width="250"/>

<img src="images/eks/S9 EKS/eks9.4.png" width="250"/>

</div>

---

## 📝 Notes

- **Amazon EKS** simplifies Kubernetes cluster provisioning and lifecycle management through managed control planes.
- **eksctl** automates cluster creation, managed node groups, and networking configuration.
- **Amazon EBS CSI Driver** enables dynamic provisioning of persistent storage using Amazon EBS volumes.
- **AWS Load Balancer Controller** automatically provisions Application Load Balancers from Kubernetes Ingress resources.
- **GitHub Container Registry (GHCR)** securely hosts private container images used during deployment.
- **Kubernetes Secrets** protect sensitive application configuration such as database credentials and API keys.
- **Amazon CloudFront** provides HTTPS, edge caching, and global content delivery for the deployed application.
- **Outcome:** Successfully deployed a production-style Resume Matcher application on Amazon EKS with persistent storage, secure image pulls, automated ingress provisioning, centralized secrets management, and end-to-end validation through CloudFront.

---

</div>

---
