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

Select 'bhaktiazurecicdregistry' container registry-> Rename Image to 'vote-service'-> 'validate and configure'.

update trigger to
trigger:
 paths:
   include:
     - result/*

update
  # Agent VM image name
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

Follow below commands to establish connection of pool to the pipeline.
Go to 'https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/linux-agent?view=azure-devops&tabs=IP-V4' link.
Go to Azure devops-> agent pool-> add pool -> self hosted-> named(azureagent)-> 
Grant access permission to all pipelines-> Create.

After that go to that created pool-> agent-> new agent-> Linux, and follow below commands.



