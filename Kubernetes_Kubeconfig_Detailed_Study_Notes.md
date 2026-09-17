# Kubernetes Kubeconfig — Detailed Study Notes
<img width="1067" height="849" alt="image" src="https://github.com/user-attachments/assets/a6b2333d-6ca1-4927-a2ea-04a4c99ea5d6" />

## 1. Overview

When working with Kubernetes, administrators and developers commonly use:

```bash
kubectl
```

`kubectl` is the command-line client used to communicate with a Kubernetes cluster through its **Kubernetes API Server**.

A common question is:

> Suppose you have 3 Kubernetes clusters — Cluster A, Cluster B, and Cluster C — and you are using the same machine to manage all three. You run:

```bash
kubectl apply -f deployment.yaml
```

> How does `kubectl` know that the request should go to Cluster A?

The answer is **Kubeconfig**.

A kubeconfig file provides `kubectl` with the information required to:

- Identify Kubernetes clusters
- Authenticate users
- Select a cluster
- Select a user/credential
- Select a namespace
- Determine which configuration should be used by default

---

# 2. What Is Kubeconfig?

A **kubeconfig** is a YAML configuration file used by Kubernetes client tools such as `kubectl` to configure access to one or more Kubernetes clusters.

The default kubeconfig location is:

```bash
~/.kube/config
```

You can also use another kubeconfig file by setting:

```bash
export KUBECONFIG=/path/to/config
```

Or for a single command:

```bash
kubectl --kubeconfig /path/to/config get pods
```

### Important

Kubeconfig does **not** itself run Kubernetes workloads.

It tells the client:

1. **Which API Server to contact**
2. **How to authenticate**
3. **Which cluster/user/namespace combination to use**

---

# 3. Basic kubectl Communication Flow

The simplified flow is:

```text
User
  |
  | kubectl command
  v
kubectl
  |
  | reads kubeconfig
  v
Current Context
  |
  +----> Cluster
  |        |
  |        +----> API Server endpoint
  |
  +----> User
  |        |
  |        +----> Authentication credentials
  |
  +----> Namespace
  |
  v
Kubernetes API Server
  |
  v
Kubernetes Control Plane
  |
  v
Cluster Resources
```

For example:

```bash
kubectl apply -f deployment.yaml
```

`kubectl` does not randomly select a Kubernetes cluster.

It uses its configuration to determine the API Server to which the request should be sent.

---

# 4. The Four Important Kubeconfig Components

A kubeconfig commonly contains four important concepts:

1. **clusters**
2. **users**
3. **contexts**
4. **current-context**

A simplified kubeconfig looks like this:

```yaml
apiVersion: v1
kind: Config

clusters:
  - name: cluster-a
    cluster:
      server: https://cluster-a-api.example.com:6443

users:
  - name: admin-a
    user:
      token: <authentication-token>

contexts:
  - name: cluster-a-context
    context:
      cluster: cluster-a
      user: admin-a
      namespace: production

current-context: cluster-a-context
```

---

# 5. Clusters

The `clusters` section contains information about Kubernetes API Servers.

Example:

```yaml
clusters:
  - name: cluster-a
    cluster:
      server: https://cluster-a-api.example.com:6443
```

Here:

```text
cluster-a
    |
    +---- API Server
            |
            +---- https://cluster-a-api.example.com:6443
```

The important field is:

```yaml
server:
```

It specifies the Kubernetes API Server endpoint.

You can have multiple clusters:

```yaml
clusters:
  - name: cluster-a
    cluster:
      server: https://cluster-a-api.example.com:6443

  - name: cluster-b
    cluster:
      server: https://cluster-b-api.example.com:6443

  - name: cluster-c
    cluster:
      server: https://cluster-c-api.example.com:6443
```

This allows the same `kubectl` installation to be configured for multiple clusters.

---

# 6. Users

The `users` section contains authentication information.

Example using a token:

```yaml
users:
  - name: admin-a
    user:
      token: <token>
```

Authentication can vary depending on the Kubernetes environment.

A kubeconfig may contain or reference:

- Client certificates
- Client keys
- Tokens
- Cloud-provider authentication mechanisms
- Exec-based credential plugins
- Other authentication configuration

Example:

```yaml
users:
  - name: admin-a
    user:
      client-certificate: /path/to/client.crt
      client-key: /path/to/client.key
```

The important idea is:

```text
User configuration
       |
       v
Authentication
       |
       v
API Server
```

Authentication answers:

> "Who are you?"

Authorization is a separate step handled by Kubernetes after authentication.

Authorization answers:

> "What are you allowed to do?"

---

# 7. Contexts

A **context** connects:

```text
Cluster + User + Namespace
```

Example:

```yaml
contexts:
  - name: cluster-a-context
    context:
      cluster: cluster-a
      user: admin-a
      namespace: production
```

This context means:

```text
cluster-a
   +
admin-a
   +
production namespace
```

In simple terms:

> Connect to Cluster A using admin-a and use the production namespace.

A context is especially useful when you manage multiple clusters or namespaces.

---

# 8. Current Context

The kubeconfig can specify a default context:

```yaml
current-context: cluster-a-context
```

This tells `kubectl`:

> Use `cluster-a-context` by default.

The flow becomes:

```text
kubectl
   |
   v
current-context
   |
   v
cluster-a-context
   |
   +---- cluster: cluster-a
   |
   +---- user: admin-a
   |
   +---- namespace: production
   |
   v
Cluster A API Server
```

Therefore:

```bash
kubectl get pods
```

will use the current context unless you explicitly specify another context.

---

# 9. Complete Example: Three Clusters

Suppose one engineer manages:

- Cluster A
- Cluster B
- Cluster C

The kubeconfig could contain:

```yaml
apiVersion: v1
kind: Config

clusters:
  - name: cluster-a
    cluster:
      server: https://cluster-a.example.com:6443

  - name: cluster-b
    cluster:
      server: https://cluster-b.example.com:6443

  - name: cluster-c
    cluster:
      server: https://cluster-c.example.com:6443

users:
  - name: admin-a
    user:
      token: <token-a>

  - name: admin-b
    user:
      token: <token-b>

  - name: admin-c
    user:
      token: <token-c>

contexts:
  - name: cluster-a-context
    context:
      cluster: cluster-a
      user: admin-a
      namespace: production

  - name: cluster-b-context
    context:
      cluster: cluster-b
      user: admin-b
      namespace: production

  - name: cluster-c-context
    context:
      cluster: cluster-c
      user: admin-c
      namespace: production

current-context: cluster-a-context
```

Now:

```bash
kubectl get pods
```

uses:

```text
current-context
      |
      v
cluster-a-context
      |
      +---- cluster-a
      |
      +---- admin-a
      |
      +---- production
      |
      v
Cluster A API Server
```

---

# 10. How kubectl Executes `kubectl apply`

Consider:

```bash
kubectl apply -f deployment.yaml
```

The simplified process is:

### Step 1 — kubectl starts

The `kubectl` client receives the command.

```bash
kubectl apply -f deployment.yaml
```

### Step 2 — kubectl loads configuration

By default, it looks for:

```bash
~/.kube/config
```

It can also use the `KUBECONFIG` environment variable or the `--kubeconfig` option.

### Step 3 — kubectl determines the current context

Example:

```yaml
current-context: cluster-a-context
```

### Step 4 — kubectl resolves the context

The context contains:

```yaml
context:
  cluster: cluster-a
  user: admin-a
  namespace: production
```

### Step 5 — kubectl finds the cluster

It finds:

```yaml
clusters:
  - name: cluster-a
    cluster:
      server: https://cluster-a.example.com:6443
```

### Step 6 — kubectl authenticates

It uses the credentials associated with:

```yaml
user: admin-a
```

### Step 7 — kubectl sends the request

The request goes to:

```text
Cluster A API Server
```

### Step 8 — API Server processes the request

The Kubernetes API Server authenticates and authorizes the request and processes the desired resource operation.

For an apply operation, Kubernetes then handles the resource through the normal control-plane workflow.

