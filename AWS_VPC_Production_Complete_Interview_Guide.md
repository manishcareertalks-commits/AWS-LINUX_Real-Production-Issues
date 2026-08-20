**AWS VPC Production Environment --- Services & Architecture Guide**

<img width="1023" height="902" alt="image" src="https://github.com/user-attachments/assets/af60b61b-0aa1-43fd-89eb-4dc0e8fa2040" />

**1. Executive Overview**

A production AWS network should be designed around four principles:

1.  **Isolation** --- workloads should not be directly exposed unless
    required.
2.  **High availability** --- distribute critical components across
    multiple Availability Zones.
3.  **Controlled connectivity** --- explicitly define how traffic
    enters, leaves, and moves inside the VPC.
4.  **Operational security** --- minimize public exposure and provide
    controlled administrative access.

The reference architecture uses:

**Route 53 → Internet Gateway → Public Subnets → NAT Gateway → Private
Subnets → EC2**

with route tables controlling traffic.

> Important: Route 53 is a DNS service. It is not part of the VPC data
> plane. The Internet Gateway is the VPC component that provides the
> path between the VPC and the internet.

------------------------------------------------------------------------

**2. AWS Region**

**What is it?**

An AWS Region is a separate geographic area containing multiple isolated
Availability Zones.

**Why use it?**

Regions allow you to choose where workloads and data are hosted based
on:

-   latency
-   regulatory requirements
-   data residency
-   service availability
-   disaster recovery requirements
-   business requirements

**Production consideration**

Select the Region based on the application's users, compliance
requirements, AWS service availability, and disaster-recovery strategy.

------------------------------------------------------------------------

**3. Availability Zones**

**What is an Availability Zone?**

An Availability Zone (AZ) is an isolated location within an AWS Region
with independent infrastructure.

A production application should avoid depending on a single AZ for
critical workloads.

**Why use multiple AZs?**

Multiple AZs provide:

-   high availability
-   fault isolation
-   resilience against an AZ-level failure
-   better distribution of workloads

**Example**

``` text
Region
├── AZ-A
│   ├── Public Subnet
│   └── Private Subnet
│
└── AZ-B
    ├── Public Subnet
    └── Private Subnet
```

For higher resilience, critical architectures may use three AZs where
supported and justified.

------------------------------------------------------------------------

**4. Amazon VPC**

**What is Amazon VPC?**

Amazon VPC (Virtual Private Cloud) is a logically isolated virtual
network in AWS.

You define:

-   IPv4/IPv6 address ranges
-   subnets
-   route tables
-   gateways
-   network connectivity
-   security boundaries

Example:

``` text
VPC CIDR: 10.0.0.0/16
```

**Why use VPC?**

A VPC provides the network foundation for AWS workloads.

It lets you control:

-   which resources are reachable from the internet
-   which resources can communicate internally
-   how outbound traffic leaves the VPC
-   how the VPC connects to on-premises or other networks

------------------------------------------------------------------------

**5. CIDR Planning**

**What is CIDR?**

CIDR (Classless Inter-Domain Routing) defines an IP address range.

Example:

``` text
10.0.0.0/16
```

This gives a large address space that can be divided into smaller subnet
ranges.

**Production CIDR planning**

Do not choose the VPC CIDR in isolation.

Review:

-   existing VPCs
-   on-premises networks
-   data centers
-   VPN connectivity
-   Direct Connect
-   future VPC peering
-   Transit Gateway connectivity
-   partner networks

**Why is overlapping CIDR dangerous?**

Suppose:

``` text
VPC-A = 10.0.0.0/16
On-Prem = 10.0.0.0/16
```

If you later need routed connectivity between them, overlapping address
space creates routing ambiguity and can prevent normal IP-based
communication.

**Recommended approach**

Reserve address space for future expansion and document the enterprise
IP allocation strategy.

------------------------------------------------------------------------

**6. Subnets**

A subnet is an IP range inside a VPC.

Example:

``` text
VPC: 10.0.0.0/16

Public-A:  10.0.1.0/24
Public-B:  10.0.2.0/24

Private-A: 10.0.11.0/24
Private-B: 10.0.12.0/24
```

**Public subnet**

A subnet is considered public when its associated route table contains a
route that can send internet-bound traffic to an Internet Gateway.

Example:

``` text
0.0.0.0/0 → Internet Gateway
```

A resource also needs the appropriate public addressing and security
configuration to actually communicate with the internet.

**Private subnet**

A private subnet does not have a direct route to an Internet Gateway.

