# CI/CD Pipeline --- Complete End-to-End Explanation
#Detail Explanation Video - https://www.instagram.com/manishcareertalks/
## Overview

The attached diagram shows a modern **CI/CD pipeline** for a
containerized application deployed to AWS using a **GitOps +
Kubernetes-style deployment workflow**.

The flow can be summarized as:

``` text
Developer
   ↓
GitHub
   ↓
Jenkins
   ↓
Maven Build
   ↓
JUnit Tests
   ↓
SonarQube
   ↓
Docker Build
   ↓
Trivy Security Scan
   ↓
Amazon ECR
   ↓
GitOps Repository
   ↓
Argo CD
   ↓
Deployment
   ↓
Post-Deployment Validation
   ↓
Prometheus
   ↓
Alertmanager
   ↓
Logging / Observability
```

The main purpose is to automate the journey from **source code → tested
application → secure container image → deployment → monitoring**.

------------------------------------------------------------------------

# 1. Application Source Code --- GitHub

### Tool shown

**GitHub**

The developer stores the application's source code in a Git repository.

Typical repository contents could include:

``` text
my-application/
├── src/
├── pom.xml
├── Dockerfile
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
└── README.md
```

A developer might make a change such as:

``` bash
git add .
git commit -m "Add new payment feature"
git push origin main
```

The push to GitHub becomes the trigger for the CI pipeline.

### Why GitHub?

Git provides:

-   Version control
-   Collaboration
-   Branching
-   Pull requests
-   Code review
-   Change history
-   Rollback capability

### Typical trigger

``` text
Developer pushes code
        ↓
GitHub webhook
        ↓
Jenkins pipeline starts
```

------------------------------------------------------------------------

# 2. Jenkins --- Check Out Code

### Tool shown

**Jenkins**

Jenkins acts as the CI/CD automation server.

Once Jenkins receives the GitHub webhook, it starts the pipeline and
checks out the required source-code version.

Conceptually:

``` bash
git clone <repository>
git checkout <commit-or-branch>
```

Jenkins now has the source code available in its workspace.

### Why checkout the exact commit?

The pipeline should build the same code that triggered the pipeline.

This makes the build:

-   Reproducible
-   Traceable
-   Auditable

For example:

``` text
Git commit: a82f91c
       ↓
Jenkins build #152
       ↓
Docker image built from a82f91c
```

------------------------------------------------------------------------

# 3. Maven --- Build the Application

### Tool shown

**Apache Maven**

Maven is commonly used to build Java applications.

The project normally contains:

``` text
pom.xml
```

A basic build command could be:

``` bash
mvn clean package
```

Maven performs tasks such as:

-   Cleaning previous build artifacts
-   Resolving dependencies
-   Compiling source code
-   Running configured build phases
-   Packaging the application

For example, a Java application may produce:

``` text
target/application.jar
```

### Typical flow

``` text
Java source code
       ↓
Maven
       ↓
Compiled classes
       ↓
JAR/WAR artifact
```

------------------------------------------------------------------------

# 4. JUnit --- Unit Testing

### Tool shown

**JUnit**

After building the application, automated unit tests validate individual
components.

Example:

``` java
@Test
void shouldCalculateOrderTotal() {
    assertEquals(100, orderService.calculateTotal(...));
}
```

The pipeline may execute:

``` bash
mvn test
```

If tests pass:

``` text
JUnit → PASS
       ↓
Continue pipeline
```

If tests fail:

``` text
JUnit → FAIL
       ↓
Pipeline stops
       ↓
Developer investigates
```

This is an important CI principle:

> **Do not deploy code that has failed automated tests.**

------------------------------------------------------------------------

# 5. SonarQube --- Static Code Analysis

### Tool shown

**SonarQube**

SonarQube analyzes the source code for potential quality and
maintainability problems.

It can identify things such as:

-   Bugs
-   Code smells
-   Security issues
-   Duplicated code
-   Maintainability problems
-   Test coverage information

A pipeline may execute a Sonar analysis using Maven, for example:

