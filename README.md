# Main structure of flow

| Action                                 | Triggered In | Result                              	    |
| -------------------------------------- | ------------ | -----------------------------------------|
| You commit code                        | Azure DevOps | Pipeline starts                     	    |
| Pipeline builds & pushes image to ACR  | ACR          | Stores `:123` version               	    |
| Pipeline updates image tag in Git YAML | Git          | `vote-deployment.yaml` has new image tag |
| ArgoCD sees change in Git              | ArgoCD       | Syncs to AKS                             |
| AKS uses image from ACR                | ACR + AKS    | Deployment is rolled out             	   |

<img src="https://github.com/user-attachments/assets/a5297db0-89c1-4e42-85fa-da643755e72e" alt="DevOps CI_CD Process Flow" width="1000"/>

# CI_CD-pipeline-instructions
Create project named 'voting-app' in Azure devops portal.

Import project using 'https://github.com/dockersamples/example-voting-app.git' url

Set main branch as default branch.

Run below commands in powershell to create resources in Azure (provided file should be in same folder where commands are executed.)
<pre>
login
1. az login

create resource group
2. az group create --name azurecicd --location eastus

create container registry
3. az deployment group create --name containerregisterycreate --resource-group azurecicd --template-file create-container-registry.json --parameters create-container-registry-parameters.json
(If ACR is deleted from Azure portal then delete that service connection and create new connection for new acr, then update id in pipeline dockerRegistryServiceConnection: '9bf0bb84-daa2-4791-819e-2feda83dfaa6')

create vm
4. az deployment group create --resource-group azurecicd --template-file azurevm.json --parameters "@azurevm.parameters.json"
</pre>

Go to Azure devops, follow below steps to create pipeline.

Click on 'Create Pipeline'-> choose azure repos git-> choose voting-app-> select a docker image (build & push an image to Azure container registry)-> select subscription and continue.

Select 'bhaktiazurecicdregistry' container registry-> Rename Image to 'result-service'-> 'validate and configure'.

name of yml file = azure-pipelines-result.yml
<pre>
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
</pre>
similarly, create another stage for push
go to setting update command to push then add

then save and run, you will get error of azure agent.

# Follow below commands to establish connection of pool to the pipeline.

Go to 'https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/linux-agent?view=azure-devops&tabs=IP-V4' link.
Go to Azure devops-> agent pool-> add pool -> self hosted-> named(azureagent)-> 
Grant access permission to all pipelines-> Create.

After that go to that created pool-> agent-> new agent-> Linux, and follow below commands.

<pre>
Connect to your linux vm. (follow below commands.)
Open git bash and redirect to the path where your private ssh key is located.
1. chmod 600 azureagent_key (for permission)
2. ssh -i azureagent_key azureuser@74.235.238.150 (to connect vm)
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
</pre>

Now run the pipeline, it should be successful, check from azure devops if pipeline shows that permission is required then grant that.)

Now create another two pipelines following above steps. (no need to create pool again, it can be used in another pipelines also)
<pre>
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
</pre>

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

# After creating kubernetes cluster run below command
<pre>
az aks get-credentials --name azuredevops  --resource-group azurecicd
kubectl get pods (not needed)
go to argo cd-> getting started-> install kubectl (curl.exe -LO "https://dl.k8s.io/release/v1.33.0/bin/windows/amd64/kubectl.exe")-> 
copy install argo cd url and run in cmd (kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)
run-> kubectl get pods -n argocd

Once all pods are in running state then move to config argocd.
kubectl get secrets -n argocd
kubectl edit secret argocd-initial-admin-secret -n argocd
copy the password = dUFLOEhtSzZQUGpDNC1Kbw==


echo dUFLOEhtSzZQUGpDNC1Kbw== | base64 --decode (gitbash-only this command-run others in cmd)
copy output = uAK8HmK6PPjC4-Jo

kubectl get svc -n argocd
kubectl edit svc argocd-server -n argocd
One file will be opened edit ClusterIP to NodePort
kubectl get svc -n argocd (verify this line-> argocd-server    NodePort    10.0.184.76    <none> 80:31029/TCP,443:30995/TCP    68m -->> copy http port = 31029)
kubectl get nodes -o wide (copy external-ip from output =   172.206.194.130)
need to access =  172.206.194.130:31029

