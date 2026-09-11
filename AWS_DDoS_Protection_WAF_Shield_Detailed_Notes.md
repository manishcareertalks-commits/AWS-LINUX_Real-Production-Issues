# AWS DDoS Protection Using AWS WAF, Shield, Route 53, ALB and Auto Scaling

## 1. Interview Scenario

### Question

> **Your AWS application is getting thousands of requests per second. How would you stop a DDoS attack?**

The key is to avoid treating every high-traffic event as a DDoS attack. A sudden traffic spike can be legitimate, such as a product launch, marketing campaign, flash sale, or viral event.

A good AWS design uses **defense in depth**:

```text
Internet
   |
   v
Amazon Route 53
   |
   v
AWS WAF
   |---- Managed Rule Groups
   |---- IP Reputation Rules
   |---- Custom Rules
   |---- Rate-Based Rules
   |
   v
Application Load Balancer
   |
   v
EC2 Auto Scaling Group
   |
   v
Application
```

For a production architecture, **Amazon CloudFront can also be placed in front of the application**, with AWS WAF attached at the edge.

---

# 2. What is a DDoS Attack?

**DDoS = Distributed Denial of Service.**

The objective is to overwhelm an application, network, server, or service with a large amount of traffic or requests so that legitimate users cannot access it.

A DDoS attack can involve:

- Huge volumes of network traffic
- Large numbers of HTTP/HTTPS requests
- Bot-generated requests
- Repeated requests from many IP addresses
- Requests targeting expensive application endpoints
- Attempts to exhaust application resources such as CPU, memory, database connections, or worker threads

### Simple example

Suppose an application normally receives:

```text
1,000 requests/second
```

During an attack:

```text
200,000 requests/second
```

If the application cannot handle the traffic, users may experience:

- High latency
- HTTP 5xx errors
- Timeouts
- Connection failures
- Increased infrastructure cost
- Application or database saturation

---

# 3. DDoS vs Legitimate Traffic Spike

This is an important interview distinction.

## Legitimate traffic spike

Example:

```text
Normal traffic  →  1K req/sec
Product launch  →  20K req/sec
```

The requests may come from real users and should generally be served.

The solution may be:

- ALB
- Auto Scaling
- CloudFront caching
- Database scaling
- Application optimization

## DDoS traffic

Example:

```text
Normal traffic  →  1K req/sec
Attack traffic  →  200K req/sec
```

A large percentage may be malicious or automated.

The solution should include:

- AWS Shield
- AWS WAF
- Rate-based rules
- IP reputation
- Managed rule groups
- Custom rules
- Monitoring and alerting
- Application-layer protections

---

# 4. Layered AWS Protection Strategy

A strong answer should explain the protection layer by layer.

```text
                 INTERNET
                    |
                    v
             +--------------+
             |   Route 53   |
             |     DNS      |
             +--------------+
                    |
                    v
             +--------------+
             | AWS Shield   |
             | DDoS Defense |
             +--------------+
                    |
                    v
             +--------------+
             |   AWS WAF    |
             |              |
             | Managed Rules|
             | IP Reputation|
             | Custom Rules |
             | Rate Limits  |
             +--------------+
                    |
                    v
             +--------------+
             |     ALB      |
             +--------------+
                    |
                    v
             +--------------+
             | Auto Scaling |
             |    Group     |
             +--------------+
                    |
                    v
             +--------------+
             | Application  |
             +--------------+
```

Each component has a different responsibility.

---

# 5. Amazon Route 53

Route 53 is the DNS layer.

Example:

```text
www.example.com
       |
       v
Route 53
       |
       v
Application endpoint
```

Route 53 provides highly available DNS services and can be used with routing policies and health checks.

### Important interview point

**Route 53 is not the primary HTTP DDoS filtering layer.**

It helps with DNS availability and routing, while services such as Shield and WAF provide dedicated DDoS/application-request protection.

Do not say:

> "Route 53 blocks all DDoS requests."

A better statement is:

> "Route 53 provides highly available DNS and routing, while AWS Shield and AWS WAF provide DDoS and application-layer request protection."

---

# 6. AWS Shield

AWS Shield is AWS's managed DDoS protection service.

There are two important levels:

## AWS Shield Standard

Shield Standard is included automatically with AWS services and provides baseline protection against common network and transport-layer DDoS attacks.

It is not something you normally "enable" manually.

### Important correction

Instead of saying:

> "Enable AWS Shield Standard."

Say:

> "AWS Shield Standard provides automatic baseline DDoS protection for eligible AWS services."

---

# 7. AWS Shield Advanced

For business-critical applications, AWS Shield Advanced provides additional DDoS protection and response capabilities.

It can provide:

- Enhanced DDoS protection
- Additional visibility
- Advanced detection and mitigation capabilities
- DDoS response support
- Integration with AWS WAF
- Additional protections for supported resources

It is a paid service.

For critical production workloads, Shield Advanced should be considered based on:

- Business impact
- Availability requirements
- Attack risk
- Cost
- Required response capabilities

---

# 8. AWS WAF

AWS WAF = **AWS Web Application Firewall**.

WAF operates at the application/request layer.

A Web ACL contains rules that inspect incoming web requests and determine whether they should be:

```text
ALLOW
BLOCK
COUNT
CHALLENGE
```

Depending on the rule and supported configuration.

A Web ACL can be associated with supported AWS resources such as:

- CloudFront distributions
- Application Load Balancers
- API Gateway
- AppSync
- Other supported AWS resources

---

# 9. AWS WAF Managed Rule Groups

AWS provides managed rule groups that help protect applications against common threats.

Examples include protections for:

- Common web exploits
- Malicious requests
- IP reputation
- Anonymous IP sources
- Known attack patterns

Managed rules reduce the need to build every security rule from scratch.

### Example flow

```text
Incoming request
       |
       v
AWS WAF
       |
       v
Managed Rule Groups
       |
       +---- Malicious ---> BLOCK
       |
       +---- Suspicious --> COUNT/CHALLENGE/BLOCK
       |
       +---- Normal ------> Continue
```

Always test managed rules carefully before moving every rule to `BLOCK`, because legitimate traffic can sometimes match security rules.

---

# 10. IP Reputation Lists

IP reputation rules use known threat intelligence to identify IP addresses associated with malicious activity.

AWS WAF provides managed IP reputation rule groups.

One important group is:

```text
AWSManagedRulesAmazonIpReputationList
```

It contains AWS-managed intelligence around IP addresses associated with malicious activity, reconnaissance, and other threats.

The rule group includes rules such as:

```text
AWSManagedIPReputationList
AWSManagedReconnaissanceList
AWSManagedIPDDoSList
```

The exact rule actions and behavior should be checked against the current AWS WAF documentation.

### Why it helps

If an attacker is repeatedly sending traffic from known malicious infrastructure:

```text
Attacker IP
     |
     v
AWS WAF IP Reputation
     |
     v
BLOCK
```

This prevents those requests from reaching the application.

### Important limitation

IP reputation is not enough by itself.

Attackers can use:

- Large botnets
- Rotating IP addresses
- Residential proxies
- Cloud infrastructure
- Distributed sources

Therefore, combine IP reputation with rate limiting, managed rules, application-specific rules, and Shield.

---

# 11. Custom AWS WAF Rules

Custom rules allow the security team to enforce application-specific behavior.

Examples:

### Block a known malicious IP

```text
IP = 203.0.113.10
       |
       v
BLOCK
```

### Restrict an administrative endpoint

```text
/admin/*
     |
     v
Allow only trusted source IPs
```

### Restrict HTTP methods

For example:

```text
Unexpected DELETE request
        |
        v
BLOCK
```

### Geo restrictions

Example:

```text
Requests from selected countries
            |
            v
     Additional inspection
```

Geographic restrictions should only be used when they match the application's legitimate user base.

---

# 12. Rate-Based Rules

This is one of the most important protections for an HTTP request flood.