It can still have outbound internet access through a NAT Gateway.

Example:

``` text
0.0.0.0/0 → NAT Gateway
```

**Important interview point**

A subnet is **not public merely because an Internet Gateway is attached
to the VPC**.

The route table association is what gives the subnet a public route.

------------------------------------------------------------------------

**7. Internet Gateway (IGW)**

**What is it?**

An Internet Gateway is a VPC component that enables communication
between resources in a VPC and the public internet when the required
routing and addressing configuration exists.

**Key properties**

-   horizontally scaled and highly available
-   attached to a VPC
-   supports internet connectivity for appropriately configured
    resources
-   works with public subnet routing

**Example**

``` text
EC2
 ↓
Public Subnet
 ↓
Route Table
 ↓
0.0.0.0/0 → IGW
 ↓
Internet
```

**Important distinction**

Attaching an IGW to a VPC does not automatically expose every resource
in the VPC.

The subnet route table, resource addressing, and security controls must
also permit the traffic.

------------------------------------------------------------------------

**8. NAT Gateway**

**What is it?**

A NAT Gateway enables resources in private subnets to initiate
connections to destinations outside the VPC, including the public
internet, without requiring those private resources to have public IP
addresses.

**Typical flow**

``` text
Private EC2
   ↓
Private Route Table
   ↓
NAT Gateway
   ↓
Public Subnet
   ↓
Internet Gateway
   ↓
Internet
```

**Where is NAT Gateway deployed?**

A public NAT Gateway is created in a public subnet.

The public subnet must have a route toward the Internet Gateway.

**High availability**

For production environments, place a NAT Gateway in each AZ used by
private workloads and route each private subnet toward the NAT Gateway
in the same AZ where practical.

This avoids unnecessary cross-AZ dependency and reduces the blast radius
of an AZ failure.

**NAT Gateway vs NAT Instance**

### NAT Gateway

-   AWS-managed
-   scalable
-   less operational overhead
-   preferred for most production architectures

### NAT Instance

-   EC2-based
-   customer-managed
-   requires patching and scaling design
-   can require failover mechanisms
-   useful only for specific legacy or customization requirements

------------------------------------------------------------------------

**9. Route Tables**

**What is a route table?**

A route table contains rules that determine where network traffic is
sent.

Example:

``` text
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         igw-xxxx
```

**Public route table**

``` text
10.0.0.0/16 → local
0.0.0.0/0   → Internet Gateway
```

**Private route table**

``` text
10.0.0.0/16 → local
0.0.0.0/0   → NAT Gateway
```

**Route selection**

AWS uses the most specific matching route.

For example:

``` text
10.0.10.0/24 → local-target
0.0.0.0/0    → NAT Gateway
```

Traffic destined for `10.0.10.x` uses the more specific `/24` route.

------------------------------------------------------------------------

**10. Amazon EC2**

**What is EC2?**

Amazon EC2 provides virtual compute instances.

In this architecture, application servers are normally placed in private
subnets.

**Why private subnets?**

Benefits include:

-   reduced public attack surface
-   no direct inbound internet path
-   centralized ingress through controlled components
-   better separation between internet-facing and internal workloads

**Production pattern**

Instead of manually managing a few standalone EC2 instances, production
applications commonly use:

-   Auto Scaling Groups
-   Application Load Balancer
-   immutable or automated deployments
-   Systems Manager
-   centralized logging and monitoring

------------------------------------------------------------------------

**11. Bastion Host / Jump Server**

**What is a Bastion Host?**

A bastion host is a controlled administrative entry point used to reach
private resources.

Traditional pattern:

``` text
Administrator
     ↓
Bastion Host
     ↓
Private EC2
```

The bastion is placed in a public subnet.

**Security considerations**

If a bastion is required:

-   restrict inbound SSH to approved source IPs
-   avoid `0.0.0.0/0` for SSH
-   use strong authentication
-   patch the host
-   monitor access
-   use short-lived credentials where possible
-   keep the host minimal

**Modern alternative**

AWS Systems Manager Session Manager can provide shell access to managed
instances without requiring inbound SSH exposure or a traditional
bastion architecture.

------------------------------------------------------------------------

**12. Amazon Route 53**

**What is Route 53?**

Route 53 is AWS's highly available DNS service.

It can:

-   resolve domain names
-   route users to applications
-   perform health checks
-   support routing policies
-   integrate with AWS services

Example:

``` text
www.example.com
       ↓
Route 53
       ↓
Application Load Balancer
       ↓
Private EC2
```

