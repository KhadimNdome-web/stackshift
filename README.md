# StackShift

**StackShift** is a modernized lift-and-shift cloud migration project built on AWS. It demonstrates how to transition legacy multi-tier applications to a cloud-native environment using automation, observability, and managed services.

---

## 📦 Stack Overview

### ✅ Services Used
- **EC2 Instances**: Tomcat, RabbitMQ, Memcached, MySQL
- **ELB (Elastic Load Balancer)**: Replaces traditional NGINX load balancer
- **Auto Scaling**: Dynamically scales EC2 instances
- **S3 / EFS**: Shared storage for application artifacts
- **Route 53**: Private DNS for internal domain resolution

---

## 🚀 Project Workflow

### 1. 🔐 Security Group Configuration

- **`vprofile-ELB-SG`** (Load Balancer)
  - Inbound: HTTP (80), HTTPS (443) from Anywhere (IPv4/IPv6)
- **`vprofile-app-sg`** (Tomcat App Layer)
  - Inbound: Port 8080 from `vprofile-ELB-SG`
  - SSH (22) from Admin IP
- **`vprofile-backend-sg`** (MySQL, RabbitMQ, Memcached)
  - Inbound: 3306 (MySQL), 5672 (RabbitMQ), 11211 (Memcached)
  - Source: `vprofile-app-sg`
  - SSH (22) from Admin IP

---

### 2. 🔑 Key Pair Creation

Created key pair named `vprofile-prod-key` for SSH access to instances.

---

### 3. 🛠 MySQL Instance Setup (`vprofile-db01`)

**AMI**: Amazon Linux

#### 🧾 User Data:
```bash
#!/bin/bash
DATABASE_PASS='admin123'
sudo dnf update -y
sudo dnf install git zip unzip mariadb105-server -y
sudo systemctl start mariadb
sudo systemctl enable mariadb
cd /tmp/
git clone -b awsliftandshift https://github.com/hkhcoder/vprofile-project.git
```

#### 🛡️ Manual Configuration:
```bash
sudo mysqladmin -u root password "$DATABASE_PASS"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User='';"
sudo mysql -u root -p"$DATABASE_PASS" -e "CREATE DATABASE accounts;"
sudo mysql -u root -p"$DATABASE_PASS" -e "GRANT ALL PRIVILEGES ON accounts.* TO 'admin'@'%' IDENTIFIED BY 'admin123';"
sudo mysql -u root -p"$DATABASE_PASS" accounts < /tmp/vprofile-project/src/main/resources/db_backup.sql
```

---

### 4. 💾 Memcached Instance Setup (`memc-stackshift`)

**AMI**: Amazon Linux

#### 🧾 User Data:
```bash
#!/bin/bash
sudo dnf install memcached -y
sudo systemctl start memcached
sudo systemctl enable memcached
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
sudo systemctl restart memcached
```

---

### 5. 📩 RabbitMQ Instance Setup (`rmq-stackshift`)

**AMI**: Amazon Linux

#### 🧾 User Data:
```bash
#!/bin/bash
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc'
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key'
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key'
curl -o /etc/yum.repos.d/rabbitmq.repo https://raw.githubusercontent.com/hkhcoder/vprofile-project/refs/heads/awsliftandshift/al2023rmq.repo
dnf update -y
dnf install socat logrotate erlang rabbitmq-server -y
systemctl enable rabbitmq-server
systemctl start rabbitmq-server
echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config
rabbitmqctl add_user test test
rabbitmqctl set_user_tags test administrator
rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
systemctl restart rabbitmq-server
```

---

### 6. 🧩 Tomcat (App) Instance Setup (`app-stackshift`)

**AMI**: Ubuntu

#### 🧾 User Data:
```bash
#!/bin/bash
sudo apt update && sudo apt upgrade -y
sudo apt install openjdk-17-jdk tomcat10 git -y
```

---

### 7. 🧭 Route 53 Private DNS Configuration

- Created Private Hosted Zone: `stackshift.com`
- Created records for:
  - `db.stackshift.com`
  - `rmq.stackshift.com`
  - `memc.stackshift.com`
  - `app.stackshift.com`
- Verified DNS resolution using:
```bash
ping -c 4 db.stackshift.com
```

---

### 8. 🪣 S3 Artifact Storage

- Created bucket: `stackshift-artifacts`
- Created IAM User: `stackshift-s3-Khadim` with access keys
- Created IAM Role: `s3-Khadim` and attached it to `app-stackshift` EC2 instance

---

### 9. 📦 Artifact Build & Deployment

- Built `.war` file using Maven: `vprofile-v2.war`
- Uploaded to S3 bucket: `stackshift-artifacts`

#### On `app-stackshift` EC2:
```bash
sudo apt install awscli -y
aws s3 cp s3://stackshift-artifacts/vprofile-v2.war /var/lib/tomcat10/webapps/
```

---

### 10. 🌐 Load Balancer Setup

- Created Application Load Balancer: `stackshift-elb`
- Spanned across AZs: `us-east-1a` to `us-east-1f`
- Verified access via browser:
```
http://stackshift-elb-1325834005.us-east-1.elb.amazonaws.com
```

---

### 11. 📈 Auto Scaling Configuration

- Created AMI from `app-stackshift`
- Created Launch Template using the AMI
- Created Auto Scaling Group: `stackshift-app-asg`
  - **Max Instances**: 4
  - **Scaling Policy**: Trigger when CPU > 50%

---

## ✅ Summary

### 📌 Key Takeaways
- Migrated a multi-tier app to AWS using EC2, ELB, and Auto Scaling
- Automated infrastructure setup via User Data scripts
- Configured internal DNS resolution with Route 53
- Managed artifact deployment using S3 and IAM roles

---

### 🔐 IAM Best Practices

- Used IAM Roles instead of Access Keys for EC2
- Applied least privilege principle to all users and roles

---

## 👨‍💻 Author

**Khadim**  
DevOps Engineer | AWS Enthusiast

---

## 📜 License

This project is licensed under the MIT License.
