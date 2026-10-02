# Service Foundry Web GitOps

GitOps configuration for [`lingarajkarorg/service-foundry-web`](https://github.com/lingarajkarorg/service-foundry-web). The Helm chart serves the app's existing NGINX image on port 80, mounts its runtime `config.json`, and exposes the app using Traefik Gateway API `HTTPRoute` resources.

## Delivery flow

- Pull requests labeled `preview` build `ghcr.io/lingarajkarorg/service-foundry-web:pr-<number>` and are discovered by the Argo CD ApplicationSet.
- Pushes to `develop` build `sha-<12-character-sha>` and update the development values.
- Pushes to `main` build that same immutable SHA tag and update staging.
- A `v*` tag promotes the existing image for that exact commit to production by copying the registry manifest and pinning its digest. It does not rebuild the app.
- Argo CD auto-syncs the values changes. Argo Rollouts uses blue-green in development and Gateway API weighted canaries in staging and production.
- A PR preview Application can be created before its `pr-<number>` image has finished building; the pod may briefly show `ImagePullBackOff` and recover after the image push.

The app repository's Actions files are kept in [`application-repo/.github/workflows`](application-repo/.github/workflows) as the source of truth for this setup. Copy those four files into `lingarajkarorg/service-foundry-web/.github/workflows/`, and copy [`application-repo/.dockerignore`](application-repo/.dockerignore) to the app repository root. Workflows in this GitOps repository cannot trigger on pushes or PRs in the separate app repository.

## Cluster values to set

The hostname values currently use `example.com`; replace that suffix in the three environment values files and `argocd/applicationset-pr.yaml`. The chart assumes a Gateway named `traefik-gateway` in namespace `traefik`, with HTTPS listener `websecure`. Adjust the chart's base `values.yaml` if your Gateway differs. Its listener must allow Routes from `sfw-*` namespaces, and DNS records must resolve to Traefik.

The cluster must have:

- Argo CD and Argo Rollouts installed.
- Gateway API `HTTPRoute` CRDs and Traefik configured as a Gateway API implementation.
- The [Argo Rollouts Gateway API traffic-routing plugin](https://rollouts-plugin-trafficrouter-gatewayapi.readthedocs.io/en/latest/installation/) installed, with its HTTPRoute permissions. Without the plugin, canary weights will not affect Traefik traffic.
- Sealed Secrets installed if Kubernetes credentials or app secrets are introduced. The current static frontend has no runtime secrets. Public GHCR is configured for friction-free pull-request previews.

The canary AnalysisTemplate calls the canary's `/healthz` endpoint, provided by the chart's NGINX config. This is a JSON HTTP smoke check, not request-rate or error-rate analysis. Connect an AnalysisTemplate to Traefik/Prometheus metrics before treating it as an SLO-based production gate.

## Configure and bootstrap

1. Set the app repository variable `GITOPS_REPOSITORY` to `lingarajkarorg/service-foundry-gitops` and add a `GITOPS_TOKEN` secret with contents write access to the GitOps repository.
2. Copy the four files in `application-repo/.github/workflows/` to the app repository's `.github/workflows/`, and copy `application-repo/.dockerignore` to its root. Create a `preview` label in the app repository. Configure the `production` GitHub Environment with required reviewers and protect `v*` tags from deletion or force updates.
3. Make the GHCR package public to allow new preview namespaces to pull images without credentials. If the package must remain private, see [Sealed Secrets](docs/sealed-secrets.md); dynamic preview namespaces need a cluster-wide registry access policy.
4. Ensure Argo CD can read this GitOps repository. For PR discovery, create an Argo CD namespace Secret named `github-token` with a `token` key that can read pull requests and labels from `lingarajkarorg/service-foundry-web`. Prefer storing it as a SealedSecret; see [Sealed Secrets](docs/sealed-secrets.md).
5. Apply the Argo CD project, Applications, and ApplicationSet:

   ```sh
   kubectl apply -n argocd -f argocd/projects/service-foundry-web.yaml
   kubectl apply -n argocd -f argocd/applications/
   kubectl apply -n argocd -f argocd/applicationset-pr.yaml
   ```

6. Push to `develop`, then `main`, to exercise development and staging. After staging verification, tag that same `main` commit with `v*` to promote its existing image digest. The GitHub `production` environment approval happens before the GitOps production values are changed.

## Helm validation

```sh
helm lint charts/service-foundry-web -f charts/service-foundry-web/values-development.yaml
helm lint charts/service-foundry-web -f charts/service-foundry-web/values-staging.yaml
helm lint charts/service-foundry-web -f charts/service-foundry-web/values-production.yaml --set image.digest=sha256:0000000000000000000000000000000000000000000000000000000000000000
helm template service-foundry-web charts/service-foundry-web -n sfw-staging -f charts/service-foundry-web/values-staging.yaml
```

The successful-build workflow uses `workflow_run.head_sha`, checks `conclusion == success`, and updates YAML with `yq`. It retries by fetching the latest GitOps `main` and reapplying its single-file change if a concurrent push wins.