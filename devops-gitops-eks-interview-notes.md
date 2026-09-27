# DevOps Interview Notes — GitOps with ArgoCD on AWS EKS

Repository: https://github.com/DheerajSam/devops-gitops-eks

This document is for **revision and interview preparation**. It contains:
1. A simple explanation of the project
2. The architecture and request/data flow
3. What each technology does
4. How the infrastructure and GitOps workflow were demonstrated
5. A practical runbook to recreate and show the project
6. Interview questions and realistic answers
7. Troubleshooting scenarios
8. A short revision sheet

---

# 1. Project in One Minute

### Project name

**GitOps with ArgoCD on AWS EKS**

### What did I build?

I built a hands-on GitOps environment where:

- Terraform provisions the AWS infrastructure.
- AWS EKS provides the managed Kubernetes cluster.
- Kubernetes manifests are stored in Git.
- ArgoCD watches the Git repository.
- ArgoCD synchronizes the desired state into EKS.
- ArgoCD can automatically correct manual changes made directly in the cluster.

The project uses a Kubernetes Deployment with 2 replicas and a LoadBalancer Service. The Terraform configuration provisions the AWS networking, EKS cluster, managed node group, and IAM roles.

The GitOps behavior was demonstrated by manually scaling the application from 2 replicas to 5. ArgoCD detected the drift and reconciled the deployment back to the Git-defined state of 2 replicas.

---

# 2. The Simple Story

The easiest way to explain the project in an interview:

> "I wanted to understand how GitOps works with a real managed Kubernetes environment rather than only using Minikube. So I used Terraform to provision an AWS EKS cluster, including the VPC, subnets, NAT Gateway, IAM roles and managed node group. I kept my Kubernetes Deployment and Service manifests in Git and used ArgoCD as the GitOps controller. ArgoCD continuously watched the repository and synchronized the desired state into EKS. I also enabled self-healing and demonstrated it by manually scaling the deployment from 2 replicas to 5. ArgoCD detected that drift and brought it back to the 2 replicas defined in Git."

If asked why the environment is not currently running:

> "It was a hands-on AWS project, so I tear the environment down after testing to avoid unnecessary AWS charges. The Terraform configuration and Kubernetes manifests are kept in Git so I can recreate the environment when I need to demonstrate it."

---

# 3. Architecture

```text
                  GitHub Repository
                         |
                         | k8s/ = desired state
                         v
                     ArgoCD
                         |
                         | sync / reconcile
                         v
                  AWS EKS Cluster
                         |
                         v
               Kubernetes Deployment
                    2 replicas
                         |
                         v
               Kubernetes Service
                  type: LoadBalancer
                         |
                         v
                AWS Network Load Balancer
```

Infrastructure side:

```text
Terraform
   |
   +-- VPC
   |    +-- 2 Public Subnets
   |    +-- 2 Private Subnets
   |    +-- Internet Gateway
   |    +-- NAT Gateway
   |
   +-- EKS Cluster
   |
   +-- Managed Node Group
   |      +-- t3.medium
   |      +-- t3.medium
   |
   +-- IAM Roles
```

The repository README documents 22 AWS-managed resources created through Terraform during the hands-on implementation.

---

# 4. Why These Technologies?

## Terraform

Terraform is used for **Infrastructure as Code**.

Instead of manually creating the VPC, EKS cluster, node group and IAM configuration in the AWS console, the infrastructure is defined in `.tf` files.

Main benefit:

```text
Terraform configuration
        |
        v
terraform apply
        |
        v
AWS infrastructure
```

The environment can later be removed using:

```bash
terraform destroy
```

---

## AWS EKS

EKS is the managed Kubernetes service used for the project.

Instead of running Kubernetes locally with Minikube, this project uses a real AWS-managed Kubernetes control plane and AWS-managed worker nodes.

The project used:

- EKS Kubernetes v1.30
- Managed node group
- 2 x `t3.medium` nodes
- AWS VPC networking

---

## Kubernetes

Kubernetes manages the application workload.

The project uses:

### Deployment

Defines:

