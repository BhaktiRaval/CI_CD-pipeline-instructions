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
5. ./config.sh
  5.1 (provide server url - from document https://dev.azure.com/{yourorganization} -> https://dev.azure.com/bhaktiraval18112001 )
  5.2 provide personal access token (get it from User settings-> personal access token-> create new one (name=azureagent, give full access) and save token) (name of agent pool - ex: azureagent) (name of agent - ex: azureagent)
6. ls (verify that config.sh exist)
7. sudo apt install docker.io (install docker)
8. sudo usermod -aG docker azureuser (give permission of azure agent to docket daemon)
10. logout
11. ssh -i azureagent_key azureuser@4.247.140.83 (provide password)
12. sudo systemctl restart docker
13.  docker pull hello-world (verify image is pulled)
14.  cd myagent
15. ./config.sh
16. ./run.sh

Now run the pipeline, it should be successful, check from azure devops if pipeline shows that permission is required then grant that.)

Now create another two pipelines following above steps. (no need to create pool again, it can be used in another pipelines also)
name of yml file = azure-pipelines-vote.yml
in include = vote/*
Docketfile = vote/Dockerfile
name of pipeline = vote-service

name of yml file = azure-pipelines-vote.yml
in include = vote/*
Docketfile = vote/Dockerfile
name of pipeline = vote-service
   





