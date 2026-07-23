# WorkflowsArgo

Manifests Argo Workflows (`WorkflowTemplate`, `CronWorkflow`, …) synchronisés
par ArgoCD vers le namespace `workflows-argo`.

## Structure

```text
manifests/
  serviceaccount.yaml   # SA + RBAC executor
  hello-template.yaml   # exemple WorkflowTemplate
```

## Déploiement

1. Plateforme TalosGitOps : `workflows-argo` dans `workflowNamespaces` + repo
   whiteliste dans l’AppProject `workloads`.
2. Application ArgoCD dans `homelab-apps` :
   `clusters/<cluster>/apps/30-workflows-argo.yaml`.
3. Push dans ce repo → ArgoCD sync auto.

## Lancer un run

```bash
argo submit --from workflowtemplate/hello -n workflows-argo
```

Ou via l’UI Argo Workflows.
