# Application Is Very Slow --- Troubleshooting & Debugging Guide

## Interview Scenario

> **Your application is very slow. What could be the possible reasons,
> and how would you debug it?**

The best approach is to **isolate the bottleneck layer by layer**
instead of immediately assuming that the application code is the
problem.

A typical request path can look like:

``` text
End Users
   ↓
Route 53
   ↓
Load Balancer
   ↓
EC2 / Web Server
   ↓
Database / Cache / S3 / External APIs
```

The objective is to identify **where the latency is introduced**, find
the root cause, fix it, and then prevent it from happening again.

------------------------------------------------------------------------

# 1. End-to-End Troubleshooting Approach

Use this sequence:

``` text
User
 ↓
DNS
 ↓
Load Balancer
 ↓
EC2 / Application
 ↓
Database / Cache / External Services
```

At every layer, ask:

-   Is the component healthy?
-   Is latency increasing?
-   Are there errors?
-   Is there resource saturation?
-   Are there connection problems?
-   Is traffic unusually high?
-   Is a dependency responding slowly?

The key principle is:

> **Don't troubleshoot randomly. Isolate the slow layer first.**

------------------------------------------------------------------------

# 2. Layer 1 --- Users / Traffic

Start by understanding **when and for whom** the application is slow.

### Questions to ask

-   Is the issue affecting all users or only some users?
-   Is it happening continuously or intermittently?
-   Did the issue start after a deployment?
-   Is traffic higher than normal?
-   Is one API/page slower than others?
-   Is the problem region-specific?
-   Is the application slow only during peak traffic?

### Check traffic patterns

Look for:

-   Sudden traffic spikes
-   Increased concurrent users
-   Unusual request rates
-   Bot traffic
-   Large numbers of requests to a particular endpoint

A sudden increase in traffic can cause:

``` text
More Users
    ↓
More Requests
    ↓
More Connections
    ↓
Resource Saturation
    ↓
Higher Latency
```

------------------------------------------------------------------------

# 3. Layer 2 --- DNS / Route 53

If the request path uses Amazon Route 53, verify that DNS resolution is
working correctly.

### What to check

-   DNS resolution
-   DNS records
-   TTL
-   Health checks
-   Routing policy
-   Failover configuration
-   Regional routing behavior

### Commands

``` bash
dig example.com
```

``` bash
nslookup example.com
```

You can also test response time:

``` bash
time dig example.com
```

### Possible problems

-   Incorrect DNS record
-   DNS misconfiguration
-   Failed health checks
-   Incorrect routing
-   Unexpected DNS target
-   High DNS resolution latency

DNS is usually not the main cause of sustained application slowness, but
it should be eliminated early in an end-to-end investigation.

------------------------------------------------------------------------

# 4. Layer 3 --- Load Balancer

If traffic reaches an AWS Load Balancer, check whether the load balancer
or its targets are contributing to latency.

### Check

-   Target health
-   Response time
-   HTTP status codes
-   Request count
-   Active connections
-   Connection errors
-   Target response time
-   Unhealthy targets

For an Application Load Balancer, useful CloudWatch metrics include:

``` text
TargetResponseTime
RequestCount
ActiveConnectionCount
HTTPCode_Target_5XX_Count
HTTPCode_ELB_5XX_Count
HealthyHostCount
UnHealthyHostCount
```

### Important question

If the load balancer is healthy but the **target response time is
high**, the bottleneck may be inside the application or its
dependencies.

Example:

``` text
Client
  ↓
ALB
  ↓
Target Response Time = 5 seconds
  ↓
EC2 / Application
  ↓
Investigate application layer
```

------------------------------------------------------------------------

# 5. Layer 4 --- EC2 / Web Server

Now investigate the actual compute layer.

The first things to check are:

-   CPU
-   Memory
-   Disk
-   Disk I/O
-   Network
-   Processes
-   Connections
-   Application logs
-   Web server logs

------------------------------------------------------------------------

## CPU Investigation

Check CPU and running processes:

``` bash
top
```

or:

``` bash
htop
```

Look for:

-   High CPU utilization
-   One process consuming excessive CPU
-   CPU constantly near capacity
-   Sudden CPU spikes

You can also check processes sorted by CPU:

``` bash
ps aux --sort=-%cpu
```

### Possible causes

