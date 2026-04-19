# Kubernetes documentation

## Technical stack

* Container orchestration: Kubernetes (kubeadm)
* OS: Ubuntu ARM64
* GitOps: FluxCD v2 (flux-operator)
* Package manager: Timoni (module OCI)
* Secrets: Sealed Secrets (Bitnami)
* Database: CloudNativePG
* Load Balancer: MetalLB (L2 mode)
* Ingress Controller: Traefik v3

## Requirements

* ARM64 nodes (all container images are built for `linux/arm64`)
* A DNS entry for `chuck.filhype.ovh`
* cert-manager + ClusterIssuer `lets-encrypt`
* MetalLB deployed in `metallb-system` (géré par Flux)
* Traefik deployed in `ingress` (géré par Flux)
* Sealed Secrets controller in `kube-system`
* CloudNativePG cluster `postgres` in namespace `postgres`
* FluxCD v2 (flux-operator) deployed in namespace `operators`

## Cluster nodes

| Hostname        | Role          | IP            |
|-----------------|---------------|---------------|
| hp              | control-plane | 192.168.1.23  |
| leeson-desktop  | worker        | 192.168.1.26  |
| leeson-jetson2  | worker        | 192.168.1.25  |

---

## Networking stack (MetalLB + Traefik)

### MetalLB

MetalLB fournit un LoadBalancer logiciel en mode L2 pour les clusters bare-metal.

| Paramètre       | Valeur                        |
|-----------------|-------------------------------|
| Namespace       | `metallb-system`              |
| Mode            | L2 Advertisement              |
| IP pool         | `192.168.1.200 – 192.168.1.210` |

Le pool d'IPs doit être en dehors de la plage DHCP du routeur. L'IP assignée à Traefik sera l'adresse publique du cluster sur le réseau local.

### Traefik

Traefik est le contrôleur d'ingress déployé via Helm dans le namespace `ingress`.

| Paramètre         | Valeur              |
|-------------------|---------------------|
| Namespace         | `ingress`           |
| IngressClass      | `traefik` (défaut)  |
| Entrypoints       | `web` (80), `websecure` (443) |
| Service type      | `LoadBalancer` (IP assignée par MetalLB) |

### Réécriture de chemin

Le backend Quarkus sert ses endpoints à la racine (`/`). L'ingress expose le service sous `/api`. La réécriture est effectuée par un **Middleware Traefik** `strip-api-prefix` dans le namespace `chuck-norris` :

```
/api        → /
/api/jokes  → /jokes
```

Annotation ingress :
```yaml
traefik.ingress.kubernetes.io/router.middlewares: chuck-norris-strip-api-prefix@kubernetescrd
```

---

## GitOps flow (FluxCD + Timoni)

### Architecture

```
GitHub                         Flux (operators ns)              Cluster
──────────────────────         ──────────────────────────       ──────────────────────
iac-chucknorris-backend   →    GitRepository
  infra/metallb/          →      Kustomization infra-metallb     → metallb-system ns
  infra/metallb-config/   →      Kustomization infra-metallb-config
  infra/traefik/          →      Kustomization infra-traefik     → ingress ns
  deploy/                 →      Kustomization chuck-norris-backend → chuck-norris ns
```

### Ordre de déploiement (dependsOn)

```
infra-metallb
    └→ infra-metallb-config   (attend que MetalLB soit prêt + CRDs installés)
           └→ infra-traefik   (attend que le pool IP soit configuré)
                  └→ chuck-norris-backend
```

### Continuous Deployment pipeline

```
1. Dev push → super-chuck-norris-backend (main)
       ↓
2. GitHub Actions: build native multi-arch image (amd64 + arm64)
   → push docker.io/leeson77/chuck-norris-backend:<sha>
       ↓
3. GitHub Actions job `update-iac`:
   → checkout filhype-organization/iac-chucknorris-backend
   → sed deploy/deployment.yaml: update image tag
   → git commit "chore: deploy backend image <sha> [skip ci]"
   → git push
       ↓
4. Flux GitRepository: détecte le nouveau commit (interval: 1m)
       ↓
5. Flux Kustomization: applique deploy/ sur le cluster (interval: 5m)
       ↓
6. K8s rolling-update: nouveau pod arm64 en ~20ms (native Quarkus)
```

### Modification des templates Timoni

Quand les fichiers CUE changent (`templates/`, `timoni.cue`, `values.cue`) :

```
1. Dev push → iac-chucknorris-backend (main)
       ↓
2. GitHub Actions job `render-manifests`:
   → timoni build → réécrit deploy/{configmap,deployment,service,ingress,middleware}.yaml
   → préserve le tag d'image courant dans deployment.yaml
   → git commit "chore: re-render manifests from timoni module [skip ci]"
       ↓
3-6. Même flux Flux que ci-dessus
```

