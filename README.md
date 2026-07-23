# WorkflowsArgo

Manifests Argo Workflows (`WorkflowTemplate`, `CronWorkflow`, …) synchronisés
par ArgoCD vers le namespace `workflows-argo`.

## Structure

```text
manifests/
  serviceaccount.yaml              # SA + RBAC executor
  artifact-repositories.yaml       # ConfigMap MinIO/S3 (placeholders)
  hello-template.yaml              # exemple WorkflowTemplate
  chaine-directe-template.yaml     # chaîne documentaire (DAG + fan-out)
  load-test-template.yaml          # stress test cluster Argo (fan-out)
```

## Déploiement

1. Plateforme TalosGitOps : `workflows-argo` dans `workflowNamespaces` + repo
   whiteliste dans l’AppProject `workloads`.
2. Application ArgoCD dans `homelab-apps` :
   `clusters/<cluster>/apps/30-workflows-argo.yaml`.
3. Push dans ce repo → ArgoCD sync auto.

## Prérequis MinIO (artifacts)

Les workflows qui passent des fichiers entre étapes (ex. `chaine-directe`)
utilisent le ConfigMap [`manifests/artifact-repositories.yaml`](manifests/artifact-repositories.yaml).

1. Configurer `bucket` et `endpoint` (**hostname seul**, sans `https://`).
   Pour MinIO en TLS : `insecure: false`. Pour HTTP : `insecure: true`.
2. Créer le Secret credentials **hors git** :

```bash
kubectl -n workflows-argo create secret generic minio-artifacts \
  --from-literal=accessKey='YOUR_ACCESS_KEY' \
  --from-literal=secretKey='YOUR_SECRET_KEY'
```

Ne jamais committer de clés MinIO dans ce dépôt.

## Lancer un run

### hello

```bash
argo submit --from workflowtemplate/hello -n workflows-argo
```

### chaine-directe

Chaîne : validation XML → XML→PS → modification PS → tri/éclatement →
3 canaux en parallèle (archivage, impression, guichet électronique), chacun
avec **0..N paquets** (`withParam`).

```bash
argo submit --from workflowtemplate/chaine-directe -n workflows-argo \
  -p input_xml_key=inbox/exemple.xml \
  -p packet_seed=42 \
  -p enable_random_failures=true \
  -p fail_rate_percent=30
```

L’entrée XML est simulée à partir de `input_xml_key` (stub). Les fichiers
entre étapes transitent bien via MinIO (artifacts Argo).

Chaque step peut échouer aléatoirement (`exit 42`) puis être retenté jusqu’à
3 fois (`retryStrategy`, backoff 2s ×2). Désactiver le chaos :
`-p enable_random_failures=false`.

| Paramètre | Défaut | Rôle |
|-----------|--------|------|
| `input_xml_key` | `inbox/exemple.xml` | Clé logique S3 du XML source |
| `packet_seed` | `42` | Conservé pour compat (le split tire 0–20 paquets/canal au hasard) |
| `simulate_missing_input` | `true` | Conservé pour compat ; entrée toujours stubbée pour l’instant |
| `enable_random_failures` | `true` | Active les échecs aléatoires pour tester les retries |
| `fail_rate_percent` | `30` | Probabilité d’échec par tentative (0–100) |

Ou via l’UI Argo Workflows.

### load-test

Stress agressif du cluster Argo / nœuds : fan-out de N pods en parallèle.
Chaque pod lance **stress-ng** (CPU + RAM + I/O + disque) pendant
`duration_seconds`, avec requests Kubernetes `1 CPU` / `1 GiB` pour saturer
le scheduler.

```bash
# Défaut déjà costaud (~25 pods × 120s)
argo submit --from workflowtemplate/load-test -n workflows-argo

# Monter encore la charge
argo submit --from workflowtemplate/load-test -n workflows-argo \
  -p workers=40 \
  -p duration_seconds=180 \
  -p vm_bytes=1G \
  -p hdd_bytes=1G
```

| Paramètre | Défaut | Rôle |
|-----------|--------|------|
| `workers` | `25` | Pods parallèles (1–500) |
| `duration_seconds` | `120` | Durée stress-ng par pod |
| `cpu_workers` | `4` | Stressors CPU / pod |
| `vm_workers` | `2` | Stressors RAM / pod |
| `vm_bytes` | `512M` | RAM par stressor vm |
| `hdd_workers` | `2` | Stressors disque / pod |
| `hdd_bytes` | `512M` | Volume I/O disque par stressor |

Artifacts MinIO : `plan`, `worker_result` (par pod), `report`.

Attention : avec les défauts, ~25 CPU / 25 GiB sont demandés au cluster.