A rate-based rule tracks requests according to its aggregation criteria and applies the configured action when the limit is exceeded.

Example from the architecture:

```text
100 requests / 5-minute evaluation window
```

or conceptually:

```text
IP address
     |
     v
Request count
     |
     +---- Below limit ---> ALLOW
     |
     +---- Above limit ---> BLOCK
```

### Example

Suppose a rate-based rule is designed around:

```text
100 requests
per IP
per evaluation window
```

An IP sends:

```text
50 requests  -> Allowed
80 requests  -> Allowed
120 requests -> Rate limit exceeded
```

The requests from the affected aggregation instance can then receive the configured rule action.

---

# 13. Do Not Use One Global Rate Limit Blindly

A common interview mistake is:

> "I will block every IP sending more than 100 requests."

That can cause false positives.

For example:

```text
Corporate NAT
    |
    +--- User 1
    +--- User 2
    +--- User 3
    +--- User 4
    +--- ...
```

Thousands of legitimate users may appear behind a smaller number of public IP addresses.

Instead, understand:

- Normal traffic pattern
- Endpoint behavior
- User behavior
- Authentication state
- Geographic distribution
- Request cost
- Application architecture

Then define an appropriate rate-limit strategy.

---

# 14. Rate Limit Specific Endpoints

Some endpoints are much more expensive than others.

For example:

```text
GET /
GET /static/app.js
GET /images/logo.png
```

may be relatively cheap.

But:

```text
POST /login
POST /checkout
POST /search
POST /forgot-password
POST /generate-report
```

may be expensive.

A better WAF strategy can apply tighter controls to high-risk or high-cost endpoints.

Example:

```text
/login
/signup
/forgot-password
/search
```

can receive stricter rate-based rules.

---

# 15. Scope-Down Statements

A scope-down statement narrows the requests that a containing WAF rule evaluates.

This is extremely useful for targeted rate limiting.

Example:

```text
Rate-Based Rule
      |
      v
Scope Down
      |
      +---- /login
      +---- /signup
      +---- /forgot-password
      |
      v
Rate Limit
```

Instead of rate limiting the entire application, you can focus on a specific category of traffic.

### Example

```text
IF URI starts with /login
AND request rate exceeds threshold
THEN BLOCK
```

This can reduce false positives and protect sensitive endpoints without unnecessarily limiting normal static traffic.

---

# 16. ALB - Application Load Balancer

The Application Load Balancer distributes incoming HTTP/HTTPS requests across healthy application targets.

Example:

```text
             ALB
              |
      +-------+-------+
      |       |       |
     EC2     EC2     EC2
```

Benefits include:

- Load distribution
- Health checks
- High availability across Availability Zones
- Integration with Auto Scaling
- TLS termination
- Integration with AWS WAF

### Important point

The ALB itself is not your complete DDoS strategy.

Use:

```text
Shield + WAF + ALB + Auto Scaling
```

rather than relying on the ALB alone.

---

# 17. Auto Scaling Group

Even after malicious traffic is filtered, legitimate traffic can increase significantly.

An Auto Scaling Group can increase the number of EC2 instances when demand rises.

Example:

```text
Normal traffic

ALB
 |
 +--- EC2
 +--- EC2
 +--- EC2


High legitimate traffic

ALB
 |
 +--- EC2
 +--- EC2
 +--- EC2
 +--- EC2
 +--- EC2
 +--- EC2
```

Auto Scaling improves application capacity, but it is **not a DDoS mitigation mechanism by itself**.

If malicious requests are allowed through unchecked:

```text
DDoS traffic
     |
     v
ALB
     |
     v
Auto Scaling
     |
     v
More EC2 instances
     |
     v
Higher cost + possible downstream saturation
```

Therefore:

> **Filter malicious traffic before scaling becomes the only response.**

---

# 18. Why Auto Scaling Alone Is Dangerous

Suppose:

```text
Normal:
10 EC2 instances

Attack:
100,000 malicious requests/sec
```

If the system simply scales based on CPU/request count:

```text
10 EC2
  |
  v
30 EC2
  |
  v
60 EC2
  |
  v
100 EC2
```

The attacker may cause:

- Infrastructure cost increase
- Database saturation
- Cache pressure
- Queue growth
- Increased latency
- Downstream service exhaustion

This is sometimes referred to as **scaling under attack**.

The better strategy is:

```text
Detect
  ↓
Filter
  ↓
Rate limit
  ↓
Block
  ↓
Then scale legitimate traffic
```

---

# 19. Monitoring With Amazon CloudWatch

You need observability during an attack.

Monitor:

- AWS WAF metrics
- Allowed requests
- Blocked requests
- Counted requests
- Rate-based rule activity
- ALB request count
- ALB target response time
- HTTP 4xx
- HTTP 5xx
- Target health
- EC2 CPU
- EC2 network metrics
- Auto Scaling activity
- Application logs

Conceptually:

```text
Traffic Spike
     |
     v
CloudWatch
     |
     +---- WAF blocked requests
     +---- ALB request count
     +---- ALB latency
     +---- 5xx errors
     +---- EC2 CPU
     +---- ASG scaling
```

---

# 20. WAF Logs

For deeper investigation, enable and analyze AWS WAF logging.

WAF logs can help answer:

- Which rule matched?
- Which IP generated the request?
- Which URI was targeted?
- Which HTTP method was used?
- Was the request allowed or blocked?
- Which headers were present?
- Which geographic information was associated with the request?

This helps tune WAF rules without blindly blocking legitimate users.

---

# 21. CloudWatch Alarms

Create alarms for abnormal traffic patterns.

Example:

```text
IF blocked WAF requests > threshold
        |
        v
CloudWatch Alarm
        |
        v
SNS / Incident workflow
```

Other useful alarms:

```text
ALB 5xx > threshold
ALB latency > threshold
Request count > threshold
EC2 CPU > threshold
Unhealthy targets > threshold
```

---

# 22. Recommended Production Architecture

A more complete production architecture can look like this:

```text
                         INTERNET
                            |
                            v
                    +---------------+
                    |   Route 53    |
                    |      DNS      |
                    +---------------+
                            |
                            v
                    +---------------+
                    |  CloudFront   |
                    |   CDN / Edge  |
                    +---------------+
                            |
                            v
                    +---------------+
                    |    AWS WAF    |
                    |               |
                    | Managed Rules |
                    | IP Reputation |
                    | Custom Rules  |
                    | Rate Limits   |
                    +---------------+
                            |
                            v
                    +---------------+
                    | AWS Shield    |
                    | Protection    |
                    +---------------+
                            |
                            v
                    +---------------+
                    |      ALB      |
                    +---------------+
                       /     |     \
                      /      |      \
                     v       v       v
                   EC2     EC2      EC2
                     \       |       /
                      \      |      /
                       +-----+-----+
                             |
                             v
                       Backend / DB
```

The exact placement and configuration depends on the AWS service architecture. In particular, if CloudFront is the public entry point, WAF can be associated with the CloudFront distribution so malicious web requests are filtered at the edge.

---

# 23. Why CloudFront Is Useful

CloudFront provides an additional edge layer.

Benefits can include:

- Global edge distribution
- Caching
- Reduced origin traffic
- TLS termination
- Integration with AWS WAF
- Additional resilience for web applications

For static content:

```text
User
 |
 v
CloudFront
 |
 +---- Cache HIT ----> Response
 |
 +---- Cache MISS ---> Origin
```

If an attacker repeatedly requests cacheable content, caching can reduce the number of requests reaching the origin.

For dynamic content, WAF rules and rate limiting become particularly important.

---

# 24. Complete Request Flow

Let's walk through a request.

## Step 1 - Client sends request

```text
Client
   |
   v
www.example.com
```

## Step 2 - Route 53 resolves DNS

```text
Route 53
   |
   v
Application endpoint
```

## Step 3 - Edge protection

If CloudFront is used:

```text
Client
  |
  v
CloudFront
```