---

## Premier déploiement (bootstrap)

### Prérequis secrets GitHub

Ajouter le secret `IAC_REPO_TOKEN` dans le repo `super-chuck-norris-backend` :

```bash
gh secret set IAC_REPO_TOKEN --repo filhype-organization/super-chuck-norris-backend
# Valeur : GitHub fine-grained PAT avec Contents: Read & Write sur iac-chucknorris-backend
```

### Appliquer les ressources Flux

Les ressources FluxCD sont dans `flux/` du repo `iac-chucknorris-backend`.
À appliquer **dans l'ordre** la première fois :

```bash
# Source Git
kubectl apply -f flux/gitrepository.yaml

# Infrastructure (ordre important)
kubectl apply -f flux/infra-metallb.yaml
kubectl apply -f flux/infra-metallb-config.yaml
kubectl apply -f flux/infra-traefik.yaml

# Application
kubectl apply -f flux/kustomization.yaml
```

### Vérifier le statut

```bash
# État de toutes les Kustomizations
kubectl get kustomization -n flux-system

# État des HelmReleases
kubectl get helmrelease -n flux-system

# Ressources ingress
kubectl get all -n ingress
kubectl get all -n metallb-system

# Ressources application
kubectl get all -n chuck-norris
kubectl get ingress -n chuck-norris
```

### Forcer une réconciliation immédiate

```bash
kubectl annotate gitrepository iac-chucknorris-backend -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date -u +%Y-%m-%dT%H:%M:%SZ)" --overwrite
```

---

## Initialisation de la base de données

La table `Joke` doit être créée avant le premier démarrage du backend
(Hibernate est en mode `validate` — il vérifie le schéma sans le créer).

```bash
# Port-forward vers le pod PostgreSQL
kubectl port-forward -n postgres postgres-1 5432:5432

# Injecter le dump SQL
psql -h localhost -p 5432 -U app -d app < /path/to/dump.sql
```

Une fois le schéma injecté, le pod redémarre automatiquement et passe en `Running`.

---

## Structure du repo IaC

```
iac-chucknorris-backend/
├── timoni.cue           # Configuration du module Timoni
├── values.cue           # Schéma + valeurs par défaut
├── prod-values.cue      # Surcharges de production
├── templates/
│   ├── configmap.cue    # ConfigMap Quarkus (variables d'env non-sensibles)
│   ├── deployment.cue   # Deployment + probes + securityContext
│   ├── service.cue      # ClusterIP service
│   └── ingress.cue      # Ingress Traefik + TLS cert-manager
├── deploy/              # Manifestes K8s pré-rendus → Flux applique ce dossier
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── deployment.yaml  # Tag d'image mis à jour automatiquement par le CI
│   ├── service.yaml
│   ├── middleware.yaml  # Traefik Middleware: strip-api-prefix (/api → /)
│   ├── ingress.yaml
│   ├── auth0-sealed-secret.yaml
│   ├── db-sealed-secret.yaml
│   └── kustomization.yaml
├── infra/
│   ├── metallb/         # HelmRelease MetalLB + namespace
│   ├── metallb-config/  # IPAddressPool + L2Advertisement
│   └── traefik/         # HelmRelease Traefik + namespace
└── flux/                # Ressources FluxCD (appliquées une fois manuellement)
    ├── gitrepository.yaml
    ├── kustomization.yaml        # App (dependsOn: infra-traefik)
    ├── infra-metallb.yaml
    ├── infra-metallb-config.yaml
    └── infra-traefik.yaml
```

### Images Docker

| Image | Registry | Architecture |
|-------|----------|--------------|
| `leeson77/chuck-norris-backend` | Docker Hub | multi-arch (amd64 + arm64) |
| `leeson77/chuck-norris-backend-timoni` | Docker Hub | N/A (OCI module) |

Format du tag backend : `<short-sha>` (ex: `3953622`)

---

## Configuration Auth0

La configuration OIDC non-sensible est dans le ConfigMap `chuck-norris-backend-config`.
Le secret (`QUARKUS_OIDC_CREDENTIALS_SECRET`) est chiffré dans `auth0-sealed-secret.yaml`
et déchiffré automatiquement par le contrôleur Sealed Secrets.

| Propriété | Valeur |
|-----------|--------|
| Client ID | `2GSRAJdfhRYURWjwlnYwJlJN4J9cEi4o` |
| Issuer | `https://dev-lesson.eu.auth0.com/` |
| Auth Server URL | `https://dev-lesson.eu.auth0.com` |
| Audience | `chuck-norris-api` |
| Application type | `service` |
| Role claim path | `scope` |

---

## Architecture Diagram

![Alt docker](assets/k8s.drawio.png)
