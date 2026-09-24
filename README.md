# AWS Deployment Guide

## Student Registration CRUD Application

This document describes the AWS deployment of the **Student Registration
CRUD application** in this repository using a two-tier application
architecture with a private MariaDB database.

**Repository:** https://github.com/Shripad191/CRUD-operations

> **Important:** This guide documents the AWS architecture and
> deployment approach. Replace all example values with the values from
> your own AWS account. Never commit passwords, private keys, `.pem`
> files, or other secrets to GitHub.

------------------------------------------------------------------------

## 1. Project Overview

The application is a full-stack CRUD system consisting of:

-   **Frontend:** React + Vite
-   **Backend:** Spring Boot 3.3.5
-   **Java:** Java 17
-   **Build tool:** Maven
-   **Database:** Amazon RDS for MariaDB
-   **Web server:** Nginx
-   **Load balancing:** Application Load Balancer (ALB)
-   **Compute:** EC2 Auto Scaling Groups
-   **Networking:** Amazon VPC, public/private subnets, Internet Gateway
    and NAT Gateway

The backend listens on **TCP 8080** and the frontend is built as static
files and served by Nginx on **TCP 80**.

The repository confirms Java 17 and Spring Boot 3.3.5 in the backend
Maven configuration, while the frontend uses React with Vite and the
`npm run build` command. The frontend API service reads its base URL
from `VITE_API_URL`.

------------------------------------------------------------------------

## 2. AWS Architecture

### High-level request flow

``` text
                         Internet
                            |
                            v
                +-----------------------+
                | Internet Gateway      |
                +-----------+-----------+
                            |
                            v
             +----------------------------+
             | Internet-facing ALB        |
             | Port 80                    |
             +-------------+--------------+
                           |
              +------------+------------+
              |                         |
        /api/* rule                 Default rule
              |                         |
              v                         v
    Backend Target Group       Frontend Target Group
              |                         |
       +------+-------+          +------+-------+
       |              |          |              |
       v              v          v              v
   Backend EC2    Backend EC2  Frontend EC2  Frontend EC2
    App-A          App-B        App-A          App-B
       |              |
       +------+-------+
              |
              v
       Amazon RDS MariaDB
        DB-A / DB-B

Private EC2 instances
        |
        v
   NAT Gateway
        |
        v
 Internet Gateway
```

### Traffic design

  Traffic                    Source        Destination      Port
  -------------------------- ------------- -------------- ------
  Public web traffic         Internet      ALB                80
  Frontend traffic           ALB           Frontend EC2       80
  API traffic                ALB           Backend EC2      8080
  Backend database traffic   Backend EC2   RDS MariaDB      3306
  Private outbound traffic   App EC2       NAT Gateway       ---
  Administration             Bastion       App EC2            22
  Administration             Your IP       Bastion            22

The database subnets have no Internet Gateway or NAT Gateway route.

------------------------------------------------------------------------

# 3. VPC and Subnet Design

Create a dedicated VPC:

``` text
VPC CIDR: 10.0.0.0/16
```

Create six subnets across two Availability Zones.

## Public subnets

  -----------------------------------------------------------------------
  Subnet            CIDR              Availability Zone Purpose
  ----------------- ----------------- ----------------- -----------------
  Public-A          `10.0.1.0/24`     AZ 1              ALB, NAT Gateway,
                                                        temporary bastion

  Public-B          `10.0.2.0/24`     AZ 2              ALB
  -----------------------------------------------------------------------

## Private application subnets

  Subnet   CIDR             Availability Zone   Purpose
  -------- ---------------- ------------------- --------------------------
  App-A    `10.0.11.0/24`   AZ 1                Frontend and backend EC2
  App-B    `10.0.12.0/24`   AZ 2                Frontend and backend EC2

## Private database subnets

  Subnet   CIDR             Availability Zone   Purpose
  -------- ---------------- ------------------- ------------------
  DB-A     `10.0.21.0/24`   AZ 1                RDS subnet group
  DB-B     `10.0.22.0/24`   AZ 2                RDS subnet group

### Why three subnet tiers?

The architecture separates resources according to their exposure:

1.  **Public tier** --- resources that need direct Internet connectivity
    or controlled administrative access.
2.  **Application tier** --- frontend and backend EC2 instances without
    public IP addresses.
3.  **Database tier** --- RDS only, with no direct Internet route.

------------------------------------------------------------------------

# 4. Internet Gateway

Create an Internet Gateway and attach it to the VPC.

The Internet Gateway provides Internet connectivity for resources in
public subnets through the public route table.

------------------------------------------------------------------------

# 5. Route Tables

## Public route table

