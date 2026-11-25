# Make Some Noise — GitOps deployment

Kubernetes manifests used to deploy **Make Some Noise**, a collaborative music application. This repository is the Git source for the application's GitOps deployment and is intended to be synchronized by **Argo CD** on a lightweight Kubernetes cluster such as **K3s** or **K3d**.

## Architecture

The deployment is split into independently deployable services:

| Component | Purpose |
| --- | --- |
| `frontend` | Web application frontend |
| `backend` | Application API and business logic |
| `studio` | Studio service |
| `nginx-media` | Media-serving service |
| `db` | PostgreSQL database |
| `ingress` | Traefik routing for frontend, API and studio paths |

The backend, database and media services use persistent volumes. The repository also contains Service, Deployment, PersistentVolume, PersistentVolumeClaim, ConfigMap and Ingress manifests.

## Repository layout

```text
manifests/
├── audio-server/       # nginx media Deployment and Service
├── backend/            # backend Deployment, Service and media PVC
├── database/           # PostgreSQL Deployment, Service, ConfigMap and PVC
├── frontend/           # frontend Deployment and Service
├── studio/             # studio Deployment and Service
├── db-persistent-volume.yaml
├── media-persistent-volume.yaml
└── ingress.yaml
```

## Prerequisites

- A running Kubernetes cluster (K3s or K3d)
- `kubectl`
- An ingress controller compatible with the Traefik annotations in `manifests/ingress.yaml`
- Argo CD, if deploying through GitOps
- Container images referenced by the manifests available to the cluster

The manifests target the `dev` namespace where specified. Create it before deployment:

```sh
kubectl create namespace dev
```

## Secrets and configuration

The backend expects a Kubernetes Secret named `back-secret` containing:

- `POSTGRES_URL`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `JWT_SECRET`
- `MEDIA_ROOT`

Do not commit real credentials to this repository. Create the secret in the destination cluster before synchronizing the application.

The PostgreSQL deployment reads its initial settings from `manifests/database/db-configmap.yaml`. Review and adapt these values for the target environment.

## Deploy with kubectl

For local development or manual validation:

```sh
kubectl apply --recursive -f manifests/
```

Check the deployed resources:

```sh
kubectl get deployments,services,pods,pvc -n dev
kubectl get ingress -n dev
```

## Deploy with Argo CD

Create an Argo CD application pointing to this repository and the `manifests` directory. Example:

```sh
argocd app create makesomenoise \
  --repo https://github.com/jbayonnegit/makesomenoise-gitops-jbayonne.git \
  --path manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev \
  --sync-policy automated
```

Then synchronize and inspect its status:

```sh
argocd app sync makesomenoise
argocd app get makesomenoise
```

With automated synchronization enabled, changes merged into this repository become the desired cluster state managed by Argo CD.

## Networking

`manifests/ingress.yaml` defines Traefik routes for `makesomenoise.com`:

- `/` → frontend
- `/api/` → backend
- `/studio/` → studio

Adapt the host name, TLS configuration and DNS resolution to the target cluster before exposing the application outside a development environment.

## Notes

This repository stores deployment manifests, not application source code. It is designed to make deployment configuration reviewable, versioned and reproducible.
