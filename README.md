# configure-kubectl

GitHub Action that authenticates to a [Cloudfleet (CFKE)](https://cloudfleet.ai) cluster using
GitHub Actions OIDC, with no long-lived credentials in your repository.

## Quick start

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: azure/setup-kubectl@v5
      - uses: cloudfleet-actions/configure-kubectl@v1
        with:
          cluster-id: ${{ vars.CFKE_CLUSTER_ID }}
          region: ${{ vars.CFKE_REGION }}
      - run: kubectl get nodes
```

## Full example: read-only cluster check

```yaml
name: Cluster sanity check
on:
  workflow_dispatch:
  schedule:
    - cron: '*/30 * * * *'

permissions:
  id-token: write       # required: lets the workflow mint a GitHub OIDC token
  contents: read

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: azure/setup-kubectl@v5

      - name: Configure kubectl
        uses: cloudfleet-actions/configure-kubectl@v1
        with:
          cluster-id: ${{ vars.CFKE_CLUSTER_ID }}
          region:     ${{ vars.CFKE_REGION }}

      - name: Identity
        run: kubectl auth whoami

      - name: Node and pod overview
        run: |
          kubectl get nodes -o wide
          kubectl get pods --all-namespaces -o wide
```

Two repository **variables** (not secrets — neither is sensitive):

| Variable | Example |
|---|---|
| `CFKE_CLUSTER_ID` | `95cc1ef4-2122-4b51-97d9-b35b531c3c45` |
| `CFKE_REGION` | `europe-central-1a` |

## Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `cluster-id` | yes | | CFKE cluster ID (UUID). |
| `region` | yes | | Cluster region (e.g. `europe-central-1a`). |

## Outputs

| Name | Description |
|---|---|
| `kubeconfig` | Path to the kubeconfig file written by this action. Also exported as `KUBECONFIG` env var. |

## Cluster admin: granting RBAC

The action authenticates the workflow as the GitHub OIDC token's `sub` claim. The user
has no permissions until you bind RBAC to that subject.

### Subject cheat sheet

The `sub` GitHub puts in the token depends on what triggered the workflow:

CFKE prefixes the GitHub `sub` with `github:` to namespace these identities away
from `system:*` and Keycloak users. So the value you bind RBAC to is
`github:` + the `sub` GitHub put in the token:

| Trigger | RBAC subject |
|---|---|
| `push` to a branch | `github:repo:OWNER/NAME:ref:refs/heads/BRANCH` |
| `push` of a tag | `github:repo:OWNER/NAME:ref:refs/tags/TAG` |
| `pull_request` | `github:repo:OWNER/NAME:pull_request` |
| Job using a deployment environment | `github:repo:OWNER/NAME:environment:ENV` |
| Reusable workflow | `github:repo:OWNER/NAME:job_workflow_ref:OTHER/REPO/.github/workflows/X.yml@REF` |

(GitLab CI identities are prefixed with `gitlab:` analogously.)

Bind RBAC to the exact subject your workflow produces. Don't grant `cluster-admin` to
`pull_request` — every PR (including from forks if you allow it) hits that.

### Example: cluster-wide read access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gha-myapp-view
subjects:
  - kind: User
    name: "github:repo:my-org/my-app:ref:refs/heads/main"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view              # built-in read-only role
  apiGroup: rbac.authorization.k8s.io
```

### Example: namespace-scoped read access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gha-myapp-view
  namespace: production
subjects:
  - kind: User
    name: "github:repo:my-org/my-app:ref:refs/heads/main"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

### Verifying the binding

The action's first run will show `Forbidden` errors with the exact `User` it was
authenticated as — confirming auth works before you debug authorisation. Or check
explicitly with:

```bash
kubectl auth whoami       # prints sub + extras
kubectl auth can-i --list # what RBAC currently allows for this identity
```

## Policy

In addition to the username (`sub`), CFKE attaches GitHub claims to the
authenticated user as `extras`, available to `ValidatingAdmissionPolicy` for
fine-grained checks RBAC can't express:

| Extra key | Source claim |
|---|---|
| `cfke.io/github-repository` | `claims.repository` |
| `cfke.io/github-ref` | `claims.ref` |
| `cfke.io/github-actor` | `claims.actor` |
| `cfke.io/github-run-id` | `claims.run_id` |
| `cfke.io/github-workflow` | `claims.workflow` |
| `cfke.io/github-event` | `claims.event_name` |

Example policy: deny modifications in the `production` namespace unless the request
comes from `refs/heads/main`:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: prod-only-from-main
spec:
  matchConstraints:
    resourceRules:
      - apiGroups: ["*"]
        apiVersions: ["*"]
        operations: ["CREATE", "UPDATE", "DELETE"]
        resources: ["*"]
  validations:
    - expression: |
        !request.userInfo.username.startsWith('repo:') ||
        ('cfke.io/github-ref' in request.userInfo.extra &&
         request.userInfo.extra['cfke.io/github-ref'][0] == 'refs/heads/main')
      message: "Production resources may only be modified from refs/heads/main"
```

## Notes

- `kubectl` is **not** installed by this action. Add `azure/setup-kubectl@v5` (or any other
  installer) before this step.
- The OIDC token GitHub mints expires in 5 minutes. `kubectl` calls after that will fail —
  re-run the action if your job is long-lived.
- The action writes `KUBECONFIG` to `$GITHUB_ENV` so subsequent steps in the same job can
  call `kubectl` without `--kubeconfig`.
