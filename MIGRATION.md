# Migracja do Struktury Base + Overlays

## 📌 Przegląd

To repozytorium używa architektury **base + overlays** zgodnie z najlepszymi praktykami Flux CD i Kustomize dla deploymentów multi-cluster.

## 🏗️ Nowa Struktura

```
external-lab-k8s/
│
├── base/                          # Bazowe definicje (multi-cluster)
│   ├── sources/                   # HelmRepositories
│   ├── grafana/                   # Grafana base
│   ├── alloy/                     # Alloy base
│   └── nginx/                     # NGINX base
│
└── clusters/
    └── svm-k8s-lab/              # Cluster-specific overlays
        ├── flux-system/
        ├── sources/              # Overlay: base/sources
        ├── grafana/              # Overlay: base/grafana + patches
        ├── alloy/                # Overlay: base/alloy + patches
        └── nginx/                # Overlay: base/nginx + patches
```

## ✨ Korzyści Nowej Struktury

### 1. Multi-Cluster Ready

```
base/grafana/          ← Wspólne dla wszystkich

clusters/
├── production/grafana/    ← Production overrides
├── staging/grafana/       ← Staging overrides
└── svm-k8s-lab/grafana/   ← Lab overrides
```

### 2. DRY (Don't Repeat Yourself)

**Zamiast:**
```yaml
# infrastructure/monitoring/grafana/values.yaml (pełna config)
# infrastructure/network/grafana/values.yaml (pełna config - duplikat)
```

**Teraz:**
```yaml
# base/grafana/helmrelease.yaml (wspólne wartości)
# clusters/svm-k8s-lab/grafana/patches.yaml (tylko różnice!)
```

### 3. Prostsze Zarządzanie

- **Base** - edytujesz raz, zmiana wszędzie
- **Patches** - tylko cluster-specific overrides
- **Łatwe diff** - widać co jest specyficzne dla klastra

## 🚀 Zastosowanie na Nowym Klastrze

### Opcja 1: Flux już działa (migracja)

Flux automatycznie wykryje nowe pliki `kustomization-flux.yaml` w każdym katalogu aplikacji.

```bash
# Sprawdź co Flux wykrył
flux get kustomizations

# Powinny być:
# - sources
# - grafana
# - alloy
# - nginx
```

### Opcja 2: Bootstrap od zera

```bash
# Bootstrap Flux
flux bootstrap github \
  --owner=YOUR_GITHUB_USER \
  --repository=external-lab-k8s \
  --branch=main \
  --path=clusters/svm-k8s-lab \
  --personal

# Aplikuj Flux Kustomizacje ręcznie (opcjonalnie)
kubectl apply -f clusters/svm-k8s-lab/sources/kustomization-flux.yaml
kubectl apply -f clusters/svm-k8s-lab/grafana/kustomization-flux.yaml
kubectl apply -f clusters/svm-k8s-lab/alloy/kustomization-flux.yaml
kubectl apply -f clusters/svm-k8s-lab/nginx/kustomization-flux.yaml

# Sprawdź status
flux get kustomizations
flux get helmreleases -A
```

## 🔄 Dodawanie Kolejnego Klastra

Przykład: Dodanie klastra "production"

```bash
# 1. Stwórz strukturę
mkdir -p clusters/production/{sources,grafana,alloy,nginx}

# 2. Skopiuj z lab jako template
cp -r clusters/svm-k8s-lab/grafana/* clusters/production/grafana/

# 3. Edytuj patches dla production
vim clusters/production/grafana/patches.yaml
# Zmień wartości na production-specific

# 4. Commit i push
git add clusters/production
git commit -m "feat: add production cluster"
git push

# 5. Bootstrap Flux na production
flux bootstrap github \
  --owner=YOUR_GITHUB_USER \
  --repository=external-lab-k8s \
  --branch=main \
  --path=clusters/production \
  --personal
```

## 📝 Typowe Operacje

