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

    The important point for my project is that Nginx receives the request first and then forwards it internally.

    Therefore, it acts as a reverse proxy rather than exposing my application directly.

4. **Making my websites scalable**

    I would not depend on a single EC2 instance.

    For example:

                 ALB
              /       \
             /         \
        EC2/Nginx    EC2/Nginx
             \         /
              \       /
             Application

    I would put these instances into an Auto Scaling Group.

    If my website normally receives 100 users but suddenly receives 5,000 users because of a promotion or popular article, Auto Scaling can launch additional instances.

    For example:

        Normal traffic:

        EC2 ── EC2


        High traffic:

        EC2 ── EC2 ── EC2 ── EC2 ── EC2

    When traffic decreases, unnecessary instances can be terminated.

    This is important for my project because I don't want to pay for five EC2 instances when I only need two.

5. **High availability**

    I would also avoid putting all my servers inside one Availability Zone.

    Instead:

            Application Load Balancer
                       /       \
                      /         \
                   AZ-A         AZ-B
                    |             |
                  EC2           EC2
                    |             |
                    +------+------+
                           |
                          RDS

    If an EC2 instance in AZ-A fails, the ALB can send traffic to the healthy instance in AZ-B.

    I would also use an appropriate Multi-AZ configuration for my RDS database.

    This means my websites are not dependent on one EC2 instance or one Availability Zone.

6. **Security in my project**

    For my e-commerce website, I would use several security layers:

        Internet
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
        Private EC2
            |
            v
        Private RDS

    I would use AWS WAF to inspect incoming web requests and help protect against common web attacks.

    I would also use AWS Certificate Manager for HTTPS certificates.

    Therefore:

        HTTP
        ↓
        HTTPS
        ↓
        Encrypted communication

    My RDS database would remain in a private subnet and would not have a public IP address.

    This is especially important because my e-commerce application contains sensitive customer information.

7. **Performance optimization**

    For my CMS, I would make greater use of caching.

    For example:

        User
        |
        v
        CloudFront
        |
        +---- Cache HIT → Return article
        |
        +---- Cache MISS
          |
          v
         ALB
          |
         Nginx
          |
        CMS EC2

    If 1,000 people request the same blog article, CloudFront doesn't necessarily need to send all 1,000 requests to my EC2 servers.

    It can serve the cached content closer to the users.

    I could also store static content such as:

    - Images
    - CSS
    - JavaScript
    - Documents

    in Amazon S3 and use CloudFront to deliver them.

8. **Cost optimization in my project**

    Cost is another important consideration for me.

    I would avoid simply launching large EC2 instances and leaving them running permanently.

    Instead, I would use:

    **Auto Scaling**: to automatically increase or decrease the number of instances.

    I would use:

    **CloudFront caching**: to reduce unnecessary requests reaching my EC2 servers.

    I would use:

    **S3**: for inexpensive storage of static files.

    I would use:

    **CloudWatch**: to monitor CPU, memory, network traffic and application performance.

    Then I can make decisions about whether my EC2 instances are over-sized or under-sized.

    For predictable workloads, I could also consider Savings Plans or Reserved Instances.

9. **How this connects to my previous AWS project**

    This architecture builds on the AWS concepts I have already been working with.

    For example, I previously worked with a public EC2 instance acting as a bastion host and a private EC2 instance. I also worked with a NAT Gateway for private resources that need outbound Internet access.

    I can apply the same principle here:

                    INTERNET
                       |
                  Public Layer
                       |
               ALB / Bastion
                       |
                Private Layer
                       |
                 Application
                       |
                 Private RDS
    The important distinction is that a bastion host is for administrative SSH access, whereas the reverse proxy/load-balancing layer is for handling application traffic.

    So I would not use my bastion host as the website's reverse proxy. I would keep those responsibilities separate.

### **My final design**

My completed AWS architecture would therefore look like this:

                        INTERNET
                            |
                       Route 53
                     /           \
                    /             \
             E-commerce           CMS
                  |                 |
             CloudFront        CloudFront
                  |                 |
                 WAF               WAF
                  |                 |
                 ALB               ALB
                  |                 |
             Nginx/EC2        Nginx/EC2
              /      \          /      \
            EC2      EC2      EC2      EC2
              \      /          \      /
             Auto Scaling     Auto Scaling
                  |                 |
                RDS               RDS
             Private DB         Private DB

### **Conclusion**

For my project, reverse proxy technology provides an important middle layer between Internet users and my application servers. Nginx can receive and forward requests, while AWS services such as ALB, WAF, CloudFront and Auto Scaling provide additional security, scalability and performance.

The e-commerce website would receive stronger security controls because it handles sensitive customer information, while the CMS would make heavier use of caching and CloudFront because its content is more suitable for caching.

By placing my application and databases in private subnets, distributing resources across Availability Zones, automatically scaling EC2 instances and using caching to reduce unnecessary requests, I can build an AWS architecture that is secure, highly available, scalable and cost-efficient without wasting resources.