``` bash
mvn sonar:sonar
```

The organization can configure a **Quality Gate**.

Example:

``` text
SonarQube Analysis
        ↓
Quality Gate
    ↙       ↘
 PASS       FAIL
  ↓           ↓
Continue     Stop
```

This prevents poor-quality code from progressing further.

------------------------------------------------------------------------

# 6. Docker --- Build the Container Image

### Tool shown

**Docker**

Once the application passes the required CI checks, Jenkins builds a
Docker image.

A typical Dockerfile could look conceptually like:

``` dockerfile
FROM eclipse-temurin:17-jre

COPY target/application.jar /app/application.jar

ENTRYPOINT ["java", "-jar", "/app/application.jar"]
```

The pipeline could execute:

``` bash
docker build -t myapp:${BUILD_NUMBER} .
```

For example:

``` text
myapp:152
```

The image packages the application and its runtime dependencies into a
deployable unit.

### Why containers?

Containers provide:

-   Consistent runtime environments
-   Portability
-   Easier deployment
-   Isolation
-   Versioned application artifacts

------------------------------------------------------------------------

# 7. Trivy --- Container Security Scan

### Tool shown

**Trivy**

Before pushing the image to the registry, the pipeline scans it for
known vulnerabilities.

Example:

``` bash
trivy image myapp:${BUILD_NUMBER}
```

Trivy can identify vulnerabilities in:

-   OS packages
-   Application dependencies
-   Container images
-   Misconfigurations
-   Secrets, depending on the scan configuration

Conceptually:

``` text
Docker Image
     ↓
   Trivy
     ↓
Vulnerability Scan
     ↓
 ┌───────────────┐
 │               │
PASS            FAIL
 │               │
 ↓               ↓
Continue       Stop
```

This adds a security gate to the CI pipeline.

------------------------------------------------------------------------

# 8. Amazon ECR --- Push Docker Image

### Tool shown

**AWS / Amazon ECR**

After the image passes security checks, it is pushed to **Amazon Elastic
Container Registry (ECR)**.

Conceptually:

``` text
Jenkins
   ↓
Docker Image
   ↓
AWS ECR
```

Example image reference:

``` text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:152
```

The image tag is extremely important.

Avoid relying only on:

``` text
latest
```

A versioned tag such as:

``` text
myapp:152
```

or a Git SHA such as:

``` text
myapp:a82f91c
```

provides better traceability.

------------------------------------------------------------------------

# 9. Update the GitOps Repository

### Tool shown

**Git**

This is a key part of the architecture.

Instead of Jenkins directly changing the production Kubernetes cluster,
the pipeline updates the **GitOps repository**.

For example, a Kubernetes manifest might contain:

``` yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp
  tag: "151"
```

After a successful build, the pipeline updates:

``` yaml
tag: "152"
```

and commits the change:

``` bash
git add .
git commit -m "Update application image to 152"
git push
```

Now Git becomes the desired-state source for deployment.

### Why GitOps?

GitOps provides:

-   Version-controlled deployment configuration
-   Audit history
-   Easy rollback
-   Declarative infrastructure/application state
-   Separation between CI and CD

------------------------------------------------------------------------

# 10. Argo CD --- Deployment

### Tool shown

**Argo CD**

Argo CD is the continuous delivery component shown in the diagram.

Argo CD watches the GitOps repository.

Conceptually:

``` text
GitOps Repository
       ↓
     Argo CD
       ↓
Kubernetes Cluster
```

Suppose Git contains:

``` yaml
image: myapp:152
```

but the cluster is running:

``` text
myapp:151
```

Argo CD detects that the cluster differs from the desired state.

It can synchronize the cluster toward the state defined in Git.

### GitOps model

``` text
        Git
         │
         │ Desired State
         ▼
      Argo CD
         │
         ▼
    Kubernetes
         │
         ▼
   Running App
```

This is different from a Jenkins pipeline directly running deployment
commands against the cluster.

------------------------------------------------------------------------

# 11. Post-Deployment Validation