---

# 11. Important Interview Point

The interviewer may ask:

> How do you make sure `kubectl apply` goes only to Cluster A?

There are several ways.

## Method 1 — Set Cluster A as Current Context

```bash
kubectl config use-context cluster-a-context
```

Then:

```bash
kubectl apply -f deployment.yaml
```

uses Cluster A.

---

## Method 2 — Specify the Context Explicitly

Instead of relying on the current context:

```bash
kubectl --context=cluster-a-context apply -f deployment.yaml
```

This is often safer for scripts because the target context is explicit.

---

## Method 3 — Specify a Kubeconfig File

```bash
kubectl --kubeconfig=/path/to/cluster-a-config apply -f deployment.yaml
```

This tells `kubectl` which kubeconfig file to use.

---

# 12. Useful Kubeconfig Commands

## View the current configuration

```bash
kubectl config view
```

---

## View the current context

```bash
kubectl config current-context
```

Example:

```text
cluster-a-context
```

---

## List all contexts

```bash
kubectl config get-contexts
```

Example:

```text
CURRENT   NAME                  CLUSTER       AUTHINFO
*         cluster-a-context     cluster-a     admin-a
          cluster-b-context     cluster-b     admin-b
          cluster-c-context     cluster-c     admin-c
```

The `*` indicates the current context.

---

## Switch to Cluster A

```bash
kubectl config use-context cluster-a-context
```

---

## Switch to Cluster B

```bash
kubectl config use-context cluster-b-context
```

---

## Switch to Cluster C

```bash
kubectl config use-context cluster-c-context
```

---

## View configured clusters

```bash
kubectl config get-clusters
```

---

## View configured users

```bash
kubectl config view
```

The user/authentication entries are part of the configuration.

---

# 13. Context vs Cluster

This is a common interview question.

### Cluster

A cluster entry primarily describes:

```text
Where is the Kubernetes API Server?
```

Example:

```yaml
clusters:
  - name: cluster-a
    cluster:
      server: https://cluster-a.example.com:6443
```

### Context

A context defines:

```text
Which cluster?
Which user?
Which namespace?
```

Example:

```yaml
contexts:
  - name: cluster-a-context
    context:
      cluster: cluster-a
      user: admin-a
      namespace: production
```

Therefore:

```text
Cluster = API Server information

User = Authentication information

Context = Cluster + User + Namespace

Current Context = Default context used by kubectl
```

---

# 14. Namespace and Context

A context can include a default namespace.

Example:

```yaml
contexts:
  - name: cluster-a-prod
    context:
      cluster: cluster-a
      user: admin-a
      namespace: production
```

Then:

```bash
kubectl get pods
```

uses:

```text
Cluster:   cluster-a
User:      admin-a
Namespace: production
```

You don't necessarily need to specify:

```bash
-n production
```

every time.

However, you can override the namespace for a particular command:

```bash
kubectl get pods -n staging
```

The explicit command-line namespace takes precedence for that command.

---

# 15. Changing the Current Context

Suppose the current context is:

```text
cluster-a-context
```

Check it:

```bash
kubectl config current-context
```

Output:

```text
cluster-a-context
```

Switch to Cluster B:

```bash
kubectl config use-context cluster-b-context
```

Now:

```bash
kubectl config current-context
```

returns:

```text
cluster-b-context
```

A command such as:

```bash
kubectl get pods
```

will now target Cluster B, assuming the context and credentials are valid.

---

# 16. Explicit Context vs Current Context

### Current context

```bash
kubectl apply -f deployment.yaml
```

The destination depends on the current context.

### Explicit context

```bash
kubectl --context=cluster-a-context apply -f deployment.yaml
```

The destination is explicitly selected.

For automation, explicitly specifying the context can reduce the risk of accidentally operating on the wrong cluster.

---

# 17. KUBECONFIG Environment Variable

You can specify a kubeconfig file with:

```bash
export KUBECONFIG=/home/user/.kube/config
```

Check it:

```bash
echo $KUBECONFIG
```

Then:

```bash
kubectl get pods
```