- Application
- Replica count
- RollingUpdate strategy
- Health probes

The desired replica count is 2.

### Service

The Service exposes the application.

It uses:

```yaml
type: LoadBalancer
```

This integrates with AWS to provision a load balancer.

---

## ArgoCD

ArgoCD is the GitOps controller.

Its job is to compare:

```text
Git desired state
        vs
Kubernetes live state
```

If they differ, ArgoCD can synchronize the cluster back to the desired state.

Important features demonstrated:

- Automated synchronization
- Self-healing
- Resource pruning

---

# 5. Desired State vs Live State

This is one of the most important concepts to understand.

Suppose Git contains:

```yaml
replicas: 2
```

That is the **desired state**.

Kubernetes is currently running:

```text
2 pods
```

So:

```text
Desired = 2
Live = 2
Status = Synced
```

Now someone manually runs:

```bash
kubectl scale deployment gitops-app --replicas=5
```

The cluster becomes:

```text
Desired = 2
Live = 5
```

There is now **drift**.

ArgoCD detects the difference and reconciles the cluster back to:

```text
Desired = 2
Live = 2
```

That is the self-healing demonstration in this project.

---

# 6. GitOps vs Traditional Deployment

Traditional imperative approach:

```text
Developer
   |
   v
kubectl apply
   |
   v
Kubernetes
```

The operator tells Kubernetes what to do.

GitOps approach:

```text
Developer
   |
   v
Git repository
   |
   v
ArgoCD
   |
   v
Kubernetes
```

Git becomes the source of truth.

In this project, the `k8s/` directory contains the desired Kubernetes configuration.

---

# 7. What Happens During a Normal GitOps Change?

Suppose I change:

```yaml
replicas: 2
```

to:

```yaml
replicas: 3
```

and commit it to Git.

The flow is:

```text
1. Change Kubernetes manifest
        |
2. Git commit / push
        |
3. ArgoCD detects repository change
        |
4. ArgoCD compares desired and live state
        |
5. ArgoCD synchronizes Kubernetes
        |
6. Deployment changes from 2 -> 3 replicas
```

The important point is that I do not need to manually run:

```bash
kubectl apply -f deployment.yaml
```

for the GitOps workflow.

---

# 8. Project Components

## `terraform/`

Contains the infrastructure configuration.

The repository documents:

- `main.tf`
- `variables.tf`
- `outputs.tf`
- `.gitignore`

The Terraform configuration handles the AWS infrastructure required for EKS.

## `k8s/`

Contains the Kubernetes desired state.

The repository documents:

```text
deployment.yaml
service.yaml
```

The Deployment handles the application workload.

The Service exposes it through an AWS LoadBalancer.

## ArgoCD

ArgoCD is installed into the EKS cluster and configured to watch the repository's `k8s/` directory.

---

# 9. Kubernetes Deployment Details

The project uses:

```text
Replicas: 2
Strategy: RollingUpdate
maxSurge: 1
maxUnavailable: 0
```

The Deployment also uses:

- Liveness probe
- Readiness probe
- `/health` endpoint

### Liveness probe

Answers:

> "Is the container/application still alive?"

If the application becomes unhealthy, Kubernetes can restart the container.

### Readiness probe

Answers:

> "Is this pod ready to receive traffic?"

A pod that is not ready should not receive normal application traffic.

---

# 10. LoadBalancer Flow

The Kubernetes Service uses:

```yaml
type: LoadBalancer
```

The simplified flow is:

```text
Internet
   |
   v
AWS Network Load Balancer
   |
   v
Kubernetes Service
   |
   v
Application Pods
```

The project demonstrated AWS Load Balancer provisioning through the Kubernetes Service rather than manually creating a load balancer separately.

---

# 11. Terraform Infrastructure Flow

When I run:

```bash
terraform init
```

Terraform initializes the working directory and downloads the required providers/modules required by the configuration.

Then:

```bash
terraform apply
```

Terraform calculates the required changes and creates the AWS infrastructure.

The project provisions:

