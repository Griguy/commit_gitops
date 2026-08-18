# cm_gitops

Argo CD app-of-apps source. `bootstrap/root-app.yaml` is applied once, by
hand, into the `argocd` namespace; it then watches `apps/` (recursively) in
this repo and syncs every `Application` it finds there.

```
bootstrap/root-app.yaml          # apply once: kubectl apply -f bootstrap/root-app.yaml
apps/
  platform/                      # cluster-wide add-ons: ALB controller, observability stack
    aws-load-balancer-controller.yaml
    kube-prometheus-stack.yaml
    otel-collector.yaml
    grafana-dashboard-app-metrics.yaml   # plain ConfigMap, not an Application -- see file
  services/                       # one Application per app service, per environment
    backend.yaml
    frontend.yaml
envs/
  dev/                             # only environment that exists today
    backend/values.yaml            # CodeBuild bumps image.tag here on every build
    frontend/values.yaml
```

## How the services get deployed

`apps/services/backend.yaml` and `frontend.yaml` are Argo CD **multi-source**
Applications: the Helm chart itself comes from `cm_charts`, pinned to a
tagged release (`backend-0.1.0`, `frontend-0.1.0` right now -- never `main`,
so a chart template edit doesn't go live until someone explicitly bumps the
pinned tag), while the values come from this repo's `envs/dev/<service>/values.yaml`
via Argo CD's `ref: values` cross-source values-file feature.

`commit_backend`/`commit_frontend`'s CodePipeline builds an image on every
push to `dev`, pushes it to ECR under an immutable git-SHA tag, then clones
this repo and bumps `envs/dev/<service>/values.yaml`'s `image.tag` to that
SHA, commits, and pushes to `main` -- Argo CD's `selfHeal: true` picks that
up automatically (polling by default; a hard refresh via
`kubectl patch application <name> -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'`
forces it immediately if you don't want to wait).

Both `envs/dev/*/values.yaml` start with `image.tag: unreleased` --
deliberately not a real tag, so a from-scratch deploy fails visibly
(`ImagePullBackOff`) until the pipeline has actually run at least once,
rather than silently pulling whatever `latest` happens to resolve to.

## Status

Everything listed above exists and is wired up. `apps/platform/*` and
`apps/services/*` are both live Applications as of this repo's current
state -- nothing here is aspirational/not-yet-added anymore.
