# 3-Tier Architecture on AWS --- Production-Oriented Study Notes
<img width="955" height="897" alt="image" src="https://github.com/user-attachments/assets/b37e8e1b-5abc-4432-8ce5-03fe822e3be1" />

## 1. Overview

A **3-tier architecture** separates an application into three logical
layers:

1.  **Presentation Tier** --- user-facing frontend
2.  **Application Tier** --- backend/business logic
3.  **Data Tier** --- databases, cache, and object storage

This separation improves **security, scalability, availability,
maintainability, and operational control**.

> **Important production note:** 3-tier architecture is a logical
> separation, not necessarily three physical servers or three subnets.
> In AWS, each tier can contain multiple services, instances, subnets,
> and Availability Zones.

------------------------------------------------------------------------

# 2. Reference Production Architecture

A typical production request flow can look like:

``` text
Users
   |
   v
Route 53
   |
   v
CloudFront
   |----------------------> S3
   |                         Frontend static assets
   |
   v
AWS WAF
   |
   v
Application Load Balancer
   |
   +-------------------+
   |                   |
   v                   v
Private Subnet AZ-A   Private Subnet AZ-B
   |                   |
   EC2 / App Servers   EC2 / App Servers
   |                   |
   +---------+---------+
             |
       Aurora / RDS
             |
       ElastiCache
             |
            S3
      Media/Documents
```

The exact path depends on the application design. For example, static
frontend assets may be served directly from CloudFront/S3 while API
requests are routed to the ALB.

------------------------------------------------------------------------

# 3. Tier 1 --- Presentation Tier

## Purpose

The presentation tier is responsible for the user interface.

Examples:

-   React
-   Angular
-   Vue
-   HTML/CSS/JavaScript
-   Static frontend assets

For a modern AWS implementation, static frontend files can be stored in
**Amazon S3** and delivered through **Amazon CloudFront**.

### Typical flow

``` text
User
  |
  v
Route 53
  |
  v
CloudFront
  |
  v
S3
  |
  v
HTML / CSS / JS / Images
```

## Why CloudFront?

CloudFront provides CDN functionality and can:

-   Cache static content at edge locations
-   Reduce latency
-   Reduce direct load on S3
-   Improve global user experience
-   Provide TLS/HTTPS integration
-   Work with AWS WAF

## Production recommendation

Do **not** make an S3 bucket publicly writable.

A stronger design is:

``` text
Internet
   |
CloudFront
   |
Origin Access Control
   |
S3 Private Bucket
```

The S3 bucket remains private and CloudFront is allowed to retrieve the
content.

## Deployment example

A CI/CD pipeline may:

``` text
Developer
   |
Git
   |
CI Pipeline
   |
Build React Application
   |
Upload build artifacts
   |
S3
   |
CloudFront Cache Invalidation
```

For large deployments, cache-busting/versioned assets are preferable to
invalidating everything unnecessarily.

------------------------------------------------------------------------

# 4. Route 53

Amazon Route 53 provides DNS functionality.

Example:

``` text
www.example.com
       |
       v
Route 53
       |
       v
CloudFront
```

For API traffic, DNS can point to an ALB:

``` text
api.example.com
       |
       v
Route 53
       |
       v
ALB
```

## Production considerations

Use:

-   Health checks where appropriate
-   Alias records for AWS resources
-   Separate DNS names for frontend/API when useful
-   Appropriate TTLs
-   Failover/routing policies when multi-region architecture requires
    them

------------------------------------------------------------------------

# 5. CloudFront --- Static and Dynamic Content

A common production architecture uses CloudFront for both static and API
traffic:

``` text
                    CloudFront
                  /            \
                 /              \
                v                v
              S3                ALB
           Frontend             APIs
```

CloudFront can use multiple origins and path-based behaviors.

Example:

``` text
/*          -> S3
/api/*      -> ALB
```

This avoids forcing every frontend request through the application
servers.

------------------------------------------------------------------------

# 6. AWS WAF

AWS WAF protects web applications from common HTTP-layer attacks and
unwanted traffic.

It can help with:

-   SQL injection protection
-   Cross-site scripting protection
-   IP-based blocking
-   Rate-based rules
-   Managed rule groups
-   Bot-related controls

Typical placement:

``` text
User
 |
Route 53
 |
CloudFront
 |
WAF
 |
ALB
 |
Application
```

WAF can be associated with CloudFront or an ALB depending on the
architecture.

## Production example

Suppose an API normally receives a few hundred requests per minute but
suddenly receives abnormal traffic from one source.

A rate-based WAF rule can help reduce abusive traffic before it reaches
the application.

> WAF is not a replacement for application-level authentication,
> authorization, validation, or DDoS protection.

------------------------------------------------------------------------

# 7. Tier 2 --- Application Tier

The application tier contains the backend/business logic.

Examples:

-   Java Spring Boot
-   Node.js
-   Python
-   .NET
-   Go
-   REST APIs
-   Microservices

The application tier should normally run in **private subnets** when it
does not need to accept direct internet traffic.

Typical architecture:

``` text
Internet
   |
   v
ALB
   |
   +------------------+
   |                  |
   v                  v
EC2 AZ-A            EC2 AZ-B
Private Subnet      Private Subnet
```

------------------------------------------------------------------------

# 8. Application Load Balancer

The ALB distributes HTTP/HTTPS traffic across healthy application
targets.

Example:

``` text
                ALB
              /     \
             /       \
          EC2-1     EC2-2
```

## Important production capabilities

### Health checks

The ALB continuously checks application health.

Example:

``` text
GET /health
```

If an instance fails the health check, ALB can stop sending new traffic
to that target.

### Path-based routing

Example:

``` text
/api/users/*     -> User Service
/api/orders/*    -> Order Service
/api/payments/*  -> Payment Service
```

### Host-based routing

Example:

``` text
api.example.com      -> API target group
admin.example.com    -> Admin target group
```

------------------------------------------------------------------------

# 9. EC2 Auto Scaling

Running one EC2 instance is a single point of failure.

Production systems normally use multiple instances across multiple
Availability Zones.

Example:

``` text
                ALB
              /     \
             v       v
          EC2-A     EC2-B
            AZ-A      AZ-B
```

An **Auto Scaling Group (ASG)** can automatically:

-   Maintain desired capacity
-   Replace unhealthy instances
-   Scale out when demand increases
-   Scale in when demand decreases

Example:

``` text
Normal traffic:
3 instances

High traffic:
6 instances

Low traffic:
2 instances
```

Scaling policies can use metrics such as:

-   CPU utilization
-   Request count per target
-   Application-specific CloudWatch metrics
-   Scheduled scaling
-   Target tracking policies

> CPU alone is not always the best scaling signal. For web applications,
> request count per target or latency can sometimes better represent
> user demand.

------------------------------------------------------------------------

# 10. Multi-AZ Design

Production applications should avoid depending on a single Availability
Zone.

Bad:

``` text
ALB
 |
EC2
 |
Database
```

Better:

``` text
             ALB
           /     \
          /       \
       AZ-A       AZ-B
        |           |
      EC2         EC2
        \           /
         \         /
          Database
```

If one AZ has an infrastructure failure, traffic can continue through
healthy resources in another AZ.

------------------------------------------------------------------------

# 11. Private Subnets and Internet Access

A common secure VPC design is:

``` text
                 Internet
                    |
                 IGW
                    |
              Public Subnet
                    |
                   ALB
                    |
        -------------------------
        |                       |
 Private Subnet AZ-A     Private Subnet AZ-B
        |                       |
      EC2                     EC2
```

Application instances do not need public IP addresses just because they
need outbound internet access.

For controlled outbound access, private subnets can use a **NAT
Gateway**.

Example:

``` text
Private EC2
    |
NAT Gateway
    |
Internet Gateway
    |
Internet
```

NAT Gateway is for outbound connectivity; it does not make the private
EC2 instance directly reachable from the internet.

------------------------------------------------------------------------

# 12. IAM Roles for EC2

Applications should avoid storing AWS access keys directly on EC2
instances.

Instead:

``` text
EC2
 |
IAM Role
 |
AWS Permissions
```

Example:

``` text
EC2 Application
       |
       v
IAM Role
       |
       +---- S3: GetObject
       |
       +---- CloudWatch: PutMetricData
```

Follow **least privilege**.

Do not give an application:

``` text
AdministratorAccess
```

when it only needs:

``` text
s3:GetObject
s3:PutObject
```

for a specific bucket/prefix.

------------------------------------------------------------------------

# 13. Tier 3 --- Data Tier

The data tier stores persistent application data.

Possible components:

-   Amazon Aurora
-   Amazon RDS
-   Amazon ElastiCache
-   Amazon S3

These services have different responsibilities.

------------------------------------------------------------------------

# 14. Amazon Aurora / RDS

A relational database can store structured transactional data.

Example e-commerce data:

``` text
Users
Products
Orders
Payments
Inventory
Addresses
```

Example relationship:

``` text
Application
    |
    v
Aurora
    |
    +--- Users
    +--- Products
    +--- Orders
    +--- Inventory
```

## Production considerations

Database design should consider:

-   Multi-AZ availability
-   Automated backups
-   Point-in-time recovery
-   Encryption at rest
-   Encryption in transit
-   Parameter configuration
-   Connection limits
-   Monitoring
-   Maintenance windows
-   Read replicas where appropriate
-   Disaster recovery

------------------------------------------------------------------------

# 15. Database Should Not Be Public

A production database should generally be placed in private subnets.

Example:

``` text
Internet
   X
   |
  ALB
   |
Application
   |
Private Database
```

Security groups should allow database traffic only from the application
tier.

Example:

``` text
App Security Group
       |
       | TCP 5432
       v
DB Security Group
```

For MySQL, the database port is commonly 3306.

For PostgreSQL, the database port is commonly 5432.

------------------------------------------------------------------------

# 16. ElastiCache / Redis

Redis can be used as a caching layer.

Example:

``` text
Application
    |
    +---- Cache Hit ----> Redis
    |
    +---- Cache Miss ---> Database
```

Suppose product information is requested frequently.

Without cache:

``` text
User
 |
API
 |
Database
```

With Redis:

``` text
User
 |
API
 |
Redis
 |
 +-- Hit  -> Return data
 |
 +-- Miss -> Database
```

This can reduce database load and improve response time.

## Important production point

Redis should not automatically be treated as the source of truth.

Usually:

``` text
Database = Source of truth
Redis    = Performance layer
```

If cached data disappears, the application should be able to retrieve it
again from the database when appropriate.

------------------------------------------------------------------------

# 17. Amazon S3 for Media and Documents

Do not store large files such as:

-   Product images
-   Videos
-   PDFs
-   User documents
-   Backups

inside relational database tables unless there is a specific reason.

A common design is:

``` text
Application
    |
    v
S3
    |
    +--- Images
    +--- Documents
    +--- Exports
```

The database can store metadata:

``` text
Document ID
User ID
S3 Object Key
Created Time
File Type
```

while the actual file remains in S3.

------------------------------------------------------------------------

# 18. End-to-End Request Flow

Consider an e-commerce user opening:

``` text
https://www.example.com
```

### Step 1 --- DNS

The browser resolves the domain using Route 53.

``` text
User
 |
Route 53
```

### Step 2 --- CloudFront

The request reaches CloudFront.

``` text
Route 53
 |
CloudFront
```

### Step 3 --- Static frontend

CloudFront retrieves cached frontend content or fetches it from S3.

``` text
CloudFront
 |
S3
```

### Step 4 --- API call

The frontend calls:

``` text
https://api.example.com/products
```

The API request goes through the API entry point, such as CloudFront/WAF
and then ALB depending on the architecture.

``` text
CloudFront / WAF
       |
       v
      ALB
```

### Step 5 --- Application server

ALB selects a healthy application instance.

``` text
ALB
 |
 +-- EC2-A
 +-- EC2-B
```

### Step 6 --- Cache

The application checks Redis.

``` text
Application
 |
Redis
```

If data is cached, return it.

### Step 7 --- Database

If there is a cache miss:

``` text
Application
 |
Aurora
```

### Step 8 --- Response

The response travels back:

``` text
Aurora / Redis
      |
 Application
      |
     ALB
      |
CloudFront
      |
    User
```

------------------------------------------------------------------------

# 19. Security Group Design

A production architecture should use layered network controls.

Example:

``` text
Internet
   |
   v
ALB Security Group
   |
   | 443
   v
App Security Group
   |
   | 5432 / DB port
   v
DB Security Group
```

Conceptually:

### ALB SG

Allow:

``` text
HTTPS 443
Source: Internet or approved sources
```

### App SG

Allow:

``` text
Application port
Source: ALB Security Group
```

### DB SG

Allow:

``` text
Database port
Source: App Security Group
```

This is preferable to allowing:

``` text
0.0.0.0/0
```

to the database.

------------------------------------------------------------------------

# 20. Network ACL vs Security Group

### Security Group

-   Stateful
-   Attached to ENI/resource
-   Commonly used for application-level network access control

### Network ACL

-   Stateless
-   Applied at subnet level
-   Supports explicit allow/deny rules

In most application architectures, security groups are the primary
control mechanism, while NACLs can provide an additional subnet-level
layer.

------------------------------------------------------------------------

# 21. Monitoring and Observability

Production architecture is incomplete without observability.

Use **CloudWatch** for:

-   CPU metrics
-   Memory metrics with appropriate agents/integrations
-   ALB request count
-   ALB latency
-   4xx/5xx errors
-   EC2 health
-   Application logs
-   Database metrics
-   Alarms

Example:

``` text
Application
    |
CloudWatch Logs
    |
Metrics / Alarms
    |
SNS / Incident Management
```

Useful alerts include:

-   High 5xx rate
-   High latency
-   Unhealthy targets
-   Database CPU/storage issues
-   Connection exhaustion
-   High memory
-   Auto Scaling capacity problems

------------------------------------------------------------------------

# 22. AWS CloudTrail

CloudTrail records AWS API activity.

Example:

``` text
Who changed the security group?
Who modified the IAM policy?
Who deleted an S3 object?
Who changed an AWS resource?
```

CloudTrail is primarily for **API activity auditing**, not application
performance monitoring.

------------------------------------------------------------------------

# 23. AWS KMS

Use AWS KMS for encryption key management.

Possible encryption use cases:

``` text
S3
RDS / Aurora
EBS
Secrets
Backups
```

Example:

``` text
Application
    |
Encrypted data
    |
AWS Service
    |
KMS Key
```

Use appropriate key policies and IAM permissions.

------------------------------------------------------------------------

# 24. AWS Systems Manager

Systems Manager can help manage EC2 instances without relying on direct
SSH access for every operational task.

Useful capabilities include:

-   Session Manager
-   Patch management
-   Run Command
-   Parameter Store
-   Automation

A production-oriented approach is:

``` text
Engineer
   |
Systems Manager Session Manager
   |
Private EC2
```

This can reduce the need for publicly exposed SSH.

------------------------------------------------------------------------

# 25. AWS Inspector

Inspector can help identify vulnerabilities in supported workloads.

It can be used as part of a vulnerability-management process for:

-   EC2 workloads
-   Container images
-   Other supported AWS resources

Security scanning should be integrated into the broader security
lifecycle rather than treated as a one-time activity.

------------------------------------------------------------------------

# 26. Backup and Disaster Recovery

Production systems need a recovery strategy.

Potential backup targets include:

-   Aurora/RDS automated backups
-   Database snapshots
-   S3 versioning
-   AWS Backup
-   Cross-region copies where required

A basic DR process should define:

``` text
RPO = How much data can we afford to lose?

RTO = How quickly must service be restored?
```

Example:

``` text
RPO: 15 minutes
RTO: 1 hour
```

The architecture should be designed and tested against these targets.

------------------------------------------------------------------------

# 27. High Availability

High availability means designing to reduce service interruption when
components fail.

Example:

``` text
                 ALB
               /     \
              /       \
           AZ-A       AZ-B
            |           |
           EC2         EC2
             \         /
              \       /
             Database
```

Avoid:

``` text
Single EC2
Single AZ
Single database instance
Single NAT Gateway for critical multi-AZ traffic without considering AZ failure implications
```

