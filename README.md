# DevOps_Microservice_Project

## Architecture

![Architeture Image](/image/dia.png)

## Servers Setup on AWS

### Jenkins

- Ubuntu 18 or 20 version
- t2.small type AWS instance minimum needed. (t2.micro will also work)
- Add Tags

Security Group:

- Allow SSH on port 22
- Allow Custom TCP on port 8080
- Allow all traffic from SonarQube Security group.

```bash
#!/bin/bash
sudo apt update
sudo apt install openjdk-11-jdk -y
sudo apt install maven -y
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
###
```

### SonarQube

- Ubuntu 18 AMI is needed.
- Minimum t2.medium instance is needed.
- Add tages

```bash
#!/bin/bash
cp /etc/sysctl.conf /root/sysctl.conf_backup
cat <<EOT> /etc/sysctl.conf
vm.max_map_count=262144
fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
EOT
cp /etc/security/limits.conf /root/sec_limit.conf_backup
cat <<EOT> /etc/security/limits.conf
sonarqube   -   nofile   65536
sonarqube   -   nproc    409
EOT
sudo apt-get update -y
sudo apt-get install openjdk-11-jdk -y
sudo update-alternatives --config java
java -version
sudo apt update
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
sudo apt install postgresql postgresql-contrib -y
#sudo -u postgres psql -c "SELECT version();"
sudo systemctl enable postgresql.service
sudo systemctl start  postgresql.service
sudo echo "postgres:admin123" | chpasswd
runuser -l postgres -c "createuser sonar"
sudo -i -u postgres psql -c "ALTER USER sonar WITH ENCRYPTED PASSWORD 'admin123';"
sudo -i -u postgres psql -c "CREATE DATABASE sonarqube OWNER sonar;"
sudo -i -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;"
systemctl restart  postgresql
#systemctl status -l   postgresql
netstat -tulpena | grep postgres
sudo mkdir -p /sonarqube/
cd /sonarqube/
sudo curl -O https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.3.0.34182.zip
sudo apt-get install zip -y
sudo unzip -o sonarqube-8.3.0.34182.zip -d /opt/
sudo mv /opt/sonarqube-8.3.0.34182/ /opt/sonarqube
sudo groupadd sonar
sudo useradd -c "SonarQube - User" -d /opt/sonarqube/ -g sonar sonar
sudo chown sonar:sonar /opt/sonarqube/ -R
cp /opt/sonarqube/conf/sonar.properties /root/sonar.properties_backup
cat <<EOT> /opt/sonarqube/conf/sonar.properties
sonar.jdbc.username=sonar
sonar.jdbc.password=admin123
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
sonar.web.host=0.0.0.0
sonar.web.port=9000
sonar.web.javaAdditionalOpts=-server
sonar.search.javaOpts=-Xmx512m -Xms512m -XX:+HeapDumpOnOutOfMemoryError
sonar.log.level=INFO
sonar.path.logs=logs
EOT
cat <<EOT> /etc/systemd/system/sonarqube.service
[Unit]
Description=SonarQube service
After=syslog.target network.target
[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonar
Group=sonar
Restart=always
LimitNOFILE=65536
LimitNPROC=4096
[Install]
WantedBy=multi-user.target
EOT
systemctl daemon-reload
systemctl enable sonarqube.service
#systemctl start sonarqube.service
#systemctl status -l sonarqube.service
apt-get install nginx -y
rm -rf /etc/nginx/sites-enabled/default
rm -rf /etc/nginx/sites-available/default
cat <<EOT> /etc/nginx/sites-available/sonarqube
server{
    listen      80;
    server_name sonarqube.abhis.cloud;
    access_log  /var/log/nginx/sonar.access.log;
    error_log   /var/log/nginx/sonar.error.log;
    proxy_buffers 16 64k;
    proxy_buffer_size 128k;
    location / {
        proxy_pass  http://127.0.0.1:9000;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;

        proxy_set_header    Host            \$host;
        proxy_set_header    X-Real-IP       \$remote_addr;
        proxy_set_header    X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto http;
    }
}
EOT
ln -s /etc/nginx/sites-available/sonarqube /etc/nginx/sites-enabled/sonarqube
systemctl enable nginx.service
#systemctl restart nginx.service
sudo ufw allow 80,9000,9001/tcp
echo "System reboot in 30 sec"
sleep 30
reboot
```

Sonar Analysis Properties file:

```bash
sonar.projectKey=vprofile
sonar.projectName=vprofile-repo
sonar.projectVersion=1.0
sonar.sources=src/
sonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/
sonar.junit.reportsPath=target/surefire-reports/
sonar.jacoco.reportsPath=target/jacoco.exec
sonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
```

### Kops Setup

Prerequisites:

