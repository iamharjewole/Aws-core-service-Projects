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

For this project, I would implement one AWS architecture that hosts both websites separately while using reverse-proxy/load-balancing layers to protect and distribute traffic to the application servers.

Proposed AWS architecture

                INTERNET
                    |
            Amazon Route 53
                DNS records
             /                  \
            /                    \
    shop.company.com       blog.company.com
            |               |
        AWS WAF            AWS WAF
            |               |
    Application Load      Application Load
    Balancer (ALB)        Balancer (ALB)
        |                       |
    Reverse Proxy Layer   Reverse Proxy Layer
    
    EC2 Auto Scaling Group  EC2 Auto Scaling Group
        (Nginx/Apache)      (Nginx/Apache)
         /    \                /    \
        /      \              /      \
    Web/App 1 Web/App 2 CMS/App 1  CMS/App 2
    |          |          |          |
    +----------+          +----------+
        |                       |
    Amazon RDS              Amazon RDS
    (E-commerce DB)          (CMS DB)

For production, I would place the databases in private subnets, while the public-facing load balancers are in public subnets and the EC2 application/reverse-proxy instances are isolated from direct Internet access as much as practical.

**How reverse proxy helps**

For the e-commerce website, Nginx can act as the reverse proxy in front of the application servers. It can:

- Hide the private IP addresses of the application servers.
- Accept HTTPS connections and forward requests internally.
- Reject malformed or unwanted requests.
- Rate-limit suspicious traffic.
- Cache suitable static content.
- Compress responses.
- Distribute requests across application instances.
- Prevent users from directly accessing the application servers.

For example:

    server {
    listen 80;
    server_name shop.company.com;

    location / {
        proxy_pass http://ecommerce_backend;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    
        }
    }

    upstream ecommerce_backend {
        server 10.0.2.10:8080;
        server 10.0.2.11:8080;
    }

For the CMS website, the same concept can be used, but caching becomes particularly useful because articles, images, CSS, JavaScript and blog pages are frequently requested.

    server {
    listen 80;
    server_name blog.company.com;

    location / {
        proxy_pass http://cms_backend;
        proxy_cache my_cache;
        proxy_cache_valid 200 10m;
        
        }
    }

    AWS services I would use

|Requirement|AWS service|
|----------|------------|
|DNS|Route 53|
|Reverse proxy/load balancing| Application Load Balancer + Nginx|
|Web/application servers|EC2|
|Automatic scaling|EC2 Auto Scaling|
|Web protection|AWS WAF|
|HTTPS certificates|AWS Certificate Manager|
|E-commerce database|Amazon RDS|
|CMS database|Amazon RDS|
|Static files|Amazon S3|
|Global caching|CloudFront|
|Monitoring|CloudWatch|
|Private networking|Amazon VPC|
|Secrets|AWS Secrets Manager|
|Serverless tasks|AWS Lambda|

**Security design**

I would divide the VPC into public and private subnets across at least two Availability Zones:

    VPC: 10.0.0.0/16

    AZ-A                        AZ-B

    Public subnet             Public subnet
    10.0.1.0/24               10.0.2.0/24
    |                             |
    +------ Application Load -----+
            Balancer

    Private subnet            Private subnet
    10.0.11.0/24              10.0.12.0/24
    |                             |
    EC2/Nginx                 EC2/Nginx
    |                             |
    Application servers  Application servers

    Private DB subnet      Private DB subnet
    10.0.21.0/24           10.0.22.0/24
       \                       /
                Amazon RDS

The security groups would follow a simple principle:

        Internet
        |
        v
    ALB
        |
        | TCP 80/443
        v
    Nginx/Application EC2
        |
        | TCP 3306/5432
        v
        RDS

The database should not accept connections from the Internet. Its security group should only permit connections from the application tier.

**Scalability**

Each website can have its own Auto Scaling Group:

        E-commerce ASG
            |
        +---+---+
        |       |
        EC2     EC2
        \       /
         \     /
            ALB

and:

        CMS ASG
            |
        +---+---+
        |       |
        EC2     EC2
         \       /
          \     /
            ALB
            
if traffic increases, Auto Scaling launches additional EC2 instances. When traffic falls, unnecessary instances can be terminated.

This avoids permanently running a large number of servers and therefore improves resource utilization and cost efficiency.

**Performance optimization**

For the CMS, I would place CloudFront in front of the application because blog content often contains cacheable objects.

        User
        |
        v
        CloudFront
        |
        +---- cached content
        |
        +---- cache miss
          |
          v
         ALB
          |
        Nginx
          |
        CMS EC2

Static files such as:

- images
- CSS
- JavaScript
- downloadable documents

can also be stored in Amazon S3 and delivered through CloudFront.

For the e-commerce site, caching needs more care. Public product images and static assets can be cached, but sensitive information such as shopping carts, account pages and payment-related responses should not be cached publicly.

**High availability**

I would deploy the infrastructure across multiple Availability Zones rather than relying on a single EC2 instance.

For example:

                 Route 53
                    |
                 CloudFront
                    |
                   ALB
              _____|_____
             /           \
          AZ-A           AZ-B
           |               |
        EC2/Nginx       EC2/Nginx
           |               |
           +-------+-------+
                   |
                  RDS
             Multi-AZ setup

If an EC2 instance fails, the load balancer can stop sending traffic to it and Auto Scaling can replace it.

If an Availability Zone experiences a problem, resources in the other AZ can continue serving traffic.

**Cost optimization**

I would avoid creating unnecessarily large EC2 instances. Instead:

- Start with appropriately sized instances.
- Monitor CPU, memory and network utilization with CloudWatch.
- Configure Auto Scaling.
- Scale out during traffic peaks.
- Scale in during quiet periods.
- Use S3 for inexpensive object storage.
- Use CloudFront caching to reduce requests reaching EC2.
- Use RDS rather than maintaining database servers manually.
- Consider Savings Plans/Reserved capacity for predictable workloads.
- Use Spot Instances only for workloads that can tolerate interruption.