Create a route table for the public subnets.

  Destination     Target
  --------------- ------------------
  `10.0.0.0/16`   Local
  `0.0.0.0/0`     Internet Gateway

Associate it with:

-   Public-A
-   Public-B

This makes these subnets public.

------------------------------------------------------------------------

## Private application route table

Create a route table for the application subnets.

  Destination     Target
  --------------- -------------
  `10.0.0.0/16`   Local
  `0.0.0.0/0`     NAT Gateway

Associate it with:

-   App-A
-   App-B

The EC2 instances remain private because they do not have public IP
addresses, while the NAT Gateway allows them to initiate outbound
Internet connections.

This is required for tasks such as:

-   Installing Java
-   Installing Maven
-   Installing Node.js
-   Installing Nginx
-   Installing Git
-   Cloning the public repository
-   Downloading Maven/npm dependencies

------------------------------------------------------------------------

## Database route table

Create a dedicated database route table.

It should contain only the local VPC route:

  Destination     Target
  --------------- --------
  `10.0.0.0/16`   Local

Associate it with:

-   DB-A
-   DB-B

### Security objective

The database subnets must **not** have:

``` text
0.0.0.0/0 -> Internet Gateway
```

or:

``` text
0.0.0.0/0 -> NAT Gateway
```

The RDS database should therefore remain private.

------------------------------------------------------------------------

# 6. NAT Gateway

For this practice deployment, use one NAT Gateway to reduce cost.

### Configuration

1.  Allocate an Elastic IP.
2.  Create the NAT Gateway in **Public-A**.
3.  Attach the Elastic IP.
4.  Update the private application route table:

``` text
0.0.0.0/0 -> NAT Gateway
```

### Availability note

Using one NAT Gateway is a deliberate low-cost practice configuration.

It is **not highly available** because the NAT Gateway exists in only
one Availability Zone. In a production architecture, separate NAT
Gateways would normally be considered for each Availability Zone.

------------------------------------------------------------------------

# 7. Security Groups

Use security-group-to-security-group references wherever possible
instead of opening application ports to the Internet.

## ALB security group

Name:

``` text
crud-alb-sg
```

Inbound:

  Protocol     Port Source
  ---------- ------ -------------
  HTTP           80 `0.0.0.0/0`

Outbound:

-   Allow required outbound traffic.

------------------------------------------------------------------------

## Frontend security group

Name:

``` text
crud-frontend-sg
```

Inbound:

  Protocol     Port Source
  ---------- ------ ------------------------
  HTTP           80 `crud-alb-sg`
  SSH            22 Bastion security group

The frontend EC2 instances should not accept HTTP directly from the
Internet.

------------------------------------------------------------------------

## Backend security group

Name:

``` text
crud-backend-sg
```

Inbound:

  Protocol       Port Source
  ------------ ------ ------------------------
  Custom TCP     8080 `crud-alb-sg`
  SSH              22 Bastion security group

Port 8080 should not be open to `0.0.0.0/0`.

------------------------------------------------------------------------

## RDS security group

Name:

``` text
crud-rds-sg
```

Inbound:

  Protocol     Port Source
  ---------- ------ -------------------
  MariaDB      3306 `crud-backend-sg`

Do **not** allow:

``` text
3306 from 0.0.0.0/0
```

Only the backend security group should be able to reach MariaDB.

------------------------------------------------------------------------

## Bastion security group

Name:

``` text
crud-bastion-sg
```

Inbound:

  Protocol     Port Source
  ---------- ------ ----------------------
  SSH            22 Your public IP `/32`

Example:

``` text
203.0.113.10/32
```

Do not use:

``` text
0.0.0.0/0
```

for SSH.

------------------------------------------------------------------------

# 8. EC2 Key Pair

Create an EC2 key pair, for example:

``` text
crud-aws-key
```

Download the private key and store it securely.

For Windows, use either:

-   Windows OpenSSH
-   PuTTY

The private key should never be committed to GitHub.

Example SSH command:

``` bash
ssh -i crud-aws-key.pem ec2-user@<BASTION_PUBLIC_IP>
```

Use the correct default username for the AMI you selected.

------------------------------------------------------------------------

# 9. Amazon RDS MariaDB

Create an Amazon RDS MariaDB database.

Recommended architecture:

-   Engine: MariaDB
-   Database name: `student_db`
-   Private subnets: DB-A and DB-B
-   RDS subnet group: dedicated DB subnet group
-   Security group: `crud-rds-sg`
-   Public access: **No**

Use a strong database password.

### Database endpoint

After RDS becomes available, copy its endpoint.

It will resemble:

``` text
database-example.xxxxxxxxxxxx.ap-south-1.rds.amazonaws.com
```

