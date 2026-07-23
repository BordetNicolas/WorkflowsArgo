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

Stress test basique du cluster Argo : fan-out de N pods en parallèle
(~10s de charge CPU chacun), puis rapport JSON vers MinIO.

```bash
argo submit --from workflowtemplate/load-test -n workflows-argo \
  -p workers=20
```

| Paramètre | Défaut | Rôle |
|-----------|--------|------|
| `workers` | `10` | Nombre de pods parallèles (1–200) |

Artifacts MinIO : `plan` (préparation), `worker_result` (par pod), `report` (synthèse).
