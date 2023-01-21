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
SSH into jenkins server and clone repo to jenkins server.
  git clone url copied from github.
  copy files (app codes) to repo or upload files to github

# Step-4: Create docker file (build instruction required to build the image)
   FROM openjdk:11-alpine

   RUN apk update && apk add /bin/sh

   RUN mkdir -p /opt/app
   ENV PROJECT_HOME /opt/app

   COPY target/spring-boot-mongo-1.0.jar $PROJECT_HOME/spring-boot-mongo.jar

   WORKDIR $PROJECT_HOME
   EXPOSE 8080
   CMD ["java" ,"-jar","./spring-boot-mongo.jar"]

 # Step-5: Create Manifest File (build instruction required to build the image)  
kind: ConfigMap
apiVersion: v1
metadata:
  name: springappconfigmap
data:
  username: springapp
  password: mongodb@123

---
apiVersion: v1
kind: Secret
metadata:
  name: springappsecrets
type: Opaque
stringData:   # We can define multiple key value pairs.
  password: devdb@123

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: "/tmp/mongodata"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-hostpath
spec:
  storageClassName: manual
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi

---
## Spring Boot App
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springappdeployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: springapp
  template:
    metadata:
      name: springapppod
      labels:
        app: springapp
    spec:
      containers:
      - name: springappcontainer
        image: acadalearning/spring-boot-mongo
        ports:
        - containerPort: 8080
        env:
        - name: MONGO_DB_USERNAME
          valueFrom:
            configMapKeyRef:
              name: springappconfigmap
              key: username
        - name: MONGO_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: springappsecrets
              key: password
        - name: MONGO_DB_HOSTNAME
          value: mongo
---
apiVersion: v1
kind: Service
metadata:
  name: springapp
spec:
  selector:
    app: springapp
  ports:
  - port: 80
    targetPort: 8080
  type: NodePort
---
# Mongo db pod with volumes(HostPath)
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: mongodbrs
spec:
  selector:
    matchLabels:
      app: mongodb
  template:
     metadata:
       name: mongodbpod
       labels:
         app: mongodb
     spec:
       volumes:
       - name: mongodb-pvc
         persistentVolumeClaim:
           claimName: pvc-hostpath
       containers:
       - name: mongodbcontainer
         image: mongo
         ports:
         - containerPort: 27017
         env:
         - name: MONGO_INITDB_ROOT_USERNAME
           valueFrom:
             configMapKeyRef:
               name: springappconfigmap
               key: username
         - name: MONGO_INITDB_ROOT_PASSWORD
           valueFrom:
             secretKeyRef:
               name: springappsecrets
               key: password
         volumeMounts:
         - name: mongodb-pvc
           mountPath: /data/db
---
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  type: ClusterIP
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
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
