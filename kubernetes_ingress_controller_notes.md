# Kubernetes Ingress Controller — Production Study Notes

## 1. Production Scenario

An application runs on **Amazon EKS** with three microservices:

| Microservice | Kubernetes Service | Service Port | URL Path |
|---|---|---:|---|
| Frontend | `frontend-service` | 3000 | `/` |
| User API | `user-service` | 8080 | `/users` |
| Payment API | `payment-service` | 9090 | `/payments` |

**Requirement:** Expose all three services publicly using **one domain and one external load balancer** rather than deploying a separate public load balancer for each application.

## Architectural Diagram with Ingress Controller -

<img width="623" height="627" alt="image" src="https://github.com/user-attachments/assets/bc46496d-2e41-451b-bf42-c99a7d0c6b5a" />

##  Architectural Diagram without  Ingress Controller -

<img width="636" height="637" alt="image" src="https://github.com/user-attachments/assets/a2fef68c-d2ea-402f-a818-0611286c3bb1" />

## 2. Solution and Traffic Flow

Install an **NGINX Ingress Controller** in EKS, expose that controller using one AWS load balancer, and define an **Ingress resource** with host- and path-based routing rules.

```text
Internet Users → Route 53 (DNS) → AWS Load Balancer
              → NGINX Ingress Controller (EKS)
                 ├── /         → frontend-service:3000 → Frontend Pods
                 ├── /users    → user-service:8080     → User Pods
                 └── /payments → payment-service:9090  → Payment Pods
```

**Key distinction:** An **Ingress** is a Kubernetes API resource describing HTTP(S) routing rules; an **Ingress Controller** watches these resources and implements their routing. Creating an Ingress alone does not deploy a controller or a load balancer.

## 3. Example Ingress Manifest (`app-ingress.yaml`)

<img width="578" height="633" alt="image" src="https://github.com/user-attachments/assets/604f7282-3eea-4e57-b3fd-b1ee0b81c676" />

   

**Manifest explanation:**

- `apiVersion` / `kind`: Declare a Kubernetes Ingress.
- `metadata`: Name it `app-ingress` in the `production` namespace.
- `ingressClassName: nginx`: Select the installed NGINX Ingress Controller.
- `tls`: Enable HTTPS using the existing `example-tls` Secret in `production`.
- `rules.host`: Match requests to `example.com`.
- `paths` / `pathType: Prefix`: Route by URL path; `/users` and `/payments` are more specific than `/`.
- `backend.service`: Reference existing Kubernetes Services and **Service ports** (not necessarily container ports).

## 4. Advantages in Production

1. **Potential cost reduction:** One public load balancer can serve multiple applications instead of three independently provisioned load balancers. Savings depend on AWS pricing, traffic, and implementation.
2. **Centralized routing:** Manage multiple services through host-based (`api.example.com`) or path-based (`/users`) routing.
3. **Centralized HTTPS:** Terminate TLS at the Ingress Controller with a Kubernetes TLS Secret, or at the external load balancer with a different certificate configuration.
4. **Traffic and security controls:** Depending on controller capabilities, support authentication, rate limiting, redirects, and observability.

## 5. Deploy and Troubleshoot

```bash
kubectl apply -f app-ingress.yaml
kubectl get ingress -n production
kubectl describe ingress app-ingress -n production
kubectl get svc,endpointslices -n production
kubectl get pods -n production
```

Check the controller's own namespace for its Pods and logs, verify DNS resolves to the public load balancer, and ensure each Service has ready backend Pods.

## 6. Production Considerations and Interview Tips

- Install and configure the Ingress Controller **before** applying the Ingress resource; use `ClusterIP` Services for the three internal applications.
- Ensure `example-tls` contains a valid certificate/key for `example.com`. If TLS is terminated at the AWS ALB instead, configure the ALB certificate and listener separately.
- **Path prefixes are not automatically stripped:** Backend APIs must handle `/users` and `/payments`, or you must configure supported rewrites.
- For AWS EKS, an alternative is the **AWS Load Balancer Controller**, which can create an ALB directly from compatible Ingress resources; do not confuse this with the NGINX architecture above.
- An ALB is **not** automatically created for each `LoadBalancer` Service. The actual AWS load balancer type depends on the installed controller and Service configuration; AWS EKS commonly uses NLBs for `LoadBalancer` Services.
- For resilience, run multiple controller replicas across nodes/AZs and configure readiness probes, monitoring, access controls, and sensible timeouts.

**Interview takeaway:** *Ingress specifies HTTP(S) routing rules; the Ingress Controller implements them. Sharing one public entry point across multiple Kubernetes Services can simplify routing, TLS management, and infrastructure cost.*