For critical production systems, evaluate the failure domain of every
component.

------------------------------------------------------------------------

# 28. Scalability

## Vertical Scaling

Increase capacity of one instance.

``` text
t3.medium
   |
   v
t3.large
```

## Horizontal Scaling

Add more instances.

``` text
2 EC2
 |
 v
6 EC2
```

For web applications, horizontal scaling with an ALB and ASG is often
preferred because it improves both scalability and resilience.

------------------------------------------------------------------------

# 29. Stateless Application Design

A major production principle is to keep application servers as stateless
as practical.

Avoid storing critical session state only on one EC2 instance.

Bad:

``` text
User -> EC2-A
          |
       Local Session
```

If EC2-A fails, the session may be lost.

Better options include:

``` text
EC2-A \
EC2-B  ---> Shared session/cache/data store
EC2-C /
```

Possible technologies include Redis or a database, depending on
requirements.

This makes horizontal scaling easier.

------------------------------------------------------------------------

# 30. Deployment Strategy

Production deployments should avoid unnecessary downtime.

Common strategies:

### Rolling Deployment

Replace/update instances gradually.

### Blue-Green Deployment

``` text
Blue  = Current Production
Green = New Version
```

Test Green, then shift traffic.

### Canary Deployment

Send a small percentage of traffic to the new version.

``` text
95% -> Version A
5%  -> Version B
```

Monitor:

-   Error rate
-   Latency
-   Business metrics
-   Logs

Then increase traffic if healthy.

------------------------------------------------------------------------

# 31. CI/CD Example

A production pipeline may look like:

``` text
Developer
   |
Git Repository
   |
CI
   |
Build + Unit Tests
   |
Security Scan
   |
Artifact/Image Build
   |
Deployment
   |
ALB / ASG
   |
Production
```

For infrastructure, Infrastructure as Code such as Terraform or AWS
CloudFormation can be used.

------------------------------------------------------------------------

# 32. Secrets Management

Never hard-code:

``` text
DB_PASSWORD=abc123
AWS_ACCESS_KEY=...
API_KEY=...
```

inside application source code.

Use appropriate secret-management mechanisms such as:

-   AWS Secrets Manager
-   AWS Systems Manager Parameter Store
-   IAM roles
-   KMS encryption

Example:

``` text
Application
    |
Secrets Manager
    |
Database Credentials
```

Applications should retrieve secrets securely with least-privilege
permissions.

------------------------------------------------------------------------

# 33. Database Connection Management

A common production problem is database connection exhaustion.

Example:

``` text
100 EC2 instances
       |
Each opens many DB connections
       |
Database connection limit reached
       |
Requests fail
```

Mitigation can include:

-   Connection pooling
-   Proper pool sizing
-   Application tuning
-   Database scaling
-   RDS Proxy where appropriate
-   Monitoring connection metrics

Scaling EC2 instances does not automatically mean the database can
handle unlimited connections.

------------------------------------------------------------------------

# 34. Caching Pitfalls

Caching improves performance but introduces consistency considerations.

Questions to answer:

-   How long should data remain cached?
-   When should cache entries expire?
-   What happens after an update?
-   What happens if Redis becomes unavailable?
-   Is stale data acceptable?

Example:

``` text
Product price updated
        |
Database updated
        |
Cache still has old value
```

The application needs an appropriate cache invalidation/TTL strategy.

------------------------------------------------------------------------

# 35. Common Production Failure Scenarios

## Scenario 1 --- ALB returns 502

Check:

1.  Target health
2.  Application process
3.  Target port
4.  Security groups
5.  Listener/target group configuration
6.  Application logs
7.  Network connectivity
8.  Timeout settings

------------------------------------------------------------------------

## Scenario 2 --- ALB returns 503

Possible causes include:

-   No healthy targets
-   Target group has no available instances
-   Application unavailable

Check:

``` text
ALB
 |
Target Group
 |
Target Health
 |
EC2
 |
Application
```

------------------------------------------------------------------------

## Scenario 3 --- Application is slow

Check:

``` text
User
 |
CloudFront
 |
ALB latency
 |
EC2 CPU/Memory
 |
Application latency
 |
Redis latency
 |
Database latency
```

