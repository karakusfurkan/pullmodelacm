# Pull Model ACM + ArgoCD Demo (Hub–Spoke on Minikube)

Bu repo, **Open Cluster Management (OCM)** ile **pull model** bir GitOps demoyu dökümante eder: Hub kümesinde OCM çalışır; spoke (edge) kümesi `klusterlet` ile Hub’a bağlanır. Hub’dan gönderilen **ManifestWork** sayesinde **Argo CD Application** objesi spoke üzerinde oluşur ve uygulama senkronize edilir.

> Bu döküman “başarılı bir kurulum” senaryosunu anlatır ve tekrar üretilebilir adımlar ve script’ler içerir.

---

## Mimari Özeti

- **Hub (minikube profil: `hub`)**
  - `ocm/cluster-manager` Helm chart’ı
  - `clusteradm` ile bootstrap token üretimi
  - `ManifestWork` ile ArgoCD `Application` objesinin *spoke*’a gönderimi

- **Spoke / Edge (minikube profil: `edge1`)**
  - `ocm/klusterlet` Helm chart’ı
  - Bootstrap kubeconfig → CSR’ların approve edilmesi → kalıcı hub kubeconfig’e geçiş
  - ArgoCD (Helm ya da operatör ile) kurulu varsayılır

---

## Hızlı Başlangıç (Özet Adımlar)

> PowerShell kullanıldığı varsayılmıştır (Windows). Linux/MacOS için komutlar benzerdir.

1) **Önkoşullar**
- `kubectl`, `minikube`, `helm`, `clusteradm`, `kustomize` yüklü
- `kubectl config` içinde iki context var:
  - `hub` (API: `https://127.0.0.1:50445` benzeri)
  - `edge1` (API: `https://127.0.0.1:<edge-port>`)

2) **Hub’a OCM Cluster Manager kur**
```powershell
kubectl config use-context hub
helm repo add ocm https://open-cluster-management.io/helm-charts
helm repo update
helm upgrade --install cluster-manager ocm/cluster-manager `
  -n open-cluster-management --create-namespace
```

3) **Bootstrap ServiceAccount oluştur (gerekirse) ve token al**
```powershell
helm upgrade cluster-manager ocm/cluster-manager `
  -n open-cluster-management --reuse-values `
  --set createBootstrapSA=true

# Güvenilir token çıkarma (tek satır):
$s = (clusteradm get token --context hub | Select-String '^token=' | Select-Object -First 1).ToString()
$TOKEN = $s -replace '^token=',''
```

4) **Spoke’a klusterlet kur ve bootstrap secrets (edge1)**
> Eğer Hub API sertifikasında `host.docker.internal` yoksa, spoke tarafında `hub` adını Windows host IP’sine çözdürmek için CoreDNS’e hosts kaydı eklemek gerekebilir (bkz. `scripts/coredns-add-hub.ps1`).

```powershell
kubectl config use-context edge1
# Bootstrap kubeconfig dosyasını oluştur
$HUB_API = "https://hub:50445"   # DNS olarak 'hub' kullanıyoruz
$KUBE = @"
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: $HUB_API
  name: hub
contexts:
- context:
    cluster: hub
    user: bootstrap
  name: bootstrap
current-context: bootstrap
kind: Config
preferences: {}
users:
- name: bootstrap
  user:
    token: "$TOKEN"
"@

$TMP = "$env:TEMP\hub-bootstrap.kubeconfig"
[IO.File]::WriteAllText($TMP, $KUBE)

kubectl create ns open-cluster-management-agent --dry-run=client -o yaml | kubectl apply -f -
kubectl -n open-cluster-management-agent create secret generic bootstrap-hub-kubeconfig `
  --from-file=kubeconfig=$TMP --dry-run=client -o yaml | kubectl apply -f -

helm upgrade --install klusterlet ocm/klusterlet `
  -n open-cluster-management-agent `
  --kube-context edge1 `
  --set clusterName=cluster1 `
  --set hubKubeconfigSecret=bootstrap-hub-kubeconfig

kubectl -n open-cluster-management-agent get pods -w
```

5) **Hub’da gelen CSR’ları onayla ve cluster’ı kabul et**
```powershell
kubectl config use-context hub
kubectl get csr
# Pending olanları approve et:
# kubectl certificate approve <CSR_ADI>

clusteradm accept --clusters cluster1
kubectl get managedclusters
# Beklenen: Joined=True, Available=True
```

6) **ArgoCD Application’ı ManifestWork ile gönder**
```powershell
kubectl config use-context hub
kubectl -n cluster1 apply -f manifests\app-guestbook-work.yaml
# ya da “dokundurmak” için:
kubectl -n cluster1 annotate manifestwork argo-guestbook touch=$(Get-Date -Format o) --overwrite
```

7) **Spoke’ta sonucu gör**
```powershell
kubectl config use-context edge1
kubectl -n argocd get applications
kubectl -n guestbook get pods
```

---

## Repo İçeriği

```
.
├─ manifests/
│  └─ app-guestbook-work.yaml         # Hub’da uygulanacak ManifestWork (ArgoCD Application objesini taşır)
├─ scripts/
│  ├─ get-token.ps1                   # Hub’dan güvenli token çıkarma
│  ├─ edge-bootstrap.ps1              # Spoke’a bootstrap secret yazma (hub:50445 + token)
│  ├─ approve-csrs.ps1                # Hub’da pending CSR’ları otomatik approve
│  ├─ restart-klusterlet.ps1          # Spoke klusterlet agent rollout restart
│  └─ coredns-add-hub.ps1             # Edge CoreDNS’e “hub -> <WindowsHostIP>” hosts kaydı ekler
└─ README.md
```

> Not: ArgoCD kurulumu için Helm tercih ediyorsanız minimal bir örnek:
```powershell
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace
```

---

## ManifestWork İçeriği (Örnek)

`manifests/app-guestbook-work.yaml` içinde, spoke’a taşınacak ArgoCD `Application` şablonu bulunur. ArgoCD CRD’leri spoke’ta kurulu olmalıdır.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: argo-guestbook
  namespace: cluster1
spec:
  workload:
    manifests:
    - apiVersion: argoproj.io/v1alpha1
      kind: Application
      metadata:
        name: guestbook
        namespace: argocd
      spec:
        destination:
          namespace: guestbook
          server: https://kubernetes.default.svc
        project: default
        source:
          repoURL: https://github.com/argoproj/argocd-example-apps
          targetRevision: HEAD
          path: guestbook
        syncPolicy:
          automated:
            prune: true
            selfHeal: true
```

---

## Sık Karşılaşılan Sorunlar

- **`Available=Unknown`** ve Lease güncellenmiyor: Spoke agent kalıcı hub kubeconfig’e geçememiş olabilir. CSR’ları approve ettiğinizden emin olun, `hub-kubeconfig-secret` oluştu mu kontrol edin ve agent’ı restart edin.
- **`x509: certificate ... not host.docker.internal`**: Hub API sertifikasında `host.docker.internal` yok. Spoke DNS’inde `hub` adını Windows host IP’sine çözdürüp server’ı `https://hub:50445` yapın (bkz. `scripts/coredns-add-hub.ps1`).

---

## Lisans

MIT