to give permission to the port
go to azure portal-> vmms-> Instances-> go into instance-> networking settings-> create port rule-> inbound port rule-> change Destination port ranges to 31029 -> Add.)

access 172.191.85.172:31029 from browser
provide
username = admin
password = uAK8HmK6PPjC4-Jo

copy saved personal access token(PAT) = 8gGwcmbSYkV6rzKw1KM0vDtPNjOZ0kjyGA6ASkNrk9XE8Fplqxf4JQQJ99BEACAAAAAAAAAAAAASAZDO1lrh
in argocd-> settings-> repositories-> connect repo-> Choose your connection method: via http/https -> type=git, project=default,
for repo url -> get it from azure devops portal from clone via http = https://bhaktiraval18112001@dev.azure.com/bhaktiraval18112001/voting-app/_git/voting-app


replace bhaktiraval18112001 to PAT = 8gGwcmbSYkV6rzKw1KM0vDtPNjOZ0kjyGA6ASkNrk9XE8Fplqxf4JQQJ99BEACAAAAAAAAAAAAASAZDO1lrh
final url = https://8gGwcmbSYkV6rzKw1KM0vDtPNjOZ0kjyGA6ASkNrk9XE8Fplqxf4JQQJ99BEACAAAAAAAAAAAAASAZDO1lrh@dev.azure.com/bhaktiraval18112001/voting-app/_git/voting-app
Click on connect (status should be successful)
</pre>

Go to Applications -> new app -> Application Name=voteapp-service, project name=default, SYNC POLICY=automatic, repo url will be provided, path=k8s-specifications, cluster url-provided, Namespace=default -> Create


In repo create new folder=scripts, file name = updateK8sManifests.sh
paste below script in that file


<pre>
#!/bin/bash
set -x
# Set the repository URL
REPO_URL="https://8gGwcmbSYkV6rzKw1KM0vDtPNjOZ0kjyGA6ASkNrk9XE8Fplqxf4JQQJ99BEACAAAAAAAAAAAAASAZDO1lrh@dev.azure.com/bhaktiraval18112001/voting-app/_git/voting-app"
# Clone the git repository into the /tmp directory
rm -rf /tmp/temp_repo
git clone "$REPO_URL" /tmp/temp_repo
# Navigate into the cloned repository directory
cd /tmp/temp_repo
# Make changes to the Kubernetes manifest file(s)
# For example, let's say you want to change the image tag in a deployment.yaml file
sed -i "s|image:.*|image: bhaktiazurecicdregistry.azurecr.io/$2:$3|g" k8s-specifications/$1-deployment.yaml
# Add the modified files
git add .
# Commit the changes
git commit -m "Update Kubernetes manifest"
# Push the changes back to the repository
git push
# Cleanup: remove the temporary directory
rm -rf /tmp/temp_repo
</pre>




In vote-service pipeline -> add new stage (task=shellscript@2) -> settings -> script path = scripts/updateK8sManifests.sh, Arguments = vote $(imageRepository) $(tag)

code of new stage is as below
 <pre>
  - stage: Update
  displayName: Update
  jobs:
  - job: Update
    displayName: Update
    steps:
    - task: ShellScript@2
      inputs:
        scriptPath: 'scripts/updateK8sManifests.sh'
        args: 'vote $(imageRepository) $(tag)'
 </pre>

<pre>
(uploading script from machine is recommended and When uploading scripts do below steps)
Use VS Code or another code editor.
In VS Code (bottom-right), set line endings to LF before saving.
Ensure encoding is UTF-8.


If you copy and paste the script then -> login into vm/agent -> then run below commands (To prevent this extra work and other errors, upload from machine.)
sudo apt install dos2unix
cd /home/azureuser/myagent/_work/2/s/scripts (can get from pipeline inspection)
dos2unix updateK8sManifests.sh
bash updateK8sManifests.sh vote votingapp 44
</pre>


run below commands in cmd
<pre>
kubectl edit cm argocd-cm -n argocd
in opened file add below data at last and save
data:
 timeout.reconciliation: 10s
 run - kubectl get pods (noticed vote-5768cb6584-z4fjh     0/1     ImagePullBackOff     0     15m)
 got azure container registry -> Setting -> access keys -> enable admin user -> get username=bhaktiazurecicdregistry and password=f9ElSFPk5V8ywCR3Jvgqa1jJHy3Lbg0sacOEvS4u26+ACRDH2jES)
