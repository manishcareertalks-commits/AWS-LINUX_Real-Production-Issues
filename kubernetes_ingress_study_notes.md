# Kubernetes Ingress and Ingress Controller — Study Notes


<img width="987" height="897" alt="image" src="https://github.com/user-attachments/assets/2e480f1b-5464-4727-9984-9d2420ff2b05" />


<img width="947" height="863" alt="image" src="https://github.com/user-attachments/assets/8df81eb9-3d51-4aec-af06-91b070cfd82e" />



## 1. What problem does Ingress solve?

Imagine an e-commerce application running on Amazon EKS with three components:

- **Frontend** — serves the website
- **Product API** — handles product requests
- **Order API** — handles order requests

These components run behind Kubernetes Services, which provide stable network endpoints for Pods. External users still need a way to reach the application.

One option is to expose each application with a separate `Service` of type `LoadBalancer`. In AWS, this commonly provisions a separate cloud load balancer for each such Service. This can increase cost and create more infrastructure to manage.

When several applications can share an entry point, **Ingress can route HTTP/HTTPS requests to different Kubernetes Services using host and path rules**.

> Ingress does not automatically mean one load balancer in every architecture. The cost and resource benefit depends on the controller and how it is exposed.

## 2. Ingress vs Ingress Controller

These two terms are related but not the same.

### Ingress

An **Ingress** is a Kubernetes API resource that describes rules for routing HTTP and HTTPS traffic to Services. Rules can match hostnames and URL paths.

An Ingress resource is configuration; by itself, it does not route traffic.

### Ingress Controller

An **Ingress Controller** is the component that watches Ingress resources and implements their rules. Examples include NGINX-based controllers and cloud-specific ingress solutions.

The controller must be installed, running, and configured to handle the Ingress resource's class.

**Remember:** Ingress defines the rules; the Ingress Controller implements them.

## 3. Architecture: before and after

### Before: separate LoadBalancer Services

A possible architecture is:

```text
                         ┌─ Load Balancer A ─ Service ─ Frontend Pods
Users ───────────────────┼─ Load Balancer B ─ Service ─ Product API Pods
                         └─ Load Balancer C ─ Service ─ Order API Pods
```

Potential trade-offs:
- Multiple external entry points
- More load balancers to configure and monitor
- Potentially higher cloud infrastructure cost

### After: shared entry point with Ingress

```text
Users
  |
  v
External Load Balancer
  |
  v
Ingress Controller (for example, NGINX)
  |
  +-- /                  --> frontend-service --> Frontend Pods
  |
  +-- /api/products      --> product-service  --> Product API Pods
  |
  +-- /api/orders        --> order-service    --> Order API Pods
```

The Ingress Controller is commonly exposed using a `Service` of type `LoadBalancer` or another suitable networking setup. With one shared entry point, multiple applications can be routed through it.

## 4. Example routing plan

For the example hostname `shop.example.com`:

| Request | Destination Service |
|---|---|
| `https://shop.example.com/` | `frontend-service` |
| `https://shop.example.com/api/products` | `product-service` |
| `https://shop.example.com/api/orders` | `order-service` |

This example assumes the backend applications can handle the paths they receive. Path rewriting is not configured below.

## 5. Example Ingress manifest

Save as `ingress.yaml`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

          - path: /api/products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 80

          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
```

### Important fields

- `apiVersion: networking.k8s.io/v1` — current stable API version for Ingress.
- `kind: Ingress` — declares an Ingress resource.
- `metadata.name` — name of the resource.
- `spec.ingressClassName: nginx` — selects an IngressClass handled by the configured controller.
- `rules.host` — hostname to match in the incoming request.
- `http.paths` — URL path matching rules.
- `pathType: Prefix` — matches the path by URL path elements, not simply any arbitrary string prefix.
- `backend.service.name` — Kubernetes Service to receive matching traffic.
- `backend.service.port.number` — port on that Service, not necessarily the container's port.

### Notes about the `/` rule

The `/` Prefix rule matches all paths for that host. Kubernetes Ingress path matching prefers the longest matching path, so `/api/products` and `/api/orders` take precedence over `/` when those paths match.

## 6. How traffic flows

1. A user requests `shop.example.com/api/products`.
2. DNS resolves the hostname to the external entry point for the Ingress Controller.
3. The external load balancer forwards traffic to the controller according to its configuration.
4. The Ingress Controller evaluates the hostname and path against the configured Ingress rules.
5. It forwards the request to `product-service` on Service port `80`.
6. The Kubernetes Service routes traffic to an eligible backend Pod.

The exact implementation and network path depend on the controller, cloud integration, and cluster networking.

## 7. Ingress, Service, and Pod — responsibilities

| Component | Responsibility |
|---|---|
| Ingress resource | Declares HTTP/HTTPS host and path routing rules |
| Ingress Controller | Watches and implements Ingress rules |
| Service | Provides a stable endpoint and selects backend Pods |
| Pod | Runs one or more application containers |
| External Load Balancer | Provides an external entry point when used in the architecture |
| DNS | Maps a hostname to the relevant entry point |

**Key point:** Ingress normally routes to a Kubernetes Service, not directly to a Pod.

## 8. What must exist for the manifest to work?

The sample Ingress manifest is not a complete application deployment. Verify these prerequisites:

- An Ingress Controller is installed and healthy.
- An `IngressClass` named `nginx` exists and is handled by the intended controller.
- The three Services exist in the same namespace as this Ingress.
- Each Service exposes port `80` and selects the correct Pods.
- The backend Pods are healthy and ready.
- DNS for `shop.example.com` points to the controller's external entry point, or the hostname is supplied in a local test.
- Network rules permit traffic to the controller and backends as required.

The hostname `shop.example.com` is an example placeholder. It will not work publicly unless you control and configure that domain.

## 9. Useful commands for a demo

Inspect the resources:

```bash
kubectl get ingress
kubectl describe ingress ecommerce-ingress
kubectl get ingressclass
kubectl get svc
kubectl get endpointslices
kubectl get pods
```

Inspect the controller (namespace and Deployment name depend on installation):

```bash
kubectl get pods -A
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

