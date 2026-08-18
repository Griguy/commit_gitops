# cm_gitops

Argo CD app-of-apps source. `bootstrap/root-app.yaml` is applied once, by
hand, into the `argocd` namespace; it then watches `apps/` in this repo and
syncs every `Application` it finds there.

```
bootstrap/root-app.yaml   # apply once: kubectl apply -f bootstrap/root-app.yaml
apps/                     # one Application manifest per thing Argo CD manages
```

## Status

- `apps/kube-prometheus-stack.yaml` — ready to sync as-is, no other
  dependencies.
- `aws-load-balancer-controller` — not added yet. It needs an IRSA IAM role
  (OIDC-federated, scoped to the controller's service account) that doesn't
  exist in the infrastructure repo yet; adding the Application before that
  role exists would leave the controller unable to call the AWS APIs it
  needs.
- `cm-backend` / `cm-frontend` — not added yet. Each app's own repo needs a
  Helm chart under it first; the Application here would just point at that
  chart's path once it exists.
