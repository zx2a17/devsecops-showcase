# DevSecOps Showcase
the most modern DevSecOps pipeline as of 2025 - Powered by OWASP JuiceShop

1) Kubernetes cluster on AWS
2) Docker container in Dockerhub
3) source code in Github
4) security scan integration
5) Grafna & Prometheus for monitoring the kube cluster

<h1 align="center">TIC-TAC-TOE-GAME</h1>

![Screenshot 2024-11-25 102833](https://github.com/user-attachments/assets/b7cae5c0-070d-47a0-9ca9-87f812a8dd4d)


1) Create a master server t2 medium and 20GB storage with
   a) Admin IAM role
   b) security group set up to have the following open:
   ```
     3000 (for app to run)
     9000 (for Sonarqube dashboard)
     22, 443, 80
     3**** (Generated randomly while creating the EKS Cluster)
     3306 (MySQL/Aurora)
     6443
     8080
     5000
     8000
   ```

3) Create a new bucket for the terraform state as backend (if wanting to use remote backend)

4) install the following on the master server by creating a script



Put in User-data when launching
```
#!/bin/bash
# install docker
sudo apt update -y
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock

# Run sonar container on port 9000

docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

```


Create a new script file using a text editor, such as nano, give the file execution rights and then execute:

```
nano install_java.sh
chmod +x install_java.sh
./install_java.sh
```

```

sudo apt update -y
sudo touch /etc/apt/keyrings/adoptium.asc
sudo wget -O /etc/apt/keyrings/adoptium.asc https://packages.adoptium.net/artifactory/api/gpg/key/public
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version

# Install Trivy
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y

# Install Terraform
sudo apt install wget -y
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Install kubectl
sudo apt update
sudo apt install curl -y
curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# Install AWS CLI 
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt-get install unzip -y
unzip awscliv2.zip
sudo ./aws/install

# Install Node.js 16 and npm
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource.gpg.key | sudo gpg --dearmor -o /usr/share/keyrings/nodesource-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/nodesource-archive-keyring.gpg] https://deb.nodesource.com/node_16.x focal main" | sudo tee /etc/apt/sources.list.d/nodesource.list
sudo apt update
sudo apt install -y nodejs

# Install Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

```
4) verify all installed correctly

```
docker --version
trivy --version
terraform --version
aws --version
kubectl version
node -v
java --version
```

4.1) clone this repo to get the source code for the app as well as the tf script for server provisioning
then,

assign IAM role to server so it can execute the server provisioning

terraform init, terraform apply - this should take about 20 min to have the cluster set up



5) Add Helm repo to prepare for Prometheus/Grafana installation
```
# associate the eks cluster
aws eks --region us-east-1 update-kubeconfig --name EKS_SHOWCASE

helm repo add stable https://charts.helm.sh/stable  
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# create namespace
kubectl create namespace monitoring   
helm install stable prometheus-community/kube-prometheus-stack -n monitoring

# change the service type to LoadBalancer
kubectl edit svc stable-kube-prometheus-sta-prometheus -n monitoring 
kubectl edit svc stable-grafana -n monitoring   

```

Accessing Grafana
The username will be admin and the password will be prom-operator for your Grafana LogIn.

6) Login to Sonar Dashboard
Load on Browser using the <ec2-public-ip:9000>
username ==> admin
password ==> admin

set up project and configure Snoar creds on github secrets
Copy the token generated here for the SONAR_TOKEN and SONAR_URL to the secret 

7) Add Docker login token as well for the docker access. (your own account)
   - note here that the source code has been pushed to docker hub and stored there already.
   - at this point you can run initial sonarqube analysis already.

8) Add the sonar-project.properties in it
sonar.projectKey=<Your - key - shown>
This will be the project key within the sonar instance (it shoudl be part of the instruction)
After adding this one, then SonarQube is now ready to scan the content of the github repo and the code within it.

9) run the self-runner instructions on github, so that github action commands will apply to the master server.
10) clone the repo (this one) onto the sonarqube master server - then give this server IAM role (this will allow terraform to use s3 as backend and provision resources)
11) cd into the terraform folder - terraform init and apply
12) then *merging to main* should run the github action and it should deploy everything
13) run ```kubectl get all``` to get the port open on the ELB and open it to accept traffice to access the app!
14) access the app via the LoadBalancer with the external IP output of the command above
   a74a209fdffea4d7b8e773212a9809d7-1738975369.us-east-1.elb.amazonaws.com


Credits:
source
https://medium.com/@kunalbarot3188/deploying-tic-tac-toe-game-ci-cd-on-githubactions-and-amazon-eks-448e4e6b060d
