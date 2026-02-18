# Kong internal API platform (self-managed)

## What is included
- Kong OSS in DB-less mode with JWT auth, rate limiting, IP allow-listing, and custom Lua logic.
- Auth service (FastAPI) with SQLite, auto-initialized database, and JWT issuance.
- Optional DDoS reverse proxy (NGINX with limit_req/limit_conn) in front of Kong.
- Kubernetes manifests for all components.

## Key behaviors
- JWT is enforced only for GET /users. Public endpoints GET /health and GET /verify bypass JWT.
- Rate limiting applies to all routes: 10 requests per minute per IP.
- IP allow-listing applies to all routes (edit the CIDR ranges in k8s/kong/configmap.yaml).
- Custom Lua logic injects headers and logs structured request info in Kong.

## Local testing (Minikube)

### Prerequisites
- Docker
- Minikube
- kubectl

### Setup steps
```bash
# 1. Start Minikube
minikube start

# 2. Point your shell to Minikube's Docker daemon
eval $(minikube docker-env)

# 3. Build the auth-service image inside Minikube's Docker
docker build -t auth-service:latest services/auth-service

# 4. Deploy all manifests
kubectl apply -k k8s

# 5. Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app=auth-service --timeout=120s
kubectl wait --for=condition=ready pod -l app=kong --timeout=120s

# 6. Port-forward Kong proxy to localhost
kubectl port-forward svc/kong 8000:8000
```

Alternative: use the DDoS proxy instead of Kong directly:
```bash
kubectl port-forward svc/ddos-proxy 8080:8080
```

### Cleanup
```bash
kubectl delete -k k8s
minikube stop
# or to delete the cluster entirely: minikube delete
```

## Example API usage
- Login: POST http://localhost:8000/login with JSON {"username":"admin","password":"password"}
- Public health: GET http://localhost:8000/health
- Public verify: GET http://localhost:8000/verify
- Protected users: GET http://localhost:8000/users with Authorization: Bearer <token>

## Configuration notes
- JWT secrets are externalized in k8s/kong/secret.yaml and injected into Kong and the auth service.
- Change CIDR allow-list in k8s/kong/configmap.yaml under the ip-restriction plugin.
- Update seed user credentials in k8s/auth-service/deployment.yaml.


# High-level architecture Flow
Client → NGINX DDoS Proxy (8080) → Kong Gateway (8000) → Auth Service
              ↓                          ↓
         Rate limit              Rate limit + JWT
         Conn limit              IP allow-list
