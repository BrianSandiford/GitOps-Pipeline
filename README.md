
# GitOps-Pipeline
In this project we will build a GitOps Pipeline.The Pipeline has two repos.A Code repo and and a Deployment repo.The code repo houses the movie site app.Code repo can be found [here](https://github.com/BrianSandiford/moviesiteapp).When a change is made to the code repo( a git push) it triggers the Jenkin CI pipeline.Jenkins builds the Docker image ,pushes it to Amazon ECR (Amazon Elastic Container Registry (ECR) ) and also updates the image tag in the Deployment repo.Deployment repo can be found [here](https://github.com/BrianSandiford/moviesiteapp-helmcharts).ArgoCD listens to the Deployment repo and once the change in image tag is noticed syncs with the kubernetes environmemt ( Amazon Elastic Kubernetes Service (EKS)).
![gitop-pipeline](https://user-images.githubusercontent.com/67350852/178124160-9e8067a0-df59-4a85-b355-09939f4cbd93.jpeg)


To replicate the movie site app in the code repo you would need an API key from the TheMovieDB site -https://developers.themoviedb.org/3/getting-started/introduction .After you have received the API key use it to generate a Secret in Kubernetes.Instructions on how to create a Secret from a config file can be found [here](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-config-file/).Secret config file stored as secret.yaml in the templates directory of the deployment repo.

A secret will also be needed to pull the image from the private registry on Amazon ECR . `aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 655895384845.dkr.ecr.us-east-2.amazonaws.com` would allow you to log into the registry and generate the file /root/.docker/config.json (path may vary).This file contains the authentication to the private registry.
` cat /root/.docker/config.json  | base64` would generate the base 64 encoded value of the config.json file.This value is used as the value for the key `.dockercfg:` in the secret1.yaml file in the deployment repo.Instructions on creating Docker config Secrets [here](https://kubernetes.io/docs/concepts/configuration/secret/#docker-config-secrets)

## Create EC2 instance

##  Install Jenkins on AWS EC2
Instructions on installing Jenkins on AWS EC2 can be found [here](https://github.com/yankils/Simple-DevOps-Project/blob/master/Jenkins/Jenkins_Installation.MD#install-jenkins-on-aws-ec2). 

Select "install suggested pluggins " after installation.

Install git plugin on Jenkins.Instructions [here](https://github.com/yankils/Simple-DevOps-Project/blob/master/Jenkins/Git_plugin_install.MD)

Install Docker Pipeline Pluggin in Jenkins.

Install CloudBees Docker Build and Publish plugin.

Install Pipeline pluggin.

## Install Docker on AWS EC2

Instructions on installing Docker on AWS EC2 can be found [here](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/docker-basics.html)

Add your jenkins user to the docker group:
``` 
sudo usermod -aG docker jenkins
```
Then restart your jenkins server to refresh the group.

## Create Kubernetes (K8s) Cluster on AWS

**Note well** must be root user

1. install AWSCLI.Instructions on installing AWSCLI version 2 can be found [here](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)

2. Login to AWS 

``` 
# Access your "My Security Credentials" section in your profile. 
# Create an access key


# Regions
https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html

```

3. Install kubectl on EC2 instance 
Instructions on installing kubectl can be found [here](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

4. Deploy Cluster with EKS CLI  https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html 
## Install EKS CTL

Instructions on installing eksctl can be found [here](https://docs.amazonaws.cn/en_us/eks/latest/userguide/eksctl.html)

Set the path as below :
```
export PATH=$PATH:/usr/local/bin/
```
```
eksctl create cluster --name getting-started-eks \
--region us-east-2 \
--version 1.22 \
--managed \
--node-type t2.medium \
--nodes 1 \
--node-volume-size 200 
```
**must run updated version of kubectl** --version 1.22

Cleanup
```
eksctl delete cluster --name getting-started-eks
```

## Create a private repository on AWS ECR

Instructions on how to create a private repository on AWS ECR can be found  [here](https://www.youtube.com/watch?v=vWSRWpOPHws)

## Define Jenkins Pipeline
- Define Jenkins Pipeline in SCM.Instructions [here](https://www.jenkins.io/doc/book/pipeline/getting-started/#through-the-classic-ui)
.
- In the Jenkinsfile within the **Code** repo the field **git-pass-credentials-ID** needs to be defined as as a Username with password credential in Jenkins `withCredentials([usernameColonPassword(credentialsId: 'git-pass-credentials-ID', variable: 'USERPASS')])'` 
.
- `aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 655895384845.dkr.ecr.us-east-2.amazonaws.com` derived from the View push commands tab in the Amazon ECR
## Install Argo CD
Instructions on installing Argo CD can be found [here](https://argo-cd.readthedocs.io/en/stable/getting_started/)

Under step 6. Create An Application From A Git Repository replace example repo https://github.com/argoproj/argocd-example-apps.git with Code repo 
https://github.com/BrianSandiford/moviesiteapp-helmcharts.git

Give your app the name moviesiteapp instead of guestbook

Change SYNC POLICY to automatic

CHECK PRUNE RESOURCES and SELF HEAL check boxes

![selfheal](https://user-images.githubusercontent.com/67350852/123559741-ed70d780-d76b-11eb-9903-a4025ed3f553.JPG)

PATH would simply be a period '.'

![path](https://user-images.githubusercontent.com/67350852/123559906-fada9180-d76c-11eb-8218-8d492fa75fee.JPG)

`kubectl -n argocd get all` should provide the server IP/hostname	`argocd login <ARGOCD_SERVER>`

Server hostname hightlighted in yellow below
![argo-server](https://user-images.githubusercontent.com/67350852/123556084-53069900-d757-11eb-9576-b4a9fb278f04.JPG)

## Webhooks
Confgure webhook in your github repository. Example [here](https://docs.github.com/en/developers/webhooks-and-events/webhooks/creating-webhooks#setting-up-a-webhook)

### Create a build triger in Jekins

Select pipeline

![pipline-jenkins](https://user-images.githubusercontent.com/67350852/182642105-56d1ce96-61b2-4b99-a070-5e004e92a335.JPG)

Select configure

![pipeline-configure](https://user-images.githubusercontent.com/67350852/182642828-46762d32-3501-4abb-86f4-c9884a920338.JPG)

Select build triggers and tick check box "GitHub hook trigger for GITScm polling"

![image](https://user-images.githubusercontent.com/67350852/182643668-addf3a60-a035-4d8b-b21e-8a0aba556547.png)

Under Pipeline Select Pipeline script from SCM ,Git and enter Repository URL as shown below

![image](https://user-images.githubusercontent.com/67350852/182644935-82c39e5f-6315-417a-ac6a-53c70a480723.png)

## SSH integration between Jenkins and GitHub

As you can no longer authenticate to github with a password SSH keys are used whenever you push to your git repo.

Jenkins must be up and running and Credentials plug-in installed in Jenkins

Create SSH keys in your Jenkins EC2 instance `ssh-keygen`

`sudo cat ~/.ssh/id_rsa.pub` then copy public key.Paste key into your git repo

Add public keys into the --> settings--> Deploy keys section of the https://github.com/BrianSandiford/moviesiteapp-helmcharts.git repo

![deploy-keys1](https://user-images.githubusercontent.com/67350852/182867139-027fa8a8-f3c5-4b6c-b450-014206540974.JPG)

**Note well** Allow write access must be ticked


![deploy-keys3](https://user-images.githubusercontent.com/67350852/182871170-81c0215b-8ab9-470b-be22-6b4b2cbb842d.JPG)


Click on Add Deploy Key ,enter public keys and save.

![deploy-keys2](https://user-images.githubusercontent.com/67350852/182869511-1d261429-8159-418f-b433-f4dc0dadf47a.JPG)

Add **private key** to jenkins. Instructions can be found [here](https://www.jenkins.io/doc/book/using/using-credentials/)

Login Jenkins. Click on Manage Jenkins then click on Manage Credentials

![jenkins-ssh1](https://user-images.githubusercontent.com/67350852/183105820-1b29a658-c785-451f-b9a9-7d89bec909a2.JPG)

![jenkins-ssh2](https://user-images.githubusercontent.com/67350852/183106480-8b22afa7-8bf2-4047-a518-809f561bd4ed.JPG)

Click on Jenkins click on Global Credentials then click on Add Credentials

![image](https://user-images.githubusercontent.com/67350852/183107797-a65e2fea-fee9-4ecd-a630-2b9d9b332ea0.png)

![image](https://user-images.githubusercontent.com/67350852/183109228-16f355c4-6335-4060-87d4-809881f87577.png)

![image](https://user-images.githubusercontent.com/67350852/183109391-2ad37653-ded0-4f5f-80e8-d41493a5b50f.png)

Choose SSH username with private key.Username can be anything

![image](https://user-images.githubusercontent.com/67350852/183110152-7cd2b452-44ab-4bcb-87b5-a82191699370.png)

Click on enter directly under private key option and Click Add.Copy and paste **private key(not public key)** of your from Jenkins instance.Click OK to save.

![image](https://user-images.githubusercontent.com/67350852/183122647-c017b766-a059-4010-b26a-512eb198597b.png)





## Finally
A git push should trigger the build.

![argocd-gitops](https://user-images.githubusercontent.com/67350852/124676715-0c1e4f00-de8d-11eb-9d24-1be0c68a1015.png)


![jenkins-gitops](https://user-images.githubusercontent.com/67350852/124677332-37ee0480-de8e-11eb-8fa9-df2d0a4d7888.JPG)
