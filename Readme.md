# GitOps Pipeline: GitHub Actions (ARC) + ArgoCD + Harbor

K3s single-node cluster üzerinde WordPress uygulamasını GitOps yaklaşımıyla deploy eden CI/CD pipeline.

---

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
                                    WordPress Deploy (K3s)
```

---

## Bileşenler

| Bileşen        | Namespace     | Açıklama                                      |
| -------------- | ------------- | --------------------------------------------- |
| K3s            | -             | Single-node cluster (necmi-desktop)           |
| Harbor         | `harbor`      | Container registry (NodePort 30002, insecure) |
| ArgoCD         | `argocd`      | GitOps CD tool                                |
| ARC Controller | `arc-systems` | GitHub Actions runner controller              |
| ARC Runners    | `arc-runners` | Self-hosted runner pods (DinD, autoscale 0-2) |
| WordPress      | `wordpress`   | Uygulama (Bitnami Helm chart)                 |

---

## Dizin Yapısı

```
gitops/
├── .github/workflows/
│   └── deploy-wordpress.yaml
├── Readme.md
├── argocd/
│   └── wordpress-app.yaml
├── arc-runner/
│   └── arc-runner-values.yaml
├── harbor/
│   └── values.yaml
└── helm/
    ├── wordpress/
    └── values/
        └── wordpress_value.yaml
```

---

## Kurulum Adımları

---

### 0. Ön Koşullar

#### 0.1 K3s Kurulumu

```bash
curl -sfL https://get.k3s.io | sh -
```

#### 0.2 Kubeconfig Ayarı

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
```

#### 0.3 K3s Insecure Registry Ayarı (Harbor HTTP)

Harbor TLS'siz çalıştığı için K3s'e insecure registry tanımlanmalıdır. Aksi hâlde image pull sırasında **401 Unauthorized** veya TLS hatası alınır.

```bash
sudo bash -c 'cat > /etc/rancher/k3s/registries.yaml << EOF
mirrors:
  "172.30.67.51:30002":
    endpoint:
      - "http://172.30.67.51:30002"

configs:
  "172.30.67.51:30002":
    auth:
      username: admin
      password: Harbor12345
    tls:
      insecure_skip_verify: true
EOF'

sudo systemctl restart k3s
```

> **Not:** Node IP adresi `172.30.67.51` olarak sabitlenmiştir. Farklı bir IP kullanıyorsan hem bu dosyada hem `harbor/values.yaml` içindeki `externalURL` değerini güncelle.

---

### 1. Harbor Kurulumu (Helm)

#### 1.1 Helm Repo Ekle

```bash
helm repo add harbor https://helm.goharbor.io
helm repo update
```

#### 1.2 Harbor Install

```bash
helm install harbor harbor/harbor \
  --namespace harbor \
  --create-namespace \
  -f harbor/values.yaml
```

#### 1.3 Kontrol

```bash
kubectl get pods -n harbor
kubectl get svc -n harbor
```

Erişim:

```
http://172.30.67.51:30002
```

Login:

* Kullanıcı: `admin`
* Şifre: `Harbor12345`

---

### 2. ARC (Actions Runner Controller)

#### 2.1 Controller Install

```bash
helm install arc \
  --namespace arc-systems \
  --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

#### 2.2 arc-runner-values.yaml Hazırlığı

`arc-runner/arc-runner-values.yaml` dosyasını aç ve `github_token` satırını gerçek GitHub PAT ile güncelle:

```yaml
githubConfigSecret:
  github_token: ghp_XXXXXXXXXXXXXXXXXXXX   # <-- buraya gerçek token
```

PAT'ın şu izinlere sahip olması gerekir: `repo`, `workflow`.

#### 2.3 Runner Scale Set Kur

```bash
helm install arc-runner-set \
  --namespace arc-runners \
  --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  -f arc-runner/arc-runner-values.yaml
```

#### 2.4 Kontrol

```bash
# Listener pod arc-systems'da görünmeli
kubectl get pods -n arc-systems