uses the configured kubeconfig loading behavior.

On systems where multiple kubeconfig files are configured, Kubernetes client configuration can also combine multiple files.

For example:

```bash
export KUBECONFIG=~/.kube/config:~/.kube/config-prod
```

The exact merge behavior depends on the Kubernetes client configuration rules, so care should be taken when managing multiple files.

---

# 18. `--kubeconfig` Option

For a specific command:

```bash
kubectl --kubeconfig=/path/to/config get pods
```

For example:

```bash
kubectl --kubeconfig=/home/user/.kube/cluster-a.yaml \
  apply -f deployment.yaml
```

This is useful when a dedicated kubeconfig is available for a particular environment.

---

# 19. Kubeconfig Security

A kubeconfig can contain sensitive authentication material.

For example:

```yaml
users:
  - name: admin
    user:
      token: <sensitive-token>
```

Or references to client certificates and keys.

Therefore:

- Do not commit sensitive kubeconfig files to public Git repositories.
- Do not share kubeconfig files casually.
- Protect file permissions.
- Rotate credentials when required.
- Follow your organization's credential-management practices.
- Prefer short-lived or identity-based authentication mechanisms where supported.

A kubeconfig file is **not automatically a harmless configuration file**; depending on its contents, it can provide access to Kubernetes resources.

---

# 20. Kubeconfig Does Not Mean "Master Server"

A common terminology issue in interviews is the phrase:

> "One common master server for three clusters."

Modern Kubernetes terminology generally uses **control plane** rather than "master."

More importantly, kubeconfig does not make multiple clusters share one control plane.

The kubeconfig tells the `kubectl` client which **Kubernetes API Server endpoint** to contact.

A simplified architecture is:

```text
                  kubectl
                     |
              reads kubeconfig
                     |
        +------------+------------+
        |            |            |
        v            v            v
    Cluster A    Cluster B    Cluster C
    API Server   API Server   API Server
```

Each cluster can have its own control plane/API Server.

---

# 21. Authentication vs Authorization

These two concepts are frequently confused.

## Authentication

Authentication determines:

> Who is making the request?

Examples include:

- Client certificates
- Bearer tokens
- Cloud IAM-based mechanisms
- OIDC-based authentication
- Exec credential plugins

## Authorization

Authorization determines:

> What is this authenticated identity allowed to do?

Kubernetes commonly uses:

```text
RBAC
```

Role-Based Access Control.

Example:

```text
User: admin-a
       |
       v
Authentication
       |
       v
Authenticated identity
       |
       v
Authorization / RBAC
       |
       v
Allowed or denied
```

---

# 22. Kubeconfig Does Not Bypass RBAC

Having a context pointing to Cluster A does not automatically mean the user can perform every operation.

For example:

```bash
kubectl apply -f deployment.yaml
```

may still fail if the authenticated identity does not have the required permissions.

You may see an error such as:

```text
Error from server (Forbidden)
```

The kubeconfig helps establish:

```text
Where to connect + how to authenticate
```

RBAC and other authorization mechanisms determine:

```text
What the identity can do
```

---

# 23. Common Troubleshooting Commands

## Check current context

```bash
kubectl config current-context
```

---

## List contexts

```bash
kubectl config get-contexts
```

---

## Check the active cluster

```bash
kubectl cluster-info
```

---

## Check the current configuration

```bash
kubectl config view
```

---

## Check a specific context

```bash
kubectl config view --minify
```

This is useful for inspecting the configuration associated with the current context.

---

## Check permissions

```bash
kubectl auth can-i create deployments
```

Check within a namespace:

```bash
kubectl auth can-i create deployments -n production
```

---

# 24. Common Interview Questions

## Q1. What is kubeconfig?

**Answer:**

Kubeconfig is a configuration file used by Kubernetes clients such as `kubectl` to configure access to Kubernetes clusters. It contains cluster information, user authentication information, contexts, and a current context.

---

## Q2. Where is the default kubeconfig file?

```bash
~/.kube/config
```

---

## Q3. What are the main components of kubeconfig?