**Important clarification**

Route 53 is not a replacement for the Internet Gateway, NAT Gateway, or
route tables.

DNS resolution and network packet routing are separate concepts.

------------------------------------------------------------------------

**13. Security Groups**

Security Groups are stateful virtual firewalls attached to resources
such as EC2 instances and network interfaces.

Example:

``` text
ALB Security Group
Inbound: TCP 443 from Internet

EC2 Security Group
Inbound: TCP 8080 from ALB Security Group
```

This is safer than allowing the EC2 server to accept application traffic
from the entire internet.

------------------------------------------------------------------------

**14. Network ACLs**

Network ACLs (NACLs) operate at the subnet level.

They are stateless, meaning inbound and outbound rules must be
considered independently.

Use NACLs when you need subnet-level traffic controls in addition to
security groups.

Do not treat NACLs as a replacement for security groups.

------------------------------------------------------------------------

**15. Production Architecture Pattern**

A common production pattern is:

``` text
                    Internet
                       |
                   Route 53
                       |
              Public Load Balancer
                  /          \
             AZ-A              AZ-B
          Public Subnet     Public Subnet
               |                 |
             NAT-A             NAT-B
               |                 |
          Private-A          Private-B
               |                 |
            EC2/ASG           EC2/ASG
```

The exact architecture depends on the application.

For example, a load balancer is commonly used for internet-facing
application traffic, while the EC2 application tier remains private.

------------------------------------------------------------------------

**16. Key Interview Rules to Remember**

### Rule 1

**IGW attached to VPC ≠ public subnet.**

### Rule 2

**Public subnet = route to Internet Gateway + appropriate resource
addressing/security configuration.**

### Rule 3

**Private subnet can have outbound internet through NAT Gateway.**

### Rule 4

**NAT Gateway belongs in a public subnet.**

### Rule 5

**Application servers should generally remain private.**

### Rule 6

**Use multiple AZs for high availability.**

### Rule 7

**Plan CIDRs before connecting networks.**

### Rule 8

**Route tables determine where packets are sent.**

### Rule 9

**Route 53 provides DNS; it does not replace network routing.**

### Rule 10

**Minimize public exposure and prefer managed services where
practical.**

------------------------------------------------------------------------

**AWS VPC Production Environment --- Interview Questions & Detailed
Answers**

**How to Use This Guide**

Answer architecture questions in this order:

1.  Clarify requirements.
2.  Plan CIDR ranges.
3.  Design multiple Availability Zones.
4.  Separate public and private tiers.
5.  Define ingress.
6.  Define egress.
7.  Define route tables.
8.  Add security controls.
9.  Add high availability.
10. Explain failure scenarios and operational trade-offs.

------------------------------------------------------------------------

**Section 1 --- Core Interview Questions**

**Q1. What is a VPC?**

### Answer

A VPC is a logically isolated network in AWS where you define IP ranges,
subnets, routes, gateways, and network connectivity.

A strong interview answer should continue beyond the definition:

> "I treat the VPC as the network foundation of the application. Before
> creating subnets, I plan the CIDR range based on current workloads and
> future connectivity requirements."

------------------------------------------------------------------------

**Q2. How would you design a production VPC?**

### Answer

I would normally start with:

-   one VPC with a carefully planned CIDR
-   at least two Availability Zones for critical workloads
-   public subnets for internet-facing infrastructure
-   private subnets for application workloads
-   route tables separated according to traffic requirements
-   NAT Gateway for private outbound internet access when required
-   security groups with least-privilege rules
-   controlled administrative access
-   monitoring and logging
-   a documented IP allocation strategy

For an internet-facing application, I would typically use:

``` text
Internet
   ↓
Route 53
   ↓
Internet-facing ALB
   ↓
Private EC2 / containers
```

NAT Gateway is used for outbound connections initiated by private
workloads.

------------------------------------------------------------------------

**Q3. What makes a subnet public?**

### Answer

The defining network property is the route table.

A subnet is public when its associated route table has a route to an
Internet Gateway, for example:

``` text
0.0.0.0/0 → Internet Gateway
```

However, a resource still needs appropriate public addressing and
security configuration to communicate with the internet.

### Common wrong answer

"Because the subnet is inside a VPC that has an Internet Gateway."

That is incomplete.

------------------------------------------------------------------------

**Q4. Can a private subnet access the internet?**

### Answer

Yes.

A private subnet can use a NAT Gateway for outbound internet
connectivity.

``` text
Private EC2
 ↓
Private Route Table
 ↓
NAT Gateway
 ↓
Internet Gateway
 ↓
Internet
```

