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

- The NGINX Ingress Controller would not normally be responsible for this connection because NGINX handles incoming traffic into my Kubernetes cluster.

- The Vendor A connection is an outbound connection, so the important components are the Kubernetes network, route tables, source NAT, and Site-to-Site VPN.

**Why this meets Vendor A's requirements**

This design satisfies the requirements because:

- The connection to Vendor A travels through an encrypted AWS Site-to-Site VPN.

- Kubernetes remains in the private subnet.

- Vendor A only needs to whitelist one private IP address.

- Kubernetes Pods do not need individual public IP addresses.

- Normal internet traffic can continue using my existing NAT Gateway.

- Vendor A traffic can be separated from normal internet traffic using routing rules.

- The source IP presented to Vendor A remains controlled and predictable.

Therefore, for my AWS project, I would use specific routing + centralized source NAT + AWS Site-to-Site VPN to make all Kubernetes requests to Vendor A appear to originate from one approved private IP address.

### My Personalized Scenario

Imagine that I am responsible for an AWS environment hosting my company's private e-commerce application and CMS.

My architecture looks roughly like this:

                    MY AWS VPC
                  10.0.0.0/16
                         │
          ┌──────────────┴──────────┐
          │                         │
     Public Subnet         Private Subnet
           │                   │
    ┌──────┴──────┐     ┌──────┴──────┐
    |Bastion Host │     │ Kubernetes  |
    |Public EC2   │     | Cluster     |
    └─────────────┘     | E-commerce  |
                        | CMS         |
                        └────┬────────┘
                    ┌────────▼──────────┐
                    │ Dedicated SNAT    │
                    │ Private IP        │
                    |  10.0.20.10       │
                    └──────────┬────────┘
                               │
                     Site-to-Site VPN
                               │
                               ▼
                    ┌─────────────────┐
                    │    Vendor A     │
                    │                 │
                    │ Private Network │
                    │ 172.20.0.0/16   │
                    └─────────────────┘

Vendor A has told me:

"We will only allow traffic coming from one private IP address."

For my project, I therefore choose:

10.0.20.10/32

as the dedicated source IP.

### Example 1: My E-commerce Application Calls Vendor A

Suppose my e-commerce application needs to communicate with Vendor A to process an order.

A customer places an order:

    Customer
    │
    ▼
    E-commerce application
    │
    │ HTTPS request
    ▼
    Kubernetes Pod

The Kubernetes pod might have an address such as:

    10.0.10.25

If I simply send the request toward Vendor A, Vendor A might see:

    Source: 10.0.10.25

But Vendor A doesn't allow that address.

Instead, I configure SNAT:

    Kubernetes Pod
    10.0.10.25
      │
      │ HTTPS
      ▼
    Dedicated SNAT
    10.0.20.10
      │
      ▼
    Site-to-Site VPN
      │
      ▼
    Vendor A

Vendor A now sees:

    Source IP: 10.0.20.10

Because 10.0.20.10 is whitelisted, Vendor A accepts the connection.

### Example 2: Multiple Kubernetes Pods

This becomes especially useful when I have many Kubernetes pods.

Suppose my e-commerce application has three pods:

    Pod 1 → 10.0.10.21
    Pod 2 → 10.0.10.22
    Pod 3 → 10.0.10.23

All three need to access Vendor A.

Without SNAT:

    10.0.10.21 ─────► Vendor A
    10.0.10.22 ─────► Vendor A
    10.0.10.23 ─────► Vendor A

I would have to ask Vendor A to whitelist three IP addresses.

That's not what Vendor A wants.

Instead:

    10.0.10.21 ─┐
    10.0.10.22 ─┼─►10.0.20.10─►VPN─► Vendor A
    10.0.10.23 ─┘

Vendor A sees:

    10.0.20.10

for all three applications.

Therefore, Vendor A only needs one whitelist entry:

    10.0.20.10/32

### Example 3: My CMS Needs Vendor A

Let's say my CMS also needs to retrieve information from Vendor A.

My CMS runs in Kubernetes:

    CMS Pod
    10.0.10.40
    
It sends:

    GET https://vendor-a-private-service/api/products

The traffic follows:

        CMS Pod
        10.0.10.40
            │
            ▼
        Dedicated SNAT
        10.0.20.10
            │
            ▼
        AWS Site-to-Site VPN
            │
            ▼
        Vendor A

Vendor A doesn't need to know anything about my Kubernetes pod.

It only needs to know:

    10.0.20.10 = trusted source

This is particularly useful because Kubernetes pods can be recreated and their IP addresses can change.

### Example 4: What Happens When a Pod Is Recreated?

This is an important real-world Kubernetes scenario.

Suppose my application initially has:

    Pod A → 10.0.10.21

Vendor A cannot reliably whitelist individual pod IPs because Kubernetes may replace the pod.

Later, Kubernetes deletes the pod and creates:

    Pod B → 10.0.10.57
    
The application is still working.

If I depended on the pod IP, Vendor A would suddenly see:

    10.0.10.57

and could reject the connection.

With my dedicated SNAT architecture:

    Pod A
    10.0.10.21
     │
     ▼
    10.0.20.10
     │
     ▼
    Vendor A

Later:

    Pod B
    10.0.10.57
     │
     ▼
    10.0.20.10
     │
     ▼
    Vendor A

