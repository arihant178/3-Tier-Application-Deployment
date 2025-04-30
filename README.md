# 3-Tier Application Deployment on AWS

This repository demonstrates how to deploy a **Java-based Student Web Application** using a **3-Tier architecture** on AWS. The architecture consists of:

- **Frontend Layer**: NGINX on a Public EC2 instance (acts as reverse proxy)
- **Application Layer**: Apache Tomcat on a Private EC2 instance (hosts the `student.war` app)
- **Database Layer**: Amazon RDS MySQL (stores student records)

---

## 📐 Architecture Overview

```
Client
  │
  ▼
NGINX (Public EC2)
  │
  ▼
Tomcat (Private EC2)
  │
  ▼
MySQL (Amazon RDS)
```

---

## ⚙️ AWS Infrastructure

### VPC Setup
- Custom VPC: `VPC-3-tier`
- Subnets:
  - `Public-Subnet-Nginx` (192.168.1.0/24)
  - `Private-Subnet-Tomcat` (192.168.2.0/24)
  - `Private-Subnet-Database` (192.168.3.0/24)
  - `Public-Subnet-LB` (192.168.4.0/24)
- Internet Gateway: `IGW-3-tier`
- NAT Gateway: `NAT-3-tier`
- Route Tables:
  - `RT-Public-Subnet`: routes to IGW
  - `RT-Private-Subnet`: routes to NAT Gateway

### Security Groups
| SG Name      | Purpose           | Inbound Rules                                  |
|--------------|-------------------|--------------------------------------------------|
| SG-Nginx     | NGINX EC2         | HTTP (80), SSH (22) from your IP               |
| SG-Tomcat    | Tomcat EC2        | 8080 from SG-Nginx, SSH (22) from your IP      |
| SG-Database  | RDS MySQL         | 3306 from SG-Tomcat                            |

---

##  Deployment Guide

### Step 1: Launch EC2 Instances
- **NGINX EC2 (Public)**: Ubuntu 22.04 in `Public-Subnet-Nginx`
- **Tomcat EC2 (Private)**: Ubuntu 22.04 in `Private-Subnet-Tomcat`

### Step 2: RDS Setup
- Engine: MySQL
- DB Name: `studentapp`
- Username: `admin`
- Password: `Passwd123$`
- Subnet Group: Private Subnets
- Public Access: NO
- Allow inbound port `3306` from `SG-Tomcat`

### Step 3: Configure Tomcat Server
```bash
sudo apt update
sudo apt install openjdk-11-jdk curl -y
mkdir -p /opt/tomcat
cd /opt/tomcat
wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.90/bin/apache-tomcat-9.0.90.tar.gz
tar -xzf apache-tomcat-9.0.90.tar.gz
cd apache-tomcat-9.0.90/webapps
curl -O https://s3-us-west-2.amazonaws.com/studentapi-cit/student.war
cd ../lib
curl -O https://s3-us-west-2.amazonaws.com/studentapi-cit/mysql-connector.jar
```

Edit `context.xml`:
```xml
<Resource name="jdbc/TestDB" auth="Container" type="javax.sql.DataSource"
               maxTotal="100" maxIdle="30" maxWaitMillis="10000"
               username="admin" password="Passwd123$" driverClassName="com.mysql.jdbc.Driver"
               url="jdbc:mysql://<RDS-ENDPOINT>:3306/studentapp"/>
```

Start Tomcat:
```bash
cd ../bin
chmod +x catalina.sh
./catalina.sh start
```

### Step 4: Setup MySQL DB
```bash
sudo apt install mysql-client -y
mysql -h <RDS-ENDPOINT> -u admin -p
```
SQL Commands:
```sql
CREATE DATABASE studentapp;
USE studentapp;
CREATE TABLE students (
  student_id INT NOT NULL AUTO_INCREMENT,
  student_name VARCHAR(100),
  student_addr VARCHAR(100),
  student_age VARCHAR(3),
  student_qual VARCHAR(20),
  student_percent VARCHAR(10),
  student_year_passed VARCHAR(10),
  PRIMARY KEY (student_id)
);
```

### Step 5: Configure NGINX Reverse Proxy
```bash
sudo apt install nginx -y
sudo nano /etc/nginx/sites-available/default
```
Edit default site:
```nginx
location / {
    proxy_pass http://<Tomcat-Private-IP>:8080/student/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```
Apply Changes:
```bash
sudo systemctl reload nginx
```

---

##  Testing & Verification
- Open browser → `http://<NGINX-EC2-Public-IP>`
- Check NGINX logs: `/var/log/nginx/error.log`
- Check Tomcat logs: `/opt/tomcat/apache-tomcat-9.0.90/logs/catalina.out`
- Use `curl` or `telnet` to test internal connectivity

---

##  Technologies Used
- **AWS EC2, VPC, RDS**
- **Ubuntu 22.04**
- **Apache Tomcat 9**
- **MySQL**
- **NGINX**
- **JDBC Connector**

---

## 📄 License
Open-source project for educational and deployment reference purposes.