The private EC2 does not need a public IP for this outbound path.

------------------------------------------------------------------------

**Q5. Can the internet directly initiate a connection to an EC2 through
NAT Gateway?**

### Answer

No.

NAT Gateway is designed for connections initiated from private resources
toward external destinations.

It is not a mechanism for accepting unsolicited inbound connections from
the public internet to private instances.

For inbound application traffic, use an appropriate public-facing entry
point such as an internet-facing load balancer and keep the application
tier private.

------------------------------------------------------------------------

**Q6. Why is NAT Gateway placed in a public subnet?**

### Answer

The NAT Gateway needs a path to the public internet.

Its public subnet has a default route to the Internet Gateway.

Private subnets route their internet-bound traffic to the NAT Gateway.

``` text
Private Subnet
      |
      v
  NAT Gateway
      |
      v
Public Subnet Route Table
      |
      v
Internet Gateway
      |
      v
Internet
```

------------------------------------------------------------------------

**Q7. Why should application servers be in private subnets?**

### Answer

The main reason is to reduce direct exposure.

Instead of:

``` text
Internet → EC2
```

we prefer:

``` text
Internet
   ↓
Load Balancer
   ↓
Private EC2
```

The EC2 security group can then allow application traffic only from the
load balancer security group.

This creates layered security.

------------------------------------------------------------------------

**Q8. What is the difference between Internet Gateway and NAT Gateway?**

  ---------------------------------------------------------------------
  Component                          Purpose
  ---------------------------------- ----------------------------------
  Internet Gateway                   Provides internet connectivity for
                                     appropriately configured VPC
                                     resources

  NAT Gateway                        Allows private resources to
                                     initiate outbound connections
                                     through a public path

  Placement                          IGW attaches to VPC; NAT Gateway
                                     is created in a subnet

  Typical use                        Public subnet connectivity
  ---------------------------------------------------------------------

------------------------------------------------------------------------

**Q9. What happens if you remove the NAT Gateway route from a private
subnet?**

Private instances lose the configured outbound internet path through
that NAT Gateway.

Internal VPC traffic can continue if the corresponding routes remain
valid.

For example, this route:

``` text
10.0.0.0/16 → local
```

continues to support local VPC routing.

But:

``` text
0.0.0.0/0 → NAT Gateway
```

would no longer be available if removed.

------------------------------------------------------------------------

**Q10. What is a route table?**

A route table contains routing rules that determine the target for
traffic matching a destination CIDR.

Example:

``` text
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         nat-xxxx
```

AWS selects the most specific matching route.

------------------------------------------------------------------------

**Section 2 --- Scenario-Based Questions**

**Scenario 1: Design a Production Web Application**

### Question

You have a public web application. Users access it from the internet.
The application runs on EC2. How would you design the VPC?

### Strong Solution

I would use at least two AZs.

``` text
                         Internet
                            |
                         Route 53
                            |
                    Internet-facing ALB
                       /           \
                     AZ-A          AZ-B
                      |              |
                 Private-A       Private-B
                      |              |
                   EC2/ASG        EC2/ASG
```

For outbound internet access:

``` text
Private-A → NAT-A → IGW
Private-B → NAT-B → IGW
```

### Why?

-   ALB handles public ingress.
-   EC2 remains private.
-   Multiple AZs provide resilience.
-   NAT provides controlled outbound access.
-   Security groups can restrict EC2 traffic to the ALB.

------------------------------------------------------------------------

**Scenario 2: One AZ Has Failed**

### Question

Your application is deployed only in AZ-A and AZ-A becomes unavailable.
What happens?

### Answer

The application may become unavailable.

The correct production response is to distribute critical workloads
across multiple AZs.

For example:

``` text
AZ-A → EC2 instances
AZ-B → EC2 instances
```

An Auto Scaling Group can maintain desired capacity across AZs.

A load balancer can distribute traffic to healthy targets.

------------------------------------------------------------------------

**Scenario 3: Private EC2 Cannot Download Packages**

### Question

An EC2 instance in a private subnet cannot download updates from the
internet. What would you check?

### Troubleshooting sequence

1.  Is the EC2 subnet associated with the expected private route table?
2.  Does the route table contain:

``` text
0.0.0.0/0 → NAT Gateway
```

3.  Is the NAT Gateway available?
4.  Is the NAT Gateway in a public subnet?
5.  Does the public subnet route table contain:

``` text
0.0.0.0/0 → Internet Gateway
```

