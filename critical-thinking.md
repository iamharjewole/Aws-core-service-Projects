# Critical Inquiry

As a cloud architect, responsible for crafting AWS cloud solutions for two distinct company websites, how can reverse proxy technology be strategically implemented to enhance security, scalability, and performance while ensuring optimal resource utilization and cost-efficiency? Design and deploy an infrastructure on AWS cloud for 2 company websites utilizing reverse proxy technology.

## Scenario

You are part of a team charged with migrating two company websites to the AWS cloud infrastructure. The first website is an e-commerce platform managing sensitive customer information, while the second website is a content management system (CMS) used for publishing articles and blogs. Both websites demand robust security protocols, scalable architecture to handle varying traffic loads, and efficient resource management to minimize costs. How can you employ reverse proxy technology within the AWS environment to meet these requirements effectively for each website while optimizing their performance and security?

### Project Goals

Understand the Project: Comprehend the project requirements from the provided scenario and infrastructure diagram.

Enhanced Security: Utilize reverse proxy technology to act as a shield between the internet and the company websites. This adds an additional layer of security by inspecting and filtering incoming traffic before reaching the web servers.

Scalability: Design the infrastructure to dynamically scale based on traffic demands. Use AWS Auto Scaling groups and Elastic Load Balancers (ELBs) in conjunction with reverse proxy servers to efficiently distribute incoming requests across multiple web servers.

Performance Optimization: Implement caching mechanisms within the reverse proxy servers to store frequently accessed content. This will reduce latency and improve response times for end-users.

Resource Utilization: Optimize resource allocation by leveraging AWS services such as Amazon EC2 for web servers, Amazon RDS for databases, and AWS Lambda for serverless functionalities. Ensure efficient utilization of compute resources to maintain cost-effectiveness.

High Availability: Configure redundancy and failover mechanisms within the infrastructure to ensure high availability of the company websites. Minimize downtime and ensure uninterrupted service for end-users.

### Solution

For my project, I would design an AWS environment for two company websites: an e-commerce website and a CMS/blog website. The e-commerce website would handle sensitive customer and transaction information, while the CMS would mainly publish articles, images and blog content.

My main objective would be to make both websites secure, highly available, scalable and cost-effective without giving every server direct exposure to the Internet.

Architecture I would use

Proposed AWS architecture

                INTERNET
                    |
            Amazon 53 DNS records
             /                  \
            /                    \
    shop.company.com       blog.mycompany.com
            |               |
        CloudFront      CloudFront  
            |               |
            WAF            WAF
            |               |
    Application Load      Application Load
    Balancer (ALB)        Balancer (ALB)
            |                       |
    Reverse Proxy Layer   Reverse Proxy Layer
        Nginx EC2           Nginx EC2
            |                   |
     +------+-----+         +-----+-----+
     |            |         |           |
    App Server App Ser.   CMS Ser.  CMS Ser.
    |          |            |          |
    +----------+            +----------+
         |                       |
    E-commerce RDS             CMS RDS

I would place these resources inside my AWS VPC and separate them into public and private subnets.

For example, my VPC could be:

    VPC: 10.0.0.0/16

    Public Subnets
    ├── Public-AZ-A
    └── Public-AZ-B

    Private Application Subnets
    ├── Private-App-AZ-A
    └── Private-App-AZ-B

    Private Database Subnets
    ├── Private-DB-AZ-A
    └── Private-DB-AZ-B

This is similar to the VPC architecture I have been working with in my other AWS exercises, where I separated public-facing resources from private EC2 instances.

1. **My e-commerce website**

    For my e-commerce website, security would be my highest priority because the application handles customer information.

    I would not expose the EC2 application servers directly to the Internet.

    Instead, the request would follow this path:

        Customer
            |
            v
        Route 53
            |
            v
        AWS WAF
            |
            v
        Application Load Balancer
            |
            v
        Nginx Reverse Proxy
            |
            v
        E-commerce EC2
            |
            v
        Amazon RDS

    The Nginx reverse proxy would act as an additional layer between users and my application.

    For example, if a customer visits:

        https://shop.mycompany.com

    the customer does not directly communicate with my application server.

    Instead:

        Customer
        ↓
        ALB
        ↓
        Nginx
        ↓
        E-commerce application

    This means that the private IP address of my application server does not need to be exposed publicly.

    Why this is useful

    If someone tried to access my application server directly, the security group would prevent the connection.

    I could configure the security groups so that:

        Internet → ALB HTTPS  443  ALLOWED

        ALB→ Nginx/EC2 App  port  ALLOWED

        Nginx→ RDS  Database port ALLOWED

        Internet→  RDS            DENIED

    This gives me multiple layers of protection.

2. **My CMS website**

    My second website would be a CMS used for publishing articles and blogs.

    The CMS doesn't handle the same level of sensitive transaction information as my e-commerce platform, so I can focus more heavily on performance and caching.

    The request could follow:

        User
        |
        v
        Route 53
        |
        v
        CloudFront
        |
        v
        AWS WAF
        |
        v
        ALB
        |
        v
        Nginx Reverse Proxy
        |
        v
        CMS EC2
        |
        v
        CMS RDS

    For example, if someone visits:

        https://blog.mycompany.com/aws-tutorial

    CloudFront can serve a cached version of the article instead of sending every request all the way to my EC2 server.

    This is particularly useful for my CMS because the same articles may be requested by thousands of visitors.

3. **How I would use Nginx as the reverse proxy**

    On my EC2 instances, I could install Nginx:

        sudo dnf update -y
        sudo dnf install nginx -y
        sudo systemctl enable nginx
        sudo systemctl start nginx

    I would then configure Nginx to forward requests to my application.

    For example:

        server {
            listen 80;
            server_name shop.mycompany.com;

        location / {
            proxy_pass http://ecommerce_backend;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            }
        }

        upstream ecommerce_backend {
            server 10.0.11.10:8080;
            server 10.0.12.10:8080;

        }
