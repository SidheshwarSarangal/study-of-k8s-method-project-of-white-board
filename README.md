# WhiteboardK8s – Full Guide: From Monolith to Kubernetes

This document is for **beginners to Kubernetes**. It walks through everything we did in this project, step by step: splitting a MERN whiteboard app into microservices, running them on Kubernetes (Kind), wiring Ingress, and fixing the issues we hit. You will see the **code**, **commands**, and **tech** used so you can understand and reproduce the setup.

---

## Please check these links before going through the further study

- [Original project](https://github.com/SidheshwarSarangal/whiteBoard.git)

- [New project, based on microservices way, with kubernetes kind cluster](https://github.com/SidheshwarSarangal/k8sWhiteBoard.git)

---

## 1. Kubernetes concepts (in context)

We use only what this project needs. No deep theory.

| Concept | What it is (here) |
|--------|--------------------|
| **Pod** | The smallest runnable unit: one or more containers. Our app runs **one container per pod** (one Node process or nginx per service). |
| **Deployment** | Manages pods: how many replicas, which image, env, restarts. We have one Deployment per service (auth, rooms, drawings, messages, realtime, users, frontend) and optionally one for MongoDB. |
| **Service** | Gives pods a **stable name and port** inside the cluster (ClusterIP). Other resources (e.g. Ingress) send traffic to the Service; Kubernetes forwards to a healthy pod. |
| **Namespace** | Logical grouping of resources. We use **whiteboard-backend** (all API services + optional MongoDB) and **whiteboard-frontend** (the SPA). |
| **Ingress** | HTTP routing from **outside** the cluster (e.g. browser) to the right Service by path. Example: `/api/auth` → auth-service. Implemented by the **NGINX Ingress Controller**. |
| **Secret** | Holds sensitive data (e.g. `MONGO_URI`, `JWT_SECRET`). Mounted as environment variables in pods. We use one Secret: **auth-secrets**. |
| **Kind** | A Kubernetes cluster that runs **inside Docker** (local dev). Images must be **built on the host** and **loaded into Kind**; the cluster does not pull from a registry by default. |
| **Helm** | A package manager for Kubernetes. Our **chart** (in `backend/chart/`) defines namespaces, Deployments, Services, and Ingress from templates driven by `values.yaml`. |

---

## 2. What we built – architecture

One entrypoint: the browser hits **http://localhost**. The Ingress Controller (NGINX) routes by path to the right Service in the right namespace. Backend services talk to MongoDB (in-cluster or Atlas).

```mermaid
flowchart LR
  subgraph browser [Browser]
    User[User]
  end
  subgraph ingress [NGINX Ingress]
    IngressController[Ingress Controller]
  end
  subgraph frontend_ns [whiteboard-frontend]
    FrontendSvc[frontend Service]
    FrontendPod[frontend Pod]
  end
  subgraph backend_ns [whiteboard-backend]
    AuthSvc[auth-service]
    RoomsSvc[rooms-service]
    DrawSvc[drawings-service]
    MsgSvc[messages-service]
    RealtimeSvc[realtime-service]
    UsersSvc[users-service]
    MongoSvc[mongodb Service]
    MongoPod[MongoDB Pod]
  end
  User -->|"http://localhost /"| IngressController
  User -->|"/api/*, /socket.io, /health"| IngressController
  IngressController -->|"/"| FrontendSvc
  IngressController -->|"/api/auth, /health"| AuthSvc
  IngressController -->|"/api/rooms"| RoomsSvc
  IngressController -->|"/api/drawings"| DrawSvc
  IngressController -->|"/api/messages"| MsgSvc
  IngressController -->|"/socket.io"| RealtimeSvc
  IngressController -->|"/api/users"| UsersSvc
  FrontendSvc --> FrontendPod
  AuthSvc --> MongoSvc
  RoomsSvc --> MongoSvc
  DrawSvc --> MongoSvc
  MsgSvc --> MongoSvc
  UsersSvc --> MongoSvc
  MongoSvc --> MongoPod
```

**Two namespaces, two Ingress resources:** Backend Ingress lives in `whiteboard-backend` and routes `/api/*`, `/socket.io`, `/health`. Frontend Ingress lives in `whiteboard-frontend` and routes `/`. Both use the same host (e.g. localhost), so the browser sees a single origin and CORS is simple.

| Service | Port | Ingress path(s) | Purpose |
|---------|------|------------------|---------|
| auth-service | 3000 | `/health`, `/api/auth` | Signup, login, JWT; health check for backend |
| rooms-service | 3001 | `/api/rooms` | Create/get/join room, list by owner |
| drawings-service | 3002 | `/api/drawings` | Save/get/delete strokes, clear canvas |
| messages-service | 3003 | `/api/messages` | Send/get chat messages |
| realtime-service | 3004 | `/socket.io` | Socket.IO: real-time drawing, room, chat |
| users-service | 3005 | `/api/users` | Collabs, user by username |
| frontend | 80 | `/` | React SPA (nginx) |
| mongodb | 27017 | (internal only) | Database (optional in-cluster) |

---

## 3. Tech stack

| Technology | Where it runs | Purpose |
|------------|----------------|---------|
| Node.js, Express | Backend pods | API servers (auth, rooms, drawings, messages, users) |
| Socket.IO | realtime-service pod | Real-time events (drawing, chat, room join/leave) |
| MongoDB (Mongoose) | In-cluster pod or Atlas | Persistence for users, rooms, drawings, messages |
| React, Vite | Frontend build | SPA; built at image build time |
| nginx | Frontend pod | Serves the built static files on port 80 |
| Docker | Host | Build images; Kind runs containers from these images |
| Kubernetes (Kind) | Docker | Local cluster; runs all Deployments/Services |
| Helm | Host | Install/upgrade the chart (namespaces, Deployments, Services, Ingress) |
| NGINX Ingress Controller | Cluster (ingress-nginx namespace) | Implements Ingress; path-based routing to our Services |
| Redis | (optional) | Socket.IO adapter for multi-replica realtime-service |

---

## 4. Step-by-step journey (what we did and why)

### Step 1 – From monolith to microservices

We split the backend into **six services**, each with its own responsibility:

- **auth-service**: signup, login, JWT. Uses MongoDB (User model).
- **rooms-service**: create room, get room, list by owner, join room, update password. Uses MongoDB (Room model).
- **drawings-service**: save/get/delete strokes, clear canvas. Uses MongoDB (Drawing model).
- **messages-service**: send/get messages. Uses MongoDB (Message model).
- **realtime-service**: Socket.IO only (drawing, room join/leave, chat). **No MongoDB**; optional Redis for multi-replica scaling.
- **users-service**: add room to collabs, get collabs, get user by username. Uses MongoDB (User model).

Each service has: `server.js` (Express, connect to DB, mount routes), `config/database.js` (MongoDB connection), routes, controllers, models, `.env.example`. The frontend stayed one React app but calls these services over HTTP and Socket.IO.

**Code:** See `backend/auth-service/server.js` (express, connectDB, routes, `GET /health`) and `backend/realtime-service/server.js` (Socket.IO, optional Redis adapter).

---

### Step 2 – Frontend API and Socket URL

We removed any hardcoded backend URL. The frontend uses a single config that reads `VITE_API_URL` at **build time** (Vite env). If empty, the app uses the same origin (so when served behind the same Ingress host, `/api/*` and `/socket.io` work without a separate backend URL).

**Code:** `frontend/src/config.js`:

```javascript
// Base URL for API and Socket.IO (same origin when behind Ingress). Set at build time via VITE_API_URL.
const baseUrl = typeof import.meta !== 'undefined' && import.meta.env && import.meta.env.VITE_API_URL
  ? import.meta.env.VITE_API_URL.replace(/\/$/, '')
  : '';

export const API_BASE = baseUrl;
export const SOCKET_URL = baseUrl || window.location.origin;
```

All API calls use `API_BASE`; the Socket.IO client uses `io(SOCKET_URL)`.

---

### Step 3 – Containerizing each service

Every backend service and the frontend have a **Dockerfile**. Backend pattern: Node image, install deps, copy app, expose port, run `node server.js`. Frontend: multi-stage (Node build, then nginx to serve the built files on port 80).

**Code:** `backend/auth-service/Dockerfile`:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

Other backend services follow the same pattern (only port and entrypoint may differ). See `frontend/Dockerfile` for the build + nginx pattern.

---

### Step 4 – Kubernetes manifests and Helm chart

We use **one Helm chart** in `backend/chart/`. It is the single place that defines how the app runs on Kubernetes.

- **values.yaml**: namespaces, secret name, ingress on/off and class, and per-service settings (image, tag, port, replicas). No hardcoded namespace; everything uses `{{ .Values.backendNamespace }}` or `{{ .Values.frontendNamespace }}`.
- **Templates**: namespace YAMLs for backend and frontend; one Ingress for backend paths, one for frontend path `/`; one Deployment + one Service per backend service; one Deployment + one Service for the frontend. Deployments pull env (MONGO_URI, JWT_SECRET) from the Secret `auth-secrets`.

**Code:** Structure of `backend/chart/values.yaml`: `backendNamespace`, `frontendNamespace`, `secretName`, `ingress.enabled`/`className`/`annotations`, then `auth`, `rooms`, `drawings`, `messages`, `realtime`, `users` (each: image, tag, port, replicas), and `frontend` (enabled, image, tag, port, replicas).

Ingress path mapping in `backend/chart/templates/ingress-backend.yaml`: `/health` (Exact) → auth-service; `/api/auth`, `/api/rooms`, `/api/drawings`, `/api/messages`, `/socket.io`, `/api/users` (Prefix) → corresponding service.

---

### Step 5 – Two namespaces and two Ingresses

We keep backend and frontend in **separate namespaces**. Each Ingress only routes to Services in its own namespace. The browser still sees one host (e.g. http://localhost) because both Ingress resources use the same host (no `host` rule = match any host). So the frontend at `/` and all backend paths are reachable at the same origin.

---

### Step 6 – Running the stack (first attempt) and the namespace issue

We ran: create Kind cluster, create namespace, create secret, build images, load into Kind, then `helm install whiteboard ./chart -n whiteboard-backend --create-namespace`.

**Problem:** We had already created `whiteboard-backend` with `kubectl create namespace`. Helm expected to either create the namespace itself (with `--create-namespace`) or manage an existing one that has **Helm ownership metadata**. A namespace created only with `kubectl` does not have that metadata, so Helm refused to install with "invalid ownership metadata".

**Fix (Option B):** Create the namespace first, then **adopt** it by Helm:

1. `kubectl create namespace whiteboard-backend`
2. `kubectl label namespace whiteboard-backend app.kubernetes.io/managed-by=Helm`
3. `kubectl annotate namespace whiteboard-backend meta.helm.sh/release-name=whiteboard meta.helm.sh/release-namespace=whiteboard-backend`
4. Run `helm install whiteboard ./chart -n whiteboard-backend` **without** `--create-namespace`.

Now Helm treats the namespace as part of the release and installs into it.

---

### Step 7 – Pods crashing (CrashLoopBackOff)

After install, pods for auth, rooms, drawings, messages, and users kept restarting; realtime and frontend were fine. Logs showed: **"MongoDB connection failed: ... IP isn't whitelisted"**.

**Cause:** The pods connect to MongoDB Atlas from the cluster’s outbound IP. Atlas **Network Access** was blocking that IP (or we had not added 0.0.0.0/0 in the correct Atlas project).

**Fix (Atlas):** In the Atlas project that owns the cluster (e.g. `cluster0.zeb2axl.mongodb.net`), go to **Network Access** and add **0.0.0.0/0** (Allow access from anywhere). Ensure the cluster is not **Paused**. Then `kubectl rollout restart deployment -n whiteboard-backend`.

**Alternative:** Use in-cluster MongoDB so we do not depend on Atlas at all (Step 10).

---

### Step 8 – 503 on signup and the health check route

The browser got **503 Service Unavailable** when calling `/api/auth/signup`. The Ingress was routing the request to auth-service, but auth-service had **no healthy pods** (they were in CrashLoopBackOff), so the Ingress returned 503.

We added a dedicated **backend health route** so we can check "is the backend up?" without hitting a real API:

- auth-service already had `GET /health` in code (returns 200 and `{ status: 'ok', service: 'auth-service' }`).
- We added a path in the **Ingress**: `/health` (Exact) → auth-service.

Now we can run `curl -s -o /dev/null -w "%{http_code}" http://localhost/health`. **200** means the backend is reachable; **503** means no healthy auth-service (e.g. still crashing).

---

### Step 9 – Accessing the app (Ingress and Kind)

We tried to open the app using the Ingress controller’s **NodePort** (e.g. 31946). `curl http://localhost:31946` returned **000** (connection failed). Reason: **Kind does not expose NodePorts to the host by default**; the node port is only visible inside the Kind node container.

**Fix (Option B):** Recreate the Kind cluster with a **config file** that maps the host’s ports 80 and 443 to the node’s ports 80 and 443, and mark the node as ingress-ready. Then install the NGINX Ingress Controller (Kind-specific manifest). The controller listens on 80/443 on the node; with the port mapping, we can use **http://localhost** (port 80) on the host.

**Code:** `kind-config.yaml` at repo root:

- `extraPortMappings`: containerPort 80 → hostPort 80, containerPort 443 → hostPort 443.
- `node-labels: "ingress-ready=true"` for the control-plane node.

Create cluster with: `kind create cluster --name whiteboard --config kind-config.yaml`. After installing the Ingress controller, the app is at **http://localhost**.

---

### Step 10 – In-cluster MongoDB (avoid Atlas)

To avoid Atlas IP whitelist and connectivity issues in local/dev, we run **MongoDB inside the cluster**. We added a YAML that defines a Deployment (one pod, image `mongo:7`) and a Service named `mongodb` in `whiteboard-backend`. Other backend services connect to it using the in-cluster DNS name: **mongodb** (same namespace). We set the Secret to `MONGO_URI=mongodb://mongodb:27017/whiteboard`, recreated the secret, and restarted the backend deployments. Data in this MongoDB is ephemeral unless you add a PersistentVolumeClaim.

**Code:** `backend/k8s/mongodb.yaml` (excerpt):

```yaml
# Deployment: one pod, mongo:7, env MONGO_INITDB_DATABASE=whiteboard
# Service: name mongodb, port 27017 → targetPort 27017, selector app: mongodb
```

Full file: see `backend/k8s/mongodb.yaml`. Apply with `kubectl apply -f backend/k8s/mongodb.yaml`.

---

### Step 11 – Per-service READMEs

Each service folder (`backend/auth-service`, `backend/rooms-service`, etc.) and `backend/k8s` has a short **README**: what the pod does, what it connects to (DB, other services), port, env vars, and main routes or events. The main README (this file) gives the overall picture and all commands. For details on a single service, see `backend/<service-name>/README.md` and `backend/k8s/README.md`.

---

## 5. Code reference (essential snippets)

- **Frontend config:** Full file at `frontend/src/config.js` (see Step 2 above).
- **Backend Dockerfile:** Full file at `backend/auth-service/Dockerfile` (see Step 3).
- **Helm values:** `backend/chart/values.yaml` – sections: namespaces, secretName, ingress, auth/rooms/drawings/messages/realtime/users (image, tag, port, replicas), frontend.
- **Ingress paths:** `backend/chart/templates/ingress-backend.yaml` – paths `/health`, `/api/auth`, `/api/rooms`, `/api/drawings`, `/api/messages`, `/socket.io`, `/api/users` and their backend services.
- **In-cluster MongoDB:** Full file at `backend/k8s/mongodb.yaml` – Deployment + Service for `mongodb` in `whiteboard-backend`.
- **Secret (in-cluster MongoDB):**

  ```bash
  kubectl create secret generic auth-secrets -n whiteboard-backend \
    --from-literal=MONGO_URI='mongodb://mongodb:27017/whiteboard' \
    --from-literal=JWT_SECRET='YOUR_JWT_SECRET_MIN_32_CHARS'
  ```

  For Atlas, use your Atlas URI in `MONGO_URI` and set Network Access (e.g. 0.0.0.0/0).

---

## 6. Commands – full run (start to end)

Run from the **repo root** unless noted. This assumes **in-cluster MongoDB** (no Atlas).

**1. Create Kind cluster (port 80/443 exposed so Ingress works on localhost)**  
```bash
kind create cluster --name whiteboard --config kind-config.yaml
```

**2. Create backend namespace and let Helm adopt it**  
```bash
kubectl create namespace whiteboard-backend
kubectl label namespace whiteboard-backend app.kubernetes.io/managed-by=Helm
kubectl annotate namespace whiteboard-backend meta.helm.sh/release-name=whiteboard meta.helm.sh/release-namespace=whiteboard-backend
```

**3. (Optional) Deploy in-cluster MongoDB**  
```bash
kubectl apply -f backend/k8s/mongodb.yaml
kubectl wait -n whiteboard-backend --for=condition=ready pod -l app=mongodb --timeout=120s
```

**4. Create the secret (in-cluster MongoDB; replace JWT_SECRET with your value)**  
```bash
kubectl create secret generic auth-secrets -n whiteboard-backend \
  --from-literal=MONGO_URI='mongodb://mongodb:27017/whiteboard' \
  --from-literal=JWT_SECRET='YOUR_JWT_SECRET_MIN_32_CHARS'
```

**5. Build all images and load them into Kind**  
```bash
docker build -t auth-service:latest backend/auth-service
docker build -t rooms-service:latest backend/rooms-service
docker build -t drawings-service:latest backend/drawings-service
docker build -t messages-service:latest backend/messages-service
docker build -t realtime-service:latest backend/realtime-service
docker build -t users-service:latest backend/users-service
docker build -t frontend:latest frontend

kind load docker-image auth-service:latest rooms-service:latest drawings-service:latest messages-service:latest realtime-service:latest users-service:latest frontend:latest --name whiteboard
```

**6. Install the Helm chart**  
```bash
cd backend && helm install whiteboard ./chart -n whiteboard-backend
```

**7. Install NGINX Ingress Controller (Kind)**  
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=120s
```

**8. Verify**  
```bash
kubectl get pods -n whiteboard-backend
kubectl get pods -n whiteboard-frontend
curl -s -o /dev/null -w "%{http_code}" http://localhost/health   # expect 200
```  
Then open **http://localhost** in the browser.

**Using Atlas instead of in-cluster MongoDB:** Skip step 3. In step 4 use your Atlas URI (e.g. `mongodb+srv://user:pass@cluster.mongodb.net/whiteboard`). In Atlas: **Network Access** → add **0.0.0.0/0** (or your IP); ensure the cluster is not paused.

**Teardown**  
```bash
helm uninstall whiteboard -n whiteboard-backend
kubectl delete namespace whiteboard-backend --ignore-not-found
kubectl delete namespace whiteboard-frontend --ignore-not-found
kind delete cluster --name whiteboard
```

---

## 7. Troubleshooting

| Issue | What to do |
|-------|------------|
| **CrashLoopBackOff** | Get logs: `kubectl logs -n whiteboard-backend deployment/auth-service --tail=50` or `--previous`. Often MongoDB: Atlas IP whitelist, wrong MONGO_URI, or cluster paused. Fix Atlas or switch to in-cluster MongoDB and restart deployments. |
| **503 Service Unavailable** | Ingress has no healthy backend. Check `kubectl get pods -n whiteboard-backend` and `kubectl get endpoints -n whiteboard-backend`. If pods are crashing, fix the cause (e.g. DB). If pods are up but a Service has no endpoints, check Service selector vs Pod labels. |
| **Helm "invalid ownership metadata"** | The namespace was created manually without Helm labels. Use Option B: create namespace, add label `app.kubernetes.io/managed-by=Helm` and annotations `meta.helm.sh/release-name`, `meta.helm.sh/release-namespace`, then `helm install` without `--create-namespace`. |
| **curl localhost returns 000 or connection refused** | Kind does not expose NodePort to the host by default. Use `kind-config.yaml` with `extraPortMappings` for 80/443 and install the Ingress controller; then use http://localhost. Or use `kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80` and open http://localhost:8080. |
| **Signup returns 500** | The request reached the backend but the server threw (e.g. DB error). Check auth-service logs for the exact error. |
| **"unable to retrieve container logs for ... --previous"** | The previous container’s logs are gone. Use current logs or stream with `-f` and delete the pod to force a restart so you see the crash in real time. |

---

## 8. Glossary (quick reference)

- **Pod:** Smallest runnable unit; one or more containers. We use one container per pod.
- **Deployment:** Manages pods (replicas, image, env). We have one per service.
- **Service:** Stable network name and port for pods (ClusterIP). Ingress sends traffic to Services.
- **Namespace:** Logical grouping. We use `whiteboard-backend` and `whiteboard-frontend`.
- **Ingress:** HTTP routing from outside the cluster by path. Implemented by the Ingress Controller (NGINX).
- **Secret:** Holds sensitive data; mounted as env in pods. We use `auth-secrets` for MONGO_URI and JWT_SECRET.
- **Kind:** Kubernetes in Docker for local dev. Images must be loaded into the cluster.
- **Helm:** Package manager for K8s; our chart defines all app resources from `values.yaml`.

**Next steps you could add:** Resource requests/limits on Deployments; readiness and liveness probes (e.g. GET /health); TLS for Ingress (e.g. cert-manager); or CI/CD to build images and run `helm upgrade`.

---

## 9. Repo layout

```
whiteboardK8s/
├── README.md                 # This file
├── kind-config.yaml         # Kind cluster: port 80/443, ingress-ready
├── backend/
│   ├── chart/               # Helm chart (values.yaml, templates/)
│   │   ├── values.yaml
│   │   └── templates/       # namespace, ingress, deployments, services
│   ├── k8s/                 # Optional Kustomize base + mongodb.yaml
│   │   ├── README.md
│   │   ├── mongodb.yaml     # In-cluster MongoDB (optional)
│   │   ├── base/
│   │   └── overlays/
│   ├── auth-service/        # Signup, login, JWT, /health
│   ├── rooms-service/
│   ├── drawings-service/
│   ├── messages-service/
│   ├── realtime-service/    # Socket.IO (no DB)
│   └── users-service/
└── frontend/                # React SPA (config.js, Dockerfile, k8s/)
```

For what each service does and how it connects, see `backend/<service>/README.md` and `backend/k8s/README.md`.
