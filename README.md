# Deploying Application on Private AKS using Private ACR and Azure DevOps for CI and ArgoCD for CD

This document outlines the steps for building and deploying an application on a private Azure Kubernetes Service (AKS) cluster using a private Azure Container Registry (ACR), Azure DevOps for Continuous Integration (CI), and ArgoCD for Continuous Deployment (CD).

## Table of Contents
1. [CI - Creating a Self-Hosted Agent](#ci---creating-a-self-hosted-agent)
2. [Creating Private ACR with Self-Hosted Agent](#creating-private-acr-with-self-hosted-agent)
3. [Creating a Private AKS Cluster](#creating-a-private-aks-cluster)
4. [Continuous Deployment (CD)](#continuous-deployment-cd)
5. [Installing and Configuring SonarQube](#installing-and-configuring-sonarqube)
6. [Installing Trivy on Self-Hosted Agent](#installing-trivy-on-self-hosted-agent)

## CI - Creating a Self-Hosted Agent

### Why Use Self-Hosted Agents?

When Azure resources are deployed in a Virtual Network (VNet) with Private Endpoints enabled, Microsoft-hosted agents cannot communicate with resources, resulting in pipeline failures. A self-hosted agent allows you to deploy applications that require custom libraries or software to build and deploy.

### Steps Involved in Creating a Self-Hosted Agent

1. **Create a Linux VM:**
   - Go to Azure Portal and log in: [Azure Portal](https://portal.azure.com/#home)
   - Search for Virtual Machines and configure your VM as shown in the screenshots below.
   - Download the private key; you will need this to log in to the VM.

2. **Configure Self Hosted Agent**
   - Follow the steps to create a PAT token for authentication from the VM to the Azure DevOps portal.
  
   - Goto Organization setting > Aget pools > Add pool > Self hosted > Provide a name & description to agent pool and click on Create button.
   - As you can see, a new Agent Pool is created. Make a note of the Agent Pool Name as we will be using the same name in YAML pipeline in pool definition.
   - Now, lets create a New agent under the Agent Pool.
   - 1. Click on Agent Pool > New Agent > Select Linux > Follow the instruction

3. **Configure the VM as a Self-Hosted Agent:**
   - Install the agent on the VM.
   - ```bash
     mkdir downloads
     cd downloads
     curl -O https://vstsagentpackage.azureedge.net/agent/3.240.1/vsts-agent-linux-x64-3.240.1.tar.gz
     tar zxvf vsts-agent-linux-x64-3.240.1.tar.gz
     ./config.sh

   - Run the Agent.
   - ```bash
     ./run.sh
   - To run the agent as a service.
   - ```bash
     sudo ./svc.sh install
     ./runsvc.sh


4. **Use Self-Hosted Agent in Azure YAML Pipeline:**
   - Define the pool in your Azure DevOps pipeline YAML:
   - ```bash
      trigger: none
      pr: none
      pool: "linux-self-hosted-agent-pool"
      steps:
        - script: echo Hello, world!
          displayName: "Run a one-line script"
   - For more detailed steps on setting up a Linux self-hosted agent, refer to this article (https://medium.com/@prasad.reddy0708/setup-linux-self-hosted-agent-for-pipelines-in-azure-devops-8c8ad6c695a3).
  
5. **Creating Private ACR:**
   - *Steps to Create Private ACR*
   - Create a Service Connection:
   - Go to Azure DevOps > Project Settings > Service Connections.
   - Use Managed Identity or Service Principal for the connection.
   - Create Virtual Networks:
   - Create two VNets: one for the self-hosted agent and another for the private ACR.
   - Create VNet Peering:
   - Establish peering between the two VNets.
   - Create the Agent VM:
   - Create a VM in the same VNet as your ACR and configure SSH access.
   - Configure ACR:
   - Create a Private ACR using the Azure Portal, ensuring that the network section allows for private access.
   - Connect ACR with Azure Pipeline:
   - Create a service connection for ACR using the username and password from ACR access keys.
   - If you want to access ACR from any VM:
   - 1. Create VNet Peering.
     2. Assign a Managed Identity to the VM.
     3. Create private links in the Azure private DNS zone.
     4. For accessing the registry from VM using Managed Identity, refer to the documentation (https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication-managed-identity?tabs=azure-cli#configure-the-vm-with-a-system-managed-identity-1).

6. **Creating AKS Cluster**
   - Steps to Create AKS Cluster
   - Create Resource Group (RG).
   - Create Virtual Network (VNet).
   - Create AKS Cluster.
   - Create a VM for accessing the cluster (the VM should be in the same VNet).
  
   - Note
   - Push the code to Git (including the Dockerfile).
   - For detailed guidance, refer to this link (https://learn.microsoft.com/en-us/azure/aks/private-clusters?tabs=default-basic-networking%2Cazure-portal).
  
7. **Continuous Deployment with ArgoCD**
   - *Steps for Setting Up ArgoCD*
   - Peer the ACR and AKS VNET.
   - Add Private Network Link in the ACR’s private DNS zone for the AKS VNet.
   - Check Connectivity using AKS Managed Identity:
   - Add role to AKS Managed Identity - AcrPull in ACR IAM.
   - Use the command
   - ```bash
     az aks check-acr -g <AKS-rg-name> -n <cluster-name> --acr acrprivateshubh.azurecr.io
   - Create a Kubernetes Docker Registry Secret for authenticating to ACR:
   - ```bash
     kubectl create secret docker-registry portfolio-img-pull-secret --namespace=shubhtest --docker-server=https://acrprivateshubh.azurecr.io --docker-username=AcrPrivateShubh --docker-password=<your-acr-password>
   - Install Nginx Ingress Controller:
   - Refer to Nginx Ingress Guide (https://spacelift.io/blog/kubernetes-ingress).
   - Deploy the Application using ArgoCD:
   - ```bash
      kubectl create namespace argocd
      kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
      kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
      kubectl get svc argocd-server -n argocd

   - Access ArgoCD:
   - Create ArgoCD Application:
   - In the AKS cluster, create a Docker secret for pulling the image:
   - ```bash
     kubectl create secret docker-registry portfolio-img-pull-secret --namespace=portfolio --docker-server=https://acrprivateshubh.azurecr.io --docker-username=AcrPrivateShubh --docker-password=<your-acr-password>
   - Edit the Deployment File to use the created secret.
   - Force Sync in ArgoCD.
   - Access the Ingress Controller's Public IP.

8. **ArgoCD Auto-Sync Feature**
   - Argo CD’s auto-sync feature operates based on the Application reconciliation timeout, which defaults to 3 minutes (180 seconds).
   - You can adjust the interval by modifying the timeout.reconciliation value in the argocd-cm ConfigMap (if the property is not present, add it) to “5s” for 5 seconds:
   - ```bash
      kubectl rollout restart deployment argocd-server -n argocd
      kubectl get pod argocd-application-controller-0 -n argocd -o yaml > app-cont.yaml
      kubectl delete pod argocd-application-controller-0 -n argocd
      kubectl apply -f app-cont.yaml
      kubectl logs -n argocd argocd-application-controller-0 -f

9. **Installing and Configuring SonarQube**
    - Install SonarQube on the Cluster:
    -    install Helm:
    -    ```bash
         curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    - Add SonarQube to Helm repo
    - ```bash
      helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
      helm repo update
    - Create Namespace
    - ```bash
      kubectl create namespace sonarqube
    - Deploy SonarQube using Helm
    - ```bash
      helm upgrade --install -n sonarqube sonarqube sonarqube/sonarqube
    - *Modify Helm Values:*
    - Create a values.yaml file
    - ```bash
        service:
        type: LoadBalancer
        externalPort: 9000
        internalPort: 9000
    - Apply the values
    - ```bash
      helm upgrade --install -n sonarqube sonarqube sonarqube/sonarqube -f values.yaml
    - *Access SonarQube:*
    - Check the service to get the access URL
    - ```bash
      kubectl get services -n sonarqube
    - *Install SonarQube Extension in Azure DevOps:*
    - Navigate to Azure DevOps project > Project settings > Extensions.
    - Search for “SonarQube” and install the extension.
    - *Create a Service Connection:*
    - Go to Project settings > Service connections.
    - Click on New service connection and select SonarQube.
    - Enter the required details
    -    SonarQube Server URL: The URL of your SonarQube instance.
    -    Token: A token generated from your SonarQube instance (create this in SonarQube under My Account > Security > Generate Tokens).
    - *Edit the Pipeline Script:*
    - Add SonarQube tasks to your Azure DevOps pipeline (ensure Java 17 is installed on the self-hosted agent).
    - Create a local project in SonarQube:
    -    Provide Project name and key, and use the same in the pipeline script.
    - Click Create and run the pipeline.

10. **Installing Trivy on Self-Hosted Agent**
    - To ensure your Docker images are secure and free from vulnerabilities, you can install Trivy on your self-hosted agent.
    - *Steps to Install Trivy*
    -    Prerequisites: Ensure you have root access to your self-hosted agent.
    -    Installation Steps: Execute the following commands to install Trivy on your self-hosted agent:
    -    ```bash
         # Download the latest version of Trivy
         wget https://github.com/aquasecurity/trivy/releases/latest/download/trivy_0.39.0_Linux-64bit.tar.gz
         
         # Extract the downloaded tar file
         tar zxvf trivy_0.39.0_Linux-64bit.tar.gz
         
         # Move the trivy binary to /usr/local/bin
         sudo mv trivy /usr/local/bin/
         
         # Verify the installation
         trivy --version

    - Reference: For more detailed information about Trivy installation and usage, refer to the following link (https://k21academy.com/docker-kubernetes/docker-image-vulnerabilities/)


