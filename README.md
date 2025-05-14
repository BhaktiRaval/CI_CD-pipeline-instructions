# CI_CD-pipeline-instructions
Create project named 'voting-app' in Azure devops portal.

Import project using 'https://github.com/dockersamples/example-voting-app.git' url

Set main branch as default branch.

Run below commands in powershell to create resources in Azure (provided file should be in same folder where commands are executed.)

login
1. az login

create resource group
2. az group create --name azurecicd --location centralindia

create container registry
3. az deployment group create --name containerregisterycreate --resource-group azurecicd --template-file create-container-registry.json --parameters create-container-registry-parameters.json

create vm
4. az deployment group create --resource-group azurecicd --template-file azurevm.json --parameters "@azurevm.parameters.json"

Go to Azure devops, follow below steps to create pipeline.

Click on 'Create Pipeline'-> choose azure repos git-> choose voting-app-> select a docker image (build & push an image to Azure container registry)-> select subscription and continue.

# Select 'bhaktiazurecicdregistry' container registry-> Rename Image to 'result-service'-> 'validate and configure'.

name of yml file = azure-pipelines-result.yml

update trigger to
trigger:
 paths:
   include:
     - result/*

imageRepository: 'resultapp'

update
  Agent VM image name
  vmImageName: 'ubuntu-latest'
  to
  pool:
  name: 'azureagent'

  update
  displayName: Build and displayName: Build the image
  remove
      pool:
      vmImage: $(vmImageName)
go to setting and update below data then add
update command to build
update Dockerfile to result/Dockerfile

similarly, create another stage for push
go to setting update command to push then add

then save and run, you will get error of azure agent.

# Follow below commands to establish connection of pool to the pipeline.
Go to 'https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/linux-agent?view=azure-devops&tabs=IP-V4' link.
Go to Azure devops-> agent pool-> add pool -> self hosted-> named(azureagent)-> 
Grant access permission to all pipelines-> Create.

After that go to that created pool-> agent-> new agent-> Linux, and follow below commands.

Connect to your linux vm. (follow below commands.)
Open git bash and redirect to the path where your private ssh key is located.
1. chmod 600 azureagent_key (for permission)
2. ssh -i azureagent_key azureuser@4.247.140.83 (to connect vm)
3. Enter password for vm

In connectd vm run below commands (Can be find when try to create new agent)
1. mkdir myagent && cd myagent
2. wget https://download.agent.dev.azure.com/agent/4.255.0/vsts-agent-linux-x64-4.255.0.tar.gz (url of Download the agent)
3. sudo apt update (only 1st time to update apt repositories)
4. tar zxvf vsts-agent-linux-x64-4.255.0.tar.gz (extract tar file)
5. sudo apt install docker.io (install docker)
6. sudo usermod -aG docker azureuser (give permission of azure agent to docket daemon)
7. logout
8. ssh -i azureagent_key azureuser@4.247.140.83 (provide password)
9. sudo systemctl restart docker
10. docker pull hello-world (verify image is pulled)
11. cd myagent
12. ./config.sh
  1 (provide server url - from document https://dev.azure.com/{yourorganization} -> https://dev.azure.com/bhaktiraval18112001 )
  2 provide personal access token (get it from User settings-> personal access token-> create new one (name=azureagent, give full access) and save token-will be needed in future) (name of agent pool - ex: azureagent) (name of agent - ex: azureagent)
13. ls (verify that config.sh exist) 
14. ./config.sh
15. ./run.sh

Now run the pipeline, it should be successful, check from azure devops if pipeline shows that permission is required then grant that.)

Now create another two pipelines following above steps. (no need to create pool again, it can be used in another pipelines also)
name of yml file = azure-pipelines-vote.yml
in include = vote/*
imageRepository: 'voteapp'
Docketfile = vote/Dockerfile
name of pipeline = vote-service

name of yml file = azure-pipelines-worker.yml
in include = worker/*
imageRepository: 'workerapp'
Docketfile = worker/Dockerfile
name of pipeline = worker-service

This pipeline will be failed because of some parameters.
change worker->Dockerfile as below
--platform=${BUILDPLATFORM} to --platform=linux
RUN dotnet restore -a $TARGETARCH to RUN dotnet restore
RUN dotnet publish -c release -o /app -a $TARGETARCH --self-contained false --no-restore to RUN dotnet publish -c release -o /app --self-contained false --no-restore


After that this last pipeline should run successfully.


# Kubernetes

Go to kubernetes services-> Create Kubernetes cluster
Kubernetes cluster name = azuredevops, Zones 1
-> next
 go to agent pool and change Minimum node count = 1, Maximum node count = 2, Max pods per node = 30, Enable public IP per node-enabled -> update (if got any quota error then change the region)
 1. k8s cluster login
 2. ArgoCD
 3. Configure ArgoCD
 4. Update (shell) -> Repo

After creating kubernetes cluster run below command
az aks get-credentials --name azuredevops  --resource-group azurecicd
kubectl get pods (not needed)
go to argo cd-> getting started-> install kubectl (curl.exe -LO "https://dl.k8s.io/release/v1.33.0/bin/windows/amd64/kubectl.exe")-> copy install argo cd url and run in cmd.
run-> kubectl get pods -n argocd

Once all pods are in running state then move to config argocd.
kubectl get secrets -n argocd
kubectl edit secret argocd-initial-admin-secret -n argocd
copy the password = UkU3RkZUUlBVbmZzd0hEcQ==
echo UkU3RkZUUlBVbmZzd0hEcQ== | base64 --decode (gitbash-only this command-run others in cmd)
copy output = RE7FFTRPUnfswHDq
kubectl get svc -n argocd
kubectl edit svc argocd-server -n argocd
One file will be opened edit ClusterIP to NodePort
kubectl get svc -n argocd (verify this line-> argocd-server  NodePort    10.0.168.192   <none>    80:32521/TCP,443:32074/TCP   55m -->> copy http port = 32521)
kubectl get nodes -o wide (copy external-ip from output =  172.190.136.83)
need to access = 172.190.136.83:32521

to give permission to the port
go to azure portal-> vmms-> Instances-> go into instance-> networking settings-> create port rule-> inbound port rule-> change Destination port ranges to 32521-> Add.)

access 172.190.136.83:32521 from browser
provide
username = admin
password = RE7FFTRPUnfswHDq
copy saved personal access token(PAT) = 8gGwcmbSYkV6rzKw1KM0vDtPNjOZ0kjyGA6ASkNrk9XE8Fplqxf4JQQJ99BEACAAAAAAAAAAAAASAZDO1lrh
in argocd-> settings-> repositories-> connect repo-> Choose your connection method: via http/https -> type=git, project=default,
for repo url -> get it from azure devops portal from clone via http = https://bhaktiraval18112001@dev.azure.com/bhaktiraval18112001/voting-app/_git/voting-app
replace bhaktiraval18112001 to PAT = 8gGwcmbSYkV6rzKw1KM0vDtPNjOZ0kjyGA6ASkNrk9XE8Fplqxf4JQQJ99BEACAAAAAAAAAAAAASAZDO1lrh
final url = https://8gGwcmbSYkV6rzKw1KM0vDtPNjOZ0kjyGA6ASkNrk9XE8Fplqxf4JQQJ99BEACAAAAAAAAAAAAASAZDO1lrh@dev.azure.com/bhaktiraval18112001/voting-app/_git/voting-app
Click on connect (status should be successful)

Go to Applications -> new app -> Application Name=voteapp-service, project name=default, SYNC POLICY=automatic, repo url will be provided, path=k8s-specifications, cluster url-provided, Namespace=default -> Create


In repo create new folder=scripts, file name = updateK8sManifests.sh
paste below script in that file



#!/bin/bash

set -x

# Set the repository URL
REPO_URL="https://8gGwcmbSYkV6rzKw1KM0vDtPNjOZ0kjyGA6ASkNrk9XE8Fplqxf4JQQJ99BEACAAAAAAAAAAAAASAZDO1lrh@dev.azure.com/bhaktiraval18112001/voting-app/_git/voting-app"

# Clone the git repository into the /tmp directory
git clone "$REPO_URL" /tmp/temp_repo

# Navigate into the cloned repository directory
cd /tmp/temp_repo

# Make changes to the Kubernetes manifest file(s)
# For example, let's say you want to change the image tag in a deployment.yaml file
sed -i "s|image:.*|image: bhaktiazurecicdregistry/$2:$3|g" k8s-specifications/$1-deployment.yaml

# Add the modified files
git add .

# Commit the changes
git commit -m "Update Kubernetes manifest"

# Push the changes back to the repository
git push

# Cleanup: remove the temporary directory
rm -rf /tmp/temp_repo




In vote-service pipeline -> add new stage (task=shellscript@2) -> settings -> script path = scripts/updateK8sManifests.sh, Arguments = vote $imageRepository $tag











   