After deployment, the pipeline/system should verify that the application
is actually healthy.

Deployment success does **not** automatically mean application success.

Useful checks include:

### Kubernetes status

``` bash
kubectl get pods
```

### Check services

``` bash
kubectl get svc
```

### Check deployment

``` bash
kubectl get deployment
```

### Inspect pod details

``` bash
kubectl describe pod <pod-name>
```

### Check application logs

``` bash
kubectl logs <pod-name>
```

### Application health endpoint

For example:

``` text
GET /health
```

Expected:

``` text
HTTP 200 OK
```

A mature deployment pipeline validates:

``` text
Deployment
    ↓
Pod Ready?
    ↓
Application Healthy?
    ↓
Health Endpoint OK?
    ↓
Traffic Working?
```

------------------------------------------------------------------------

# 12. Prometheus --- Monitoring

### Tool shown

**Prometheus**

Prometheus is used to collect and store time-series metrics.

Typical metrics include:

-   CPU utilization
-   Memory utilization
-   Request count
-   Request latency
-   Error rate
-   Pod availability
-   HTTP status codes

For example:

``` text
http_requests_total
http_request_duration_seconds
process_cpu_seconds_total
```

Prometheus periodically scrapes metrics from configured targets.

Conceptually:

``` text
Application
     ↓
 /metrics
     ↓
Prometheus
     ↓
Time-series metrics
```

------------------------------------------------------------------------

# 13. Alertmanager --- Alerts

### Tool shown

**Alertmanager**

Prometheus can evaluate alerting rules.

Example:

``` text
High error rate
      ↓
Prometheus alert
      ↓
Alertmanager
      ↓
Notification
```

Possible notifications include:

-   Email
-   Slack
-   PagerDuty
-   Other incident-management systems

Example alert scenario:

``` text
HTTP 5xx rate > threshold
          ↓
       Alert fires
          ↓
     Alertmanager
          ↓
   On-call engineer
```

Alertmanager also helps with:

-   Grouping alerts
-   Deduplication
-   Routing
-   Silencing
-   Notification management

------------------------------------------------------------------------

# 14. Logging & Observability

The bottom of the screenshot is partially covered by the video overlay,
but it shows a **Logging & Observability** stage and a Grafana-style
visualization.

This stage is important because metrics alone do not tell the complete
story.

A production observability stack commonly combines:

``` text
Metrics
   +
Logs
   +
Traces
   ↓
Observability
```

Examples of tools that may be used include:

-   Grafana
-   Loki
-   Elasticsearch
-   OpenSearch
-   Fluent Bit
-   Fluentd
-   OpenTelemetry
-   Jaeger

The exact tools depend on the organization's architecture.

------------------------------------------------------------------------

# Complete CI/CD Flow

The complete flow shown in the image can be understood as:

``` text
1. Developer writes code
          ↓
2. Push code to GitHub
          ↓
3. GitHub triggers Jenkins
          ↓
4. Jenkins checks out source code
          ↓
5. Maven builds application
          ↓
6. JUnit executes unit tests
          ↓
7. SonarQube performs code analysis
          ↓
8. Docker builds container image
          ↓
9. Trivy scans image for vulnerabilities
          ↓
10. Docker image pushed to Amazon ECR
          ↓
11. GitOps repository updated
          ↓
12. Argo CD detects Git change
          ↓
13. Argo CD synchronizes deployment
          ↓
14. Application deployed
          ↓
15. Post-deployment validation
          ↓
16. Prometheus collects metrics
          ↓
17. Alertmanager handles alerts
          ↓
18. Logging & observability provide visibility
```

------------------------------------------------------------------------

# CI vs CD in This Architecture

## Continuous Integration --- CI

The CI portion primarily validates and packages the application.

``` text
GitHub
  ↓
Jenkins
  ↓
Maven
  ↓
JUnit
  ↓
SonarQube
  ↓
Docker
  ↓
Trivy
```

The goal is:

> **Build fast, test automatically, identify quality/security problems,
> and produce a deployable artifact.**

