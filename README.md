
# End-to-End Bank Application Deployment on AWS EKS


## This is a multi-tier bank an application written in Java (Springboot).

### Login page

![App Screenshot](https://github.com/2604manishyadav/Springboot-Bankapp/blob/main/images/LoginPage.PNG)

### Transaction Logs

![App Screenshot](https://github.com/2604manishyadav/Bankapp/blob/d9ad7e37fdf190da4cb08ef20296bb0a53391c5c/Transaction-logs.PNG)

### Tech stack used in this project:

- Github (Code)  
- Terraform (Infrastructure)  
- Docker (Containerization)  
- Jenkins (CI)  
- ArgoCD (CD)  
- AWS EKS (Kubernetes)  
- Helm (Monitoring using grafana and prometheus)


# STEPS TO Deploy

### Create a Master machine on AWS (t2.large) and 29 GB of storage using Terraform

### Installation of Terraform

Install Terraform in your local machine using shell script
 
#### terraform_install.sh 

Verify Installation

    terraform --version

### Install awscli using shell script

#### awscli-install.sh

Prerequisite for configure awscli

IAM User with access keys and secret access keys

#### Configure awscli

    aws configure

### Create ec2 instance using terraform configuration file

    terraform init 
    terraform plan  
    terraform apply

### Install eksctl (used to create and manage EKS clusters)

    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo mv /tmp/eksctl /usr/local/bin
    eksctl version

### EKS cluster creation command without nodegroup

    eksctl create cluster --name=bankapp-cluster --region=eu-west-1 --without-nodegroup --version=1.32

### Associate OIDC to EKS cluster

    eksctl utils associate-iam-oidc-provider --cluster bankapp-cluster --approve

### Update the vpc-cni addon in your EKS

    eksctl update addon --name vpc-cni --cluster bankapp-cluster --force

### Install kubectl

    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    chmod +x kubectl
    sudo mv kubectl /usr/local/bin/

### Create nodegroup for your cluster

    eksctl create nodegroup --cluster=bankapp-cluster \
    --region=eu-west-1 \
    --name=bankapp-ng \
    --node-type=t2.medium \
    --nodes=2 \
    --nodes-min=2 \
    --nodes-max=2 \
    --node-volume-size=20 \
    --ssh-access \
    --ssh-public-key=bankapp-automate-key


#### NOTE :   

Make sure public key is available in your aws key pair 

### Install Jenkins using shell script

jenkin-install.sh 

#### Once jenkins install, change default port of Jenkins from 8080 to 8081 , 8080 port already assigned bankapp 

Edit below file
sudo vim /usr/lib/systemd/system/jenkins.service file
![App Screenshot](https://github.com/2604manishyadav/Bankapp/blob/f3e37cf842890342e5a81e2528f1298ba0bbc7e4/Update-Jenkins-Port.PNG)

#### Reload deamon and restart jenkins
    sudo systemctl daemon-reload 
    sudo systemctl restart jenkins

### Install Docker

    sudo apt-get install docker.io -y
    sudo newgrp -aG docker Jenkins
    newgrp docker

#### Create CI Pipeline using Jenkins for pushing image to DockerHub, jenkins configuration file shared 
![App Screenshot](https://github.com/2604manishyadav/Bankapp/blob/c6c27ead04bcd0716a48c132fab209a559a837e6/BankappCI.PNG)

#### Add credential (Username and password) of DockerHub in credential of Jenkins

![App Screenshot](https://github.com/2604manishyadav/Bankapp/blob/c6c27ead04bcd0716a48c132fab209a559a837e6/AddCredentialsof%20DockerHub.PNG)

### Install & Configure ArgoCD

#### Create a namespace for ArgoCD
    kubectl create ns argocd

#### Apply ArgoCD manifest files    
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

#### Verify argocd Pods is running
    kubctl get pods -n argocd

#### Install argocd CLI

    curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
    chmod +x /usr/local/bin/argocd

#### Change argocd service Port type from clusterip to nodeport

     kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

#### Extarct the initial password for login to argocd , Username will be "admin"
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

#### Login to argocd
    argocd login <ARGOCD_SERVER_IP>:<PORT> --username=admin  

#### Update password
     argocd account update-password

#### get list of cluster
     argocd cluster list

#### get your cluster name
     kubectl config get-contexts

#### Add your eks cluster in argocd
     argocd cluster add Project-User@bankapp-cluster.eu-west-1.eksctl.io --name bankapp-cluster

#### Add your github repository in argocd
     argocd repo add https://github.com/2604manishyadav/Bankapp.git

#### Create application
    argocd app create bankapp \
    --repo https://github.com/2604manishyadav/Bankapp.git \
    --path kubernetes \
    --revision mega-project \
    --dest-server https://C32D2EA5FF2136CA3E1F4838B1551148.sk1.eu-west-1.eks.amazonaws.com \
    --dest-namespace bankapp-namespace \
    --sync-policy automated 

####  Sucessfully create application in ArgoCD

![App Screenshot](https://github.com/2604manishyadav/Bankapp/blob/c93d706b6c8702b1abc0c0ccc56c1cbbb7fe2466/ApplicationDeploy.PNG)    

#### Change port type of deployment service from ClusterIP to Nodeport for accessing Bankapp application
     kubectl patch svc bankapp-service -n bankapp-namespace -p '{"spec": {"type": "NodePort"}}'

#### Bankapp Deployed in EKS Cluster

![App Screenshot](https://github.com/2604manishyadav/Bankapp/blob/e727b7a439b45d570d233342889acac4345f5d88/BankappDeploy.PNG)


## Monitor EKS cluster, kubernetes components and workloads using prometheus and grafana via HELM (On Master machine)

#### Install Helm chart
     curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
     chmod 700 get_helm.sh
     ./get_helm.sh

#### Add Helm Stable Charts for Your Local Client
     helm repo add stable https://charts.helm.sh/stable

#### Add prometheus to Helm repository
     helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

#### Create Prometheus namespace
     kubectl create namespace prometheus

#### Install Prometheus using Helm
     helm install stable prometheus-community/kube-prometheus-stack -n prometheus

#### Change type port of stable grafana service from clusterip to nodeport
     kubectl patch svc stable-grafana -n prometheus -p '{"spec": {"type": "NodePort"}}'

#### To get initial password for login to Grafana , username will be 'admin'
     kubectl get secret stable-grafana -n prometheus -o jsonpath="{.data.admin-password}" | base64 -d; echo


### Grafana Dashboard

![App Screenshot](https://github.com/2604manishyadav/Bankapp/blob/668a73247811eb569cc841220439c4e34037d119/Grafana-Dashboard.PNG)

### Grafana Cluster view

![App Screenshot](https://github.com/2604manishyadav/Bankapp/blob/668a73247811eb569cc841220439c4e34037d119/GrafanaCluster.PNG)



     
     
     
     


 