# Runner pod'ları job geldiğinde arc-runners'da ephemeral olarak spawn olur
kubectl get pods -n arc-runners
```

---

### 3. ArgoCD Kurulumu (Helm)

#### 3.1 Helm Repo Ekle

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

#### 3.2 ArgoCD Install

```bash
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace
```

#### 3.3 ArgoCD UI Erişimi

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Tarayıcıdan `https://localhost:8080` adresine git.

İlk giriş için şifreyi al:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o go-template='{{.data.password | base64decode}}'
```

* Kullanıcı: `admin`
* Şifre: yukarıdaki komutun çıktısı

#### 3.4 ArgoCD Application Deploy

```bash
kubectl apply -f argocd/wordpress-app.yaml
```

---

### 4. WordPress Erişimi

ArgoCD ilk sync'i tamamladıktan sonra WordPress'e port-forward ile erişebilirsin:

```bash
kubectl port-forward svc/wordpress -n wordpress 8081:80
```

Tarayıcıdan `http://localhost:8081` adresine git.

> **Not:** Pipeline henüz çalışmadıysa (Harbor'da image yoksa) WordPress pod'u `ImagePullBackOff` hatası verir. `helm/**` altında bir değişiklik push ederek CI'ı tetikle, ArgoCD sync ettikten sonra WordPress ayağa kalkar.

WordPress admin paneli:

```
http://localhost:8081/wp-admin
```

* Kullanıcı: `user`
* Şifre (otomatik üretilir, almak için):

```bash
kubectl get secret wordpress -n wordpress \
  -o go-template='{{.data.wordpressPassword | base64decode}}'
```

---

## GitHub Repo Secrets

GitHub → Repository → Settings → Secrets → Actions

| Secret            | Değer                        |
| ----------------- | ---------------------------- |
| `HARBOR_USERNAME` | admin                        |
| `HARBOR_PASSWORD` | Harbor12345                  |
| `GH_PAT`          | GitHub Personal Access Token |

---

## Pipeline Akışı

1. `helm/**` altında bir değişiklik `main` branch'e push edilir
2. GitHub Actions workflow tetiklenir, ARC runner pod'u (DinD) ayağa kalkar
3. Runner `bitnami/wordpress:latest` image'ini pull eder
4. Image `172.30.67.51:30002/library/wordpress:<sha>` olarak tag'lenir ve Harbor'a push edilir
5. `wordpress_value.yaml` içindeki image tag güncellenir ve repo'ya commit/push edilir
6. ArgoCD değişikliği algılar ve otomatik sync ile WordPress'i yeni image ile deploy eder

### İlk Pipeline Tetikleme

`helm/**` altında herhangi bir değişiklik yap ve `main`'e push et:

```bash
# Örnek: wordpress_value.yaml'a boş bir satır ekle
echo "" >> helm/values/wordpress_value.yaml
git add helm/values/wordpress_value.yaml
git commit -m "ci: trigger initial pipeline"
git push origin main
```

Ya da GitHub UI'dan **Actions → Deploy WordPress → Run workflow** ile manuel tetikle.

---

## Doğrulama Komutları

```bash
# ARC runner durumu
kubectl get pods -n arc-systems
kubectl get pods -n arc-runners

# ArgoCD uygulama durumu
kubectl get applications -n argocd

# WordPress pod'ları
kubectl get pods -n wordpress

# Harbor'da image kontrolü
curl -s -u admin:Harbor12345 http://172.30.67.51:30002/api/v2.0/projects/library/repositories
```

---

## Sorun Giderme

### Image Pull — 401 Unauthorized

K3s'in `registries.yaml` dosyasında Harbor credential'ı eksik demektir. Adım **0.3**'ü uygula ve K3s'i yeniden başlat.


### ArgoCD Sync Hatası — valueFiles bulunamadı

`wordpress-app.yaml` içinde referans verilen bir values dosyası repoda yoksa ArgoCD sync hata verir. Referansı kaldır ya da dosyayı oluştur.

---

## Final Mimari

GitHub → ARC (DinD) → Harbor → Git Update → ArgoCD → K3s

Gerçek bir GitOps CI/CD pipeline implementasyonu.