Do not add:

``` text
http://
```

or:

``` text
https://
```

The backend connects to MariaDB using:

``` text
jdbc:mariadb://<RDS_ENDPOINT>:3306/student_db
```

### Credential security

The database password must never appear in:

-   `application.properties`
-   Git history
-   screenshots
-   launch-template screenshots
-   README files
-   shell history
-   public GitHub commits

Use environment variables or a secrets-management solution instead.

------------------------------------------------------------------------

# 10. Backend Application Configuration

The repository backend is configured for:

``` text
Spring Boot: 3.3.5
Java: 17
Server port: 8080
MariaDB JDBC driver
```

The backend Maven configuration confirms these dependencies and the Java
version.

The repository's current `application.properties` also contains database
connection information. Because that file is publicly visible, **rotate
any credential that has been committed there before treating the
deployment as anything beyond a practice environment**.

Recommended runtime environment variables:

``` bash
export SPRING_DATASOURCE_URL="jdbc:mariadb://<RDS_ENDPOINT>:3306/student_db?sslMode=trust"
export SPRING_DATASOURCE_USERNAME="<DB_USERNAME>"
export SPRING_DATASOURCE_PASSWORD="<DB_PASSWORD>"
```

Spring Boot maps these environment variables to:

``` text
spring.datasource.url
spring.datasource.username
spring.datasource.password
```

------------------------------------------------------------------------

# 11. Backend Target Group

Create an Application Load Balancer target group for the backend.

Suggested configuration:

  Setting                 Value
  ----------------------- --------------
  Target type             Instances
  Protocol                HTTP
  Port                    8080
  Health check protocol   HTTP
  Health check path       `/api/users`
  Health check port       Traffic port

Initially, the target group can be empty.

After the backend Auto Scaling Group launches instances, instances
should register automatically.

A successful response from `/api/users` indicates that the backend
application is responding.

------------------------------------------------------------------------

# 12. Frontend Target Group

Create a second target group.

  Setting                 Value
  ----------------------- --------------
  Target type             Instances
  Protocol                HTTP
  Port                    80
  Health check protocol   HTTP
  Health check path       `/`
  Health check port       Traffic port

The frontend target group can initially be empty.

------------------------------------------------------------------------

# 13. Backend Launch Template

Create a launch template for backend EC2 instances.

The instances should:

-   Run in private application subnets.
-   Have no public IP.
-   Use `crud-backend-sg`.
-   Install Java 17.
-   Install Maven.
-   Install Git.
-   Clone the public GitHub repository.
-   Build the backend.
-   Start the Spring Boot application on port 8080.

### Example deployment flow

``` bash
sudo apt update
sudo apt install -y git openjdk-17-jdk maven

git clone https://github.com/Shripad191/CRUD-operations.git
cd CRUD-operations/backend

mvn clean package -DskipTests
```

Set the database configuration through environment variables:

``` bash
export SPRING_DATASOURCE_URL="jdbc:mariadb://<RDS_ENDPOINT>:3306/student_db"
export SPRING_DATASOURCE_USERNAME="<DB_USERNAME>"
export SPRING_DATASOURCE_PASSWORD="<DB_PASSWORD>"
```

Start the application:

``` bash
nohup java -jar target/*.jar > /var/log/student-backend.log 2>&1 &
```

### Verify the backend locally on the EC2 instance

``` bash
curl http://localhost:8080/api/users
```

Expected result for a new database:

``` json
[]
```

You can also verify that the process is listening:

``` bash
sudo ss -lntp | grep 8080
```

------------------------------------------------------------------------

# 14. Backend Auto Scaling Group

Create a backend Auto Scaling Group.

Configuration:

-   Launch template: backend launch template
-   Subnets: App-A and App-B
-   Security group: `crud-backend-sg`
-   No public IP
-   Attach to backend target group
-   Desired capacity: use your deployed practice value
-   Minimum capacity: use your deployed practice value
-   Maximum capacity: use your deployed practice value

After launching, verify:

1.  EC2 instance is running.
2.  Instance is registered in the backend target group.
3.  Target becomes healthy.
4.  Java process is running.
5.  Port 8080 is listening.
6.  `/api/users` returns successfully.
7.  Backend can connect to RDS.

------------------------------------------------------------------------

# 15. Bastion Host

A temporary bastion EC2 instance can be used for administration and
troubleshooting.

Place it in:

``` text
Public-A
```

Assign a public IPv4 address.

Attach:

``` text
crud-bastion-sg
```

The bastion should only accept SSH from your current public IP.

### Example troubleshooting flow

