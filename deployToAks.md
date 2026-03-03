Hello World — .NET 8 on AKS with ACR
Complete guide: code → Docker → ACR → AKS → CI/CD
📁 Project Structure
helloworld-app/
├── src/
│   ├── Program.cs                  ← .NET 8 Minimal API
│   └── HelloWorldApp.csproj        ← Project file
├── k8s/
│   ├── namespace.yaml              ← Kubernetes namespace
│   ├── deployment.yaml             ← Pod spec, replicas, probes
│   ├── service.yaml                ← LoadBalancer (public IP)
│   └── hpa.yaml                    ← Auto-scaling policy
├── .github/workflows/
│   └── ci-cd.yml                   ← GitHub Actions pipeline
├── Dockerfile                      ← Multi-stage build
└── .dockerignore
🧠 Part 1: The Application (src/Program.cs)
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddHealthChecks();
var app = builder.Build();

app.MapHealthChecks("/health");       // Used by K8s probes

app.MapGet("/", () => Results.Ok(new {
    message   = "Hello, World!",
    version   = Environment.GetEnvironmentVariable("APP_VERSION"),
    pod       = Environment.GetEnvironmentVariable("POD_NAME"),   // injected by K8s
    timestamp = DateTime.UtcNow
}));
Why Minimal API?
No controllers needed for simple apps
Faster startup, less memory
APP_VERSION and POD_NAME come from environment variables — injected
at different stages (build-time and runtime respectively)
Health Check endpoint /health
Returns HTTP 200 when the app is healthy.
Kubernetes uses this to decide whether to send traffic (readiness)
or restart the pod (liveness).
🐳 Part 2: Dockerfile (Multi-Stage Build)
┌─────────────────────────────────────┐
│  Stage 1: BUILD (sdk:8.0)           │
│  • Restores NuGet packages          │
│  • Compiles + publishes to /app/    │
│  • ~900 MB image (discarded)        │
└──────────────┬──────────────────────┘
               │ COPY --from=build
┌──────────────▼──────────────────────┐
│  Stage 2: RUNTIME (aspnet:8.0)      │
│  • Only published DLLs copied in    │
│  • No SDK, no source code           │
│  • ~220 MB final image              │
│  • Runs as non-root user            │
└─────────────────────────────────────┘
Key decisions:
| Choice | Reason |
|--------|--------|
| aspnet:8.0 runtime (not sdk) | 4× smaller image, smaller attack surface |
| Non-root user appuser | Security best practice |
| ARG APP_VERSION | Version baked in at build time |
| Port 8080 | Avoids needing root for port 80 |
Build locally:
docker build \
  --build-arg APP_VERSION=1.0.0 \
  --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -t helloworld:1.0.0 .

docker run -p 8080:8080 helloworld:1.0.0
# Visit http://localhost:8080
☁️ Part 3: Azure Setup
3.1 — Install tools
# Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# kubectl
az aks install-cli

# Login
az login
az account set --subscription "<YOUR_SUBSCRIPTION_ID>"
3.2 — Create Resource Group
az group create \
  --name rg-helloworld \
  --location eastus
3.3 — Create Azure Container Registry (ACR)
# Create ACR (name must be globally unique, alphanumeric only)
az acr create \
  --resource-group rg-helloworld \
  --name myhelloworldacr \
  --sku Basic \
  --admin-enabled true

# Get the login server URL
az acr show \
  --name myhelloworldacr \
  --query loginServer \
  --output tsv
# Output: myhelloworldacr.azurecr.io
3.4 — Create AKS Cluster
az aks create \
  --resource-group rg-helloworld \
  --name aks-helloworld \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --generate-ssh-keys \
  --attach-acr myhelloworldacr   # ← Grants AKS pull access to ACR

# Connect kubectl to the cluster
az aks get-credentials \
  --resource-group rg-helloworld \
  --name aks-helloworld

# Verify
kubectl get nodes
--attach-acr automatically creates an AKS service principal role
assignment on the ACR so pods can pull images without a pull secret.
🏷️ Part 4: Versioning Strategy
Image Tags in ACR
Every build pushes two tags:
myhelloworldacr.azurecr.io/helloworld:1.0.42   ← IMMUTABLE (never overwrite)
myhelloworldacr.azurecr.io/helloworld:latest    ← FLOATING (always newest)
Tag
Purpose
Overwritten?
1.0.42
Production deploys, rollbacks
❌ Never
latest
Local dev, quick tests
✅ Each build
Version numbering: MAJOR.MINOR.BUILD
MAJOR: Breaking API changes (bump manually)
MINOR: New features, backwards compatible (bump manually)
BUILD: github.run_number — auto-increments on every push
List all versions in ACR
az acr repository show-tags \
  --name myhelloworldacr \
  --repository helloworld \
  --orderby time_desc \
  --output table
