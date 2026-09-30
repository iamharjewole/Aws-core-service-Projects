# The Challenge

Our organization needs to establish a connection with Vendor A to access their services. However, Vendor A has strict security requirements that present a challenge for our current infrastructure setup.

## Vendor A's Requirements

Secure Connection: All communication must occur over a secure AWS site-to-site VPN connection. This ensures data confidentiality and integrity during transmission.

Private IP Whitelisting: To access their resources within the VPN, Vendor A insists on whitelisting only a single, specific private IP address. This means that all requests originating from our organization must appear to come from this pre-approved IP address.

### Our Current Setup

Kubernetes Cluster: Our applications and services run within a Kubernetes cluster deployed in a private subnet within our AWS environment. This provides an additional layer of security as our services are not directly exposed to the public internet.

Network Address Translation (NAT) Gateway: We utilize a NAT Gateway to enable our services in the private subnet to initiate outbound connections to the internet while preventing unsolicited inbound traffic.

NGINX Ingress Controller: An NGINX Ingress Controller manages incoming traffic to our Kubernetes cluster, routing requests to the appropriate services.

### The Problem

The combination of Vendor A's security requirements and our existing infrastructure creates a conflict. Our Kubernetes services, residing in a private subnet, do not have individual public IP addresses. Additionally, the NAT Gateway, while enabling outbound connections, might not provide the necessary level of IP address control to satisfy Vendor A's whitelisting requirement.

How can we configure our environment to meet Vendor A's security requirements and enable our Kubernetes services to access their resources through the VPN connection while adhering to their private IP whitelisting constraint?

### **Solution**

I would solve this problem by using an AWS Site-to-Site VPN together with a dedicated private IP address for communication with Vendor A.

My Kubernetes cluster is deployed in a private subnet, so the Kubernetes services do not have public IP addresses. At the moment, the private subnet can use the NAT Gateway for outbound internet access. However, I would not send Vendor A's traffic through the NAT Gateway because Vendor A requires the connection to use a private IP over the VPN.

I would configure the network so that traffic going to Vendor A's private network is routed through the Site-to-Site VPN.

The design would look like this:

    Kubernetes Pods 
        | 
        v 
    Private Subnet 
        | 
        v 
    Dedicated Private IP (e.g. 10.0.10.50) 
        | 
        v 
    AWS Site-to-Site VPN 
        | 
        v 
    Vendor A VPN Gateway 
        | 
        v 
    Vendor A Services

I would use a dedicated private IP, for example 10.0.10.50, as the source IP for the traffic going to Vendor A. Vendor A would then whitelist this IP address.

The important part is that Kubernetes Pod IP addresses can change, so I would not ask Vendor A to whitelist individual Pod addresses. Instead, I would use a centralized source-NAT solution so that the different Kubernetes workloads appear to Vendor A as the same approved private IP.

**Routing**

I would also create a specific route for Vendor A's network in the route table associated with the Kubernetes private subnet.

For example, if Vendor A's network is:

172.20.0.0/16

the route would direct that traffic toward the VPN:

Destination       Target
172.20.0.0/16     VPN/TGW
0.0.0.0/0         NAT Gateway

This means that normal internet traffic from my Kubernetes services can continue to use the existing NAT Gateway, while traffic intended for Vendor A follows the VPN.

    Kubernetes
    | 
    +--Vendor A traffic --> Site-to-Site VPN 
    | 
    +---- Internet traffic ----> NAT Gateway

**Role of NGINX**

The NGINX Ingress Controller would not normally be responsible for this connection because NGINX handles incoming traffic into my Kubernetes cluster.

The Vendor A connection is an outbound connection, so the important components are the Kubernetes network, route tables, source NAT, and Site-to-Site VPN.

**Why this meets Vendor A's requirements**

This design satisfies the requirements because:

The connection to Vendor A travels through an encrypted AWS Site-to-Site VPN.
Kubernetes remains in the private subnet.
Vendor A only needs to whitelist one private IP address.
Kubernetes Pods do not need individual public IP addresses.
Normal internet traffic can continue using my existing NAT Gateway.
Vendor A traffic can be separated from normal internet traffic using routing rules.
The source IP presented to Vendor A remains controlled and predictable.

Therefore, for my AWS project, I would use specific routing + centralized source NAT + AWS Site-to-Site VPN to make all Kubernetes requests to Vendor A appear to originate from one approved private IP address.