The Kubernetes pod changes, but the Vendor-facing IP does not.

That's exactly what I want.

### Example 5: What About My Existing NAT Gateway?

I would not remove my existing NAT Gateway.

I need it for normal Internet traffic.

For example, my Kubernetes application might need to download something from the Internet:

Kubernetes
    │
    ▼
NAT Gateway
    │
    ▼
Internet

For Vendor A, however:

    Kubernetes
    │
    ▼
    Dedicated SNAT
    │
    ▼
    Site-to-Site VPN
    │
    ▼
    Vendor A

I therefore create separate routes.

For example:

|Destination|Route|
|-----------|-----|
|10.0.0.0/16|Local VPC|
|172.20.0.0/16|Site-to-Site VPN|
|0.0.0.0/0|NAT Gateway|

Assume Vendor A uses:

    172.20.0.0/16

When my Kubernetes application wants to reach:

    172.20.10.50

AWS sees that the destination belongs to:

    172.20.0.0/16

and sends the traffic through the VPN.

It doesn't send it to the NAT Gateway.

### Example 6: Why NGINX Ingress Isn't the Solution

In my project, I might have NGINX Ingress managing incoming traffic.

For example:

    Internet
    │
    ▼
    NGINX Ingress
    │
    ▼
    E-commerce Service
    │
    ▼
    Kubernetes Pod

NGINX is useful for incoming traffic.

But my Vendor A problem is about outgoing traffic:

    Kubernetes Pod
      │
      ▼
    Vendor A

Therefore, I wouldn't try to solve the Vendor A whitelist requirement by putting NGINX in the middle.

Instead:

    Incoming:

    Internet → NGINX → Kubernetes


    Outgoing to Vendor A:

    Kubernetes → SNAT → VPN → Vendor A

This keeps the architecture easier to understand and troubleshoot.

### Example 7: Vendor A's Firewall

I would also ask Vendor A to configure its firewall approximately like this:

    Vendor A Firewall
    ──────────────────────────────

    Source:      10.0.20.10/32
    Destination: Vendor service
    Protocol:    TCP
    Port:        443
    Action:      ALLOW

Everything else can be denied.

    For example:

    10.0.20.10 → Vendor A:443     ALLOW
    10.0.10.21 → Vendor A:443     DENY
    10.0.10.22 → Vendor A:443     DENY
    Internet    → Vendor A:443     DENY

This gives Vendor A a very small trusted source range.

### Example 8: The Return Traffic

One thing I would be careful about is the return route.

It's not enough for me to configure:

    AWS → Vendor A

Vendor A must also know how to return traffic.

For example:

    AWS:

    10.0.20.10
      │
      ▼
    VPN
      │
      ▼
    Vendor A

Vendor A needs a route similar to:

    10.0.20.10/32 → VPN tunnel

So the response can come back:

    Vendor A
    │
    ▼
    VPN
    │
    ▼
    10.0.20.10
    │
    ▼
    Kubernetes Pod

Without the return route, I could have a situation where the request leaves AWS successfully but the response never reaches my application.

My Final Architecture

I would describe the final solution like this:

                    AWS VPC
                10.0.0.0/16
                    │
        ┌─────────────┴─────────────┐
        │                           │
       Public Subnet        Private Subnet
        │                       │
    ┌─────▼─────┐       ┌──────▼────────┐
    │ Bastion   │       │ Kubernetes    │
    │ EC2       │       │ Cluster       │
    └───────────┘       │               │
                        │ E-commerce    │
                        │ CMS           │
                        └──────┬────────┘
                               │
                        Vendor traffic only
                               │
                               ▼
                    ┌──────────────────┐
                    │ Dedicated SNAT   │
                    │                  │
                    │ 10.0.20.10/32    │
                    └────────┬─────────┘
                             │
                        Route Table
                             │
                             ▼
                      Site-to-Site VPN
                             │
                      Encrypted IPsec
                             │
                             ▼
                    ┌──────────────────┐
                    │    Vendor A      │
                    │                  │
                    │ Whitelist:       │
                    │ 10.0.20.10/32    │
                    └──────────────────┘

    Normal Internet traffic:

    Kubernetes ──► NAT Gateway ──► Internet

### In simple terms

In my environment, I would keep my Kubernetes cluster private and keep the existing NAT Gateway for normal Internet access. For Vendor A, I would create a dedicated private SNAT path with a fixed private IP, such as 10.0.20.10. I would route Vendor A's private network through the AWS Site-to-Site VPN and ask Vendor A to whitelist only 10.0.20.10/32. This means that regardless of which Kubernetes pod sends the request, Vendor A always sees the same approved private IP.

A practical way to remember it

Think of my Kubernetes pods as employees inside my company and Vendor A as a secure building that only allows visitors with one approved ID.

The employees can have different names and IDs:

    Employee A → 10.0.10.21
    Employee B → 10.0.10.22
    Employee C → 10.0.10.23

But the company sends all of them through one controlled security checkpoint:

    Security checkpoint
            │
            ▼
    10.0.20.10
            │
            ▼
            Vendor A

Vendor A doesn't need to trust every employee individually. It only trusts the company's approved gateway identity: 10.0.20.10.

That is the main idea behind using private SNAT + Site-to-Site VPN + routing for your scenario.