Rollback to a previous version
# Update the image tag on the running deployment
kubectl set image deployment/helloworld \
  helloworld=myhelloworldacr.azurecr.io/helloworld:1.0.39 \
  -n helloworld

# Watch the rollout
kubectl rollout status deployment/helloworld -n helloworld
🚢 Part 5: Manual First Deploy
5.1 — Push image to ACR
# Login to ACR
az acr login --name myhelloworldacr

# Tag your local image
docker tag helloworld:1.0.0 myhelloworldacr.azurecr.io/helloworld:1.0.0
docker tag helloworld:1.0.0 myhelloworldacr.azurecr.io/helloworld:latest

# Push both tags
docker push myhelloworldacr.azurecr.io/helloworld:1.0.0
docker push myhelloworldacr.azurecr.io/helloworld:latest
5.2 — Update deployment.yaml
Replace the placeholder in k8s/deployment.yaml:
# Change this line:
image: <ACR_LOGIN_SERVER>/helloworld:latest
# To:
image: myhelloworldacr.azurecr.io/helloworld:1.0.0
5.3 — Apply manifests to AKS
# Create namespace first
kubectl apply -f k8s/namespace.yaml

# Deploy everything
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml

# Watch pods come up
kubectl get pods -n helloworld -w
5.4 — Get the public IP
kubectl get service helloworld-svc -n helloworld
# EXTERNAL-IP column shows your Azure public IP (may take 2-3 min)

# Test it
curl http://<EXTERNAL-IP>/
curl http://<EXTERNAL-IP>/version
curl http://<EXTERNAL-IP>/health
⚙️ Part 6: GitHub Actions CI/CD Pipeline
6.1 — Set up GitHub Secrets
Go to: GitHub repo → Settings → Secrets and variables → Actions
Secret Name
How to get it
AZURE_CREDENTIALS
az ad sp create-for-rbac --name "sp-helloworld" --role contributor --scopes /subscriptions/<SUB_ID>/resourceGroups/rg-helloworld --sdk-auth
ACR_LOGIN_SERVER
myhelloworldacr.azurecr.io
ACR_USERNAME
az acr credential show --name myhelloworldacr --query username -o tsv
ACR_PASSWORD
az acr credential show --name myhelloworldacr --query passwords[0].value -o tsv
AKS_RESOURCE_GROUP
rg-helloworld
AKS_CLUSTER_NAME
aks-helloworld
6.2 — Pipeline flow
git push to main
       │
       ▼
 [Job 1] build-and-test
   dotnet restore → build → test
       │
       ▼
 [Job 2] docker-build-push
   Generate version tag (1.0.<run_number>)
   docker build --build-arg APP_VERSION=1.0.42
   docker push :1.0.42  AND  :latest  → ACR
       │
       ▼
 [Job 3] deploy-to-aks
   az aks get-credentials
   sed replace image tag in deployment.yaml
   kubectl apply → AKS
   kubectl rollout status (waits for success)
6.3 — Trigger a deploy
git add .
git commit -m "feat: update greeting message"
git push origin main
# CI/CD pipeline auto-runs → image built → deployed to AKS
🔍 Part 7: Kubernetes Manifest Explained
deployment.yaml key sections
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # Spin up 1 new pod BEFORE killing old
    maxUnavailable: 0  # Zero downtime — always keep 2 healthy pods
env:
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name  # K8s injects the actual pod name
readinessProbe:          # "Am I ready to receive traffic?"
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

livenessProbe:           # "Am I still alive?"
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
hpa.yaml — Auto-scaling
minReplicas: 2    # Always at least 2 pods running
maxReplicas: 10   # Can scale up to 10 under load
averageUtilization: 70  # Scale when CPU > 70%
🛠️ Part 8: Useful Commands
# See all pods and their status
kubectl get pods -n helloworld

# Stream live logs from all pods
kubectl logs -f -l app=helloworld -n helloworld

# Describe a pod (great for debugging)
kubectl describe pod <pod-name> -n helloworld

# See deployment history
kubectl rollout history deployment/helloworld -n helloworld

# Rollback to previous version
kubectl rollout undo deployment/helloworld -n helloworld

# Scale manually
kubectl scale deployment helloworld --replicas=5 -n helloworld

# See resource usage
kubectl top pods -n helloworld

# Delete everything
kubectl delete namespace helloworld
💡 Summary: Data Flow
Developer laptop
    │  git push
    ▼
GitHub Actions
    │  dotnet build + test
    │  docker build (multi-stage)
    │  docker push :1.0.42 + :latest
    ▼
Azure Container Registry (ACR)
    │  myhelloworldacr.azurecr.io/helloworld:1.0.42
    ▼
Azure Kubernetes Service (AKS)
    │  kubectl apply deployment.yaml (image tag = 1.0.42)
    │  RollingUpdate: new pods start → health check passes → old pods removed
    ▼
Azure LoadBalancer  ←  Public IP  ←  Users