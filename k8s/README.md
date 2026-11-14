# Full-Stack Chat App on Kubernetes

Kubernetes manifests for deploying a real-time full-stack chat application. The setup is tested locally on **minikube (Docker driver)** and is intended to move to **GKE on Google Cloud** later.

## Project overview

| Component | Technology | Notes |
|---|---|---|
| Frontend | React (served on port 80) | Image: `sourabhshedge/chatapp-frontend` |
| Backend | Node.js, Express, Socket.IO | Image: `sourabhshedge/chatapp-backend`, port 5001 |
| Database | MongoDB 8.0 | StatefulSet with a 5Gi persistent volume |
| Routing | NGINX Ingress | `/` to frontend, `/api` and `/socket.io` to backend |

Application features: real-time messaging, user authentication with JWT, profile pictures, and online/offline status.

## Architecture

```
Browser
   |
   v
Ingress (nginx)
   |-- /            --> frontend Service (80)  --> frontend Deployment
   |-- /api         --> backend Service  (5001) --> backend Deployment --+
   |-- /socket.io   --> backend Service  (5001) --> backend Deployment --+
                                                                         |
                                            mongodb headless Service (27017)
                                                                         |
                                            mongodb StatefulSet + PVC (5Gi)
```

All resources live in the `chat-app` namespace.

## Repository layout (`k8s/`)

| File | Purpose |
|---|---|
| `namespace.yml` | Creates the `chat-app` namespace |
| `secrets.yml` | Root user, app user, app database name and JWT secret |
| `mongo-init-configmap.yml` | Init script that creates the MongoDB app user |
| `mongodb-depl.yml` | MongoDB StatefulSet, volume claim template and headless Service |
| `backend-depl.yml` | Backend Deployment |
| `backend-service.yml` | Backend ClusterIP Service |
| `frontend-depl.yml` | Frontend Deployment |
| `frontend-service.yml` | Frontend ClusterIP Service |
| `ingress.yml` | Ingress routing rules |

Do not keep `mongodb-service.yml`, `mongodb-pv.yml` or `mongodb-pvc.yml`. The StatefulSet file already provides the Service and the storage.

## Database users

MongoDB has two users:

- **Root user** (`MONGO_INITDB_ROOT_*`): created by the MongoDB image on first start. Used only for administration. The backend does not use it.
- **App user** (`MONGO_APP_*`, default `chatapp_user`): created by `mongo-init-configmap.yml` with `readWrite` on the `chatapp` database only. The backend connects as this user.

The backend connection string is built from the secret:

```
mongodb://$(MONGO_USERNAME):$(MONGO_PASSWORD)@mongodb-statefulset-0.mongodb.chat-app.svc.cluster.local:27017/$(MONGO_DB)?authSource=$(MONGO_DB)
```

> The init script runs only when the MongoDB data directory is empty. See [Troubleshooting](#troubleshooting) if the volume already has data.

## Prerequisites

- Docker Desktop
- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

## Setup

### 1. Start minikube and enable the ingress controller

```powershell
minikube start --driver=docker
minikube addons enable ingress
kubectl get pods -n ingress-nginx        # wait until the controller is Running
```

### 2. Set your secrets

Edit `secrets.yml` and replace every `ChangeMe...` value. Use letters and digits only in passwords, because special characters (`@ : / %`) break the MongoDB connection string unless URL-encoded. Do not commit real secrets to Git.

### 3. Deploy

Apply in this order:

```powershell
kubectl apply -f namespace.yml
kubectl apply -f secrets.yml
kubectl apply -f mongo-init-configmap.yml
kubectl apply -f mongodb-depl.yml
kubectl apply -f backend-depl.yml -f backend-service.yml
kubectl apply -f frontend-depl.yml -f frontend-service.yml
kubectl apply -f ingress.yml
```

### 4. Check the rollout

```powershell
kubectl get pods -n chat-app
kubectl get svc,ingress -n chat-app
kubectl logs mongodb-statefulset-0 -n chat-app | Select-String "created user"
kubectl logs deploy/backend-deployment -n chat-app
```

All pods should show `Running` and `1/1` Ready.

### 5. Access the app on localhost

With the Docker driver on Windows, the minikube IP is not reachable from the host. Use one of these:

**Option A: `minikube tunnel`** (leave the window open; may need admin rights)

```powershell
minikube tunnel
```

Open <http://localhost>.

**Option B: port-forward the ingress controller** (no admin rights needed)

```powershell
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
```

Open <http://localhost:8080>.

## Health probes

- **MongoDB:** `tcpSocket` startup, readiness and liveness probes on port 27017.
- **Backend:** `tcpSocket` probes on port 5001, because the backend has no `/ready` or `/health` route. After you add those routes, switch the probes to `httpGet`.
- **Frontend:** no probes configured yet.

## Troubleshooting

**Backend crashes with an authentication error**
`chatapp_user` does not exist because MongoDB had already initialised its volume. Either:

1. Wipe the data and start fresh (deletes all data):
   ```powershell
   kubectl delete statefulset mongodb-statefulset -n chat-app
   kubectl delete pvc mongodb-pvc-mongodb-statefulset-0 -n chat-app
   kubectl apply -f mongodb-depl.yml
   ```
2. Or create the user manually, using the real values from your secret:
   ```powershell
   kubectl exec -it mongodb-statefulset-0 -n chat-app -- mongosh -u root_user -p '<root-password>' --authenticationDatabase admin --eval 'db.getSiblingDB("chatapp").createUser({user:"chatapp_user",pwd:"<app-password>",roles:[{role:"readWrite",db:"chatapp"}]})'
   ```

**Readiness probe fails with 404**
The backend has no route at the probed path. Use the `tcpSocket` probes in `backend-depl.yml`, or add the route to the backend code.

**Backend pod in CrashLoopBackOff**
```powershell
kubectl logs deploy/backend-deployment -n chat-app --previous
kubectl describe pod -l app=backend -n chat-app
```

**Page loads but login or chat fails**
Check that the frontend calls the backend through a relative `/api` path or the correct host, and that Socket.IO connects to the same origin.

**Ingress has no address**
The ingress controller is not ready yet. Check `kubectl get pods -n ingress-nginx`.

## Clean up

```powershell
kubectl delete namespace chat-app      # removes everything, including the database volume
minikube stop
```

## Roadmap

- [ ] Pin versioned image tags instead of `:latest`
- [ ] Add `/health` and `/ready` routes to the backend and switch to `httpGet` probes
- [ ] Add `securityContext` (`runAsNonRoot`) to the app pods
- [ ] Manage secrets with Secret Manager or Sealed Secrets instead of plain files
- [ ] Deploy to GKE with Artifact Registry
- [ ] Add monitoring and cost optimisation