```text
VPC
 ├── Public Subnet
 ├── Public Subnet
 ├── Private Subnet
 ├── Private Subnet
 ├── Internet Gateway
 └── NAT Gateway

EKS
 ├── Control Plane
 └── Managed Node Group
       ├── t3.medium
       └── t3.medium

IAM
 ├── EKS role
 └── Worker node role
```

---

# 12. Full Hands-On Runbook

Use this section when you actually want to recreate the project.

## Prerequisites

You need:

- AWS account
- AWS CLI
- Terraform
- kubectl
- Git
- AWS credentials configured locally

Verify:

```bash
aws --version
terraform --version
kubectl version --client
git --version
```

Verify AWS access:

```bash
aws sts get-caller-identity
```

---

## Step 1 — Clone the repository

```bash
git clone https://github.com/DheerajSam/devops-gitops-eks.git
cd devops-gitops-eks
```

---

## Step 2 — Provision AWS infrastructure

```bash
cd terraform
terraform init
terraform apply
```

Review the Terraform plan and confirm the resources before applying.

---

## Step 3 — Configure kubectl

The project uses the AWS `ap-south-1` region.

```bash
aws eks update-kubeconfig   --region ap-south-1   --name devops-gitops-eks
```

Verify:

```bash
kubectl get nodes
```

Expected result:

```text
2 worker nodes
STATUS: Ready
```

---

# 13. Install ArgoCD

Create the namespace:

```bash
kubectl create namespace argocd
```

Install ArgoCD:

```bash
kubectl apply -n argocd   -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml   --server-side
```

Check the pods:

```bash
kubectl get pods -n argocd
```

Wait until the ArgoCD components are running.

---

# 14. Get ArgoCD Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret   -o jsonpath="{.data.password}" | base64 -d
```

Save the password temporarily for the demo.

---

# 15. Access ArgoCD

Run:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

Log in with:

```text
Username: admin
Password: <password retrieved above>
```

---

# 16. Create the ArgoCD Application

Create an ArgoCD Application that points to:

```text
Repository:
https://github.com/DheerajSam/devops-gitops-eks.git

Path:
k8s/
```

Enable:

- Automated Sync
- Self Heal
- Prune

The important relationship is:

```text
GitHub repository
      |
      | k8s/
      v
    ArgoCD
      |
      v
     EKS
```

---

# 17. Verify the Application

Check Kubernetes:

```bash
kubectl get deployments
kubectl get pods
kubectl get services
```

Check the Service:

```bash
kubectl get service
```

Because the Service is a LoadBalancer, AWS should provision the corresponding load balancer.

Check ArgoCD:

```text
Application = Healthy
Sync Status = Synced
```

---

# 18. Demonstrate Self-Healing

First check:

```bash
kubectl get deployment
```

The desired replica count should be 2.

Now manually introduce drift:

```bash
kubectl scale deployment gitops-app --replicas=5
```

Check:

```bash
kubectl get pods
kubectl get deployment
```

There will temporarily be 5 replicas.

ArgoCD detects that:

```text
Git desired state = 2
Cluster live state = 5
```

and reconciles the deployment.

After synchronization:

```text
Git desired state = 2
Cluster live state = 2
```

The original project demonstration documented this reconciliation occurring within roughly 20 seconds.

---

# 19. Demonstrate a GitOps Change

For a cleaner demonstration, make a legitimate change in Git.

Example:

```yaml
replicas: 3
```

Commit and push:

```bash
git add k8s/deployment.yaml
git commit -m "Scale application to three replicas"
git push
```

Then observe ArgoCD.

Expected flow:

```text
Git change
   ↓
ArgoCD detects change
   ↓
Application becomes OutOfSync
   ↓
ArgoCD synchronizes
   ↓
Deployment changes to 3 replicas
```

This is a good way to explain the difference between:

- **GitOps change** — change Git and let ArgoCD deploy it
- **Manual drift** — change the cluster and let ArgoCD correct it

---

# 20. Cleanup

Do not leave the EKS environment running unnecessarily.

First remove ArgoCD:

```bash
kubectl delete -n argocd   -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Then:

```bash
kubectl delete namespace argocd
```