``` text
Your computer
     |
     | SSH
     v
Bastion
     |
     | SSH
     v
Private Backend EC2
```

From the backend instance, useful checks include:

``` bash
curl http://localhost:8080/api/users
```

and:

``` bash
sudo tail -f /var/log/student-backend.log
```

The bastion is an administrative component, not an application
component.

------------------------------------------------------------------------

# 16. Frontend Launch Template

The repository frontend uses Vite.

The `package.json` defines:

``` bash
npm run build
```

and the production build is generated in the Vite `dist` directory.

Create a frontend launch template that:

1.  Installs Git.
2.  Installs Node.js and npm.
3.  Installs Nginx.
4.  Clones the public repository.
5.  Installs frontend dependencies.
6.  Sets the production API base URL.
7.  Builds the React application.
8.  Copies the generated `dist` files into Nginx's web root.
9.  Starts/enables Nginx.

Example:

``` bash
sudo apt update
sudo apt install -y git nginx nodejs npm

git clone https://github.com/Shripad191/CRUD-operations.git
cd CRUD-operations/frontend

npm install

export VITE_API_URL="/api"

npm run build

sudo rm -rf /var/www/html/*
sudo cp -r dist/* /var/www/html/

sudo systemctl enable nginx
sudo systemctl restart nginx
```

The repository's frontend API service reads `VITE_API_URL` and appends
endpoints such as `/users` and `/register`. Therefore, setting:

``` text
VITE_API_URL=/api
```

allows the browser to send requests to the same ALB host while the ALB
routes `/api/*` to the backend target group. citeturn8view0

This avoids hard-coding the private backend EC2 address into the
browser.

------------------------------------------------------------------------

# 17. Frontend Auto Scaling Group

Create a frontend Auto Scaling Group.

Configuration:

-   Launch template: frontend launch template
-   Subnets: App-A and App-B
-   Security group: `crud-frontend-sg`
-   No public IP
-   Attach to frontend target group
-   Desired/min/max capacity: use your deployed practice values

Verify:

1.  EC2 instance is running.
2.  Nginx is active.
3.  Port 80 is listening.
4.  `dist` exists.
5.  Instance is registered in the frontend target group.
6.  Target becomes healthy.

Useful checks:

``` bash
sudo systemctl status nginx
```

``` bash
sudo ss -lntp | grep :80
```

``` bash
ls -la /var/www/html
```

------------------------------------------------------------------------

# 18. Application Load Balancer

Create an **internet-facing Application Load Balancer**.

### Availability Zones

Select:

-   Public-A
-   Public-B

### Security group

Use:

``` text
crud-alb-sg
```

### Listener

Create:

``` text
HTTP : 80
```

### Target groups

Attach:

-   Frontend target group
-   Backend target group through a listener rule

------------------------------------------------------------------------

# 19. ALB Listener Rules

The ALB is responsible for exposing one public endpoint while keeping
the EC2 application tier private.

## Default rule

``` text
HTTP : 80
       |
       v
Frontend Target Group
```

## API rule

Create a higher-priority rule:

``` text
IF Path is /api/*
THEN Forward to Backend Target Group
```

The API rule must have a higher priority than the default frontend rule.

### Final routing

``` text
http://<ALB-DNS>/
        |
        +----> Frontend EC2 instances

http://<ALB-DNS>/api/users
        |
        +----> Backend EC2 instances
```

This is the key routing mechanism that allows the React frontend and
Spring Boot backend to share the same public origin.

------------------------------------------------------------------------

# 20. End-to-End Deployment Flow

Once all services are healthy, the complete flow is:

``` text
Browser
   |
   | HTTP : 80
   v
Application Load Balancer
   |
   +----------------------+
   |                      |
   | /api/*               | /
   v                      v
Backend TG            Frontend TG
   |                      |
   v                      v
Backend EC2           Nginx EC2
   |
   | MariaDB : 3306
   v
Amazon RDS
```

Private EC2 instances use:

``` text
Private subnet -> NAT Gateway -> Internet Gateway
```

only for outbound Internet access.

------------------------------------------------------------------------

# 21. Deployment Validation

## 21.1 Check ALB

Copy the ALB DNS name from:

``` text
EC2 Console
→ Load Balancers
→ Your ALB
→ DNS name
```

Open:

``` text
http://<ALB-DNS>/
```

The Student Registration frontend should load.

------------------------------------------------------------------------

## 21.2 Check backend API

Open:

``` text
http://<ALB-DNS>/api/users
```

For a fresh database, an expected response is:

``` json
[]
```

This confirms:

-   ALB API rule works.
-   Backend target is reachable.
-   Spring Boot is running.
-   Backend is listening on 8080.
-   RDS connectivity is working sufficiently for the query.