## Step 4 - AWS WAF evaluates the request

```text
             AWS WAF
                |
       +--------+--------+
       |        |        |
    Managed   IP Rep.  Custom
     Rules     Rules    Rules
       |        |        |
       +--------+--------+
                |
                v
        Rate-Based Rule
```

## Step 5 - Malicious request

```text
Malicious Request
       |
       v
AWS WAF
       |
       X
     BLOCK
```

The request does not need to reach the application.

## Step 6 - Legitimate request

```text
Legitimate Request
       |
       v
AWS WAF
       |
       ✓
       |
       v
ALB
       |
       v
Healthy EC2
```

---

# 25. What Happens During a DDoS Attack?

Imagine:

```text
Normal traffic:
5,000 requests/sec

Attack traffic:
500,000 requests/sec
```

The layered defense should behave approximately like:

```text
500K requests/sec
       |
       v
AWS edge / Shield protection
       |
       v
AWS WAF
       |
       +---- Known malicious IPs ----> BLOCK
       |
       +---- Malicious patterns ------> BLOCK
       |
       +---- Excessive rate ----------> BLOCK
       |
       +---- Suspicious traffic ------> COUNT/CHALLENGE/BLOCK
       |
       +---- Legitimate traffic ------> ALB
                                      |
                                      v
                                Auto Scaling
```

The goal is to prevent as much malicious traffic as practical from consuming application resources.

---

# 26. Important DDoS Design Principle

Do not think:

> "How can I block every request?"

Think:

> **"How can I absorb legitimate traffic while filtering malicious traffic as early as possible?"**

This leads to a layered architecture:

```text
EDGE PROTECTION
       ↓
APPLICATION FIREWALL
       ↓
RATE LIMITING
       ↓
LOAD BALANCING
       ↓
AUTO SCALING
       ↓
APPLICATION
       ↓
MONITORING
```

---

# 27. Common WAF Rule Categories

A practical Web ACL can contain rules such as:

### Rule 1 - AWS Managed Rules

Protect against common malicious patterns.

```text
Managed Rule Group
       |
       +---- BLOCK
```

### Rule 2 - IP Reputation

```text
Known malicious IP
       |
       v
BLOCK
```

### Rule 3 - Rate Limiting

```text
Too many requests
       |
       v
BLOCK
```

### Rule 4 - Application-specific rule

```text
Suspicious /admin request
       |
       v
Additional restriction
```

### Rule 5 - Geographic rule

```text
Selected geography
       |
       v
Additional inspection/restriction
```

---

# 28. Count vs Block

During rule development, it is often safer to observe first.

## Count

```text
Request
  |
  v
Rule matches
  |
  v
COUNT
  |
  v
Request continues
```

Useful for:

- Testing
- Monitoring false positives
- Understanding traffic

## Block

```text
Request
  |
  v
Rule matches
  |
  v
BLOCK
```

The request is stopped by the WAF.

A common production approach is:

```text
Test / Observe
      ↓
Count
      ↓
Analyze logs
      ↓
Tune rule
      ↓
Block
```

Do not blindly switch a broad rule to `BLOCK`.

---

# 29. Example Rate-Based Design

Suppose the application has these endpoints:

```text
/login
/search
/payment
/report
```

Different endpoints have different resource costs.

A possible conceptual strategy:

```text
/login
  |
  +---- strict rate limit


/payment
  |
  +---- strict rate limit


/report
  |
  +---- very strict rate limit


/static/*
  |
  +---- higher threshold
```

This is usually better than treating every URL exactly the same.

The actual thresholds should be based on measured application behavior.

---

# 30. Important Caveat: DDoS Is Not Only an HTTP Problem

AWS WAF is primarily an application-layer protection service.

DDoS can also target:

- Network layer
- Transport layer
- Application layer
- DNS infrastructure
- Other service dependencies

Therefore:

```text
AWS WAF
```

alone should not be described as a complete DDoS solution.

A better answer is:

```text
AWS Shield
+
AWS WAF
+
CloudFront where appropriate
+
ALB
+
Auto Scaling
+
CloudWatch
+
Application-level controls
```

---

# 31. Protect the Database and Downstream Services

A common mistake is to protect only EC2.

Consider:

```text
Internet
   |
   v
WAF
   |
   v
ALB
   |
   v
EC2
   |
   v
Database
```

Even if EC2 scales successfully, an attack can overload:

- RDS connections
- Database CPU
- Redis
- Elasticsearch/OpenSearch
- Internal APIs
- Message queues
- Third-party APIs

Therefore, rate limiting should sometimes be applied specifically to requests that consume expensive backend resources.

---

# 32. Protect Expensive APIs

Suppose:

```text
POST /generate-report
```

takes 5 seconds and performs several database queries.

An attacker sends:

```text
10,000 requests/sec
```

Even if EC2 scales, the backend can become saturated.

A better design can include:

```text
Client
  |
  v
WAF
  |
  +---- Rate limit /generate-report
  |
  v
API
  |
  v
Queue
  |
  v
Workers
```

This protects the application by controlling expensive operations.

---

# 33. Interview Answer - Short Version

If asked:

> **"How would you stop a DDoS attack on AWS?"**

A strong answer is:

> "I would use a layered DDoS protection strategy. AWS Shield provides baseline DDoS protection, and for critical workloads I would evaluate Shield Advanced. I would put AWS WAF in front of the application with managed rule groups, IP reputation rules, custom application-specific rules, and rate-based rules. For sensitive endpoints such as login or expensive APIs, I would use stricter rate limits and scope-down statements. I would place the application behind an ALB and use an Auto Scaling Group for legitimate traffic spikes. I would also monitor WAF, ALB and application metrics through CloudWatch and use WAF logs for tuning and investigation. If the application is internet-facing and globally distributed, I would consider CloudFront with WAF at the edge."

---

# 34. Interview Answer - Senior-Level Version

A more senior answer should focus on **defense in depth and avoiding false positives**:

> "First, I would determine whether the traffic spike is legitimate or malicious by looking at request patterns, source distribution, endpoints, error rates and application behavior. At the AWS edge, Shield provides baseline DDoS protection, while Shield Advanced can be evaluated for critical workloads. I would use CloudFront where appropriate and attach AWS WAF with managed rule groups, IP reputation, custom rules and carefully tuned rate-based rules. I would use scope-down statements for high-risk endpoints rather than applying one aggressive global threshold. The application would sit behind an ALB with Auto Scaling for legitimate demand. Finally, I would use CloudWatch and WAF logs for detection, alerting, investigation and continuous rule tuning. The goal is to filter malicious traffic as early as possible while preserving legitimate user traffic."

---

# 35. Common Interview Mistakes

## Mistake 1

> "Auto Scaling will solve the DDoS."

### Why it is wrong

Auto Scaling adds capacity but does not identify malicious traffic.

---

## Mistake 2

> "WAF blocks all DDoS attacks."

### Why it is wrong

WAF focuses on web/application-layer requests. DDoS protection requires multiple layers.

---

## Mistake 3

> "Route 53 blocks DDoS."

### Better explanation

Route 53 provides DNS availability and routing capabilities. Shield and WAF provide dedicated DDoS/application protection.

---

## Mistake 4

> "Block every IP sending more than 100 requests."

### Why it is risky

Multiple legitimate users can share one public IP.

---

## Mistake 5

> "Shield Advanced is free."

### Correct concept

Shield Standard is included automatically for eligible AWS services; Shield Advanced is a paid offering.

---

## Mistake 6

> "More EC2 instances always means more protection."

### Why it is wrong

Scaling can increase cost while allowing malicious traffic to continue consuming downstream resources.

---

# 36. Production Checklist

## Edge / DDoS

- [ ] Understand whether CloudFront is appropriate
- [ ] Use AWS Shield baseline protection
- [ ] Evaluate Shield Advanced for critical workloads
- [ ] Design for multi-AZ availability

## AWS WAF