Then destroy the AWS infrastructure:

```bash
cd terraform
terraform destroy
```

Verify:

```bash
aws eks list-clusters --region ap-south-1
```

---

# 21. Important Terraform State Point

The repository intentionally excludes Terraform state files from Git.

Do not commit:

```text
terraform.tfstate
terraform.tfstate.backup
.terraform/
```

For a team environment, the natural next step would be remote Terraform state with locking, such as an S3-based state backend and an appropriate locking mechanism.

For this personal hands-on project, the important point is:

> Terraform state contains infrastructure information and should not be casually committed to a public repository.

---

# 22. Interview Questions and Answers

## Q1. Explain your project.

**Answer:**

> "I built a hands-on GitOps project using Terraform, AWS EKS, Kubernetes and ArgoCD. Terraform provisions the AWS infrastructure including the VPC, subnets, NAT Gateway, EKS cluster, managed node group and IAM roles. The Kubernetes Deployment and Service are maintained in Git. ArgoCD watches the repository and synchronizes the desired state into EKS. I enabled automated sync, self-healing and pruning. I demonstrated self-healing by manually scaling the deployment from 2 replicas to 5, and ArgoCD detected the drift and brought it back to the 2 replicas defined in Git."

---

## Q2. Why did you choose EKS instead of Minikube?

**Answer:**

> "I had already worked with local Kubernetes environments, but I wanted to understand how Kubernetes works in a real cloud environment. EKS gave me experience with AWS networking, managed worker nodes, IAM and AWS LoadBalancer integration. It also made the GitOps demonstration closer to a real cloud deployment."

---

## Q3. What did Terraform provision?

**Answer:**

> "The project provisions the VPC, two public and two private subnets across two Availability Zones, Internet Gateway, NAT Gateway, EKS cluster, managed node group and IAM roles. The repository documents 22 AWS-managed resources created through Terraform."

---

## Q4. Why do you need public and private subnets?

**Answer:**

> "The architecture separates resources by network exposure. Public subnets can host resources that need public connectivity, while private subnets provide a more restricted network location for internal resources. The NAT Gateway allows resources in private subnets to initiate outbound internet connectivity without requiring public IPs."

---

## Q5. What is the role of the NAT Gateway?

**Answer:**

> "The NAT Gateway provides outbound internet connectivity for resources in private subnets. The private resources can reach external services without being directly exposed through public IP addresses."

---

## Q6. What is the difference between EKS control plane and worker nodes?

**Answer:**

> "The EKS control plane manages the Kubernetes API and cluster control components. The worker nodes provide the compute capacity where application pods actually run. In this project I used an EKS managed node group with two t3.medium nodes."

---

## Q7. What is GitOps?

**Answer:**

> "GitOps means Git is used as the source of truth for the desired state of the environment. Instead of manually changing the cluster, I make the desired configuration change in Git and a controller such as ArgoCD reconciles the cluster to that state."

---

## Q8. What is ArgoCD?

**Answer:**

> "ArgoCD is a GitOps continuous delivery tool for Kubernetes. It watches the Git repository, compares the desired state in Git with the live cluster state and synchronizes the cluster when required."

---

## Q9. What is self-healing in ArgoCD?

**Answer:**

> "Self-healing means ArgoCD can detect manual drift in the Kubernetes cluster and automatically reconcile it back to the desired state stored in Git."

---

## Q10. Explain your self-healing demonstration.

**Answer:**

> "The Git-defined Deployment had 2 replicas. I manually ran `kubectl scale deployment gitops-app --replicas=5`. That created a difference between the Git desired state and the live cluster. ArgoCD detected the drift and reconciled the Deployment back to 2 replicas. The demonstration showed that Git remained the source of truth."

---

## Q11. What is pruning in ArgoCD?

**Answer:**

> "Pruning removes resources from the cluster when those resources have been removed from the desired state in Git, assuming pruning is enabled for the ArgoCD Application."

---

## Q12. What is the difference between sync and self-heal?

**Answer:**

