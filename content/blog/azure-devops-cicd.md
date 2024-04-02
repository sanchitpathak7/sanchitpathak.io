+++
title = "Azure DevOps CI/CD Project for Distributed Microservice Application"
+++

Azure DevOps is a suite of services provided by Microsoft Azure that allows teams to plan projects, collaborate on code development, and deploy applications. It includes services like Azure Repos for version control, Azure Pipelines for continuous integration and deployment (CI/CD), Azure Boards for project management, Azure Artifacts for package management, and Azure Test Plans for test management. 

Overall, it provides a comprehensive set of tools for software development and delivery, supporting agile practices and DevOps principles.

In this blog post I am going over how to setup end to end CI/CD project using Azure Devops.

![](https://github.com/sanchitpathak7/blogsite/assets/44384286/6488929e-8b3d-4bfc-a081-daafb4947213)
Source: https://github.com/dockersamples/example-voting-app

### Pre-Requisities
- Import the source repository into Azure DevOps portal:
![](https://github.com/sanchitpathak7/blogsite/assets/44384286/48b71e34-4747-4ed0-92e3-14c117a6061c)

- Select `main` as the default branch:
![](https://github.com/sanchitpathak7/blogsite/assets/44384286/9e224f3c-5518-435c-9636-befbfd67a28a)

- Create resource group `azurecicd` for the project:
```
$ az group create --name azurecicd --location eastus
{
  "id": "/subscriptions/6fecc3e6-8a71-4138-b2a4-6527f4151493/resourceGroups/azurecicd",
  "location": "eastus",
  "managedBy": null,
  "name": "azurecicd",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": "Microsoft.Resources/resourceGroups"
}
```

- Create container registry `sanchitazurecicd` in Azure Portal:
```
$ az acr create --name sanchitazurecicd --resource-group azurecicd --sku Standard 
{
  "adminUserEnabled": false,
  "anonymousPullEnabled": false,
  "creationDate": "2024-04-01T05:27:16.531227+00:00",
  "dataEndpointEnabled": false,
  "dataEndpointHostNames": [],
  "encryption": {
    "keyVaultProperties": null,
    "status": "disabled"
  },
  "id": "/subscriptions/6fecc3e6-8a71-4138-b2a4-6527f4151493/resourceGroups/azurecicd/providers/Microsoft.ContainerRegistry/registries/sanchitazurecicd",
  "identity": null,
  "location": "eastus",
  "loginServer": "sanchitazurecicd.azurecr.io",
  "metadataSearch": "Disabled",
  "name": "sanchitazurecicd",
  "networkRuleBypassOptions": "AzureServices",
  "networkRuleSet": null,
  "policies": {
    "azureAdAuthenticationAsArmPolicy": {
      "status": "enabled"
    },
    "exportPolicy": {
      "status": "enabled"
    },
    "quarantinePolicy": {
      "status": "disabled"
    },
    "retentionPolicy": {
      "days": 7,
      "lastUpdatedTime": "2024-04-01T05:27:28.285623+00:00",
      "status": "disabled"
    },
    "softDeletePolicy": {
      "lastUpdatedTime": "2024-04-01T05:27:28.285666+00:00",
      "retentionDays": 7,
      "status": "disabled"
    },
    "trustPolicy": {
      "status": "disabled",
      "type": "Notary"
    }
  },
  "privateEndpointConnections": [],
  "provisioningState": "Succeeded",
  "publicNetworkAccess": "Enabled",
  "resourceGroup": "azurecicd",
  "sku": {
    "name": "Standard",
    "tier": "Standard"
  },
  "status": null,
  "systemData": {
    "createdAt": "2024-04-01T05:27:16.531227+00:00",
    "createdBy": "sanchitpathak17@gmail.com",
    "createdByType": "User",
    "lastModifiedAt": "2024-04-01T05:27:16.531227+00:00",
    "lastModifiedBy": "sanchitpathak17@gmail.com",
    "lastModifiedByType": "User"
  },
  "tags": {},
  "type": "Microsoft.ContainerRegistry/registries",
  "zoneRedundancy": "Disabled"
}
```

---
### Implementing Continuous Integration (CI)
- Creating VM `azureagent` for running pipelines:
```
$ az vm create --resource-group azurecicd --name azureagent --image Ubuntu2204 --vnet-name demo-vnet --subnet default --generate-ssh-keys --output json --verbose
Use existing SSH public key file: <REDACTED>
{
  "fqdns": "",
  "id": "/subscriptions/6fecc3e6-8a71-4138-b2a4-6527f4151493/resourceGroups/azurecicd/providers/Microsoft.Compute/virtualMachines/azureagent",
  "location": "eastus",
  "macAddress": "00-0D-3A-57-DA-96",
  "powerState": "VM running",
  "privateIpAddress": "10.0.0.4",
  "publicIpAddress": "13.90.150.209",
  "resourceGroup": "azurecicd",
  "zones": ""
}
```

- Configure new Agent Pool named `azureagent` in project settings and run the instructions on the created VM `azureagent`:
![](https://github.com/sanchitpathak7/blogsite/assets/44384286/64148cb9-7c6a-471f-b6ff-9e009afe516d)

- Connect to the VM and download the binary:
```
$ az ssh vm --resource-group azurecicd --vm-name azureagent --subscription 6fecc3e6-8a71-4138-b2a4-6527f4151493
...
sanchitpathak17@gmail.com@azureagent:~$ wget https://vstsagentpackage.azureedge.net/agent/3.236.1/vsts-agent-linux-x64-3.236.1.tar.gz
--2024-04-01 17:15:18--  https://vstsagentpackage.azureedge.net/agent/3.236.1/vsts-agent-linux-x64-3.236.1.tar.gz
Resolving vstsagentpackage.azureedge.net (vstsagentpackage.azureedge.net)... 72.21.81.200, 2606:2800:11f:17a5:191a:18d5:537:22f9
Connecting to vstsagentpackage.azureedge.net (vstsagentpackage.azureedge.net)|72.21.81.200|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 161771925 (154M) [application/octet-stream]
Saving to: ‘vsts-agent-linux-x64-3.236.1.tar.gz’
vsts-agent-linux-x64-3.236.1.tar.gz             100%[====================================================================================================>] 154.28M   114MB/s    in 1.4s    
2024-04-01 17:15:20 (114 MB/s) - ‘vsts-agent-linux-x64-3.236.1.tar.gz’ saved [161771925/161771925]
```

- Untar the binary:
```
sanchitpathak17@gmail.com@azureagent:~/myagent$ tar zxvf vsts-agent-linux-x64-3.236.1.tar.gz
...
sanchitpathak17@gmail.com@azureagent:~/myagent$ ls
bin  config.sh  env.sh  externals  license.html  run-docker.sh  run.sh  vsts-agent-linux-x64-3.236.1.tar.gz
```

- Create a Personal Access Token in Azure DevOps Settings and run the config script:
```
sanchitpathak17@gmail.com@azureagent:~/myagent$ ./config.sh
     _ _
 / _ \                     | ___ (_)          | (_)
/ /_\ \_____   _ _ __ ___  | |_/ /_ _ __   ___| |_ _ __   ___  ___
|  _  |_  / | | | '__/ _ \ |  __/| | '_ \ / _ \ | | '_ \ / _ \/ __|
| | | |/ /| |_| | | |  __/ | |   | | |_) |  __/ | | | | |  __/\__ \
\_| |_/___|\__,_|_|  \___| \_|   |_| .__/ \___|_|_|_| |_|\___||___/
                                   | |
        agent v3.236.1             |_|          (commit 1d7a476)
>> End User License Agreements:
Building sources from a TFVC repository requires accepting the Team Explorer Everywhere End User License Agreement. This step is not required for building sources from Git repositories.
A copy of the Team Explorer Everywhere license agreement can be found at:
  /home/sanchitpathak17/myagent/license.html
Enter (Y/N) Accept the Team Explorer Everywhere license agreement now? (press enter for N) > Y
>> Connect:
Enter server URL > https://dev.azure.com/sanchitpathak17
Enter authentication type (press enter for PAT) > 
Enter personal access token > ****************************************************
Connecting to server ...

>> Register Agent:
Enter agent pool (press enter for default) > azureagent
Enter agent name (press enter for azureagent) > azureagent
Scanning for tool capabilities.
Connecting to the server.
Successfully added the agent
Testing agent connection.
Enter work folder (press enter for _work) > 
2024-04-01 17:22:27Z: Settings Saved.
```

- Setup Docker on the VM:
```
sanchitpathak17@gmail.com@azureagent:~/myagent$ sudo apt install docker.io
sanchitpathak17@gmail.com@azureagent:~/myagent$ sudo usermod -aG docker sanchitpathak17@gmail.com
sanchitpathak17@gmail.com@azureagent:~/myagent$ sudo systemctl restart docker
sanchitpathak17@gmail.com@azureagent:~/myagent$ systemctl status docker
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2024-04-01 17:26:25 UTC; 3s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 4349 (dockerd)
      Tasks: 8
     Memory: 31.1M
        CPU: 314ms
     CGroup: /system.slice/docker.service
             └─4349 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

- Exit the session, log back in again and then run the executable:
```
sanchitpathak17@gmail.com@azureagent:~/myagent$ ./run.sh 
Scanning for tool capabilities.
Connecting to the server.
2024-04-01 17:26:44Z: Listening for Jobs
```

- Agent Pool is now online
![](https://github.com/sanchitpathak7/blogsite/assets/44384286/ae52cbc9-3233-4c6d-b195-76aac3ff3248)

- Configuring path based trigger while setting up Azure DevOps Pipelines for `Results` App.
```
$ voting-app/azure-pipelines-result.yml

# Docker
# Build and push an image to Azure Container Registry
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

trigger:
  paths:
    include:
      - result/*

resources:
- repo: self

variables:
  # Container registry service connection established during pipeline creation
  dockerRegistryServiceConnection: '37ddf373-922d-4add-b43a-076639b876e8'
  imageRepository: 'resultapp'
  containerRegistry: 'sanchitazurecicd.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/result/Dockerfile'
  tag: '$(Build.BuildId)'

pool:
  name: 'azureagent'

stages:
- stage: Build
  displayName: Build
  jobs:
  - job: Build
    displayName: Build
    steps:
    - task: Docker@2
      displayName: Build an image to container registry
      inputs:
        containerRegistry: '$(dockerRegistryServiceConnection)'
        repository: '$(imageRepository)'
        command: 'build'
        Dockerfile: 'result/Dockerfile'
        tags: '$(tag)'
- stage: Push
  displayName: Push
  jobs:
  - job: Push
    displayName: Push
    steps:
    - task: Docker@2
      displayName: Push an image to container registry
      inputs:
        containerRegistry: '$(dockerRegistryServiceConnection)'
        repository: '$(imageRepository)'
        command: 'push'
        tags: '$(tag)'
```

- Pipeline Run Succeeded:
![](https://github.com/sanchitpathak7/blogsite/assets/44384286/62447958-9f09-438e-b36f-6e7824b9f040)

- Configuring path based trigger while setting up Azure DevOps Pipelines for `Voting` App.
```
# Docker
# Build and push an image to Azure Container Registry
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

trigger:
  paths:
    include:
      - vote/*

resources:
- repo: self

variables:
  # Container registry service connection established during pipeline creation
  dockerRegistryServiceConnection: 'eae15d9c-952e-4b9e-bdc3-808d2a63c8f9'
  imageRepository: 'votingapp'
  containerRegistry: 'sanchitazurecicd.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/result/Dockerfile'
  tag: '$(Build.BuildId)'

pool:
  name: 'azureagent'

stages:
- stage: Build
  displayName: Build an image
  jobs:
  - job: Build
    displayName: Build
    steps:
    - task: Docker@2
      displayName: Build an image
      inputs:
        containerRegistry: '$(dockerRegistryServiceConnection)'
        repository: '$(imageRepository)'
        command: 'build'
        Dockerfile: 'vote/Dockerfile'
        tags: '$(tag)'
- stage: Push
  displayName: Push an image
  jobs:
  - job: Push
    displayName: Push
    steps:
    - task: Docker@2
      displayName: Push an image
      inputs:
        containerRegistry: '$(dockerRegistryServiceConnection)'
        repository: '$(imageRepository)'
        command: 'push'
        Dockerfile: 'vote/Dockerfile'
        tags: '$(tag)'
```

- Configuring path based trigger while setting up Azure DevOps Pipelines for `Worker` App.
```
# Docker
# Build and push an image to Azure Container Registry
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

trigger:
  paths:
    include:
      - worker/*

resources:
- repo: self

variables:
  # Container registry service connection established during pipeline creation
  dockerRegistryServiceConnection: '357976e2-83df-43b0-8208-8236ec12a82d'
  imageRepository: 'workerapp'
  containerRegistry: 'sanchitazurecicd.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/result/Dockerfile'
  tag: '$(Build.BuildId)'

pool:
  name: 'azureagent'

stages:
- stage: Build
  displayName: Build an image
  jobs:
  - job: Build
    displayName: Build
    steps:
    - task: Docker@2
      displayName: Build an image
      inputs:
        containerRegistry: '$(dockerRegistryServiceConnection)'
        repository: '$(imageRepository)'
        command: 'build'
        Dockerfile: 'worker/Dockerfile'
        tags: '$(tag)'
```

- The worker pipeline failed with error:
```
failed to parse platform : "" is an invalid component of "": platform specifier component must match "^[A-Za-z0-9_-]+$": invalid argument
```

- Updated the Dockerfile to specify the platform for worker app in Repo section. Azure automatically picked up the change and ran the pipeline successfully.
![](https://github.com/sanchitpathak7/blogsite/assets/44384286/890feb70-12de-4f02-b999-03eb9d9faaad)

- All the pipeline success confirmation:
![](https://github.com/sanchitpathak7/blogsite/assets/44384286/d04aa637-b67d-4b26-90a7-233cf2f15bc5)

- At this stage, all GitHub CI pipelines are migrated to Azure DevOps.
---

### Implementing Continuous Delivery (CD)
- Now, for the CD part, first let's create a AKS i.e. Azure Kubernetes Cluster.
```
$ az aks list --resource-group azuredevops_group --output table
Name         Location    ResourceGroup      KubernetesVersion    CurrentKubernetesVersion    ProvisioningState    Fqdn
-----------  ----------  -----------------  -------------------  --------------------------  -------------------  ----------------------------------------------
azuredevops  westus2     azuredevops_group  1.28.5               1.28.5                      Succeeded            azuredevops-dns-0y5c1lu4.hcp.westus2.azmk8s.io
```
```
$ az aks get-credentials --name azuredevops --resource-group azuredevops_group
Merged "azuredevops" as current context in /Users/sanchit/.kube/config

$ kubectl get nodes -o wide
NAME                                STATUS   ROLES   AGE    VERSION   INTERNAL-IP   EXTERNAL-IP     OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
aks-agentpool-12250849-vmss000000   Ready    agent   4m5s   v1.28.5   10.224.0.4    20.187.35.201   Ubuntu 22.04.4 LTS   5.15.0-1058-azure   containerd://1.7.7-1
```
- Install ArgoCD

Reference: https://argo-cd.readthedocs.io/en/stable/getting_started/

```
$ kubectl create namespace argocd
namespace/argocd created

$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

```
$ kubectl get all -n argocd
NAME                                                    READY   STATUS    RESTARTS   AGE
pod/argocd-application-controller-0                     1/1     Running   0          35s
pod/argocd-applicationset-controller-75b78554fd-bz86c   1/1     Running   0          38s
pod/argocd-dex-server-869fff9967-9cj42                  1/1     Running   0          37s
pod/argocd-notifications-controller-5b8dbb7c86-l2sc5    1/1     Running   0          37s
pod/argocd-redis-66d9777b78-rfnb2                       1/1     Running   0          37s
pod/argocd-repo-server-7b8d97c767-prh92                 1/1     Running   0          36s
pod/argocd-server-5c797497fb-x77xj                      1/1     Running   0          36s

NAME                                              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
service/argocd-applicationset-controller          ClusterIP   10.0.184.112   <none>        7000/TCP,8080/TCP            41s
service/argocd-dex-server                         ClusterIP   10.0.115.30    <none>        5556/TCP,5557/TCP,5558/TCP   40s
service/argocd-metrics                            ClusterIP   10.0.249.23    <none>        8082/TCP                     40s
service/argocd-notifications-controller-metrics   ClusterIP   10.0.30.135    <none>        9001/TCP                     40s
service/argocd-redis                              ClusterIP   10.0.54.169    <none>        6379/TCP                     40s
service/argocd-repo-server                        ClusterIP   10.0.7.221     <none>        8081/TCP,8084/TCP            39s
service/argocd-server                             ClusterIP   10.0.149.92    <none>        80/TCP,443/TCP               39s
service/argocd-server-metrics                     ClusterIP   10.0.146.7     <none>        8083/TCP                     39s

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argocd-applicationset-controller   1/1     1            1           39s
deployment.apps/argocd-dex-server                  1/1     1            1           38s
deployment.apps/argocd-notifications-controller    1/1     1            1           38s
deployment.apps/argocd-redis                       1/1     1            1           38s
deployment.apps/argocd-repo-server                 1/1     1            1           37s
deployment.apps/argocd-server                      1/1     1            1           37s

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/argocd-applicationset-controller-75b78554fd   1         1         1       39s
replicaset.apps/argocd-dex-server-869fff9967                  1         1         1       38s
replicaset.apps/argocd-notifications-controller-5b8dbb7c86    1         1         1       38s
replicaset.apps/argocd-redis-66d9777b78                       1         1         1       38s
replicaset.apps/argocd-repo-server-7b8d97c767                 1         1         1       37s
replicaset.apps/argocd-server-5c797497fb                      1         1         1       37s

NAME                                             READY   AGE
statefulset.apps/argocd-application-controller   1/1     37s
```

Modify the service type to NodePort for argocd-server service and vote service and then add the port numbers as a inbound allow rule on the Node Instance's NSG.

- Configure ArgoCD to connect with Azure Repo
    - First, get the secret from K8s secret resource `argocd-initial-admin-secret` in the argocd namespace and use the nodeIP:NodePort in U/I to load ArgoCD page.
![](https://github.com/sanchitpathak7/blogsite/assets/44384286/7706991c-0197-401f-85a1-aee75d36e50f)

    - Next connect to the Azure Repo using Access Token:
    ![](https://github.com/sanchitpathak7/blogsite/assets/44384286/01630e6e-6dd1-4017-8bb8-533b68d2ce58)
    ![](https://github.com/sanchitpathak7/blogsite/assets/44384286/65d224b9-e856-4f67-bbdd-878ffcbfe234)

    - Deploy AzureRepo manifests using ArgoCD to the AKS Cluster:
        - Repo
        ![](https://github.com/sanchitpathak7/blogsite/assets/44384286/91f00419-7efc-4025-82e0-6a8e3f0bec02)
        - Create the Application
        ![](https://github.com/sanchitpathak7/blogsite/assets/44384286/7e2fe0d6-b755-4a80-89e5-eeee52970db2)
        - ArgoCD Status
        ![](https://github.com/sanchitpathak7/blogsite/assets/44384286/725739e0-bc8e-4539-8307-40208b3574e4)
    - From kubectl perspective, all resources are deployed.
```
$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
db-6d9f87bb9b-4ngfl       1/1     Running   0          2m18s
redis-77fccb7f9-69wgp     1/1     Running   0          2m18s
result-54b5ccfc95-xcbvc   1/1     Running   0          2m18s
vote-5655bd759-8ln2p      1/1     Running   0          2m18s
worker-7dd74bcbbb-d9slw   1/1     Running   0          2m18s
 sanchit@FVFH81G2Q6LW  ~/devopsprojects/azure-arm  kubectl get all 
NAME                          READY   STATUS    RESTARTS   AGE
pod/db-6d9f87bb9b-4ngfl       1/1     Running   0          2m26s
pod/redis-77fccb7f9-69wgp     1/1     Running   0          2m26s
pod/result-54b5ccfc95-xcbvc   1/1     Running   0          2m26s
pod/vote-5655bd759-8ln2p      1/1     Running   0          2m26s
pod/worker-7dd74bcbbb-d9slw   1/1     Running   0          2m26s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/db           ClusterIP   10.0.125.79    <none>        5432/TCP         2m26s
service/kubernetes   ClusterIP   10.0.0.1       <none>        443/TCP          40m
service/redis        ClusterIP   10.0.145.41    <none>        6379/TCP         2m26s
service/result       NodePort    10.0.191.177   <none>        5001:31001/TCP   2m26s
service/vote         NodePort    10.0.65.200    <none>        5000:31000/TCP   2m26s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/db       1/1     1            1           2m26s
deployment.apps/redis    1/1     1            1           2m26s
deployment.apps/result   1/1     1            1           2m26s
deployment.apps/vote     1/1     1            1           2m26s
deployment.apps/worker   1/1     1            1           2m26s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/db-6d9f87bb9b       1         1         1       2m26s
replicaset.apps/redis-77fccb7f9     1         1         1       2m26s
replicaset.apps/result-54b5ccfc95   1         1         1       2m26s
replicaset.apps/vote-5655bd759      1         1         1       2m26s
replicaset.apps/worker-7dd74bcbbb   1         1         1       2m26s
```
Vote U/I Loads as expected:
![](https://github.com/sanchitpathak7/blogsite/assets/44384286/11793893-0a3d-47cf-b03c-9c35f2fbb243)

Now, at this point any change to the Azure Repo resources in `k8s-specifications`, ArgoCD will pickup the change and apply it on the AKS cluster.

- Update operation

Now, we want to see what we need to do to ensure automatic updates to the pipeline and K8s resources if we make changes to vote application code to vote for "Coffee/Tea" instead of "Cats/Dogs".

Before working on changing the pipelines for Update stage, let's ensure our AKS cluster has authentication to pull images from our ACR.
```
$ kubectl create secret docker-registry sanchitazurecicd-acr-secret --docker-server sanchitazurecicd.azurecr.io --docker-username sanchitazurecicd --docker-password=<REDACTED>
secret/sanchitazurecicd-acr-secret created
```
Ensure that the required deployment YAMLs also has the above created secret set in the ImagePullSecret parameter.

Create a `updateK8sManifest.sh` script in the same repo. Example shown below:
```
#!/bin/bash
set -x
# Set the repo URL
REPO_URL="https://<TOKEN>@dev.azure.com/<PROJECT>/voting-app/_git/voting-app"
# Clone the git repo into the /tmp directory
git clone "$REPO_URL" /tmp/temp_repo
# Navigate into the cloned repo directory
cd /tmp/temp_repo
# To change the image tag in a deployment.yaml file. $1 represents the service name. $2 is repository name. $3 is the build tag.
sed -i "s|image:.*|image: <ACR_LOGIN_SERVER>/$2:$3|g" k8s-specifications/$1-deployment.yaml
# Add the modified files
git add .
# Commit the changes
git commit -m "Updated Kubernetes manifest"
# Push the changes back into the repo
git push
# Cleanup: remove the tmp directory
rm -rf /tmp/temp_repo
```

Next, update the vote-service pipeline to include update stage with input as the script file with the arguments.
```
...
- stage: Update
  displayName: Update
  jobs:
  - job: Update
    displayName: Update
    steps:
    - task: ShellScript@2
      inputs:
        scriptPath: 'manifest-update-scripts/updateK8sManifest.sh'
        args: 'vote $(imageRepository) $(tag)'
```
All 3 stages are successful on the new run for vote-service:
![](https://github.com/sanchitpathak7/blogsite/assets/44384286/f724dee1-53e9-412f-a824-0e78dacc67de)

Now, we can see new vote deployment image tag got created in ACR `votingapp:10` and the YAML file is also updated correctly and the new updated pod is running.
```
$ kubectl get pod vote-67b9fcd78b-kpr6s -o yaml | grep -i image
  - image: sanchitazurecicd.azurecr.io/votingapp:10
    imagePullPolicy: Always
```
![](https://github.com/sanchitpathak7/blogsite/assets/44384286/702f25d1-febe-403c-b3e6-dda6a649745c)




### Conclusion
Below picture (not very asthetic) summarizes in short the entire CI/CD workflow that we acheived.
![](https://github.com/sanchitpathak7/blogsite/assets/44384286/5e62429d-532f-4806-95ed-6daa35d9782a)
Hope this was helpful. Thank you for reading till the end!