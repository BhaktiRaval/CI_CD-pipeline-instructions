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