-   CPU-intensive application code
-   Infinite loop
-   Too many concurrent requests
-   Expensive calculations
-   Background process consuming CPU
-   Insufficient EC2 capacity

------------------------------------------------------------------------

# 6. Memory Investigation

Check memory:

``` bash
free -h
```

Look for:

-   Low available memory
-   High swap usage
-   Memory leaks
-   Processes consuming excessive RAM
-   OOM-related events

Find memory-heavy processes:

``` bash
ps aux --sort=-%mem
```

### Possible impact

``` text
High Memory Usage
       ↓
Memory Pressure
       ↓
Swap / OOM
       ↓
Application Performance Degrades
```

------------------------------------------------------------------------

# 7. System Performance

Use:

``` bash
vmstat 1 5
```

This gives a quick view of:

-   CPU
-   Memory
-   Processes
-   Swap
-   System activity

Useful indicators include:

-   High CPU wait
-   High I/O wait
-   Swapping
-   Excessive process activity

------------------------------------------------------------------------

# 8. Disk Space

Check disk usage:

``` bash
df -h
```

Also check inode usage:

``` bash
df -i
```

A server can have available disk space but still run into problems if it
runs out of inodes.

Possible causes:

-   Application logs filling the disk
-   Temporary files
-   Large uploads
-   Old backups
-   Log rotation failure

------------------------------------------------------------------------

# 9. Disk I/O

High disk latency can make an otherwise healthy application appear slow.

Use:

``` bash
iostat -xz 1 5
```

Check:

-   Disk utilization
-   Read/write activity
-   I/O wait
-   Average request latency
-   Queue depth

Possible causes:

-   Slow EBS volume
-   High disk utilization
-   Burstable volume performance issues
-   Excessive logging
-   Database disk pressure

------------------------------------------------------------------------

# 10. Network Investigation

Check network behavior and connections.

Useful commands:

``` bash
ss -tulpn
```

``` bash
netstat -i
```

Depending on the issue, investigate:

-   Active connections
-   Listening ports
-   Connection backlog
-   Network errors
-   Packet loss
-   Network latency
-   Bandwidth utilization

Possible causes:

-   Too many connections
-   Network congestion
-   Packet loss
-   Incorrect security rules
-   Application connection leaks

------------------------------------------------------------------------

# 11. Application / Web Server Logs

Logs are critical because metrics tell you **that** something is wrong,
while logs often help explain **why**.

For Nginx:

``` bash
tail -f /var/log/nginx/access.log
```

``` bash
tail -f /var/log/nginx/error.log
```

For Apache:

``` bash
tail -f /var/log/httpd/access_log
```

``` bash
tail -f /var/log/httpd/error_log
```

Look for:

-   HTTP 5xx errors
-   Timeout errors
-   Slow requests
-   Connection failures
-   Application exceptions
-   Database connection errors
-   External API failures

------------------------------------------------------------------------

# 12. Kernel / System Logs

Check kernel-related messages:

``` bash
dmesg | tail
```

Look for:

-   OOM events
-   Disk errors
-   Network issues
-   Driver problems
-   Kernel-level failures

------------------------------------------------------------------------

# 13. AWS CloudWatch Investigation

Use CloudWatch to correlate infrastructure metrics over the same time
period.

Check:

### EC2

-   CPUUtilization
-   NetworkIn
-   NetworkOut
-   DiskReadOps
-   DiskWriteOps
-   DiskReadBytes
-   DiskWriteBytes
-   Status checks

### Load Balancer

-   RequestCount
-   TargetResponseTime
-   ActiveConnectionCount
-   HTTP error metrics
-   HealthyHostCount
-   UnHealthyHostCount

### Other AWS services

Depending on the architecture, also investigate:

-   RDS
-   ElastiCache
-   DynamoDB
-   S3
-   Lambda
-   API Gateway
-   SQS
-   SNS
-   NAT Gateway

------------------------------------------------------------------------

# 14. Database Slowness

A very common reason for a slow application is a slow database.

Possible causes:

-   Slow SQL queries
-   Missing indexes
-   Lock contention
-   High connection count
-   CPU saturation
-   Memory pressure
-   Disk I/O
-   Connection pool exhaustion
-   Large table scans

Typical flow:

``` text
Application
    ↓
Database Query
    ↓
Slow SQL
    ↓
Application Waits
    ↓
User Sees High Latency
```

### What to investigate