```text
Clusters
Users
Contexts
Current Context
```

---

## Q4. What is a context?

A context associates:

```text
Cluster + User + Namespace
```

and provides a convenient configuration for a particular Kubernetes environment.

---

## Q5. How do you check the current context?

```bash
kubectl config current-context
```

---

## Q6. How do you list contexts?

```bash
kubectl config get-contexts
```

---

## Q7. How do you switch clusters?

```bash
kubectl config use-context cluster-a-context
```

---

## Q8. How do you explicitly target a cluster?

```bash
kubectl --context=cluster-a-context get pods
```

---

## Q9. How do you use a specific kubeconfig?

```bash
kubectl --kubeconfig=/path/to/config get pods
```

---

## Q10. Does kubeconfig authenticate and authorize the user?

Kubeconfig provides authentication configuration, but **authorization is handled separately** by the Kubernetes API Server and its configured authorization mechanisms such as RBAC.

---

# 25. Interview Scenario

### Question

You have:

```text
Cluster A
Cluster B
Cluster C
```

and a single workstation.

You run:

```bash
kubectl apply -f deployment.yaml
```

How do you make sure the command goes to Cluster A?

### Answer

First, ensure the kubeconfig contains a context that points to Cluster A.

Check the available contexts:

```bash
kubectl config get-contexts
```

Then either make Cluster A the current context:

```bash
kubectl config use-context cluster-a-context
```

and run:

```bash
kubectl apply -f deployment.yaml
```

Or explicitly specify the context:

```bash
kubectl --context=cluster-a-context apply -f deployment.yaml
```

The second approach makes the intended target explicit in the command itself.

---

# 26. Mental Model to Remember

Remember this simple relationship:

```text
KUBECONFIG
    |
    +------------------+
    |                  |
    v                  v
 CLUSTERS            USERS
    |                  |
    |                  |
    +--------+---------+
             |
             v
         CONTEXT
             |
             +---- Cluster
             +---- User
             +---- Namespace
             |
             v
     CURRENT-CONTEXT
             |
             v
          kubectl
             |
             v
      Kubernetes API Server
```

The shortest way to remember it:

```text
Cluster = WHERE

User = WHO

Context = WHERE + WHO + NAMESPACE

Current Context = WHICH CONTEXT BY DEFAULT
```

---

# 27. Quick Revision

| Component | Purpose |
|---|---|
| `clusters` | Defines Kubernetes API Server information |
| `users` | Defines authentication configuration |
| `contexts` | Connects cluster, user and namespace |
| `current-context` | Defines the default context |
| `KUBECONFIG` | Can specify kubeconfig file(s) |
| `--kubeconfig` | Specifies a kubeconfig for a command |
| `--context` | Explicitly selects a context |

---

# 28. Most Important Commands

```bash
# Show current context
kubectl config current-context

# List contexts
kubectl config get-contexts

# Switch context
kubectl config use-context cluster-a-context

# List clusters
kubectl config get-clusters

# View kubeconfig
kubectl config view

# View only the current context configuration
kubectl config view --minify

# Check cluster information
kubectl cluster-info

# Explicitly select a context
kubectl --context=cluster-a-context get pods

# Use a specific kubeconfig
kubectl --kubeconfig=/path/to/config get pods

# Check authorization
kubectl auth can-i create deployments
```

---

# 29. Final Takeaway

When multiple Kubernetes clusters are managed from the same machine, `kubectl` needs configuration that tells it where to connect and how to authenticate.

That configuration is provided through **kubeconfig**.

The key flow is:

```text
kubectl
   ↓
Kubeconfig
   ↓
Current Context
   ↓
Cluster + User + Namespace
   ↓
API Server
   ↓
Kubernetes Cluster
```

For the command:

```bash
kubectl apply -f deployment.yaml
```

if the current context points to Cluster A, `kubectl` uses the Cluster A API Server for that request.

For safer, explicit targeting:

```bash
kubectl --context=cluster-a-context apply -f deployment.yaml
```

This makes the intended Kubernetes context visible directly in the command.
