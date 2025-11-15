# Quick Start Guide

## 📦 Struktura Repozytorium

```
base/                      ← Bazowe definicje (multi-cluster)
├── sources/              ← HelmRepositories
├── grafana/              ← Grafana base
├── alloy/                ← Alloy base
└── nginx/                ← NGINX base

clusters/svm-k8s-lab/      ← Cluster-specific overlays
├── flux-system/          ← Flux components
├── sources/              ← Overlay: base/sources
├── grafana/              ← Overlay: base/grafana + patches
├── alloy/                ← Overlay: base/alloy + patches
└── nginx/                ← Overlay: base/nginx + patches
```

## 🚀 Zastosowanie na Klastrze

### 1. Bootstrap Flux (jeśli jeszcze nie zrobione)

```bash
flux bootstrap github \
  --owner=YOUR_GITHUB_USER \
  --repository=external-lab-k8s \
  --branch=main \
  --path=clusters/svm-k8s-lab \
  --personal
```

### 2. Zastosuj Flux Kustomizacje

Flux automatycznie wykryje pliki `kustomization-flux.yaml`, ale możesz je też zastosować ręcznie:

```bash
# Zastosuj wszystkie kustomizacje
kubectl apply -f clusters/svm-k8s-lab/sources/kustomization-flux.yaml
kubectl apply -f clusters/svm-k8s-lab/grafana/kustomization-flux.yaml
kubectl apply -f clusters/svm-k8s-lab/alloy/kustomization-flux.yaml
kubectl apply -f clusters/svm-k8s-lab/nginx/kustomization-flux.yaml
```

### 3. Sprawdź Status

```bash
# Sprawdź Flux Kustomizacje
flux get kustomizations

# Sprawdź HelmReleases
flux get helmreleases -A

# Sprawdź wszystko
watch flux get all
```

## 🔄 Edycja Konfiguracji

### Zmiana wartości dla wszystkich klastrów

Edytuj `base/[app]/helmrelease.yaml`:

```bash
# Przykład: Zwiększ persistence dla Grafana wszędzie
vim base/grafana/helmrelease.yaml
# Zmień spec.values.persistence.size: 10Gi → 20Gi

git add base/grafana/helmrelease.yaml
git commit -m "chore: increase grafana persistence to 20Gi"
git push
```

### Zmiana wartości tylko dla svm-k8s-lab

Edytuj `clusters/svm-k8s-lab/[app]/patches.yaml`:

```bash
# Przykład: Dodaj ingress dla Grafana tylko w lab
vim clusters/svm-k8s-lab/grafana/patches.yaml
# Dodaj ingress config w spec.values

git add clusters/svm-k8s-lab/grafana/patches.yaml
git commit -m "feat: enable grafana ingress for lab cluster"
git push
```

## 🆕 Dodawanie Nowej Aplikacji

### Przykład: Prometheus

```bash
# 1. Dodaj HelmRepository
cat > base/sources/helmrepo-prometheus.yaml <<EOF
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: prometheus-community
  namespace: flux-system
spec:
  interval: 1h
  url: https://prometheus-community.github.io/helm-charts
EOF

# Dodaj do kustomization
vim base/sources/kustomization.yaml  # Dodaj: - helmrepo-prometheus.yaml

# 2. Stwórz base
mkdir -p base/prometheus

cat > base/prometheus/namespace.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
EOF

cat > base/prometheus/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: monitoring
resources:
  - namespace.yaml
  - helmrelease.yaml
EOF

cat > base/prometheus/helmrelease.yaml <<EOF
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: prometheus
  namespace: monitoring
spec:
  interval: 30m
  chart:
    spec:
      chart: kube-prometheus-stack
      version: '61.x'
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
        namespace: flux-system
  values:
    prometheus:
      prometheusSpec:
        retention: 7d
EOF

# 3. Stwórz overlay
mkdir -p clusters/svm-k8s-lab/prometheus

cat > clusters/svm-k8s-lab/prometheus/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../base/prometheus
patches:
  - path: patches.yaml
    target:
      kind: HelmRelease
      name: prometheus
EOF

cat > clusters/svm-k8s-lab/prometheus/patches.yaml <<EOF
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: prometheus
  namespace: monitoring
spec:
  values:
    prometheus:
      prometheusSpec:
        storageSpec:
          volumeClaimTemplate:
            spec:
              storageClassName: "local-path"
              resources:
                requests:
                  storage: 50Gi
EOF

cat > clusters/svm-k8s-lab/prometheus/kustomization-flux.yaml <<EOF
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: prometheus
  namespace: flux-system
spec:
  interval: 10m
  serviceAccountName: kustomize-controller
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./clusters/svm-k8s-lab/prometheus
  prune: true
  wait: false
  dependsOn:
    - name: sources
EOF

# 4. Commit i push
git add base/prometheus base/sources/helmrepo-prometheus.yaml
git add clusters/svm-k8s-lab/prometheus
git commit -m "feat: add prometheus monitoring"
git push

# 5. Aplikuj Flux Kustomizację
kubectl apply -f clusters/svm-k8s-lab/prometheus/kustomization-flux.yaml

# Lub poczekaj ~1 min aż Flux sam wykryje
```

## 🔍 Debugging

```bash
# Logi Flux
flux logs --level=error

# Force reconcile
flux reconcile source git flux-system
flux reconcile kustomization grafana --with-source

# Sprawdź co Kustomize wyprodukuje (lokalnie)
cd clusters/svm-k8s-lab/grafana
kustomize build .

# Sprawdź HelmRelease
kubectl describe helmrelease grafana -n monitoring

# Sprawdź Helm
helm list -n monitoring
```

## 🎯 Typowe Operacje

### Upgrade wersji aplikacji

```bash
# W base/[app]/helmrelease.yaml zmień version
vim base/grafana/helmrelease.yaml
# spec.chart.spec.version: '8.x' → '9.x'

git add base/grafana/helmrelease.yaml
git commit -m "chore: upgrade grafana to v9"
git push
```

### Dodaj nowy namespace/grupę

```bash
# W base/ dodaj namespace w helmrelease.yaml każdej app
# W clusters/[name]/ możesz dodać patches dla namespace jeśli potrzeba
```

### Tymczasowo wyłącz aplikację

```bash
# Opcja 1: Suspend Flux Kustomization
flux suspend kustomization grafana

# Opcja 2: Usuń kustomization-flux.yaml
kubectl delete -f clusters/svm-k8s-lab/grafana/kustomization-flux.yaml

# Przywróć
flux resume kustomization grafana
# lub
kubectl apply -f clusters/svm-k8s-lab/grafana/kustomization-flux.yaml
```

## 📚 Więcej Informacji

- [README.md](README.md) - Pełna dokumentacja
- [STRUCTURE.md](STRUCTURE.md) - Szczegóły struktury
- [MIGRATION.md](MIGRATION.md) - Przewodnik migracji
