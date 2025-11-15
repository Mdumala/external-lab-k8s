# External Lab Kubernetes - Flux CD

Repository zarządzające klastrem Kubernetes lab używając Flux CD z architekturą base + overlays.

## 🏗️ Struktura Repozytorium

```
.
├── base/                             # Bazowe definicje aplikacji (multi-cluster)
│   ├── sources/                      # HelmRepositories
│   │   ├── kustomization.yaml
│   │   ├── helmrepo-grafana.yaml
│   │   └── helmrepo-nginx.yaml
│   │
│   ├── grafana/                      # Bazowa konfiguracja Grafana
│   │   ├── namespace.yaml
│   │   ├── kustomization.yaml
│   │   └── helmrelease.yaml
│   │
│   ├── alloy/                        # Bazowa konfiguracja Alloy
│   │   ├── namespace.yaml
│   │   ├── kustomization.yaml
│   │   └── helmrelease.yaml
│   │
│   └── nginx/                        # Bazowa konfiguracja NGINX
│       ├── namespace.yaml
│       ├── kustomization.yaml
│       ├── rbac.yaml
│       └── helmrelease.yaml
│
└── clusters/
    └── svm-k8s-lab/                  # Konfiguracja specyficzna dla klastra
        ├── flux-system/              # Flux system components
        │   ├── gotk-components.yaml
        │   ├── gotk-sync.yaml
        │   └── kustomization.yaml
        │
        ├── sources/                  # Overlay dla sources
        │   ├── kustomization.yaml         ← references base/sources
        │   └── kustomization-flux.yaml    ← Flux CRD
        │
        ├── grafana/                  # Overlay dla Grafana
        │   ├── kustomization.yaml         ← references base/grafana
        │   ├── patches.yaml               ← cluster-specific values
        │   └── kustomization-flux.yaml    ← Flux CRD
        │
        ├── alloy/                    # Overlay dla Alloy
        │   ├── kustomization.yaml
        │   ├── patches.yaml
        │   └── kustomization-flux.yaml
        │
        └── nginx/                    # Overlay dla NGINX
            ├── kustomization.yaml
            ├── patches.yaml
            └── kustomization-flux.yaml
```

## 📋 Organizacja

### Zasady Architektury Base + Overlays