- [ ] Create a Web ACL
- [ ] Add AWS Managed Rules where appropriate
- [ ] Add IP reputation protections
- [ ] Add application-specific custom rules
- [ ] Add rate-based rules
- [ ] Use scope-down statements where appropriate
- [ ] Start broad rules in Count mode when testing
- [ ] Review false positives before Block mode

## Application

- [ ] Use ALB
- [ ] Configure health checks
- [ ] Use Auto Scaling
- [ ] Protect expensive APIs
- [ ] Use caching where appropriate
- [ ] Protect database connections
- [ ] Add application-level throttling where necessary

## Monitoring

- [ ] Enable WAF logging
- [ ] Monitor blocked requests
- [ ] Monitor allowed requests
- [ ] Monitor ALB request count
- [ ] Monitor ALB 4xx/5xx
- [ ] Monitor latency
- [ ] Monitor EC2 CPU/network
- [ ] Monitor Auto Scaling activity
- [ ] Configure CloudWatch alarms
- [ ] Create an incident response process

---

# 37. Key Commands / AWS CLI Concepts

The exact CLI commands depend on whether the WAF scope is `REGIONAL` or `CLOUDFRONT`, and on the resource being protected.

Useful AWS CLI areas to know include:

```bash
aws wafv2 list-web-acls
```

```bash
aws wafv2 get-web-acl
```

```bash
aws wafv2 list-available-managed-rule-groups
```

For rate-based rule investigations, AWS WAF also provides APIs/CLI functionality to retrieve managed keys for a rate-based statement.

Example concept:

```bash
aws wafv2 get-rate-based-statement-managed-keys
```

Always verify the current CLI syntax and required parameters for your region, scope, and Web ACL.

---

# 38. Final Architecture Summary

The architecture shown in the reel can be remembered as:

```text
              DDoS / BOT TRAFFIC
                       |
                       v
                 Amazon Route 53
                       |
                       v
                  AWS Shield
                       |
                       v
                  AWS WAF
          +------------+------------+
          |            |            |
     Managed Rules  IP Reputation  Custom Rules
          |            |            |
          +------------+------------+
                       |
                Rate-Based Rules
                       |
             +---------+---------+
             |                   |
        Malicious            Legitimate
             |                   |
             v                   v
           BLOCK                 ALB
                                 |
                                 v
                           Auto Scaling
                                 |
                    +------------+------------+
                    |            |            |
                   EC2          EC2          EC2
                    |            |            |
                    +------------+------------+
                                 |
                                 v
                            Application
                                 |
                                 v
                              Backend
```

---

# 39. The Core Message

The most important idea is:

> **Don't rely on a single AWS service to stop a DDoS attack.**

Use layered protection:

```text
Shield
  ↓
WAF
  ↓
Managed Rules
  ↓
IP Reputation
  ↓
Custom Rules
  ↓
Rate-Based Rules
  ↓
ALB
  ↓
Auto Scaling
  ↓
Monitoring
```

And for globally distributed web applications:

```text
Route 53
   ↓
CloudFront
   ↓
AWS WAF
   ↓
ALB
   ↓
Auto Scaling
   ↓
Application
```

The objective is to **detect and filter malicious traffic as early as possible**, while allowing legitimate users to continue accessing the application.

---

# 40. AWS Documentation

For deeper study, refer to the current AWS documentation:

- AWS WAF Web ACLs: https://docs.aws.amazon.com/waf/latest/developerguide/web-acl.html
- AWS WAF Rate-Based Rules: https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based.html
- AWS WAF Scope-Down Statements: https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-scope-down-statements.html
- AWS WAF IP Reputation Rule Groups: https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-ip-rep.html
- AWS WAF DDoS Prevention Rule Group: https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-anti-ddos.html
- AWS Best Practices for DDoS Resiliency: https://docs.aws.amazon.com/whitepapers/latest/aws-best-practices-ddos-resiliency/welcome.html

> **Note:** AWS services, rule groups, limits and recommended configurations can change. Always verify the current AWS documentation before implementing a production security architecture.