------------------------------------------------------------------------

## Continuous Delivery / Deployment --- CD

The CD/GitOps portion takes the validated artifact toward the
environment.

``` text
ECR
 ↓
GitOps Repository
 ↓
Argo CD
 ↓
Kubernetes
 ↓
Post-Deployment Validation
 ↓
Monitoring
 ↓
Alerts
```

The goal is:

> **Deploy the desired application version safely and continuously while
> maintaining visibility into the running system.**

------------------------------------------------------------------------

# Why Use GitOps Instead of Direct Jenkins Deployment?

A traditional pipeline might do:

``` text
Jenkins
   ↓
kubectl apply
   ↓
Kubernetes
```

The GitOps approach shown here is:

``` text
Jenkins
   ↓
Update Git
   ↓
GitOps Repository
   ↓
Argo CD
   ↓
Kubernetes
```

The second approach has an important advantage:

**Git becomes the source of truth for the desired deployment state.**

This gives teams:

-   Auditability
-   Version history
-   Easier rollback
-   Better separation of responsibilities
-   Declarative deployments
-   Easier troubleshooting

------------------------------------------------------------------------

# Example End-to-End Scenario

Imagine a developer changes:

``` text
PaymentService.java
```

### Step 1 --- Developer pushes code

``` bash
git push origin main
```

### Step 2 --- Jenkins starts

Jenkins receives the GitHub webhook.

### Step 3 --- Build

``` bash
mvn clean package
```

### Step 4 --- Unit tests

``` bash
mvn test
```

If tests fail, stop.

### Step 5 --- Code quality

SonarQube analyzes the code.

If the Quality Gate fails, stop.

### Step 6 --- Build Docker image

``` bash
docker build -t myapp:a82f91c .
```

### Step 7 --- Security scan

``` bash
trivy image myapp:a82f91c
```

If the configured security gate fails, stop.

### Step 8 --- Push image

``` text
Amazon ECR
└── myapp:a82f91c
```

### Step 9 --- Update GitOps

``` yaml
image:
  tag: a82f91c
```

Commit and push the change.

### Step 10 --- Argo CD detects Git change

Argo CD sees:

``` text
Desired: myapp:a82f91c
Actual:  myapp:old-version
```

It synchronizes the deployment.

### Step 11 --- Validate

Check:

``` bash
kubectl get pods
```

and application health.

### Step 12 --- Monitor

Prometheus observes:

``` text
CPU
Memory
Requests
Latency
Errors
Availability
```

### Step 13 --- Alert

If the error rate becomes too high:

``` text
Prometheus
    ↓
Alertmanager
    ↓
On-call Engineer
```

------------------------------------------------------------------------

# Important Production Improvements

The diagram represents a strong conceptual pipeline, but a production
implementation can add additional controls.

## 1. Artifact Versioning

Prefer immutable image tags:

``` text
myapp:<git-sha>
```

rather than relying on:

``` text
myapp:latest
```

------------------------------------------------------------------------

## 2. Dependency Scanning

Scan application dependencies in addition to container images.

------------------------------------------------------------------------

## 3. Secrets Management

Never store production credentials directly inside Git.

Use an appropriate secrets-management solution.

------------------------------------------------------------------------

## 4. Infrastructure as Code

Infrastructure can also be managed through tools such as:

``` text
Terraform
CloudFormation
Pulumi
```

------------------------------------------------------------------------

## 5. Deployment Strategies

Instead of immediately sending all traffic to a new release,
organizations can use:

-   Rolling deployment
-   Blue/Green deployment
-   Canary deployment

Example:

``` text
Version 1 → 90% traffic
Version 2 → 10% traffic
```

If Version 2 is healthy:

``` text
Version 1 → 0%
Version 2 → 100%
```

------------------------------------------------------------------------

## 6. Automatic Rollback

If post-deployment health checks fail:

``` text
Deployment
    ↓
Health check fails
    ↓
Rollback
    ↓
Previous stable version
```

This is an important reliability capability.

------------------------------------------------------------------------

