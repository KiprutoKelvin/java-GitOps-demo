# GitOps Demo Service (Java / Spring Boot)

An end-to-end GitOps deployment pipeline for a containerized Spring Boot microservice, built to demonstrate DevOps skills: containerization, Kubernetes, CI, and GitOps continuous delivery with ArgoCD.

## Stack

- **Language/Framework:** Java 21, Spring Boot 3 (Web + Actuator)
- **Build tool:** Maven
- **Container:** Multi-stage Docker build (Maven build stage → Alpine JRE runtime, non-root user)
- **Orchestration:** Kubernetes (Deployment, Service, rolling updates, health probes)
- **CI:** GitHub Actions (test → build → push versioned image to Docker Hub)
- **CD:** ArgoCD (auto-sync + self-heal from Git)

## Project layout

```
.
├── src/                      # Spring Boot source
├── Dockerfile                # Multi-stage build
├── .github/workflows/ci.yml  # CI pipeline
├── k8s/                      # Kubernetes manifests (Deployment, Service, Kustomization)
└── argocd/application.yaml   # ArgoCD Application definition
```

## 1. Run locally

```bash
mvn spring-boot:run
curl http://localhost:8080/api/hello
curl http://localhost:8080/actuator/health
```

## 2. Build and run the container

```bash
docker build -t gitops-demo-service:local .
docker run -p 8080:8080 gitops-demo-service:local
```

## 3. Set up CI (GitHub Actions)

1. Push this repo to GitHub.
2. In the repo settings, add these secrets:
   - `DOCKERHUB_USERNAME`
   - `DOCKERHUB_TOKEN` (a Docker Hub access token, not your password)
3. Push to `main` — the workflow tests, builds, and pushes `your-user/gitops-demo-service:latest` and `:<git-sha>` to Docker Hub.

## 4. Spin up a local cluster (Kind or Minikube)

**Kind:**
```bash
kind create cluster --name gitops-demo
```

**Minikube:**
```bash
minikube start
```

## 5. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Port-forward the ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

Get the initial admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Log in at `https://localhost:8081` with username `admin`.

## 6. Point the manifests at your repo

Before applying anything, edit:

- `k8s/deployment.yaml` → replace `<YOUR_DOCKERHUB_USERNAME>` with your actual Docker Hub image path.
- `argocd/application.yaml` → replace `<YOUR_GITHUB_USERNAME>` with your GitHub username/repo.

Commit and push these changes.

## 7. Register the app with ArgoCD

```bash
kubectl apply -f argocd/application.yaml
```

ArgoCD will pull the manifests from `k8s/` in your Git repo, create the `gitops-demo` namespace, and deploy the Deployment + Service. Because `syncPolicy.automated` is enabled, any future push to `k8s/` (e.g. bumping the image tag after a new CI build) is picked up automatically — no `kubectl apply` needed. `selfHeal: true` also means manual `kubectl edit` changes on the cluster get reverted back to match Git.

## 8. Verify

```bash
kubectl get pods -n gitops-demo
kubectl port-forward svc/gitops-demo-service -n gitops-demo 8080:80
curl http://localhost:8080/api/hello
```

## Talking points for a "building in public" write-up

- Multi-stage Docker build keeps the runtime image lean and drops build tooling (Maven, JDK compiler) from the final image.
- Non-root container user reduces the blast radius of a container escape.
- Startup/liveness/readiness probes wired to Spring Boot Actuator give Kubernetes an accurate signal of app health, enabling safe rolling updates.
- CI is decoupled from CD: GitHub Actions only builds and publishes an image; ArgoCD is solely responsible for getting that image running in the cluster, watching Git as the single source of truth.
- `selfHeal` + `prune` demonstrate real GitOps semantics: the cluster state converges to Git state, not the other way around.