6.  Does the NAT Gateway have the required public connectivity
    configuration?
7.  Are security groups/NACLs blocking traffic?
8.  Is DNS resolution working?
9.  Is the destination itself reachable?

This sequence demonstrates systematic troubleshooting rather than
guessing.

------------------------------------------------------------------------

**Scenario 4: Private EC2 Needs Internet but Must Not Be Public**

### Question

A security team says EC2 must not have public IP addresses, but the
servers need to download patches. What would you do?

### Answer

Keep the EC2 instances private.

Deploy NAT Gateway in a public subnet and route private subnet
internet-bound traffic through it.

``` text
EC2
 ↓
Private RT
 ↓
NAT Gateway
 ↓
IGW
 ↓
Internet
```

For stronger security, consider whether the workload can use private AWS
endpoints or internal artifact repositories instead of public internet
access.

------------------------------------------------------------------------

**Scenario 5: SSH Access to Private EC2**

### Question

Operations needs SSH access to private EC2 instances. What are your
options?

### Answer

Preferred modern option:

**AWS Systems Manager Session Manager**, when the operational and
connectivity requirements are satisfied.

Traditional option:

``` text
Admin → Bastion → Private EC2
```

If a bastion is used:

-   restrict SSH source IPs
-   use strong authentication
-   patch the host
-   log administrative access
-   avoid exposing SSH to the entire internet

------------------------------------------------------------------------

**Scenario 6: CIDR Overlap**

### Question

Your existing VPC is `10.0.0.0/16`. The network team wants to connect an
on-premise network that is also `10.0.0.0/16`. What is the problem?

### Answer

The address spaces overlap.

This creates routing ambiguity and prevents normal IP-based
communication between overlapping ranges.

### What would you do?

Ideally, redesign the address allocation before the networks are
connected.

If the existing ranges cannot be changed, more advanced migration or
translation approaches may be required, depending on the exact
connectivity requirement.

The key architectural lesson is:

**IP planning must happen before network connectivity is built.**

------------------------------------------------------------------------

**Scenario 7: NAT Gateway Failure**

### Question

You have one NAT Gateway in AZ-A. Private workloads in AZ-B use it. AZ-A
fails. What is the concern?

### Answer

Private workloads in AZ-B may lose their configured outbound internet
path.

A stronger production design is:

``` text
AZ-A Private → NAT-A
AZ-B Private → NAT-B
```

Each private subnet uses the NAT Gateway in its own AZ where practical.

This reduces:

-   cross-AZ dependency
-   failure blast radius
-   unnecessary cross-AZ data transfer

------------------------------------------------------------------------

**Scenario 8: The Application Is Publicly Reachable**

### Question

Your application EC2 instances have public IPs and are directly
reachable from the internet. What would you change?

### Answer

I would first understand why they are public.

For a standard web application, I would move the application tier into
private subnets and expose an appropriate load balancer publicly.

``` text
Internet
   ↓
ALB
   ↓
Private EC2
```

Then:

``` text
ALB SG → EC2 SG
```

rather than:

``` text
Internet → EC2 SG
```

This reduces the attack surface.

------------------------------------------------------------------------

**Scenario 9: Route Table Looks Correct but Application Cannot Reach
Internet**

### Question

A private EC2 has:

``` text
0.0.0.0/0 → NAT Gateway
```

but internet access still fails. What do you investigate?

### Answer

Check systematically:

### Network path

``` text
EC2
 ↓
Private Route Table
 ↓
NAT Gateway
 ↓
Public Route Table
 ↓
IGW
 ↓
Internet
```

Verify every hop.

### Also check

-   NAT Gateway state
-   subnet associations
-   IGW attachment
-   route table associations
-   security groups
-   NACLs
-   DNS resolution
-   OS-level firewall
-   proxy configuration
-   destination availability

A route entry alone does not guarantee end-to-end connectivity.

------------------------------------------------------------------------

**Scenario 10: Route 53 vs Internet Gateway**

### Question

What is the difference between Route 53 and Internet Gateway?

### Answer

They solve different problems.

**Route 53:**

DNS name resolution and DNS-based routing.

**Internet Gateway:**

Network connectivity between the VPC and the internet for appropriately
configured resources.

Example:

``` text
example.com
     ↓
Route 53
     ↓
ALB public endpoint
     ↓
Private application tier
```

Route 53 resolves the name; the Internet Gateway participates in the
actual network path.

------------------------------------------------------------------------

**Section 3 --- Senior-Level Interview Questions**

**Q11. Why would you create separate route tables for each AZ?**