Do not assume EC2 CPU is the only cause.

------------------------------------------------------------------------

## Scenario 4 --- Database is overloaded

Possible causes:

-   Missing indexes
-   Expensive queries
-   Too many connections
-   Traffic spike
-   Cache miss storm
-   Insufficient DB capacity

Investigate the database workload before simply increasing instance
size.

------------------------------------------------------------------------

## Scenario 5 --- One Availability Zone fails

A properly designed multi-AZ architecture should continue serving
traffic through healthy resources.

Validate:

-   ALB targets in multiple AZs
-   ASG capacity in multiple AZs
-   Database HA configuration
-   Subnet routing
-   NAT architecture
-   Dependencies

------------------------------------------------------------------------

# 36. Security Checklist

Before production, verify:

-   [ ] HTTPS enabled
-   [ ] TLS certificates configured
-   [ ] WAF rules reviewed
-   [ ] S3 buckets private where appropriate
-   [ ] CloudFront origin access configured
-   [ ] EC2 instances have no unnecessary public IPs
-   [ ] Database is not publicly accessible
-   [ ] Security groups follow least privilege
-   [ ] IAM follows least privilege
-   [ ] Secrets are not stored in source code
-   [ ] Encryption at rest enabled where required
-   [ ] Encryption in transit enabled
-   [ ] CloudTrail enabled
-   [ ] Monitoring and alarms configured
-   [ ] Vulnerability scanning configured
-   [ ] Backups configured
-   [ ] Restore process tested

------------------------------------------------------------------------

# 37. Performance Checklist

Monitor:

-   [ ] CloudFront cache hit ratio
-   [ ] ALB request count
-   [ ] ALB target response time
-   [ ] 4xx/5xx rates
-   [ ] EC2 CPU
-   [ ] EC2 memory
-   [ ] Network throughput
-   [ ] Redis hit/miss ratio
-   [ ] Redis memory
-   [ ] Database CPU
-   [ ] Database connections
-   [ ] Database latency
-   [ ] Slow queries
-   [ ] S3 request patterns

------------------------------------------------------------------------

# 38. Availability Checklist

-   [ ] Multiple Availability Zones
-   [ ] Multiple application instances
-   [ ] Auto Scaling enabled
-   [ ] ALB health checks configured
-   [ ] Database HA configured
-   [ ] Backups enabled
-   [ ] Disaster recovery plan documented
-   [ ] RTO/RPO defined
-   [ ] Failure scenarios tested
-   [ ] Monitoring and alerting tested

------------------------------------------------------------------------

# 39. Cost Optimization

Production architecture should balance reliability, performance,
security, and cost.

Potential optimization areas:

### CloudFront

Cache static content aggressively where appropriate.

### EC2

Use right-sized instances and scaling policies.

### NAT Gateway

Review traffic patterns and architecture because NAT Gateway data
processing can become expensive at scale.

### Database

Right-size Aurora/RDS and monitor utilization.

### S3

Use lifecycle policies for objects that become less frequently accessed.

### Logs

Define CloudWatch log retention based on operational and compliance
requirements.

------------------------------------------------------------------------

# 40. What Each AWS Service Is Doing

  Service           Primary Responsibility
  ----------------- ------------------------------------
  Route 53          DNS
  CloudFront        CDN / edge delivery
  S3                Static files / object storage
  WAF               Web-layer protection
  ALB               HTTP/HTTPS load balancing
  EC2               Application compute
  Auto Scaling      Capacity and instance replacement
  IAM Role          AWS permissions for workloads
  Aurora/RDS        Relational database
  ElastiCache       In-memory caching
  CloudWatch        Monitoring, metrics, logs, alarms
  CloudTrail        AWS API auditing
  KMS               Encryption key management
  Systems Manager   Operations and instance management
  Inspector         Vulnerability management
  AWS Backup        Centralized backup management

------------------------------------------------------------------------

# 41. Interview Explanation

If an interviewer asks:

> **"Explain a production 3-tier architecture on AWS."**

A strong answer can be:

> "I would separate the application into presentation, application, and
> data tiers. For the presentation layer, I can host static frontend
> assets in private S3 and deliver them through CloudFront, with Route
> 53 providing DNS. For API traffic, I can use CloudFront and WAF where
> appropriate, followed by an Application Load Balancer. The ALB
> distributes traffic across application instances running in private
> subnets across multiple Availability Zones and managed through an Auto
> Scaling Group.
>
> The application tier accesses the data tier through tightly controlled
> security groups. Aurora or RDS handles transactional relational data,
> Redis can be used for caching, and S3 can store media and documents. I
> would use IAM roles instead of hard-coded AWS credentials, CloudWatch
> for monitoring, CloudTrail for API auditing, KMS for encryption,
> Systems Manager for operations, and automated backups plus a tested
> disaster-recovery strategy.
>
> The key production goals are high availability, horizontal
> scalability, least-privilege security, observability, performance, and
> controlled recovery from failures."

------------------------------------------------------------------------

# 42. Important Corrections to the Simplified Diagram

The provided diagram is useful for explaining the **concept of 3
tiers**, but production architecture should be refined in several areas.

### 1. CloudFront does not have to send every request to ALB

A common production design is:

``` text
CloudFront
   |
   +---- S3       -> Static frontend
   |
   +---- ALB      -> API/backend
```

### 2. S3 frontend should preferably remain private

Use CloudFront with appropriate origin access controls rather than
exposing the bucket unnecessarily.

### 3. Application instances should generally be private

The ALB can be internet-facing while EC2 instances remain in private
subnets.

### 4. Database should be private

The database should not accept direct internet traffic.

### 5. ElastiCache is not normally a peer database

Redis is generally used as a cache/session/data acceleration layer
rather than the primary relational data store.

### 6. Auto Scaling is not normally "between" two ASGs

An actual production design may use one Auto Scaling Group spanning
multiple Availability Zones, or multiple groups for distinct
workloads/services.

### 7. Route 53 has multiple roles

Route 53 is primarily DNS. Health checks and routing policies can
support availability/failover designs, but it is not the same thing as
an application health-check system.

------------------------------------------------------------------------

# 43. Production Mental Model

When designing a 3-tier AWS architecture, think in this order:

``` text
1. User traffic
       |
2. DNS
       |
3. Edge / CDN
       |
4. Web security
       |
5. Load balancing
       |
6. Application compute
       |
7. Cache
       |
8. Database / Storage
       |
9. Monitoring
       |
10. Backup + Disaster Recovery
```

Then ask five questions for every tier:

### Availability

> What happens if this component fails?

### Scalability

> What happens if traffic increases 10x?

### Security

> Who can access this component?

### Observability

> How will I know it is failing?

### Recovery

> How will I restore service and data?

------------------------------------------------------------------------

# 44. Quick Revision

``` text
Route 53
   |
CloudFront
   |
   +------> S3
   |        Frontend
   |
   +------> WAF
              |
             ALB
              |
       +------+------+
       |             |
      EC2           EC2
       |             |
       +------+------+
              |
       +------+------+
       |             |
     Redis         Aurora
       |
      S3
(Media/Documents)
```

### Remember

**Presentation Tier** → S3 + CloudFront

**Application Tier** → ALB + EC2 + Auto Scaling

**Data Tier** → Aurora/RDS + Redis + S3

**Security** → WAF + IAM + Security Groups + KMS

**Operations** → CloudWatch + CloudTrail + Systems Manager + Inspector

**Resilience** → Multi-AZ + Auto Scaling + Backups + DR

------------------------------------------------------------------------

# 45. Final Production Takeaway

A production-ready 3-tier architecture is not simply:

``` text
Frontend
   |
Backend
   |
Database
```

It is about designing **how the system behaves when traffic grows,
components fail, attackers send unwanted traffic, deployments go wrong,
or data needs to be recovered.**

The architecture should therefore combine:

``` text
3-Tier Separation
       +
Multi-AZ High Availability
       +
Auto Scaling
       +
Least-Privilege Security
       +
Caching
       +
Observability
       +
Backup / DR
       +
Safe CI/CD
```

That is the difference between a **diagram that works** and an
**architecture that is production-ready**.
