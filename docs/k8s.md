# Kubernetes documentation

## Technical stack

* Container orchestration: Kubernetes (kubeadm)
* OS: Ubuntu ARM64
* GitOps: FluxCD v2
* Package manager: Timoni (module OCI)
* Secrets: Sealed Secrets (Bitnami)
* Database: CloudNativePG
* Load Balancer: MetalLB (L2 mode)
* Ingress Controller: Traefik v3

## Requirements

* ARM64 nodes (all container images are built for `linux/arm64`)
* A DNS entry for `chuck.filhype.ovh`
* cert-manager + ClusterIssuer `lets-encrypt`
* MetalLB deployed in `metallb-system` (géré par Flux via k8s-configuration)
* Traefik deployed in `ingress` (géré par Flux via k8s-configuration)
* Sealed Secrets controller in `kube-system`
* CloudNativePG cluster `postgres` in namespace `postgres`
* FluxCD v2 deployed in namespace `flux-system`

## Cluster nodes

| Hostname        | Role          | IP            |
|-----------------|---------------|---------------|
| hp              | control-plane | 192.168.1.23  |
| leeson-desktop  | worker        | 192.168.1.26  |
| leeson-jetson2  | worker        | 192.168.1.25  |

---

## Networking stack (MetalLB + Traefik)

> Manifestes et bootstrap Flux dans le repo [`k8s-configuration`](https://github.com/filhype-organization/k8s-configuration).

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

### Repos

| Repo                      | Rôle                                          |
|---------------------------|-----------------------------------------------|
| `k8s-configuration`       | Bootstrap Flux, infra (MetalLB, Traefik, Cloudflared) |
| `iac-chucknorris-backend` | Manifestes K8s du backend (`deploy/`)         |

### Architecture Flux

```
flux-system
├── GitRepository: iac-k8s-configuration  → k8s-configuration
│     ├── Kustomization: infra-metallb
│     ├── Kustomization: infra-metallb-config
│     ├── Kustomization: infra-traefik
│     └── Kustomization: infra-cloudflared
│
└── GitRepository: iac-chucknorris-backend → iac-chucknorris-backend
      └── Kustomization: chuck-norris-backend → deploy/
```

### Ordre de déploiement (dependsOn)

```
infra-metallb
    └→ infra-metallb-config
           └→ infra-traefik
                  └→ infra-cloudflared
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

## Structure du repo IaC backend

```
iac-chucknorris-backend/
├── timoni.cue           # Configuration du module Timoni
├── values.cue           # Schéma + valeurs par défaut
├── prod-values.cue      # Surcharges de production
├── templates/
│   ├── configmap.cue
│   ├── deployment.cue
│   ├── service.cue
│   └── ingress.cue
└── deploy/              # Manifestes K8s pré-rendus → Flux applique ce dossier
    ├── namespace.yaml
    ├── configmap.yaml
    ├── deployment.yaml  # Tag d'image mis à jour automatiquement par le CI
    ├── service.yaml
    ├── middleware.yaml  # Traefik Middleware: strip-api-prefix (/api → /)
    ├── ingress.yaml
    ├── auth0-sealed-secret.yaml
    ├── db-sealed-secret.yaml
    └── kustomization.yaml
```

### Images Docker

| Image | Registry | Architecture |
|-------|----------|--------------|
| `leeson77/chuck-norris-backend` | Docker Hub | multi-arch (amd64 + arm64) |
| `leeson77/chuck-norris-backend-timoni` | Docker Hub | N/A (OCI module) |

Format du tag backend : `<short-sha>` (ex: `3953622`)

---

## Premier déploiement

### Bootstrap Flux

```bash
# Dans le repo k8s-configuration
kubectl apply -k flux/
```

### Prérequis secrets GitHub

```bash
gh secret set IAC_REPO_TOKEN --repo filhype-organization/super-chuck-norris-backend
# Valeur : GitHub fine-grained PAT avec Contents: Read & Write sur iac-chucknorris-backend
```

### Vérifier le statut

```bash
kubectl get gitrepository,kustomization -n flux-system
kubectl get helmrelease -A
kubectl get all -n chuck-norris
kubectl get ingress -n chuck-norris
```

---

## Initialisation de la base de données

La table `Joke` doit être créée avant le premier démarrage du backend
(Hibernate est en mode `validate` — il vérifie le schéma sans le créer).

```bash
kubectl port-forward -n postgres postgres-1 5432:5432
psql -h localhost -p 5432 -U app -d app < /path/to/dump.sql
```

---

## Configuration Auth0

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