### Answer

Separate route tables provide more granular control.

For example:

``` text
Private-A → NAT-A
Private-B → NAT-B
```

This allows each AZ to use its local NAT Gateway.

It also makes failure isolation and troubleshooting clearer.

------------------------------------------------------------------------

**Q12. Should every subnet have its own route table?**

### Answer

Not necessarily.

Multiple subnets can share the same route table if they require
identical routing behavior.

However, separate route tables are useful when different subnets need
different routing policies.

The goal is not "one route table per subnet."

The goal is **clear and intentional routing**.

------------------------------------------------------------------------

**Q13. How would you reduce NAT Gateway costs?**

### Answer

First understand where the traffic is going.

Options may include:

-   AWS PrivateLink interface endpoints
-   S3/DynamoDB gateway endpoints where appropriate
-   private artifact repositories
-   caching
-   reducing unnecessary internet traffic
-   localizing traffic to AWS services through private connectivity

Do not optimize by simply removing NAT if the application requires
outbound internet access.

------------------------------------------------------------------------

**Q14. How would you design for least privilege?**

### Answer

Use layered controls:

-   security groups
-   NACLs where appropriate
-   private subnets
-   restricted administrative access
-   IAM least privilege
-   load balancer-to-application security group rules
-   VPC endpoints where appropriate
-   logging and monitoring

Example:

``` text
ALB SG
  ↓ only application port
EC2 SG
  ↓ only database port
Database SG
```

Avoid broad rules such as allowing all traffic from `0.0.0.0/0` unless
there is a clear requirement.

------------------------------------------------------------------------

**Section 4 --- Troubleshooting Questions**

**Q15. EC2 Has No Internet Connectivity. What Is Your Troubleshooting
Framework?**

### Answer

I would avoid changing random settings.

I would trace the packet path:

``` text
EC2
 ↓
ENI / Security Group
 ↓
Subnet
 ↓
Route Table
 ↓
NAT or IGW
 ↓
Destination
```

Then verify:

1.  subnet route table
2.  default route
3.  NAT/IGW status
4.  security groups
5.  NACLs
6.  DNS
7.  OS firewall
8.  destination

This demonstrates structured troubleshooting.

------------------------------------------------------------------------

**Q16. EC2 Cannot Communicate With Another EC2 in the Same VPC**

### Answer

Check:

-   both instances' security groups
-   NACLs
-   subnet route tables
-   local VPC route
-   host firewall
-   application listening port
-   correct private IP/DNS
-   whether the instances actually belong to the expected VPC

Remember that the VPC's `local` route normally provides connectivity
between VPC CIDR ranges.

------------------------------------------------------------------------

**Section 5 --- Architecture Trade-Off Questions**

**Q17. Bastion Host or Session Manager?**

### Bastion

Advantages:

-   familiar SSH workflow
-   useful for legacy environments
-   can support traditional administration models

Disadvantages:

-   public-facing infrastructure
-   patching required
-   credential/access management
-   additional attack surface

### Session Manager

Advantages:

-   no inbound SSH exposure required
-   centralized access control
-   integrates with IAM
-   better auditability when configured correctly

For modern AWS environments, I would evaluate Session Manager first.

------------------------------------------------------------------------

**Q18. One NAT Gateway or One per AZ?**

### One NAT Gateway

Pros:

-   lower direct NAT cost

Cons:

-   AZ dependency
-   possible cross-AZ traffic
-   larger failure blast radius

### NAT Gateway per AZ

Pros:

-   better AZ isolation
-   improved resilience
-   avoids unnecessary cross-AZ dependency

Cons:

-   higher infrastructure cost

For a highly available production workload, I generally prefer one NAT
Gateway per AZ when private workloads depend on outbound internet
access.

------------------------------------------------------------------------

**Section 6 --- Interview Design Challenge**

**Q19. Design a Highly Available AWS VPC From Scratch**

### Requirement

You have:

-   public users
-   web application
-   EC2 application tier
-   high availability requirement
-   outbound internet requirement
-   secure administration

### Solution

``` text
                         Internet
                            |
                         Route 53
                            |
                   Internet-facing ALB
                      /            \
                    AZ-A           AZ-B
                     |               |
              Public Subnet     Public Subnet
                     |               |
                   NAT-A           NAT-B
                     |               |
              Private Subnet    Private Subnet
                     |               |
                  EC2/ASG          EC2/ASG
```

### Routing

Public:

``` text
0.0.0.0/0 → IGW
```

Private-A:

``` text
0.0.0.0/0 → NAT-A
```

Private-B:

``` text
0.0.0.0/0 → NAT-B
```

### Security

-   ALB accepts HTTPS from the internet.
-   EC2 accepts application traffic only from ALB security group.
-   Administrative access uses Session Manager where possible.
-   No direct public IPs on application instances.

### Availability

-   multiple AZs
-   load balancing
-   Auto Scaling
-   NAT per AZ where appropriate

------------------------------------------------------------------------

**Section 7 --- Tricky Interview Questions**

**Q20. Is an EC2 with a public IP automatically reachable from the
internet?**

### Answer

No.

A public IP alone is not sufficient.

The network path, route table, security group, NACL, host firewall, and
application listener all matter.

------------------------------------------------------------------------

**Q21. Can a private subnet have a route to `0.0.0.0/0`?**

### Answer

Yes.

The destination is not what determines whether the subnet is public.

For example:

``` text
Private Subnet:
0.0.0.0/0 → NAT Gateway
```

This is still a private subnet because it has no direct route to an
Internet Gateway.

------------------------------------------------------------------------

**Q22. Can a public subnet contain a database?**

### Answer

Technically possible, but generally undesirable for a standard
production architecture.

A database should normally be placed in private subnets and exposed only
to the required application tier.

------------------------------------------------------------------------

**Q23. Why not put everything in public subnets?**

### Answer

It increases the attack surface and makes security harder to manage.

A layered architecture is better:

``` text
Public layer
    ↓
Application/private layer
    ↓
Database/private layer
```

Only components that genuinely require public connectivity should be
public.

------------------------------------------------------------------------

**Section 8 --- Senior Architect Answer Framework**

When an interviewer asks:

**"Design an AWS VPC for production."**

Do not immediately start listing AWS services.

Use this structure:

### Step 1 --- Requirements

Ask:

-   Is the application internet-facing?
-   How many users?
-   Availability requirement?
-   Compliance requirements?
-   Expected traffic?
-   On-premises connectivity?
-   Disaster recovery requirement?
-   IPv4/IPv6 requirement?

### Step 2 --- CIDR

Plan the address space based on current and future connectivity.

### Step 3 --- AZ Strategy

Use multiple AZs for critical workloads.

### Step 4 --- Subnet Strategy

Separate:

-   public infrastructure
-   application workloads
-   data workloads

### Step 5 --- Ingress

Use an appropriate public entry point such as an ALB when required.

### Step 6 --- Egress

Use NAT Gateway or private connectivity based on the workload.

### Step 7 --- Routing

Define route tables intentionally.

### Step 8 --- Security

Use least privilege with security groups, NACLs where appropriate, IAM,
and controlled administration.

### Step 9 --- Resilience

Eliminate single points of failure.

### Step 10 --- Operations

Add:

-   logging
-   monitoring
-   alerting
-   automated deployment
-   patch management
-   backups
-   incident response

------------------------------------------------------------------------

**Final Interview Cheat Sheet**

``` text
VPC
 └── Multiple Availability Zones
      ├── Public Subnets
      │    ├── Load Balancer
      │    └── NAT Gateway
      │
      └── Private Subnets
           └── Application Servers

Public Route Table
 └── 0.0.0.0/0 → IGW

Private Route Table
 └── 0.0.0.0/0 → NAT Gateway

Administrative Access
 └── Prefer Session Manager where appropriate
```

**One-Sentence Interview Answer**

> "For a production VPC, I would start with non-overlapping CIDR
> planning, deploy public and private subnets across multiple
> Availability Zones, keep application workloads private, use an
> appropriate public entry point for inbound traffic, NAT Gateway or
> private connectivity for controlled outbound access, intentional route
> tables, least-privilege security controls, and eliminate single points
> of failure."

------------------------------------------------------------------------

**AWS VPC --- Production Scenario Workbook**

**Scenario 1 --- Build a New Production Environment**

### Requirement

A company is launching an internet-facing application on EC2.

### Architecture

``` text
Internet
   |
Route 53
   |
Internet-facing ALB
   |
+---------------------------+
| AWS Region                |
|                           |
| AZ-A              AZ-B   |
| Public            Public |
|   |                 |     |
| NAT-A             NAT-B  |
|   |                 |     |
| Private           Private|
| EC2/ASG           EC2/ASG|
+---------------------------+
```

### Decisions

-   CIDR is reviewed with the network team.
-   Workloads span multiple AZs.
-   Application EC2 instances are private.
-   ALB provides public ingress.
-   NAT provides outbound access where required.
-   Route tables are separated by traffic role.

