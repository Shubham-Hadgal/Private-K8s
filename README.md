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

2. **Create a PAT (Personal Access Token):**
   - Follow the steps to create a PAT token for authentication from the VM to the Azure DevOps portal.

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
   - Steps to Create Private ACR
   - 1. Create a Service Connection:
        Go to Azure DevOps > Project Settings > Service Connections.
        Use Managed Identity or Service Principal for the connection.

