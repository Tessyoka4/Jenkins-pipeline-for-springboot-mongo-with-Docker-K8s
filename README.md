# Jenkins-pipeline-for-springboot-mongo-with-Docker-K8s
This is a jenkins pipeline project for a spring-boot-mongo application with Docker &amp; K8s integrated to deploy the application.


# Pre-requisites
AWS account
Github Repository
Docker Repository
IDE: VS Code

# Step-1: Provision Jenkins Server on AWS
Launch an Ec2 Ubuntu instance (T2 medium) and for SG, open ports 22 (allow ssh traffic) and port 8080. We will install Jenkins & Docker on the server using bootstrap with userdata script to install docker or SSH into the istance and run the commands below to install jenkins and docker.

# install Jenkins
a) Update system package
   sudo apt update 
b) Install Java Version 8 or 11
   sudo apt install openjdk-11-jdk -y
c) Confirm java is installed
   java -version
d) Add Jenkins repo to Ubuntu by importing GPG key to verify package integrity & then add Jenkins repo to source list
   curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc &gt; /dev/null
   echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list &gt; /dev/null
e) Update repo again and install jenkins
   sudo apt update
   sudo apt install jenkins -y
   sudo systemctl status jenkins
   sudo systemctl enable --now jenkins # run this command if jenkins service is not running or active
f) Modify firewalls to allow jenkins port if not opened
   sudo ufw allow 8080
   sudo ufw status
   sudo ufw enable # run this command if status is inactive 
g) open Jenkins ip on web browser to set up jenkins
   http://ip_address:8080
h) cat the path seen on the page to unlock jenkins password
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
i) Enter the password output in the Administrator password field and click Continue. Install suggested pluggins and create first admin by entering credentials you want    to use as jenkins admin. set up instance configuration, save and start using jenkins.

 # install Docker on Jenkins
 a) Install Docker
    curl -fsSL get.docker.com | /bin/bash  
 b) Add Jenkins User to docker group  
    sudo usermod -aG docker jenkins  
 c) Restart Jenkins
    sudo systemctl restart jenkins

# Step-2: Provision K8s Master Node & Worker Nodes Servers
SSH into the servers and run the following commands on the CLI of both servers
sudo apt-get update -y  
sudo apt-get install -y apt-transport-https  
sudo su -  

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list  
deb https://apt.kubernetes.io/ kubernetes-xenial main  
EOF  

 # Note: when you paste the above line wait a while before hitting the enter key 
apt-get update -y  
Disable swap memory
swapoff -a  
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab 

# Enable IP memories:
modprobe br_netfilter
sysctl -p  
sudo sysctl net.bridge.bridge-nf-call-iptables=1  

# Add ubuntu user to the docker group
apt  install docker.io -y

This is optional
usermod -aG docker ubuntu  

systemctl restart docker
systemctl enable docker.service  

# Install k8s modules:
apt-get install -y kubelet kubeadm kubectl kubernetes-cni  

systemctl daemon-reload
systemctl start kubelet   
systemctl enable kubelet.service  

# Execute below commands only in master node as root user.  
kubeadm init  
Exit root user (exit) & execute as normal user  
mkdir -p $HOME/.kube  
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config  
sudo chown $(id -u):$(id -g) $HOME/.kube/config  

kubectl get nodes
kubectl get pods --all-namespaces

kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version |  base64 | tr -d '\n')"  

kubectl get nodes  
kubectl get pods --all-namespaces 
  
# on worker nodes, exit as root user and run commands to join worker node to master node.
kubectl get nodes  
kubectl get pods --all-namespaces 
  
# Step-3: Create a GitHub private repository (to store source code files)
SSH into jenkins server and clone repo to jenkins server
  git clone url copied from github
  copy files (app codes) to repo or upload files to github

# Step-4: Create docker files 
Dockerfile for Postgres
Create a dockerfile-postgresql with below content under postgresql directory.

FROM postgres
COPY ./postgresql/init.sql /docker-entrypoint-initdb.d/
EXPOSE 5432
Dockerfile for React
Create a dockerfile-react with below content under react directory.

FROM node:14
# Create app directory
WORKDIR /app
COPY ./react/client/package*.json ./
RUN yarn install
# copy all files into the image
COPY ./react/client/ .
EXPOSE 3000
CMD ["yarn", "run", "start"]
Dockerfile for Nodejs
Create a dockerfile-nodejs with below content under nodejs directory.

FROM node:14
# Create app directory
WORKDIR /usr/src/app
COPY ./nodejs/server/package*.json ./
RUN npm install
# If you are building your code for production
# RUN npm ci --only=production

# copy all files into the image
COPY ./nodejs/server/ .
EXPOSE 5000
CMD ["node","app.js"]
Step-5: Create Ansible config and Dynamic Inventory file
We will create ansible.cfg file under our repository.

