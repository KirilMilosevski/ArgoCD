HOMELAB GITOPS: ARGO CD + VAULT (DEV) + EXTERNAL SECRETS + STUDENT API
======================================================================

This repository drives a small GitOps homelab:

- Argo CD for continuous delivery
- Vault (dev mode) + External Secrets Operator (ESO) for app secrets
- student-api (Helm chart) + PostgreSQL
- A GitHub Actions workflow that re-tags your Docker Hub image and bumps the Helm
  image.tag (runs on your self-hosted runner) so Argo CD auto-syncs deploys.


REPO LAYOUT
-----------
.github/workflows/   -> CI workflows (retag + bump values.yaml)
argocd/              -> Argo CD manifests (namespace, Application, etc.)
github-runner/       -> Runner helper files (optional)
student-stack/       -> Helm chart for student-api + postgres + ESO
  templates/
    app-deployment.yaml
    postgres-*.yaml
    externalsecret-app.yaml
    externalsecret-db.yaml
    ...
  secretstore.yaml       (ClusterSecretStore for Vault)
  values.yaml            (image repo/tag, DB values, ESO refs)


BOOTSTRAP (QUICK PATH)
----------------------
1) Install Argo CD
   kubectl create ns argocd
   helm -n argocd repo add argo https://argoproj.github.io/argo-helm
   helm -n argocd install argocd argo/argo-cd

   Port-forward & login:
   kubectl -n argocd port-forward svc/argocd-server 8080:443
   # UI: https://localhost:8080  (self-signed; proceed)
   argocd login localhost:8080 --username admin --password "$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)" --insecure --grpc-web

2) Install External Secrets + Vault (dev)
   kubectl create ns external-secrets
   helm -n external-secrets repo add external-secrets https://charts.external-secrets.io
   helm -n external-secrets install external-secrets external-secrets/external-secrets

   kubectl create ns vault
   helm -n vault repo add hashicorp https://helm.releases.hashicorp.com
   helm -n vault install vault hashicorp/vault --set "server.dev.enabled=true"

3) Configure ESO to read Vault
   Apply the ClusterSecretStore in the chart:
   helm -n student-api upgrade --install student-api ./student-stack

   Ensure ClusterSecretStore name is "vault-clusterstore" and it points at
   http://vault.vault.svc:8200 with token "root" (see student-stack/secretstore.yaml).

4) Seed secrets into Vault (KV v2 path: secret/student-api)
   kubectl -n vault exec -it deploy/vault -- sh -lc '
     vault login root
     vault kv put secret/student-api DB_USER=postgres DB_PASS=postgres DB_NAME=studentdb
     vault kv get secret/student-api
   '

5) Create the Argo CD Application
   Apply the manifests in argocd/ (namespace + Application pointing to this repo and
   the student-stack chart). After sync you should see:
   - Deployments for student-api and postgres
   - Two Kubernetes Secrets:
       app-credentials -> DB_USER, DB_PASS
       db-credentials  -> POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB


CI/CD: RETAG DOCKER HUB "LATEST" -> BUMP HELM TAG (SELF-HOSTED RUNNER)
-----------------------------------------------------------------------
This repo ships a workflow that does not require a Dockerfile here.

File: .github/workflows/retag-and-bump.yml

WHAT IT DOES:
1. On a GitHub-hosted runner, re-tags docker.io/kiril011/student-api:latest to a unique
   immutable tag YYYYMMDDHHMMSS.
2. On your self-hosted runner, updates student-stack/values.yaml with that new tag,
   commits, and pushes.
3. Argo CD detects the commit and auto-syncs your app.

REQUIRED REPO SECRETS:
- DOCKERHUB_USERNAME
- DOCKERHUB_TOKEN (Docker Hub access token)

RUN IT:
- Manually: Actions -> "Retag latest image and bump Helm tag" -> Run workflow
- Or wait for the scheduled run (hourly by default).

VERIFY:
  # New commit in repo updating student-stack/values.yaml
  argocd app history student-api
  argocd app get student-api --refresh
  kubectl -n student-api get deploy student-api-student-stack     -o jsonpath='{.spec.template.spec.containers[0].image}{"
"}'
  # Expect: kiril011/student-api:<timestamp>