> "Sync is the process of bringing the cluster in line with the desired state. Self-heal specifically handles drift that happens in the live cluster, such as someone manually changing a resource with kubectl."

---

## Q13. Why use a Kubernetes Deployment?

**Answer:**

> "A Deployment manages the desired number of pod replicas and provides controlled updates through a deployment strategy. It also makes it easy to scale the application and maintain the desired state."

---

## Q14. Why use a Service?

**Answer:**

> "Pods are ephemeral and their IP addresses can change. A Kubernetes Service provides a stable way to expose the application. In this project I used a LoadBalancer Service so AWS could provision a Network Load Balancer."

---

## Q15. Why use `type: LoadBalancer`?

**Answer:**

> "Because I wanted the application to be reachable through an AWS load balancer rather than exposing the application through a NodePort or manually configuring a separate load balancer."

---

## Q16. What is the purpose of the readiness probe?

**Answer:**

> "The readiness probe tells Kubernetes whether the pod is ready to receive traffic. If the application is not ready, Kubernetes can keep that pod out of normal service traffic."

---

## Q17. What is the purpose of the liveness probe?

**Answer:**

> "The liveness probe checks whether the application is still healthy enough to remain running. If the container repeatedly fails the liveness check, Kubernetes can restart it."

---

## Q18. Why did you use `maxUnavailable: 0`?

**Answer:**

> "It means the rolling update should not intentionally reduce the number of available replicas below the desired level during the update. Combined with the rolling update strategy, it helps maintain availability during a deployment."

---

## Q19. What happens if someone manually changes the Kubernetes deployment?

**Answer:**

> "If the change causes drift from the Git-defined desired state and self-healing is enabled, ArgoCD detects the difference and reconciles the resource back to the Git state."

---

## Q20. What happens if someone changes Git?

**Answer:**

> "ArgoCD detects the repository change, compares the new desired state with the live cluster and, with automated sync enabled, applies the required Kubernetes changes."

---

## Q21. What is the difference between Terraform and ArgoCD in this project?

**Answer:**

> "Terraform manages the infrastructure layer, such as the AWS VPC and EKS infrastructure. ArgoCD manages the Kubernetes application state inside the cluster. So Terraform handles infrastructure provisioning, while ArgoCD handles GitOps-based application reconciliation."

A simple way to remember:

```text
Terraform -> AWS infrastructure
ArgoCD    -> Kubernetes application state
```

---

## Q22. Why not use Terraform to deploy the Kubernetes application too?

**Answer:**

> "Terraform can manage Kubernetes resources, but I wanted to demonstrate a clear separation between infrastructure provisioning and application delivery. Terraform creates the EKS environment, while ArgoCD continuously manages the application state from Git."

---

## Q23. Is this CI/CD?

**Answer:**

> "It is a GitOps continuous delivery workflow. The important difference is that the deployment side is pull-based: ArgoCD observes Git and reconciles the cluster. A traditional pipeline often pushes deployment commands into the cluster."

---

## Q24. What is pull-based deployment?

**Answer:**

> "The deployment controller inside or connected to the cluster observes the desired state and pulls the configuration from Git. In this project ArgoCD performs that role."

---

## Q25. What happens if ArgoCD is temporarily unavailable?

**Answer:**

> "The existing Kubernetes workloads continue running because ArgoCD is not the Kubernetes runtime itself. However, Git-based synchronization and reconciliation are unavailable until ArgoCD is healthy again."

---

## Q26. What happens if the Git repository is temporarily unavailable?

**Answer:**

> "The currently deployed Kubernetes resources continue running. ArgoCD cannot retrieve new desired state changes until it can access the repository again."

---

## Q27. What happens if a pod becomes unhealthy?

**Answer:**

> "The Deployment uses health probes against the application's `/health` endpoint. Kubernetes uses the probe results to determine whether the container is alive and whether the pod is ready to receive traffic."

---

## Q28. How would you troubleshoot an ArgoCD application showing OutOfSync?

**Answer:**