-   Query latency
-   Slow query logs
-   Database CPU
-   Database connections
-   Lock waits
-   IOPS
-   Connection pool usage

For RDS, use CloudWatch and database-specific performance tools such as
Performance Insights where available.

------------------------------------------------------------------------

# 15. Cache Problems

If the architecture uses a cache such as Redis or Memcached, check:

-   Cache hit ratio
-   Cache misses
-   Memory usage
-   Evictions
-   Connection count
-   Latency

Example:

``` text
Cache Hit Ratio ↓
       ↓
More Database Queries
       ↓
Database Load ↑
       ↓
Application Latency ↑
```

A cache problem can therefore create an indirect database bottleneck.

------------------------------------------------------------------------

# 16. External API Dependencies

Your application may depend on:

-   Payment APIs
-   Authentication services
-   Third-party APIs
-   SaaS platforms
-   Internal microservices

If an external dependency is slow:

``` text
Application
     ↓
External API
     ↓
Response takes 5 seconds
     ↓
Application request takes 5+ seconds
```

Check:

-   Dependency latency
-   Timeout rate
-   Error rate
-   Retry count
-   Connection failures
-   Circuit-breaker activity

Be especially careful with retries.

For example:

``` text
1 request
   ↓
External API fails
   ↓
Retry
   ↓
Retry
   ↓
Retry
```

Retries can multiply traffic and make an existing outage worse.

------------------------------------------------------------------------

# 17. AWS Service Limits / Throttling

Check whether an AWS service is being throttled or approaching a quota.

Potential examples:

-   API rate limits
-   Lambda concurrency
-   DynamoDB capacity
-   API Gateway limits
-   EC2 network limits
-   NAT Gateway limitations
-   Service quotas

The pattern can look like:

``` text
Traffic ↑
   ↓
Service Limit Reached
   ↓
Throttling
   ↓
Retries
   ↓
Latency ↑
```

Always check AWS service metrics and relevant service quotas when
infrastructure looks healthy but requests are still slow.

------------------------------------------------------------------------

# 18. Security Groups / NACLs

Network security configuration can also cause connectivity or timeout
issues.

Check:

### Security Groups

-   Inbound rules
-   Outbound rules
-   Correct ports
-   Correct source/destination

### Network ACLs

-   Inbound rules
-   Outbound rules
-   Ephemeral port requirements
-   Explicit deny rules

A security rule problem generally causes connectivity failures or
timeouts rather than ordinary application latency, but it should be
considered when requests are hanging.

------------------------------------------------------------------------

# 19. Common Reasons for a Slow Application

### Infrastructure

-   High CPU
-   High memory usage
-   Disk I/O saturation
-   Insufficient resources
-   Network congestion

### Application

-   Inefficient code
-   Memory leaks
-   Expensive operations
-   Thread/process exhaustion
-   Connection leaks

### Database

-   Slow queries
-   Missing indexes
-   Lock contention
-   High connection count
-   Database resource saturation

### Network

-   High latency
-   Packet loss
-   Connection problems
-   Incorrect routing

### Traffic

-   Sudden traffic spike
-   Too many concurrent users
-   Abnormal/bot traffic

### AWS

-   Service throttling
-   Quota limits
-   Resource exhaustion

### Dependencies

-   Slow external API
-   Slow internal microservice
-   Dependency timeout
-   Excessive retries

### Configuration

-   Incorrect load-balancer configuration
-   Security group/NACL issues
-   Poor connection-pool configuration
-   Incorrect application/server settings

------------------------------------------------------------------------

# 20. End-to-End Debugging Workflow

Use a structured workflow instead of checking everything randomly.

## Step 1 --- Reproduce the Issue

Determine:

-   When does it happen?
-   Which endpoint is slow?
-   How long does it take?
-   Is it always slow or intermittent?
-   Are all users affected?

------------------------------------------------------------------------

## Step 2 --- Isolate the Layer

Follow:

``` text
User
 ↓
DNS
 ↓
Load Balancer
 ↓
EC2 / Application
 ↓
Database / Cache
 ↓
External Dependencies
```

Find the first layer where latency increases significantly.

------------------------------------------------------------------------

## Step 3 --- Check Metrics

Correlate:

``` text
CPU
Memory
Disk
Network
Connections
Request Count
Response Time
Error Rate
Database Metrics
```