------------------------------------------------------------------------

**Scenario 2 --- AZ Failure**

### Problem

AZ-A becomes unavailable.

### Expected behavior

The load balancer should stop sending traffic to unhealthy targets in
AZ-A and continue serving healthy targets in AZ-B.

### Architecture requirement

Do not make one AZ a single point of failure.

------------------------------------------------------------------------

**Scenario 3 --- NAT Failure**

### Problem

A private workload loses outbound internet access.

### Investigation

``` text
Private EC2
 ↓
Private Route Table
 ↓
NAT Gateway
 ↓
Public Route Table
 ↓
IGW
```

Check each layer.

### Production improvement

Use NAT Gateways per AZ where the resilience and cost model justify it.

------------------------------------------------------------------------

**Scenario 4 --- No Internet From Private EC2**

Check:

-   route table association
-   `0.0.0.0/0 → NAT Gateway`
-   NAT state
-   NAT placement
-   public subnet route
-   IGW
-   security controls
-   DNS
-   OS configuration

------------------------------------------------------------------------

**Scenario 5 --- Direct Internet Access Is Required for a Legacy
Server**

### Architect response

Do not immediately make the server public.

First ask why direct inbound access is required.

Investigate whether the requirement can be fulfilled through:

-   load balancer
-   VPN
-   Session Manager
-   private connectivity
-   reverse proxy
-   application gateway

Expose only the minimum required surface.

------------------------------------------------------------------------

**Scenario 6 --- On-Premises Connectivity**

### Problem

The company wants to connect:

``` text
AWS VPC ↔ On-Premises Data Center
```

### First question

What are the CIDR ranges?

If they overlap, solve the IP addressing problem before assuming VPN or
Direct Connect will solve it.

### Architect mindset

Connectivity technology comes after IP and routing design.

------------------------------------------------------------------------

**Scenario 7 --- Reduce Internet Dependency**

A private EC2 fleet frequently accesses AWS services.

### Architect response

Do not automatically send every request through NAT.

Evaluate VPC endpoints/private connectivity where appropriate.

Benefits can include:

-   reduced NAT dependency
-   improved network path
-   reduced public internet exposure
-   potentially lower data processing costs

------------------------------------------------------------------------

**Scenario 8 --- Security Review**

### Findings

-   EC2 has public IP.
-   SSH is open to `0.0.0.0/0`.
-   Application server accepts traffic from anywhere.

### Recommended direction

``` text
Internet
   ↓
ALB
   ↓
Private EC2
```

Restrict:

``` text
ALB SG → EC2 SG
```

and administrative access through controlled mechanisms such as Session
Manager where appropriate.

------------------------------------------------------------------------

**Scenario 9 --- Route Table Misconfiguration**

### Problem

Private subnet accidentally has:

``` text
0.0.0.0/0 → Internet Gateway
```

### Risk

The subnet now has a direct public route.

Review whether resources have public addresses and whether the subnet
was intended to be private.

Correct the route according to the intended architecture.

------------------------------------------------------------------------

**Scenario 10 --- Production Review Checklist**

**Network**

-   [ ] CIDR planned
-   [ ] No known overlapping ranges
-   [ ] Multiple AZs
-   [ ] Public/private separation
-   [ ] Route tables documented

**Internet**

-   [ ] IGW attached
-   [ ] Public routes intentional
-   [ ] NAT design reviewed
-   [ ] Outbound access minimized

**Security**

-   [ ] No unnecessary public IPs
-   [ ] Security groups least privilege
-   [ ] SSH not broadly exposed
-   [ ] Administrative access controlled
-   [ ] NACLs reviewed where used

**Availability**

-   [ ] Workloads distributed across AZs
-   [ ] Load balancing configured where appropriate
-   [ ] NAT resilience considered
-   [ ] No single AZ dependency

**Operations**

-   [ ] Monitoring
-   [ ] Logging
-   [ ] Alerting
-   [ ] Patch strategy
-   [ ] Backup/recovery strategy
-   [ ] Infrastructure as Code

------------------------------------------------------------------------

**Final Senior Architect Perspective**

The strongest VPC interview answers do not focus only on memorizing:

**VPC → Subnet → IGW → NAT → Route Table.**

They explain **why each component exists**, how traffic flows, what
happens when something fails, and what trade-offs were considered.

A senior-level answer should always connect:

**Requirement → Architecture → Traffic Flow → Security → Availability →
Failure Mode → Trade-off.**