Get the controller Service's external address, if one is used:

```bash
kubectl get svc -n ingress-nginx
```

Test a request with a Host header when you know the controller's reachable IP:

```bash
curl -H "Host: shop.example.com" http://<INGRESS-ADDRESS>/
curl -H "Host: shop.example.com" http://<INGRESS-ADDRESS>/api/products
curl -H "Host: shop.example.com" http://<INGRESS-ADDRESS>/api/orders
```

Replace `<INGRESS-ADDRESS>` with the actual address. These tests assume the controller is reachable over HTTP and that the backends respond on the configured paths.

## 10. Common production troubleshooting

### Ingress returns 404

Check:
- Is the request using the expected hostname?
- Does the Ingress contain a matching host and path rule?
- Is the intended Ingress Controller handling this resource?
- Is the 404 generated by the controller or by the backend application?

### Ingress returns 502 or 503

Check:
- Does the backend Service exist in the same namespace?
- Is the Service port correct?
- Do EndpointSlices contain ready endpoints?
- Can the controller reach the backend Pods?
- Are the application and upstream protocol/port configured correctly?
- What do the controller and application logs show?

A 502 or 503 can have multiple causes; do not assume an empty endpoint list is the cause without checking the evidence. The exact response depends on the controller and configuration.

### TLS / HTTPS problem

Check:
- Does the Ingress TLS configuration reference the correct Secret?
- Is the certificate valid for the requested hostname?
- Has the certificate expired?
- Is the request reaching the expected controller and TLS listener?

The example manifest above does not configure TLS.

### Request reaches the wrong application

Check:
- Host header and hostname rule
- Path and `pathType`
- IngressClass/controller ownership
- Any controller-specific rewrite or annotation settings

## 11. Ingress TLS example (optional)

A TLS configuration commonly looks like this, assuming a TLS Secret named `shop-example-tls` already exists in the same namespace:

```yaml
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - shop.example.com
      secretName: shop-example-tls
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

This is a **partial `spec` example**, not a replacement for the full three-route manifest. TLS termination behavior depends on the selected controller.

## 12. Limitations and modern alternatives

- The Kubernetes Ingress API primarily covers HTTP and HTTPS routing.
- Advanced traffic management features vary by controller and may rely on controller-specific annotations or custom resources.
- Kubernetes **Gateway API** is a newer, more expressive API for many traffic-routing use cases. Whether to use Ingress or Gateway API depends on your platform, controller support, and requirements.
- For AWS, the AWS Load Balancer Controller can provision and manage AWS load balancers for Kubernetes resources. Its behavior and resource model differ from using an NGINX Ingress Controller behind a load balancer. Do not assume that every controller uses the same AWS architecture.

## 13. Interview-ready explanation

> “Ingress is a Kubernetes API resource that defines HTTP and HTTPS routing rules based on hostnames and paths. The Ingress Controller watches those resources and implements the rules by directing requests to the appropriate Kubernetes Services. It can let multiple applications share an external entry point, reducing the need for a separate external load balancer per application when the architecture supports it. The controller must be installed and configured; creating an Ingress object alone does not expose an application.”

## 14. Quick revision

- **Ingress** = routing rules.
- **Ingress Controller** = implements the rules.
- **Service** = stable network endpoint for a set of Pods.
- **Pod** = runs the application containers.
- **Host rule** = routes based on domain name.
- **Path rule** = routes based on URL path.
- **TLS** = can enable HTTPS termination when configured.
- **One shared entry point** = possible architecture, not an automatic guarantee.
- **Ingress resource alone** = not enough; a suitable controller is required.