use below command and add required parameters and run in cmd
kubectl create secret docker-registry <secret-name> \
    --namespace <namespace> \
    --docker-server=<container-registry-name>.azurecr.io \
    --docker-username=<service-principal-ID> \
    --docker-password=<service-principal-password>
 
kubectl create secret docker-registry acr-secret --namespace default --docker-server=bhaktiazurecicdregistry.azurecr.io --docker-username=bhaktiazurecicdregistry --docker-password=f9ElSFPk5V8ywCR3Jvgqa1jJHy3Lbg0sacOEvS4u26+ACRDH2jES  (AKS uses these credentials to authenticate to ACR)
(can get reference from kubernetes docs image pull secrets)
</pre>
Edit k8s-specifications/vote-deployment.yaml -> spec part file as below
<pre>
    spec:
      containers:
      - image: bhaktiazurecicdregistry.azurecr.io/votingapp:32
        name: vote
        ports:
        - containerPort: 80
          name: vote
      imagePullSecrets:
      - name: acr-secret
</pre>
Now to verify working change vote -> app.py ->
<pre>
from
option_a = os.getenv('OPTION_A', "Cats")
option_b = os.getenv('OPTION_B', "Dogs")
to
option_a = os.getenv('OPTION_A', "Rain")
option_b = os.getenv('OPTION_B', "Snow")
pipeline will be triggered.
 </pre>

Verify process by running below commands
<pre>
kubectl get deploy vote -o yaml (containers: - image: bhaktiazurecicdregistry.azurecr.io/votingapp:33) - latest deployed image will be shown 
kubectl get pods (all pods should be in running state)
 </pre>

run below commands to get final url where project is deployed
<pre>
kubectl get svc (vote         NodePort    10.0.69.180    <none>        8080:31000/TCP   3h20m) (get port number of vote service = 31000)
kubectl get node -o wide (get external ip address =   172.206.194.130  )
final url = 172.206.194.130:31000
(need to open 31000 port)
go to vmss -> instances -> network settings -> create port rule-> inbound port rule-> change Destination port ranges to 31000 -> Add.
</pre>
Now at  172.206.194.130:31000 we can see updated portal.




# Advance
🎯 Objective:
Pause your pipeline before deploying to production (or a sensitive stage)
Manually approve (or timeout after a day)
In our case -->> Pause the pipeline after Push, request manual approval, and then proceed to Update. (Approval only works for deployment jobs, so in our pipeline we have updated 'Update job' as deployment job)

🔹 Step 1: Create the Environment in Azure DevOps
Go to your Azure DevOps project
Navigate to: Pipelines → Environments
Click ➕ New Environment
Enter:
Name: approval-env
Resource Type:
Select None (for approval only)
Or choose Kubernetes to link an AKS namespace
Click Create

🔹 Step 2: Add Approval to the Environment
Go to Pipelines → Environments → approval-env
Click ... → Approvals and checks → + Add
Select Approvals
Configure:
Approvers: yourself or team members
Timeout: set to 1 day (or your preference)
Optionally enable requestor cannot approve
Click Create

🔹 Step 3: Update the YAML Pipeline
Update your pipeline YAML to associate the Update stage with the environment:
<pre>
 - stage: Update
  displayName: Update
  dependsOn: Push
  jobs:
  - deployment: UpdateK8s
    displayName: Update Kubernetes Manifests
    environment:
      name: 'approval-env'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: ShellScript@2
            displayName: Update Manifests
            inputs:
              scriptPath: 'scripts/updateK8sManifests.sh'
              args: 'vote $(imageRepository) $(tag)'
</pre>


<ul>
<li>If you use only dependsOn: Build without specifying a condition, the stage runs after Build finishes, even if Build failed.</li>
<li>If you use dependsOn: Build and condition: succeeded(), it runs only if Build succeeded.</li>
<li>Approval is tied to the environment on the deployment job, not the entire stage.</li>
<li>If your stage has multiple jobs, only the deployment job referencing the environment waits for approval.</li>
<li>If your stage has just one deployment job targeting the environment, effectively the whole stage is gated by the approval.</li>
<li>If reques is not approved in configured timeline then that stage will be skipped.</li>
</ul>