[defaults]
host_key_checking=False
inventory=inventory_aws_ec2.yml
deprecation_warnings=False
interpreter_python=auto_silent
remote_user=ec2-user
We will also create a dynamic inventory file in our repository. Jenkins will create our application servers with terraform, aws will identify those servers with the given tags and add to our dynamic inventory.

plugin: aws_ec2
regions:
  - "us-east-1"
filters:
  tag:stack: ansible_project
keyed_groups:
  - key: tags.Name
  - key: tags.environment
compose:
  ansible_host: public_ip_address
Step-6: Create Terraform file to create 3 nodes for each each tier of application
We will create 3 servers with Jenkins using the terraform we have installed while provisioning Jenkins server. Create main.tf with the content given under todo-app folder.



    # we may need to uninstall any existing docker files from the centos repo first.
    - name: Remove docker if installed from CentOS repo
      yum:
        name: "{{ item }}"
        state: removed
      with_items:
        - docker
        - docker-client
        - docker-client-latest
        - docker-common
        - docker-latest
        - docker-latest-logrotate
        - docker-logrotate
        - docker-engine

    - name: Install yum utils
      yum:
        name: "{{ item }}"
        state: latest
      with_items:
        - yum-utils
        - device-mapper-persistent-data
        - lvm2
        - unzip

    - name: Add Docker repo
      get_url:
        url: https://download.docker.com/linux/centos/docker-ce.repo
        dest: /etc/yum.repos.d/docer-ce.repo

    - name: Install Docker
      package:
        name: docker-ce
        state: latest

    - name: Install pip
      package:
        name: python3-pip
        state: present
        update_cache: true

    - name: Install docker sdk
      pip:
        name: docker

    - name: Add user ec2-user to docker group
      user:
        name: ec2-user
        groups: docker
        append: yes

    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: yes

    - name: install aws cli
      get_url:
        url: https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip
        dest: /home/ec2-user/awscliv2.zip

    - name: unzip zip file
      unarchive:
        src: /home/ec2-user/awscliv2.zip
        dest: /home/ec2-user
        remote_src: True

    - name: run the awscli
      shell: ./aws/install

    - name: log in to AWS ec2-user
      shell: |
        export PATH=/usr/local/bin:$PATH
        source ~/.bash_profile
        aws ecr get-login-password --region {{ aws_region }} | docker login --username AWS --password-stdin {{ ecr_registry }}

- name: postgre database config
  hosts: _ansible_postgresql
  become: true
  vars:
    postgre_container: /home/ec2-user/postgresql
    container_name: rumeysa_postgre
    image_name: <your_AWS_account_number>.dkr.ecr.us-east-1.amazonaws.com/rumeysa-repo/todo-app:postgr
  tasks:
    - name: remove {{ container_name }} container and {{ image_name }} if exists
      shell: "docker ps -q --filter 'name={{ container_name }}' && docker stop {{ container_name }} && docker rm -fv {{ container_name }} && docker image rm -f {{ image_name }} || echo 'Not Found'"

    - name: Launch postgresql docker container
      docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports:
          - "5432:5432"
        env:
          POSTGRES_PASSWORD: "Pp123456789"
        volumes:
          - /db-data:/var/lib/postgresql/data

- name: Nodejs Server configuration
  hosts: _ansible_nodejs
  become: true
  vars:
    container_path: /home/ec2-user/nodejs
    container_name: rumeysa_nodejs
    image_name: <your_AWS_account_number>.dkr.ecr.us-east-1.amazonaws.com/rumeysa-repo/todo-app:nodejs
  tasks:
    - name: remove {{ container_name }} container and {{ image_name }} if exists
      shell: "docker ps -q --filter 'name={{ container_name }}' && docker stop {{ container_name }} && docker rm -fv {{ container_name }} && docker image rm -f {{ image_name }} || echo 'Not Found'"

    - name: Launch postgresql docker container
      docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports:
          - "5000:5000"

- name: React UI Server configuration
  hosts: _ansible_react
  become: true
  vars:
    container_path: /home/ec2-user/react
    container_name: rumeysa_react
    image_name: <your_AWS_account_number>.dkr.ecr.us-east-1.amazonaws.com/rumeysa-repo/todo-app:react
  tasks:
    - name: remove {{ container_name }} container and {{ image_name }} image if exists
      shell: "docker ps -q --filter 'name={{ container_name }}' && docker stop {{ container_name }} && docker rm -fv {{ container_name }} && docker image rm -f {{ image_name }} || echo 'Not Found'"

    - name: Launch react docker container
      docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports:
          - "3000:3000"
Step-8: Create templates for nodejs and react
We will create node-env-template under repository which will be used by Jenkins and DB_HOST will be replaced after Jenkins created Postgre server with Terraform.

SERVER_PORT=5000
DB_USER=postgres
DB_PASSWORD=Pp123456789
DB_NAME=rumeysatodo
DB_HOST=${DB_HOST}
DB_PORT=5432
We will create react-env-template under repository which will be used by Jenkins.

REACT_APP_BASE_URL=http://${NODE_IP}:5000/
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