### Zmiana wartości dla wszystkich klastrów

```bash
# Edytuj base
vim base/grafana/helmrelease.yaml
# spec.values.persistence.size: 10Gi → 20Gi

git add base/grafana/helmrelease.yaml
git commit -m "chore: increase grafana persistence to 20Gi"
git push

# Zmiana aplikuje się na wszystkich klastrach!
```

### Zmiana wartości tylko dla jednego klastra

```bash
# Edytuj patch
vim clusters/svm-k8s-lab/grafana/patches.yaml
# Dodaj/zmień wartości w spec.values

git add clusters/svm-k8s-lab/grafana/patches.yaml
git commit -m "feat: enable grafana ingress for lab"
git push

# Zmiana tylko dla svm-k8s-lab
```

### Dodanie nowej aplikacji

Sprawdź [QUICKSTART.md](QUICKSTART.md) - sekcja "Dodawanie Nowej Aplikacji"

## 🔍 Weryfikacja Lokalnie

Możesz sprawdzić co Kustomize wyprodukuje przed commit:

```bash
# Zbuduj overlay lokalnie
cd clusters/svm-k8s-lab/grafana
kustomize build .

# To pokaże finalny YAML po merge base + patches
```

## ⚠️ Ważne Uwagi

### Namespace Deduplikacja

Każda aplikacja w `base/` ma swój `namespace.yaml`. Może się wydawać, że duplikujesz:

```
base/grafana/namespace.yaml    → monitoring
base/alloy/namespace.yaml      → monitoring
```

**To jest OK!** Kustomize automatycznie deduplikuje - zostanie stworzony tylko jeden namespace.

### Strategic Merge Patches

Patches używają strategic merge:

```yaml
# base: replicas: 1, cpu: 100m
# patch: cpu: 200m

# Wynik: replicas: 1, cpu: 200m (merge!)
```

Nie musisz powtarzać całego `spec.values` - tylko to co chcesz zmienić!

### HelmRepository Location

Wszystkie HelmRepositories są w namespace `flux-system`:

```yaml
# base/sources/helmrepo-grafana.yaml
metadata:
  namespace: flux-system  # ← Zawsze flux-system
```

## 🎯 Best Practices

### ✅ DO

1. **W base/**: Sensowne defaulty działające na większości klastrów
2. **W patches**: Tylko różnice - nie cały values
3. **Testuj lokalnie**: `kustomize build` przed push
4. **Semantic versioning**: `8.x` nie `latest`
5. **Małe commity**: Jedna zmiana = jeden commit

### ❌ DON'T

1. Nie duplikuj całego `spec.values` w patches
2. Nie hardcode cluster-specific wartości w base
3. Nie używaj inline values + ConfigMap (utrudnia patching)
4. Nie commituj secrets (użyj SOPS/SealedSecrets)

## 🐛 Troubleshooting

### "invalid reference" error

```bash
# Sprawdź ścieżkę w kustomization.yaml
cat clusters/svm-k8s-lab/grafana/kustomization.yaml

# Powinna być:
# resources:
#   - ../../../base/grafana
```

### Patches nie działają

```bash
# Sprawdź czy target się zgadza
# patches.yaml:
# - target:
#     kind: HelmRelease    # ← Musi być dokładnie to
#     name: grafana        # ← Musi być dokładnie ta nazwa
```

### Flux nie widzi zmian

```bash
# Force reconcile
flux reconcile source git flux-system
flux reconcile kustomization grafana --with-source

# Sprawdź logi
flux logs --level=error
```

## 📚 Dodatkowe Zasoby

- [README.md](README.md) - Główna dokumentacja
- [STRUCTURE.md](STRUCTURE.md) - Szczegóły architektury
- [QUICKSTART.md](QUICKSTART.md) - Szybki start
- [Flux Documentation](https://fluxcd.io/docs/)
- [Kustomize Overlays](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#overlay)