------------------------------------------------------------------------

## 21.3 Test registration

Submit a student through the frontend.

Example data:

``` text
Name: Test Student
Email: test@example.com
Course: Computer Science
Highest Education: B.Tech
Percentage: 85
Branch: CSE
Mobile Number: 9876543210
```

The frontend should send:

``` text
POST /api/register
```

and the backend should persist the record in RDS.

------------------------------------------------------------------------

## 21.4 Test users endpoint

Open:

``` text
http://<ALB-DNS>/api/users
```

The newly created record should appear in the response.

------------------------------------------------------------------------

## 21.5 Test deletion

Use the Delete action in the frontend.

The frontend calls:

``` text
DELETE /api/users/<id>
```

After deletion, refresh the application or users list and verify that
the record is gone.

------------------------------------------------------------------------

# 22. Validation Checklist

Use this checklist before considering the deployment complete.

### Networking

-   [ ] VPC created with `10.0.0.0/16`
-   [ ] Six subnets created
-   [ ] Two public subnets
-   [ ] Two private application subnets
-   [ ] Two private database subnets
-   [ ] Internet Gateway attached
-   [ ] Public route table configured
-   [ ] Private application route table configured
-   [ ] Database route table contains only local route
-   [ ] NAT Gateway created in Public-A
-   [ ] Elastic IP attached to NAT Gateway

### Security

-   [ ] ALB accepts HTTP from Internet
-   [ ] Frontend accepts port 80 only from ALB security group
-   [ ] Backend accepts 8080 only from ALB security group
-   [ ] Backend SSH accepts traffic only from bastion security group
-   [ ] RDS accepts 3306 only from backend security group
-   [ ] Bastion SSH is restricted to your public IP
-   [ ] No RDS port 3306 rule from `0.0.0.0/0`
-   [ ] No application EC2 public IPs

### Compute

-   [ ] Backend launch template created
-   [ ] Backend ASG spans App-A and App-B
-   [ ] Frontend launch template created
-   [ ] Frontend ASG spans App-A and App-B
-   [ ] Backend target is healthy
-   [ ] Frontend target is healthy
-   [ ] Nginx is serving frontend files
-   [ ] Spring Boot is listening on 8080

### Database

-   [ ] RDS MariaDB is available
-   [ ] RDS is in the DB subnet group
-   [ ] RDS is not publicly accessible
-   [ ] Database is named `student_db`
-   [ ] Backend can connect to RDS

### Load balancing

-   [ ] ALB is internet-facing
-   [ ] ALB spans both public subnets
-   [ ] HTTP listener exists on port 80
-   [ ] `/api/*` rule forwards to backend
-   [ ] Default rule forwards to frontend
-   [ ] API rule has higher priority than default rule

### Application

-   [ ] `/` loads the React application
-   [ ] `/api/users` responds
-   [ ] Registration works
-   [ ] User list updates
-   [ ] Delete works
-   [ ] Frontend communicates through the ALB

------------------------------------------------------------------------

# 23. Troubleshooting

## Backend target is unhealthy

Check:

``` bash
sudo systemctl status nginx
```

For the backend:

``` bash
sudo ss -lntp | grep 8080
```

Then:

``` bash
curl http://localhost:8080/api/users
```

Verify:

-   RDS is available.
-   Backend is running.
-   Backend listens on `0.0.0.0:8080`.
-   Backend security group permits 8080 from `crud-alb-sg`.
-   RDS security group permits 3306 from `crud-backend-sg`.
-   Target group health check path is correct.
-   Backend application logs contain no database connection errors.

------------------------------------------------------------------------

## Frontend target is unhealthy

Check:

``` bash
sudo systemctl status nginx
```

Then:

``` bash
curl http://localhost/
```

Verify:

-   Nginx is running.
-   Port 80 is listening.
-   `/var/www/html` contains the built frontend.
-   `dist` was created by `npm run build`.
-   Frontend security group permits HTTP from the ALB security group.
-   Target group health check path is `/`.

------------------------------------------------------------------------

## API returns 404

Check the ALB listener rule:

``` text
Path: /api/*
Target: Backend Target Group
```

Make sure the API rule has a higher priority than the default frontend
rule.

Also verify the frontend build used:

``` bash
VITE_API_URL=/api
```

The repository's API service constructs requests using this variable,
for example `/users`, `/register`, and `/users/<id>`. citeturn8view0

------------------------------------------------------------------------

## Database connection fails

Verify:

``` text
RDS endpoint
Database name = student_db
Database username
Database password
Port = 3306
```

Also verify:

