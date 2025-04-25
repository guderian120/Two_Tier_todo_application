# 🚀 AWS VPC with Dockerized NGINX & MySQL Architecture

This project demonstrates a complete VPC architecture in AWS with Dockerized applications running across public and private subnets. It includes EC2 provisioning, VPC setup, secure network configuration, and containerized service deployment using Docker Compose.

## 📌 Project Overview

- Provision a custom VPC with a public and private subnet
- Configure route tables, internet gateway, NAT gateway, and network ACLs
- Launch EC2 instances in each subnet
- Deploy Docker containers:  
  - **NGINX & Node.js App** in the **public subnet**
  - **MySQL** in the **private subnet**
- Configure reverse proxy with NGINX and enable secure internal communication

---

## 🧱 Architecture Diagram



AWS VPC (10.0.0.0/24) 

       ├── Public Subnet (10.0.0.0/25) │ 

                └── EC2 Instance (NGINX + Node App) │

		 - Elastic IP / Public IP │ - Security Group allows HTTP/HTTPS 

	├── Private Subnet (10.0.0.128/25) │ 

		└── EC2 Instance (MySQL) │

		 - No public IP │ - Security Group allows port 3306 from NGINX SG 

		└── NAT Gateway - Enables outbound internet from private subnet

---

## ⚙️ Setup Instructions

### 1. VPC & Network Configuration

- Create VPC: `10.0.0.0/24`
- Subnets:
  - Public: `10.0.0.0/25`
  - Private: `10.0.0.128/25`
- Attach Internet Gateway to the VPC
- Create a route table for the public subnet:
  - Destination: `0.0.0.0/0`
  - Target: Internet Gateway
- Create NAT Gateway in the public subnet and associate it with the private route table
- Configure Network ACLs and Security Groups:
  - Public SG: Allow inbound HTTP/HTTPS
  - Private SG: Allow MySQL (3306) from public SG only

---

## 💻 EC2 Configuration

### Public Instance (NGINX + Node App)

- AMI: `ami-0ce8c2b29fcc8a346`
- Install Git and Docker:
  ```bash
  sudo yum install git -y
  sudo yum install docker -y
  sudo systemctl start docker
  sudo systemctl enable docker


Install Docker Compose:
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

docker-compose.yml
version: '3.8'

services:
  app:
    image: node:18-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - "127.0.0.1:3000:3000"
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos
    depends_on:
      mysql:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000"]
      interval: 10s
      timeout: 5s
      retries: 5

  mysql:
    image: mysql:8.0
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-psecret"]
      interval: 5s
      timeout: 10s
      retries: 10

volumes:
  todo-mysql-data:


Private Instance (MySQL Only)

SSH from public to private using pre-copied key:
ssh -i <private-key.pem> ec2-user@<PRIVATE_IP>

Install Docker & Docker Compose (same steps as above)
docker-compose-mysql.yml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - app_network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-psecret"]
      interval: 5s
      timeout: 10s
      retries: 10

networks:
  app_network:
    driver: bridge

volumes:
  mysql_data:


🌐 NGINX Reverse Proxy Configuration

nginx.conf
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;

        location / {
            proxy_pass http://app:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
    }
}

docker-compose-nginx.yml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    networks:
      - app_network
    environment:
      MYSQL_HOST: 10.0.2.100  # Private IP of MySQL instance

networks:
  app_network:
    driver: bridge


✅ Verification & Health Checks

Verify containers are running:
docker-compose ps

Ensure services are healthy using:
docker inspect --format='{{json .State.Health}}' <container_id>

Confirm the web app is accessible from the browser using the public EC2 IP or domain

📚 Future Enhancements

Add SSL support using Let's Encrypt
Automate setup with Terraform or AWS CDK
CI/CD integration for automatic deployment

📄 License

This project is licensed under the MIT License. See the LICENSE file for details.

🙌 Acknowledgements

Thanks to AWS, Docker, and the open-source community for making this architecture possible!	
