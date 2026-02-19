# GitOps Pipeline: GitHub Actions (ARC) + ArgoCD + Harbor

K3s single-node cluster üzerinde WordPress uygulamasını GitOps yaklaşımıyla deploy eden CI/CD pipeline.

## Mimari

```
GitHub Push (helm/**)
       │
       ▼
GitHub Actions (ARC Self-Hosted Runner, K8s DinD)
       │
       ├─ docker pull bitnami/wordpress:latest
       ├─ docker tag & push → Harbor (172.30.67.51:30002)
       └─ values/wordpress_value.yaml tag güncelle & git push
                                                │
                                                ▼
                                    ArgoCD (otomatik sync)
                                                │
                                                ▼
                                    WordPress Deploy (K8s)
```

## Bilesenler

| Bilesen | Namespace | Aciklama |
|---------|-----------|----------|
| K3s | - | Single-node cluster (necmi-desktop) |
| Harbor | `harbor` | Container registry (NodePort 30002, insecure) |
| ArgoCD | `argocd` | GitOps CD tool (v3.3.0) |
| ARC Controller | `arc-systems` | GitHub Actions runner controller |
| ARC Runners | `arc-runners` | Self-hosted runner pods (DinD, autoscale 0-2) |
| WordPress | `wordpress` | Uygulama (Bitnami Helm chart) |

## Dizin Yapisi

```
gitops/
├── .github/workflows/
│   └── deploy-wordpress.yaml   
├── Readme.md
├── argocd/
│   └── wordpress-app.yaml          # ArgoCD Application manifest
├── arc-runner/
│   └── arc-runner-values.yaml
├── harbor/
│   └── values.yaml                  # Harbor helm values
└── helm/
    ├── wordpress/                   # Bitnami WordPress helm chart
    └── values/
        ├── wordpress_value.yaml     # WordPress helm values (image registry)
```

## Erisim Bilgileri

| Servis | URL | Kullanici | Sifre |
|--------|-----|-----------|-------|
| Harbor | http://172.30.67.51:30002 | admin | Harbor12345 |
| ArgoCD | https://localhost:8080 (port-forward) | admin | `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" \| base64 -d` |

## Kurulum Adimlari

### 1. K3s Insecure Registry

```bash
sudo tee /etc/rancher/k3s/registries.yaml > /dev/null << 'EOF'
mirrors:
  "172.30.67.51:30002":
    endpoint:
      - "http://172.30.67.51:30002"

configs:
  "172.30.67.51:30002":
    tls:
      insecure_skip_verify: true
EOF

sudo systemctl restart k3s
```

### 2. ARC (Actions Runner Controller)

```bash
# Controller
helm install arc --namespace arc-systems --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller

# Runner Scale Set (DinD mode)
helm install arc-runner-set --namespace arc-runners --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  -f gitops/arc-runner-values.yaml
```

### 3. ArgoCD Application

```bash
kubectl apply -f gitops/argocd/wordpress-app.yaml
```

### 4. GitHub Repo Secrets

GitHub > cantaskin/Gitops > Settings > Secrets > Actions:

| Secret | Deger |
|--------|-------|
| `HARBOR_USERNAME` | admin |
| `HARBOR_PASSWORD` | Harbor12345 |
| `GH_PAT` | GitHub Personal Access Token |

## Pipeline Akisi

1. `helm/**` altinda bir degisiklik `main` branch'e push edilir
2. GitHub Actions workflow tetiklenir, ARC runner pod'u (DinD) ayaga kalkar
3. Runner `bitnami/wordpress:latest` image'ini pull eder
4. Image `172.30.67.51:30002/library/wordpress:<sha>` olarak tag'lenir ve Harbor'a push edilir
5. `wordpress_value.yaml` icindeki image tag guncellenir ve repo'ya commit/push edilir
6. ArgoCD degisikligi algilar ve otomatik sync ile WordPress'i yeni image ile deploy eder

## Dogrulama Komutlari

```bash
# ARC runner durumu
kubectl get pods -n arc-systems
kubectl get pods -n arc-runners

# ArgoCD uygulama durumu
kubectl get applications -n argocd

# WordPress pod'lari
kubectl get pods -n wordpress

# Harbor'da image kontrolu
curl -s -u admin:Harbor12345 http://172.30.67.51:30002/api/v2.0/projects/library/repositories

```