- Domain for Kubernetes DNS Record (ex: groophy.in)
- Create Linux Virtual Machine and setup
  - Kubectl
  - Kops
  - ssh-keys
  - awscli
- Login into AWS Account and Setup
  - s3 Bucket
  - IAM User for AWS CLI. Add following roles for new user.
    - AmazonS3FullAccess
    - AmazonEC2FullAccess
    - AmazonRoute53FullAccess
    - IAMFullAccess
    - AmazonVPCFullAccess
  - Route53 Hosted Zone

Kops Server Configuration:

- Ubuntu Server 20.04 (t2.micro). Add Tags
- Add New Kops Security Group (kops-SG).
  - SSH 22
- Launch Instance

s3 Bucket:

- Create one s3 bucket with name --> project-kops-state
- Enable the Bucket Versioning

IAM User:

1. Open IAM in AWS
2. CLick User --> add user --> Username(kopsadmin)
3. Check Programmatic Access --> Next.
4. Click Attach existing policies --> check Administrators access --> Next.
5. Click Create.

Route53:

1. Click on create hosted zone.
2. Enter domain name. (ex: kubemajor.groophy.in)
3. Check the Public hosted zone.
4. Click create hosted zone.

```bash
sudo apt update
sudo apt install awscli -y
aws --version
aws configure

# Kuberctl installation
sudo curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

sudo chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kuberctl --help

# Kops Installation
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64

sudo chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops
kops --help
kops
nslookup -type=ns kubemajor.abhis.cloud

# s3 Bucket Creation
aws s3 mb s3://pro-kop-abc
aws s3api put-bucket-versioning --bucket pro-kop-abc --versioning-configuration Status=Enabled
export KOPS_STATE_STORE=s3://pro-kop-abc

# Create SSH Key
ssh-keyegn

# Cluster Creation
kops create cluster --cloud=aws --zones=ap-south-1a --name=kubemajor.abhis.cloud --dns-zone=kubemajor.abhis.cloud --dns public

kops create cluster --cloud=aws --zones=ap-south-1a --networking calico --name=kubemajor.abhis.cloud --dns-zone=kubemajor.abhis.cloud --dns public

kops create cluster --cloud=aws --zones=ap-south-1a,ap-south-1b --networking calico --master-size t3.medium --master-count 3 --node-size t3.xlarge --node-count 3 --name=kubemajor.abhis.cloud --dns-zone=kubemajor.abhis.cloud --dns public

kops create cluster --name=kubemajor.abhis.cloud --state=s3://project-kops-state --zones=us-east-1a,us-east-1b --node-count=2 --node-size=t2.micro --master-size=t2.micro --dns-zone=kubemajor.abhis.cloud

kops update cluster kubemajor.abhis.cloud --yes --admin
kops validate cluster
kubectl get nodes
kubectl cluster-info
kops delete cluster kubemajor.abhis.cloud --yes

kops update cluster --name kubemajor.abhis.cloud --state=s3://project-kops-state --yes --admin

kops validate cluster --state=s3://project-kops-state

cat .kube/config
kubectl get nodes

kops delete cluster --name=kubemajor.abhis.cloud --state=s3://project-kops-state --yes
```

Note: Delete the AWS public hosted zone manually.

## Main Server Configuration

### Jenkins

````bash
# Install the docker
sudo apt-get update
sudo apt-get remove docker docker-engine docker.io
sudo apt install docker.io
sudo systemctl start docker
sudo systemctl enable docker
systemctl status docker

# add jenkins in docker group
sudo -i
usermod -aG docker jenkins
id jenkins
reboot
```bash

Install the Jenkins Plugins:
* Docker
* Docker Pipeline
* Pipeline Utility steps
````

### Kops Cluster

Install the all the required dependencies:

```bash
export KOPS_STATE_STORE=s3://pro-kop-abc

kops create cluster --cloud=aws --zones=us-east-1c --networking calico --master-size t2.medium --master-count 1 --node-size t2.medium --node-count 2 --name=mypro.abhis.cloud --dns-zone=mypro.abhis.cloud --dns public

kops update cluster --name mypro.abhis.cloud --yes --admin

# Helm Instllation
wget https://get.helm.sh/helm-v3.9.3-linux-amd64.tar.gz
tar xvf helm-v3.9.3-linux-amd64.tar.gz
cd linux-amd64/
sudo mv helm /usr/local/bin/helm
cd
helm

kubectl get nodes
kops get cluster
kops validate cluster

## git
git clone https://
mkdir helm
cd helm
```

Other Commands Important Kops:1

```bash
kubectl create namespace prod
kubectl get namespace
helm list --namespace prod
kubectl get all --namespace prod
helm delete vprofile-stack --namespace prod
```