1. **base/**: Bazowe definicje aplikacji - wspólne dla wszystkich klastrów
   - Namespace + HelmRelease w jednym miejscu
   - Domyślne wartości Helm w `spec.values`
   - Brak cluster-specific konfiguracji

2. **clusters/[cluster-name]/**: Overlays specyficzne dla klastra
   - Referencja do base przez `resources: - ../../../base/[app]`
   - Patches w `patches.yaml` z cluster-specific wartościami
   - `kustomization-flux.yaml` - Flux Kustomization CRD

3. **Namespace flux-system**: Wszystkie Flux komponenty + HelmRepositories

### Struktura Aplikacji

#### W base/[app]/:
- `namespace.yaml` - Definicja namespace
- `kustomization.yaml` - Lista resources
- `helmrelease.yaml` - HelmRelease z domyślnymi wartościami
- Opcjonalnie: `rbac.yaml`, `configmap.yaml`, etc.

#### W clusters/[cluster]/[app]/:
- `kustomization.yaml` - Referencja do base + patches
- `patches.yaml` - Strategic merge patches dla HelmRelease
- `kustomization-flux.yaml` - Flux Kustomization CRD

### Flux Kustomizations

Każda aplikacja ma własną Flux Kustomizację w `kustomization-flux.yaml`:
- `interval: 10m` - Częstotliwość synchronizacji
- `path: ./clusters/[cluster]/[app]` - Ścieżka do overlay
- `dependsOn` - Zależności (np. sources)

## 🚀 Dodawanie Nowej Aplikacji

### Przykład: Dodanie Prometheus

#### Krok 1: Dodaj HelmRepository do base/sources (jeśli potrzebny)

```bash
# base/sources/helmrepo-prometheus.yaml
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

# Dodaj do base/sources/kustomization.yaml
echo "  - helmrepo-prometheus.yaml" >> base/sources/kustomization.yaml
```

#### Krok 2: Stwórz bazową definicję w base/prometheus/

```bash
mkdir -p base/prometheus
```

**base/prometheus/namespace.yaml**:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

**base/prometheus/kustomization.yaml**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: monitoring
resources:
  - namespace.yaml
  - helmrelease.yaml
```

**base/prometheus/helmrelease.yaml**:
```yaml
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
      interval: 12h
  install:
    remediation:
      retries: 3
  upgrade:
    cleanupOnFail: true
    remediation:
      retries: 3
  values:
    # Domyślne wartości - wspólne dla wszystkich klastrów
    prometheus:
      prometheusSpec:
        retention: 7d
        storageSpec:
          volumeClaimTemplate:
            spec:
              resources:
                requests:
                  storage: 50Gi
```

#### Krok 3: Stwórz overlay w clusters/svm-k8s-lab/prometheus/

**clusters/svm-k8s-lab/prometheus/kustomization.yaml**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../base/prometheus

patches:
  - path: patches.yaml
    target:
      kind: HelmRelease
      name: prometheus
```

**clusters/svm-k8s-lab/prometheus/patches.yaml**:
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: prometheus
  namespace: monitoring
spec:
  values:
    # Cluster-specific overrides dla svm-k8s-lab
    prometheus:
      prometheusSpec:
        storageSpec:
          volumeClaimTemplate:
            spec:
              storageClassName: "local-path"
              resources:
                requests:
                  storage: 100Gi  # Więcej dla lab
```

**clusters/svm-k8s-lab/prometheus/kustomization-flux.yaml**:
```yaml
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
```

#### Krok 4: Commit i push

```bash
git add base/prometheus base/sources/helmrepo-prometheus.yaml
git add clusters/svm-k8s-lab/prometheus
git commit -m "feat: add prometheus monitoring stack"
git push
```

## 🔄 Workflow

### Normalna praca

1. Zmiany w **base/** - Edytuj bazowe wartości wspólne dla wszystkich klastrów
2. Zmiany w **clusters/[name]/** - Edytuj patches specyficzne dla klastra
3. Commit i push
4. Flux automatycznie wykrywa zmiany (interval: 10m dla aplikacji, 1m dla flux-system)
5. Aplikuje w kolejności `dependsOn`

### Ręczne synchronizacje

```bash
# Synchronizacja konkretnej aplikacji
flux reconcile kustomization grafana

# Synchronizacja wszystkich aplikacji
flux reconcile kustomization --all

# Synchronizacja źródła Git
flux reconcile source git flux-system

# Sprawdzenie statusu
flux get kustomizations
flux get helmreleases -A

# Debug
flux logs --level=error
```

## 📦 Istniejące Aplikacje

### Monitoring (namespace: monitoring)
- **Grafana**: Wizualizacja i dashboardy
  - Base: [base/grafana/](base/grafana/)
  - Overlay: [clusters/svm-k8s-lab/grafana/](clusters/svm-k8s-lab/grafana/)
- **Alloy**: Collector dla metryk i logów
  - Base: [base/alloy/](base/alloy/)
  - Overlay: [clusters/svm-k8s-lab/alloy/](clusters/svm-k8s-lab/alloy/)

### Network (namespace: network)
- **NGINX Ingress**: Load balancer z Cilium LB-IPAM
  - Base: [base/nginx/](base/nginx/)
  - Overlay: [clusters/svm-k8s-lab/nginx/](clusters/svm-k8s-lab/nginx/)

## 🔧 Konfiguracja

### Dostosowanie wartości

**Dla wszystkich klastrów**: Edytuj `spec.values` w `base/[app]/helmrelease.yaml`

**Dla konkretnego klastra**: Edytuj `spec.values` w `clusters/[name]/[app]/patches.yaml`

Przykład:
```yaml
# clusters/svm-k8s-lab/grafana/patches.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: grafana
spec:
  values:
    persistence:
      size: 20Gi  # Override wartości z base
```

### Secrets

Dla wartości wrażliwych użyj Sealed Secrets lub SOPS:

```yaml
# W base/[app]/helmrelease.yaml
spec:
  values:
    # Publiczne wartości
  valuesFrom:
    - kind: Secret
      name: app-secrets  # SealedSecret lub SOPS
```

## 🎯 Najlepsze Praktyki

1. ✅ **Base dla wspólnych wartości**, overlays dla cluster-specific
2. ✅ Inline values w HelmRelease (nie ConfigMap) - łatwiejsze patches
3. ✅ Używaj wersji semantycznych (`25.x` zamiast `latest`)
4. ✅ Strategic merge patches dla prostych override'ów
5. ✅ Każda aplikacja w osobnej Flux Kustomizacji
6. ✅ Dodaj `dependsOn` dla właściwej kolejności
7. ✅ Namespace w base, nawet jeśli duplikujesz (Kustomize deduplikuje)

## 📚 Dodatkowe Zasoby

- [Flux Documentation](https://fluxcd.io/docs/)
- [Flux Best Practices](https://fluxcd.io/flux/guides/)
- [Kustomize Documentation](https://kubectl.docs.kubernetes.io/references/kustomize/)
