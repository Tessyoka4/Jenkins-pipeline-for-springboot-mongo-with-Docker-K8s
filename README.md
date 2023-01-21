# Jenkins-pipeline-for-springboot-mongo-with-Docker-K8s
This is a jenkins pipeline project for a spring-boot-mongo application with Docker &amp; K8s integrated to deploy the application.

# Pre-requisites
AWS account/
Github Repository/
Docker Repository/
IDE: VS Code

# Step-1: Provision Jenkins Server on AWS
Launch an Ec2 Ubuntu instance (T2 medium) and for SG, open ports 22 (allow ssh traffic) and port 8080. We will install Jenkins & Docker on the server using bootstrap with userdata script to install docker or SSH into the instance and some commands to install jenkins and docker on the server.

# Step-2: Provision K8s Master Node & Worker Nodes Servers
SSH into the servers and configure k8s.
  
# Step-3: Create a GitHub private repository (to store source code files)
SSH into jenkins server and clone repo to jenkins server.
  git clone url copied from github.
  copy files (app codes) to repo or upload files to github

# Step-4: Create docker file (build instruction required to build the image)

# Step-5: Create Manifest File (for deployment of the application)  

Step-9: Configure Jenkins server
Connect to the jenkins server http://<jenkins-server public ip>:8080.

Connect Jenkins server with SSH and take the initial password with the following commands:

ssh -i FirstKey.pem ec2-user@<jenkins server public ip>
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Use initial password in http://<jenkins-server public ip>:8080

Configure jenkins server. (create user > install suggested plugins > start jenkins)

Learn Ansible and Terraform's executable path from the instance. We use them in jenkins plugins.

which ansible         # output: /usr/local/bin/ansible
which terraform       # output: /usr/bin/terraform
Go to the http://<jenkins-server public ip>:8080.

Add the Ansible and Terraform plugins. 

Follow these steps manage jenkins - manage plugins - add ansible and terraform plugins (locale plugin if it is necessary) - global tool configuration.

)

Step-10: Add credentials to Jenkins
We will go to the Manage Jenkins --> Manage credentials and add github token first.

Type: username with password
username: ec2-user
password: paste your TOKEN
ID: github-token
Description: github-token
Similarly we will add our private key that we use to create Jenkins server as Global credentials.

Type: SSH key with username
ID: ansible-key # same id will be used in Jenkinsfile
Description: ansible-key
user: ec2-user
Private Key: <Paste content of your PEM file downloaded locally after creating a keypair in AWS>
Step-11: Create Jenkinsfile under repo
We will create a Jenkinsfile now with the content given under todo-app folder which will deploy our application by using Terraform and Ansible.

We can commit/push all files we have created under our repository now.

Step-12: Create Pipeline Job in Jenkins
Create pipeline job in Jenkins with below properties.

Name: todo-app-pipeline
Kind: pipeline
GitHub project: <url_of_your_todo_app_repo>
Pipeline: Pipeline script from SCM
SCM: Git
Repo URL: <url_of_your_todo_app_repo>
Credentials: github-token
Branch: */main
Path: Jenkinsfile
Step-13: Build job and test application from browser
It is time to test our pipeline. Click Build Job.We can check resources from AWS Console. Our nodes for 3-tier of application is created.