> "First I would check the ArgoCD application and identify which resource differs. Then I would compare the Git manifest with the live Kubernetes resource. I would use `kubectl get`, `kubectl describe` and, where appropriate, inspect the ArgoCD diff. I would determine whether the difference is an expected Git change or manual drift before synchronizing."

Useful commands:

```bash
kubectl get deployment
kubectl describe deployment gitops-app
kubectl get pods
kubectl get service
```

---

## Q29. How would you troubleshoot pods not starting?

**Answer:**

> "I would start with the pod status and events."

```bash
kubectl get pods
kubectl describe pod <pod-name>
```

Then I would check logs:

```bash
kubectl logs <pod-name>
```

I would look for issues such as image pull failures, scheduling problems, configuration errors, insufficient resources or failed probes.

---

## Q30. How would you troubleshoot a LoadBalancer Service that does not get an external address?

**Answer:**

> "I would first check the Service and its events."

```bash
kubectl get service
kubectl describe service <service-name>
```

Then I would verify that the EKS nodes are healthy and that the AWS networking and permissions required for load balancer provisioning are correct.

---

## Q31. How would you troubleshoot an EKS node that is NotReady?

**Answer:**

```bash
kubectl get nodes
kubectl describe node <node-name>
```

I would check node conditions, events and resource pressure. I would also verify the EC2 instance and EKS managed node group status from AWS.

---

## Q32. Why should Terraform state not be committed?

**Answer:**

> "Terraform state contains information about managed infrastructure and can contain sensitive or operational details. It should not be casually committed to a public repository. In a team environment I would use remote state and state locking."

---

## Q33. What would you improve in this project for a team environment?

**Answer:**

> "The next improvements would be remote Terraform state with locking, a more structured environment separation, stronger secret management and a more formal CI/CD workflow around infrastructure and Kubernetes manifest changes. I would add those based on actual team requirements rather than adding complexity just for the demo."

---

## Q34. Is this production?

**Answer:**

> "No. It is a hands-on cloud and GitOps implementation. I used real AWS EKS infrastructure to understand the workflow, but I tear the environment down after testing to avoid ongoing costs. The configuration is kept reproducible in Git."

This is the safest way to answer. Do not describe the personal project as a production system.

---

## Q35. How long did the EKS environment run?

**Answer:**

Do not invent a duration.

Say:

> "It was created for hands-on testing and demonstrations. I tear it down after the session to avoid unnecessary AWS charges."

---

# 23. Scenario-Based Questions

## Scenario 1 — ArgoCD keeps changing replicas back

**Question:**

A developer runs:

```bash
kubectl scale deployment gitops-app --replicas=5
```

but it keeps going back to 2. Why?

**Answer:**

> "Because Git defines 2 replicas and ArgoCD self-healing is enabled. The manual change creates drift, so ArgoCD keeps reconciling the live state back to the Git state."

---

## Scenario 2 — Developer wants 5 replicas

**Question:**

What should they do?

**Answer:**

> "Change the desired replica count in the Kubernetes manifest in Git, commit and push it. ArgoCD will detect the change and synchronize the cluster."

---

## Scenario 3 — ArgoCD says Synced but application is unavailable

**Answer:**

> "Synced only means the live Kubernetes resource configuration matches Git. It does not automatically mean the application is functioning correctly. I would check pod status, readiness/liveness probes, logs, Service status and the LoadBalancer."

---

## Scenario 4 — Terraform succeeds but kubectl cannot connect

**Answer:**

> "I would first verify that the EKS cluster exists and then configure the local kubeconfig."

```bash
aws eks update-kubeconfig   --region ap-south-1   --name devops-gitops-eks
```

Then:

```bash
kubectl get nodes
```

I would also verify AWS credentials and the active AWS region/account.

---

## Scenario 5 — Terraform destroy fails

**Answer:**

> "I would inspect the Terraform error rather than repeatedly running destroy. I would identify which resource is preventing deletion, check dependencies and AWS-side resources, and then resolve the underlying dependency before retrying."

Useful commands:

```bash
terraform plan
terraform state list
terraform destroy
```

---

# 24. Questions You Should Be Able to Draw on a Whiteboard

Be able to explain these without looking at notes:

### Architecture

```text
Git
 ↓
ArgoCD
 ↓
EKS
 ↓
Deployment
 ↓
Service
 ↓
AWS Load Balancer
```

### Infrastructure

```text
Terraform
 ↓
VPC
 ├── Public subnets
 ├── Private subnets
 ├── IGW
 └── NAT
 ↓
EKS
 ↓
Managed Node Group
```

### GitOps

```text
Git desired state
       ↓
    ArgoCD
       ↓
Live Kubernetes state
       ↑
   self-healing
```

---

# 25. Important Commands to Remember

## AWS

```bash
aws sts get-caller-identity

aws eks update-kubeconfig   --region ap-south-1   --name devops-gitops-eks

aws eks list-clusters --region ap-south-1
```

## Terraform

```bash
terraform init
terraform plan
terraform apply
terraform destroy
terraform state list
```

## Kubernetes

```bash
kubectl get nodes
kubectl get pods
kubectl get deployments
kubectl get services

kubectl describe pod <pod-name>
kubectl describe deployment <deployment-name>
kubectl describe service <service-name>

kubectl logs <pod-name>

kubectl scale deployment gitops-app --replicas=5
```

## ArgoCD

```bash
kubectl get pods -n argocd

kubectl port-forward svc/argocd-server -n argocd 8080:443
```

---

# 26. 30-Second Revision

Remember these five points:

1. **Terraform** creates the AWS infrastructure.
2. **EKS** provides the managed Kubernetes environment.
3. **Git** contains the desired Kubernetes state.
4. **ArgoCD** synchronizes Git with EKS.
5. **Self-healing** corrects manual drift back to Git.

The key demonstration:

```text
Git says: 2 replicas
       ↓
kubectl changes it to 5
       ↓
ArgoCD detects drift
       ↓
ArgoCD reconciles
       ↓
Back to 2 replicas
```

---

# 27. 2-Minute Interview Explanation

If the interviewer says:

**"Tell me about one of your DevOps projects."**

Use this structure:

> "One of my projects was a GitOps implementation on AWS EKS. I used Terraform to provision the infrastructure, including the VPC, public and private subnets, NAT Gateway, EKS cluster, managed node group and IAM roles.
>
> For the Kubernetes layer, I created a Deployment with two replicas, rolling updates and health probes, and a LoadBalancer Service to expose the application through AWS.
>
> I then installed ArgoCD and configured it to watch the Kubernetes manifests in Git. I enabled automated sync, self-healing and pruning.
>
> The main thing I wanted to demonstrate was reconciliation. I manually scaled the deployment from two replicas to five using kubectl. That created drift because Git still defined two replicas. ArgoCD detected the drift and automatically reconciled the cluster back to two replicas.
>
> The project helped me understand the separation between infrastructure provisioning with Terraform and application delivery with GitOps and ArgoCD. Since this is a hands-on AWS project, I tear the environment down after testing to avoid ongoing AWS costs, but the Terraform and Kubernetes configuration remain in Git so I can recreate it."

---

# 28. What Not to Say

Avoid saying:

- "This was a production EKS cluster."
- "I managed a production Kubernetes platform."
- "I handled production traffic."
- "I implemented enterprise-grade GitOps."
- "I maintained this cluster 24/7."

Instead say:

- "hands-on EKS implementation"
- "real AWS environment used for testing"
- "demonstrated GitOps reconciliation"
- "Terraform-provisioned infrastructure"
- "ArgoCD self-healing demonstration"

Keep the explanation aligned with what was actually implemented.

---

# 29. Final Mental Model

Think of the project as two layers.

## Infrastructure Layer

```text
Terraform
   ↓
AWS
   ↓
VPC + Networking + EKS + Nodes + IAM
```

## Application Delivery Layer

```text
Git
 ↓
ArgoCD
 ↓
Kubernetes
 ↓
Deployment + Service
 ↓
Application
```

And the key GitOps principle:

```text
Git = Desired State
ArgoCD = Reconciliation
Kubernetes = Runtime
```

That is the core of the entire project.
