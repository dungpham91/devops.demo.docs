# [ArgoCD](https://github.com/dungpham91/devops.demo.argocd)

![argocd-result.jpg](../images/argocd/argocd-result.jpg)

By the time we deploy ArgoCD, we have completed:

- Created the infrastructure on AWS using Terraform, now the EKS cluster is ready for you to use.

- Created images for the frontend and backend, pushed them to ECR, ready to deploy it to the EKS cluster.

### Table of contents

- [1. Structure of the repository](#1-structure-of-the-repository)
- [2. How to use this repository](#2-how-to-use-this-repository)
  - [2.1 Create ACM cert](#21-create-acm-cert)
  - [2.2 Check if the IAM user has access to EKS resources](#22-check-if-the-iam-user-has-access-to-eks-resources)
  - [2.3 Setup on Bastion instance](#23-setup-on-bastion-instance)
  - [2.4 Install some tools on EKS cluster](#24-install-some-tools-on-eks-cluster)
    - [2.4.1 ALB Ingress Controller](#241-alb-ingress-controller)
    - [2.4.2 ArgoCD](#242-argocd)
- [3. Check the results](#3-check-the-results)

## 1. Structure of the repository

Below is an explanation of the directories and files contained in the repository.

![argocd-repo-structure.jpg](../images/argocd/argocd-repo-structure.jpg)

## 2. How to use this repository

> GitOps implementation is relatively complex at first, due to the dependence on many different steps. After the setup is complete, we can deploy the following applications automatically and simply.

### 2.1 Create ACM cert

> We can of course use Terraform to create, but in the context of the demo, I didn't cram everything into Terraform, while only clicking for about 5 minutes.

First, you need to create an ACM cert to use for future ingresses.

![acm-cert.jpg](../images/argocd/acm-cert.jpg)

### 2.2 Check if the IAM user has access to EKS resources

Because the next steps will require you to use `kubectl` and `aws` commands, you need to make sure your IAM has access to the resources inside the EKS cluster.

![eks-access.jpg](../images/argocd/eks-access.jpg)

### 2.3 Setup on Bastion instance

![connect-bastion.jpg](../images/argocd/connect-bastion.jpg)

Since all our resources (except ALB) are located inside private subnets, we cannot directly access the EKS cluster.

Even the Bastion instance doesn't have a public IP and SSH key.

I'm using SSM to connect to it and this is the safest and fastest way to do it. There are many different solutions like VPN, EIC, PAM or Cloudflare Zero Trust Tunnel to do this but the complexity and cost are much higher. So I chose SSM for this demo, still ensuring security.

After connecting to Bastion, you will need to install some tools: `eksctl`, `helm`, `kubectl`.

```sh
cd

# eksctl
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm -f eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin

# helm
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh

# kubectl
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client

# config iam credentials
aws configure

# update kubeconfig
aws eks --region ap-southeast-1 update-kubeconfig --name devopslite-dev-eks-cluster
kubectl get svc
```

Then you need to update the kubeconfig of the EKS cluster to be able to work with it.

### 2.4 Install some tools on EKS cluster

> Again, I would like to reiterate that we can absolutely use Terraform to install these tools, including ArgoCD. But because of the time and scope of the demo, plus the version changes in the past year of the tools, it took me a lot of time to do it. So I chose to install them manually, instead of putting them all in Terraform. If I have time, I will write Terraform for them.

Because the installation process of these tools is quite long, I do not want to repeat what the documentation has already detailed.

So I will only provide links to the tools that need to be installed on the EKS cluster.

#### 2.4.1 ALB Ingress Controller

Ingress Controller is definitely needed first, because we only have EKS cluster but not Ingress Controller to expose services to the internet.

https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html

![alb-ingress-controller.jpg](../images/argocd/alb-ingress-controller.jpg)

#### 2.4.2 ArgoCD

This demo of mine uses ArgoCD to automate application deployment using GitOps. So we need it.

https://argo-cd.readthedocs.io/en/stable/getting_started

After installing ArgoCD's Helm chart on your EKS cluster, you will need to apply the [argocd-ingress.yaml](../argocd-manifests/argocd-ingress.yaml) file to be able to access the UI from the internet.

```sh
kubectl apply -f argocd-ingress.yaml
```

![argocd-01.jpg](../images/argocd/argocd-01.jpg)

![argocd-02.jpg](../images/argocd/argocd-02.jpg)

You can ArgoCD admin password with following command and then login UI

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

![argocd-03.jpg](../images/argocd/argocd-03.jpg)

Then, you apply the [argocd-connect-github.yaml](../argocd-manifests/argocd-connect-github.yaml) file so that the ArgoCD server connects to the repository containing the Helm charts and ArgoCD application files.

The value `ghp_wQArfr2xzFSP5pphXXXXXXXXXXXXXXXXXXXX` in that file, you will need to create a `classic PAT` on your GitHub account with `repo` permissions.

![argocd-04.jpg](../images/argocd/argocd-04.jpg)

Next, you apply file [argocd-project-dev.yaml](../argocd-manifests/argocd-project-dev.yaml) to create Project on ArgoCD. You can create many different projects such as dev, qa, uat, prod to differentiate applications by environment.

![argocd-05.jpg](../images/argocd/argocd-05.jpg)

Finally, apply file [argocd-parent-app.yaml](../argocd-manifests/argocd-parent-app.yaml) to create a Parent App, it will manage the child apps in folder https://github.com/dungpham91/devops.demo.argocd/tree/main/env/dev

This means that every time you add a child app (.yaml file for ArgoCD) to the `env/dev` directory above, this Parent App will automatically sync it to the EKS cluster without you having to do anything.

![argocd-06.jpg](../images/argocd/argocd-06.jpg)

## 3. Check the results

- External Secret Operator

I use the External Secret Operator to allow applications running in EKS to read secrets stored in AWS Secret Manager. This helps to centrally manage secrets and avoid disclosure from the EKS cluster.

![result-01.jpg](../images/argocd/result-01.jpg)

- Monitoring stack

I use Helm chart kube-prometheus-stack to deploy monitoring service inside EKS cluster.

![result-02.jpg](../images/argocd/result-02.jpg)

![result-03.jpg](../images/argocd/result-03.jpg)

![result-04.jpg](../images/argocd/result-04.jpg)

- Frontend

![result-05.jpg](../images/argocd/result-05.jpg)

![result-06.jpg](../images/argocd/result-06.jpg)

- Backend

![result-07.jpg](../images/argocd/result-07.jpg)

![result-08.jpg](../images/argocd/result-08.jpg)

- PostgreSQl

![result-09.jpg](../images/argocd/result-09.jpg)