# Interview Questions Based on This Architecture

## Q1. Why do we use Jenkins if GitHub already stores the code?

**Answer:** GitHub provides source-code management, while Jenkins
provides automation. Jenkins can compile, test, scan, package, and
trigger downstream CI/CD activities.

------------------------------------------------------------------------

## Q2. Why use both SonarQube and Trivy?

**Answer:** They solve different problems.

**SonarQube** focuses primarily on source-code quality and static
analysis.

**Trivy** can scan container images and their components for known
vulnerabilities and other security issues.

------------------------------------------------------------------------

## Q3. Why push the Docker image to ECR?

**Answer:** ECR provides a managed container registry where versioned
Docker images can be securely stored and retrieved by the deployment
environment.

------------------------------------------------------------------------

## Q4. Why does Jenkins update Git instead of directly deploying to Kubernetes?

**Answer:** In a GitOps model, Git is the source of truth for the
desired state. Jenkins updates the GitOps repository, and Argo CD
reconciles that state with the Kubernetes cluster.

------------------------------------------------------------------------

## Q5. What is Argo CD doing?

**Answer:** Argo CD continuously monitors the Git repository and
synchronizes the desired application state from Git to the Kubernetes
environment.

------------------------------------------------------------------------

## Q6. What happens if the unit test fails?

The pipeline should stop before the application progresses to later
stages.

``` text
JUnit FAIL
   ↓
Pipeline STOP
   ↓
Developer fixes code
```

------------------------------------------------------------------------

## Q7. What happens if Trivy finds a critical vulnerability?

Depending on organizational policy, the pipeline can fail the security
gate and prevent the image from being promoted/deployed.

------------------------------------------------------------------------

## Q8. Why is post-deployment validation required?

Because a successful deployment command does not guarantee that the
application is actually functioning correctly.

You need to verify:

``` text
Pods
Services
Readiness
Health endpoints
Application behavior
Error rates
```

------------------------------------------------------------------------

# Final Architecture

``` text
                 ┌──────────────┐
                 │   Developer  │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │   GitHub     │
                 └──────┬───────┘
                        │ Webhook
                        ▼
                 ┌──────────────┐
                 │   Jenkins    │
                 └──────┬───────┘
                        │
            ┌───────────┼───────────┐
            ▼           ▼           ▼
         Maven        JUnit      SonarQube
            │           │           │
            └───────────┼───────────┘
                        ▼
                 ┌──────────────┐
                 │    Docker    │
                 └──────┬───────┘
                        ▼
                 ┌──────────────┐
                 │    Trivy     │
                 └──────┬───────┘
                        ▼
                 ┌──────────────┐
                 │    AWS ECR   │
                 └──────┬───────┘
                        │
                        │ Image version
                        ▼
                 ┌──────────────┐
                 │  GitOps Repo │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │   Argo CD    │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Kubernetes   │
                 │   Cluster    │
                 └──────┬───────┘
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
          Validation Prometheus Logging
                        │
                        ▼
                  Alertmanager
                        │
                        ▼
                   Engineers
```

------------------------------------------------------------------------

# One-Line Interview Explanation

> **"A developer pushes code to GitHub, Jenkins checks it out and runs
> the Maven build, JUnit tests, SonarQube quality checks, Docker build
> and Trivy security scan; the validated image is pushed to ECR, Jenkins
> updates the GitOps repository, Argo CD deploys the desired state to
> Kubernetes, and Prometheus, Alertmanager and logging provide
> post-deployment observability."**

------------------------------------------------------------------------

## Key Takeaway

A mature CI/CD pipeline is not simply:

``` text
Code → Build → Deploy
```

It is:

``` text
Code
 ↓
Build
 ↓
Test
 ↓
Quality
 ↓
Security
 ↓
Package
 ↓
Registry
 ↓
GitOps
 ↓
Deploy
 ↓
Validate
 ↓
Monitor
 ↓
Alert
```

This is what turns a basic deployment process into a **repeatable,
traceable, secure, and observable software delivery pipeline**.
