# **End to end CICD pipeline setup and implementation using Jenkins, Sonarqube, ArgoCD and Kubernetes**

This project is to demonstrate implementation of a complete CICD pipeline. Jenkins is used for CI, Sonarqube is for static code analysis, 
ArgoCD is used for continuous delivery and the application is deployed in a local kubernetes cluster

## **Setting up project environment** 

1. Initializing EC2 Instance. 

Since minimum hardware requirement for sonarqube is quite big, a t2.large instance is selected instead of t2.micro

Log into AWS console using IAM User account. 
Create an EC2 instance
Select an OS image - Ubuntu
Create a new key pair with any name & download .pem file
Use the same key pair in the EC2 instance creation screen. 
Instance type - t2.large
Connecting to the instance using ssh

Use the below command to connect to EC2 instance using SSH

```
ssh -i <keypair_name>.pem ubunutu@<IP_ADDRESS>
```

### **Configuring Ubuntu on EC2 instance and installing dependencies**

Update the outdated packages and dependencies. 

```
sudo apt update
```

### **Installing Git**

Install git using the below command. 

```
sudo apt install git
```

### **Installing Jenkins**

```
sudo apt install openjdk-17-jre
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```

### **Starting Jenkins service**

```
sudo systemctl enable Jenkins
sudo systemctl start Jenkins
```

### **Installing sonarqube**

For sonarqube, a new user is created using adduser command and a password is also set. 
After the new user is created, we need to switch to the newly created user and then run the following commands. 
The username need not be sonarqube but for the project, the username we have used is sonarqube. 

```
apt install unzip
adduser sonarqube
sudo su - sonarqube
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
unzip *
chmod -R 755 /home/sonarqube/sonarqube-9.4.0.54424
chown -R sonarqube:sonarqube /home/sonarqube/sonarqube-9.4.0.54424
cd sonarqube-9.4.0.54424/bin/linux-x86-64/
./sonar.sh start
```

### **Installing Docker**


```
sudo apt update
sudo apt install docker.io
```

```
sudo su - 
usermod -aG docker jenkins
usermod -aG docker ubuntu
systemctl restart docker
```
The above commands add jenkins user and ubuntu user to docker group so that docker commands can be executed without root privileges.
 
### **Installing Kubernetes(Minikube)**

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube 

sudo snap install kubectl --classic
minikube start --driver=docker
```


### **Installing ArgoCD**

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get all -n argocd

kubectl port-forward svc/argocd-server -n argocd --address 0.0.0.0 8080:443
kubectl edit secret argocd-initial-admin-secret -n argocd
echo (secret encrypt code) | base64 ==decode
```

The last three commands edit the argocd-server so that it listens on port 8080 and also shows the adgocd gui password. 



## **Integrating docker, git and sonarqube with Jenkins for CI**

*Logging into jenkins*

Log into Jenkins using the link
> http://<ec2-public-ip>:8080

As per the instruction use the below command to get the admininstrator password

```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Create an admin user after entering the password. 


After login, go to Manage Jenkins > Manage Plugins > Available plugins and install "Docker pipeline" and "Sonarqube scanner" plugin

Restart Jenkins after installation of both plugins by going to the following link

> http://<ec2-public-ip>:8080/restart


Go to Jenkins Dashboard and create new item. 

Choose "Pipeline" project, enter a name and click on "OK"

Select the "Github project" checkbox and enter github repository link. 
In the pipeline section, choose pipeline script from SCM and enter the details of the repository, branch name and path to jenkins file. 


> java-maven-sonar-argocd-helm-k8s/spring-boot-app/JenkinsFile

and then click on save and close


**Logging in sonarqube**

log into sonarqube using the following link 

> http://<ec2-public-ip>:9000

The default username is admin. Password is admin. 


Go to My Account> Security

Enter a token name, generate token and copy the token. 

Go to jenkins> manage jenkins and click on "Credentials"
Choose system> Global credentials> add credentials and add the sonarqube token as secret text with "sonarqube" as ID


In the same page where sonarqube credentials were added, add another credential of kind "Username with password" and enter docker credentials
Repeat the same step to create another credential of kind "secret" and enter github personal access token. 
To get personal access token, go to github profile icon> select settings> developer settings> Personal access token

Restart Jenkins again after that. 


After Jenkins has been restarted, build now button can be used to trigger the build which will checkout code, build the code, 
perform static code analysis, build the docker image,  push the docker image to dockerhub and update the manifest file in the spring-boot-app-manifests folder. 


**Setting up ArgoCD for continuous delivery**

Log into Argo CD with the username "admin" and admin password and click on new app


Use the following information to create the application. 
 - Application name: Any application name
 - Project Name: default
 - Sync Policy : Automatic

Select Self Heal. 

Then provide the Repository url and also the path to the manifest file deployment.yml (java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests)

Cluster URL : https://kubernetes.default.svc
Namespace : any name as per requirement and then click on create. 
Argo CD will start deploying the application as per the resources specificed in manifest file. 