Always compare the metrics with the exact period when users reported
slowness.

------------------------------------------------------------------------

## Step 4 --- Analyze Logs

Check:

-   Application logs
-   Web server logs
-   Load balancer logs
-   Database logs
-   System logs

Search for:

-   Timeout
-   5xx
-   Connection refused
-   Connection reset
-   Slow query
-   Out of memory
-   Throttling

------------------------------------------------------------------------

## Step 5 --- Identify the Bottleneck

Example:

``` text
ALB
 ↓
Target Response Time = 6 sec
 ↓
EC2 CPU = 30%
Memory = Normal
 ↓
Application logs show DB query = 5.5 sec
 ↓
Database is the bottleneck
```

This prevents wasting time tuning EC2 when the real problem is the
database.

------------------------------------------------------------------------

# 21. Root Cause Analysis

Once the bottleneck is identified, determine the **root cause**, not
just the symptom.

Example:

### Symptom

``` text
Application is slow
```

### Observation

``` text
Database queries are taking 5 seconds
```

### Root cause

``` text
Missing index causes full table scan
```

### Fix

``` text
Add appropriate index
Optimize query
```

### Prevention

``` text
Monitoring
Alerts
Query performance monitoring
Load testing
Performance regression testing
```

------------------------------------------------------------------------

# 22. Example Interview Answer

A strong interview response can be:

> "I would troubleshoot a slow application layer by layer rather than
> immediately assuming the application itself is the problem.
>
> First, I would identify whether the issue affects all users, a
> specific endpoint, or a specific time period.
>
> Then I would follow the request path from DNS and the load balancer to
> the EC2/application layer and finally to databases, caches, and
> external dependencies.
>
> At the infrastructure layer, I would check CPU, memory, disk I/O,
> network, connections, and CloudWatch metrics. On the server I would
> use commands such as `top`, `free -h`, `vmstat`, `df -h`, `iostat`,
> and `ss`.
>
> I would also analyze application and web-server logs for errors,
> timeouts, and slow requests.
>
> If infrastructure looks healthy, I would investigate database query
> latency, connection pools, cache hit ratio, and external API latency.
>
> Finally, I would correlate metrics and logs to isolate the bottleneck,
> identify the root cause, apply the fix, and monitor the system to make
> sure the issue does not recur."

------------------------------------------------------------------------

# 23. Quick Troubleshooting Cheat Sheet

  -----------------------------------------------------------------------
  Layer                   What to Check           Useful Commands /
                                                  Metrics
  ----------------------- ----------------------- -----------------------
  Users                   Traffic, affected       Access logs, traffic
                          users, timing           metrics

  DNS                     Resolution, routing,    `dig`, `nslookup`
                          health checks           

  Load Balancer           Target health, latency, CloudWatch ALB metrics
                          errors                  

  CPU                     CPU saturation,         `top`, `htop`, `ps`
                          processes               

  Memory                  RAM, swap, memory-heavy `free -h`, `ps`
                          processes               

  System                  CPU/I/O/memory behavior `vmstat`

  Disk                    Space, inodes           `df -h`, `df -i`

  Disk I/O                Latency, utilization,   `iostat -xz`
                          IOPS                    

  Network                 Connections, ports,     `ss`, `netstat`
                          errors                  

  Logs                    Errors, timeouts, slow  `tail`, log search
                          requests                

  Database                Query latency, locks,   DB metrics/tools
                          connections             

  Cache                   Hit ratio, evictions,   Cache metrics
                          memory                  

  AWS                     Throttling, quotas,     CloudWatch / Service
                          limits                  Quotas

  Security                Rules, connectivity     SG/NACL review

  External APIs           Latency, timeout,       Application/APM metrics
                          retries                 
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 24. SRE Perspective

As an SRE, the goal is not simply:

> **"Fix the slow application."**

The goal is:

``` text
Detect
  ↓
Isolate
  ↓
Diagnose
  ↓
Fix
  ↓
Validate
  ↓
Monitor
  ↓
Prevent Recurrence
```

The most important mindset is:

> **Find the bottleneck first, then fix the root cause.**

For production systems, also consider:

-   SLI/SLO impact
-   Error-budget consumption
-   Alerting
-   Distributed tracing
-   APM
-   Capacity planning
-   Load testing
-   Performance baselines
-   Post-incident review
-   Preventive automation