-   RDS is private.
-   Backend and RDS are in the same VPC.
-   RDS is attached to the intended DB subnet group.
-   RDS security group allows 3306 from the backend security group.
-   The backend is using environment variables with the correct values.

From the backend instance, basic network troubleshooting can include:

``` bash
nc -vz <RDS_ENDPOINT> 3306
```

Do not expose port 3306 to the Internet just to make troubleshooting
easier.

------------------------------------------------------------------------

## Frontend loads but registration fails

Check the browser's Developer Tools:

``` text
Developer Tools
→ Network
→ register request
```

Confirm that the request is going to:

``` text
/api/register
```

and not:

``` text
http://<PRIVATE_BACKEND_IP>:8080/register
```

Also verify:

``` text
VITE_API_URL=/api
```

was set **before** running:

``` bash
npm run build
```

Vite embeds build-time environment variables into the generated frontend
bundle.

------------------------------------------------------------------------

# 24. Security Notes

This deployment is suitable as a learning/practice architecture. It
should not be treated as a production-ready security baseline without
additional hardening.

## Secrets

Never commit:

-   Database passwords
-   Private keys
-   `.pem` files
-   AWS access keys
-   Secret tokens
-   `.env` files containing credentials

### Critical repository cleanup

The current public repository contains database connection information
in `backend/src/main/resources/application.properties`, including a
plaintext password. This credential should be considered exposed.

Before reusing the environment:

1.  Change/rotate the database password in RDS.
2.  Remove the plaintext credential from the source code.
3.  Replace hard-coded credentials with environment variables or AWS
    Secrets Manager.
4.  Remove the secret from Git history if it was committed in earlier
    revisions.
5.  Review GitHub secret scanning/security alerts.
6.  Check other branches and commits for the same credential.

Changing the password alone is not sufficient if the old password
remains in Git history.

------------------------------------------------------------------------

# 25. Recommended Production Improvements

The practice deployment intentionally uses a simple architecture. A
production design could improve it with:

-   HTTPS using ACM certificates.
-   HTTP-to-HTTPS redirect.
-   Route 53 custom domain.
-   AWS Secrets Manager for database credentials.
-   IAM roles instead of embedded AWS credentials.
-   CloudWatch Logs and metrics.
-   CloudWatch alarms.
-   VPC Flow Logs.
-   AWS WAF on the ALB.
-   Separate NAT Gateway per Availability Zone.
-   Multi-AZ RDS deployment where appropriate.
-   RDS automated backups and retention.
-   Encrypted RDS storage.
-   Systems Manager Session Manager instead of a bastion host.
-   Immutable AMIs or a CI/CD pipeline for application deployment.
-   Infrastructure as Code using Terraform or AWS CloudFormation.
-   Automated testing before deployment.
-   Dependency and container/image vulnerability scanning where
    applicable.

------------------------------------------------------------------------

# 26. Cost Considerations

This practice architecture contains several AWS resources that can incur
charges.

The most important resources to clean up after testing include:

-   NAT Gateway
-   Elastic IP
-   RDS
-   EC2 instances
-   Application Load Balancer
-   Load balancer target resources
-   CloudWatch resources if additional monitoring is configured

The single NAT Gateway is intentionally used to reduce practice cost,
but NAT Gateway hourly and data-processing charges can still accumulate.

------------------------------------------------------------------------

# 27. Cleanup Procedure

When the practice deployment is complete, clean up in dependency order.

### 1. Scale Auto Scaling Groups down

Reduce:

``` text
Frontend ASG -> 0
Backend ASG -> 0
```

Then delete both Auto Scaling Groups.

### 2. Delete the Application Load Balancer

Delete:

``` text
ALB
```

### 3. Delete target groups

Delete:

``` text
Frontend Target Group
Backend Target Group
```

### 4. Delete bastion

Terminate the temporary bastion EC2 instance.

### 5. Delete RDS

Delete the RDS MariaDB instance according to your backup requirements.

Then delete:

``` text
RDS subnet group
```

### 6. Delete NAT Gateway

Delete the NAT Gateway.

Wait until it is fully deleted.

### 7. Release Elastic IP

Release the Elastic IP that was allocated for the NAT Gateway.

### 8. Delete route tables

Delete custom:

``` text
Public route table
Private application route table
Database route table
```

### 9. Delete subnets

Delete:

``` text
Public-A
Public-B
App-A
App-B
DB-A
DB-B
```

### 10. Delete Internet Gateway

Detach the Internet Gateway from the VPC, then delete it.

### 11. Delete security groups

Delete the custom security groups after dependent resources have been
removed.

### 12. Delete the VPC

Finally delete the practice VPC.

