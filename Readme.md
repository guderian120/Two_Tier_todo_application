```
# EC2 Instance Management and Docker Compose Application

This project demonstrates the setup of a two-tier application hosted on AWS EC2 instances with Docker Compose, featuring a front-end application in a public subnet and a database in a private subnet for enhanced security. The goal is to set up a robust and secure architecture using AWS VPC, subnets, EC2, Docker, and Docker Compose.

## Table of Contents
- [Before Partitioning the Application](#before-partitioning-the-application)
  - [VPC and Subnet Setup](#vpc-and-subnet-setup)
  - [Launching EC2 Instance in the Public Subnet](#launching-ec2-instance-in-the-public-subnet)
  - [Testing the Application](#testing-the-application)
  
- [After Partitioning the Application](#after-partitioning-the-application)
  - [Private Subnet Setup for Database](#private-subnet-setup-for-database)
  - [Configuring Nginx Reverse Proxy](#configuring-nginx-reverse-proxy)
  - [Testing Partitioned Application](#testing-partitioned-application)

---

## Before Partitioning the Application

### VPC and Subnet Setup

1. **Create a VPC**: The application was initially set up in a single VPC called `Docker compose vpc`, with a CIDR block of `10.0.0.0/24`, allowing for up to 254 hosts.

2. **Create Two Subnets**:
   - **Public Subnet**: `10.0.0.0/25` (128 IPs)
   - **Private Subnet**: `10.0.0.128/25` (128 IPs)

3. **Create and Attach an Internet Gateway**: An Internet Gateway was created and attached to the VPC to provide internet access.

4. **Route Table Configuration**: 
   - For the public subnet, the route table was configured with a route to the Internet Gateway (0.0.0.0/0).
   - The private subnet was configured to use a NAT Gateway for outbound internet traffic.

5. **Security Group Configuration**: 
   - A security group was created for the public subnet, allowing HTTP and HTTPS traffic.
   - The security group for the private subnet allows only connections from the public subnet’s security group.

### Launching EC2 Instance in the Public Subnet

1. **Launch EC2 Instance**: An Amazon Linux EC2 instance was launched in the public subnet using the AMI `ami-0ce8c2b29fcc8a346`.

2. **Install Docker and Docker Compose**:
   - Docker and Docker Compose were installed on the EC2 instance.
   - To fix the issue of Docker Compose not being a recognized command, the following command was used to download the latest binary and make it executable:
     ```bash
     sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
     sudo chmod +x /usr/local/bin/docker-compose
     ```

3. **Create Docker Compose Configuration**: 
   - A `docker-compose.yml` file was created to define two services: `app` (a Node.js application) and `mysql` (a MySQL database). 
   - Health checks were added for both services to ensure they are up and running.

4. **Test the Application**: The application was initially tested without partitioning:
   - The application was accessible via the EC2 instance’s public IP, and both the app and MySQL services worked together.

---

## After Partitioning the Application

### Private Subnet Setup for Database

1. **Launch EC2 Instance in the Private Subnet**: 
   - A second EC2 instance was launched in the private subnet for the MySQL database.

2. **Docker Setup on Private Instance**:
   - Docker was installed on the private instance, and Docker Compose was configured similarly to the public instance.
   - The MySQL service was configured in a `docker-compose.yml` file to run on the private instance.
   - The `mysql` service in the private subnet was configured to be accessed by the `app` service in the public subnet.

### Configuring Nginx Reverse Proxy

1. **Configure Nginx as a Reverse Proxy**: 
   - Nginx was set up on the public instance to act as a reverse proxy for the Node.js application running in the public subnet.
   - The Nginx configuration file (`nginx.conf`) was modified to proxy requests to the `app` service.

   Example of the Nginx configuration:
   ```nginx
   server {
       listen 80;
       server_name localhost;
       
       location / {
           proxy_pass http://app:3000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }
   }
   ```

2. **Test the Partitioned Application**: 
   - After partitioning the services, the app service in the public subnet was accessed via Nginx, which routed requests to the Node.js application running on the same subnet.
   - The MySQL service in the private subnet was successfully accessed by the app service in the public subnet using the private IP address (`10.0.2.100`).
   
   The partitioning ensured that the database remained isolated in the private subnet and was not directly accessible from the internet, thus enhancing security.

---

## Architecture Diagram

```plaintext
AWS VPC (10.0.0.0/16)
├── Public Subnet (10.0.1.0/24)  
│   └── Nginx (with EC2 instance + Docker)  
│       - Has Elastic IP / Public IP  
│       - Security Group allows HTTP/HTTPS  
├── Private Subnet (10.0.2.0/24)  
│   └── MySQL (with EC2 instance + Docker)  
│       - No public IP  
│       - Security Group allows 3306 from Nginx's SG  
└── NAT Gateway (for private subnet outbound traffic)
```

---

## Conclusion

This application demonstrates a secure architecture using AWS EC2, Docker, and Docker Compose. The partitioning of the application into public and private subnets ensures that the MySQL database is isolated from the public internet, while the front-end application remains accessible. This setup also enhances the scalability and security of the system.
```

---