> **Warning:** Do not accidentally delete the AWS default VPC if you use
> it for other projects.

------------------------------------------------------------------------

# 28. Suggested Repository Documentation Structure

A clean repository structure could be:

``` text
CRUD-operations/
├── backend/
├── frontend/
├── test/
├── compose.yml
├── README.md
└── AWS-Deployment.md
```

The `AWS-Deployment.md` file should focus specifically on:

-   AWS architecture
-   Networking
-   Security groups
-   RDS
-   EC2 Auto Scaling
-   ALB routing
-   Deployment validation
-   Troubleshooting
-   Cleanup
-   Screenshots

Keep normal application-development instructions in `README.md`.

------------------------------------------------------------------------

# 29. AWS Deployment Screenshots

The following screenshots are recommended for documenting the
deployment.

## Essential screenshots

### 01 --- VPC overview

Show:

-   VPC name
-   VPC ID
-   CIDR `10.0.0.0/16`
-   Region

Suggested filename:

``` text
docs/aws/01-vpc-overview.png
```

------------------------------------------------------------------------

### 02 --- Subnets

Show all six subnets in one AWS Console view if possible.

The screenshot should clearly show:

``` text
Public-A   10.0.1.0/24
Public-B   10.0.2.0/24
App-A      10.0.11.0/24
App-B      10.0.12.0/24
DB-A       10.0.21.0/24
DB-B       10.0.22.0/24
```

Suggested filename:

``` text
docs/aws/02-subnets.png
```

------------------------------------------------------------------------

### 03 --- Route tables

Show the three route tables and their subnet associations.

Suggested filename:

``` text
docs/aws/03-route-tables.png
```

------------------------------------------------------------------------

### 04 --- NAT Gateway

Show:

-   NAT Gateway
-   Public subnet
-   Elastic IP
-   Available state

Suggested filename:

``` text
docs/aws/04-nat-gateway.png
```

------------------------------------------------------------------------

### 05 --- Security groups

Show the inbound rules for:

-   `crud-alb-sg`
-   `crud-frontend-sg`
-   `crud-backend-sg`
-   `crud-rds-sg`
-   `crud-bastion-sg`

If one screenshot cannot clearly show everything, use two or three
screenshots.

Suggested filenames:

``` text
docs/aws/05-security-groups.png
docs/aws/05b-security-group-rules.png
```

------------------------------------------------------------------------

### 06 --- RDS

Show:

-   MariaDB engine
-   Available status
-   DB instance
-   Connectivity/security group section
-   Private access configuration

Do **not** show the database password.

Suggested filename:

``` text
docs/aws/06-rds-mariadb.png
```

------------------------------------------------------------------------

### 07 --- Backend target group

Show:

-   Target group name
-   Port 8080
-   Health check path
-   Healthy backend targets

Suggested filename:

``` text
docs/aws/07-backend-target-group.png
```

------------------------------------------------------------------------

### 08 --- Frontend target group

Show:

-   Target group name
-   Port 80
-   Health check path `/`
-   Healthy frontend targets

Suggested filename:

``` text
docs/aws/08-frontend-target-group.png
```

------------------------------------------------------------------------

### 09 --- Auto Scaling Groups

Show both:

-   Backend ASG
-   Frontend ASG

Include desired/min/max capacity and subnet configuration where
possible.

Suggested filename:

``` text
docs/aws/09-auto-scaling-groups.png
```

------------------------------------------------------------------------

### 10 --- EC2 instances

Show the running instances and their subnet/private-IP information.

The screenshot should demonstrate that application instances are
private.

Suggested filename:

``` text
docs/aws/10-private-ec2-instances.png
```

------------------------------------------------------------------------

### 11 --- Application Load Balancer

Show:

-   Internet-facing scheme
-   ALB state: Active
-   Availability Zones
-   Security group
-   Listener port 80

Suggested filename:

``` text
docs/aws/11-load-balancer.png
```

------------------------------------------------------------------------

### 12 --- ALB listener rules

This is one of the most important screenshots.

Show:

``` text
Priority 1:
Path /api/* -> Backend Target Group

Default:
-> Frontend Target Group
```

Suggested filename:

``` text
docs/aws/12-alb-listener-rules.png
```

------------------------------------------------------------------------

### 13 --- ALB DNS and deployed frontend

Open:

``` text
http://<ALB-DNS>/
```

Show the working Student Registration application.

Suggested filename:

``` text
docs/aws/13-live-frontend.png
```

------------------------------------------------------------------------

### 14 --- API response

Open:

``` text
http://<ALB-DNS>/api/users
```

Show the successful JSON response.

Suggested filename:

``` text
docs/aws/14-api-response.png
```

------------------------------------------------------------------------

### 15 --- CRUD operation

Show the application after creating a student.

The screenshot should demonstrate that the record appears in the users
table.

Suggested filename:

``` text
docs/aws/15-crud-create.png
```

A second application screenshot can show deletion:

``` text
docs/aws/16-crud-delete.png
```

------------------------------------------------------------------------

# 30. Recommended Screenshot Order in GitHub

For a professional project presentation, show the screenshots in this
order:

``` text
1. Architecture / VPC
2. Subnets
3. Route tables
4. NAT Gateway
5. Security groups
6. RDS
7. Backend target group
8. Frontend target group
9. Auto Scaling Groups
10. Private EC2 instances
11. Application Load Balancer
12. ALB listener rules
13. Live frontend
14. API response
15. CRUD operation
```

This tells a clear story:

``` text
Network
  ↓
Security
  ↓
Database
  ↓
Compute
  ↓
Load Balancer
  ↓
Application
  ↓
CRUD validation
```

------------------------------------------------------------------------

# 31. Screenshot Safety Checklist

Before committing screenshots to GitHub, verify that they do **not**
expose:

-   Database password
-   AWS access key
-   AWS secret key
-   EC2 private key
-   `.pem` file
-   Secret Manager secret values
-   Environment variable values containing credentials
-   RDS username/password combination
-   Shell history containing passwords
-   Session tokens
-   Authorization headers

It is generally fine to show:

-   AWS resource names
-   VPC CIDR
-   Subnet CIDRs
-   Security group names
-   Target group names
-   ALB DNS name
-   Private IP addresses for a portfolio/demo, although they can be
    redacted if preferred
-   AWS region

------------------------------------------------------------------------

# 32. Final Deployment Result

The completed practice deployment provides a multi-tier AWS
architecture:

``` text
                         Internet
                            |
                            v
                     Internet Gateway
                            |
                            v
                  Public Subnets
              +-----------------------+
              | Application Load      |
              | Balancer              |
              +-----------+-----------+
                          |
             +------------+------------+
             |                         |
             v                         v
      Frontend Target            Backend Target
             |                         |
       +-----+-----+             +-----+-----+
       |           |             |           |
       v           v             v           v
    App-A       App-B         App-A       App-B
    Nginx       Nginx       Spring Boot  Spring Boot
                                  |
                                  v
                         Private DB Subnets
                           +-----------+
                           | RDS MariaDB|
                           +-----------+

Private App Subnets
        |
        v
   NAT Gateway
        |
        v
 Internet Gateway
```

The key architectural properties are:

-   Public ALB
-   Private application EC2 instances
-   Private RDS database
-   Security-group-based traffic control
-   Two Availability Zones for application resources
-   Auto Scaling for frontend and backend
-   NAT-based outbound Internet access
-   Path-based ALB routing
-   Single public endpoint for the React frontend and Spring Boot API

------------------------------------------------------------------------

## 33. Deployment Summary

  Component                    AWS Service / Configuration
  ---------------------------- -----------------------------
  Network                      Amazon VPC
  VPC CIDR                     `10.0.0.0/16`
  Availability Zones           2
  Public subnets               2
  Private app subnets          2
  Private DB subnets           2
  Internet access              Internet Gateway
  Private outbound access      NAT Gateway
  Frontend                     EC2 ASG + Nginx
  Backend                      EC2 ASG + Spring Boot
  Database                     Amazon RDS MariaDB
  Load balancing               Application Load Balancer
  Frontend port                80
  Backend port                 8080
  Database port                3306
  Frontend routing             Default ALB rule
  API routing                  `/api/*` ALB rule
  Database access              Backend security group only
  Administration               Temporary bastion
  Deployment style             Private EC2 + ALB
  Practice cost optimization   Single NAT Gateway

------------------------------------------------------------------------

## 34. Evidence of Successful Deployment

A successful deployment should be demonstrated with all of the
following:

``` text
✓ VPC and six subnets created
✓ Public/private routing configured
✓ NAT Gateway working
✓ Security groups restricting traffic
✓ RDS MariaDB available
✓ Backend EC2 instances healthy
✓ Frontend EC2 instances healthy
✓ ALB active
✓ /api/* routed to backend
✓ / routed to frontend
✓ /api/users returns successfully
✓ Registration persists data
✓ Delete operation removes data
✓ Application accessible through ALB DNS
```

This deployment demonstrates practical knowledge of AWS networking, EC2,
Auto Scaling, RDS, security groups, NAT, load balancing, Linux
administration, Spring Boot deployment, React/Vite production builds,
and end-to-end troubleshooting.
