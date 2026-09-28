# AGENTS.md

Guidance for AI coding agents working in this repository.

## Project overview

`k8s-hybrid-workload` is a Pulumi Infrastructure-as-Code demo written in
TypeScript that showcases the **Pulumi Kubernetes Operator (PKO) v2**. It is
**two Pulumi programs**, not one:

| Program | Project | Runtime | Role |
| --- | --- | --- | --- |
| `index.ts` (repo root) | `k8s-hybrid-workload` | nodejs / npm | **Deployer** — defines a PKO `Stack` CR (`pulumi.com/v1:Stack`) + Secrets + RBAC and applies them to a cluster where the operator is already installed. |
| `workload/index.ts` | `k8s-hybrid-app` | nodejs / npm | **Workload** — the program the operator clones from Git and runs *in-cluster*. Provisions AWS (DynamoDB + Lambda + API Gateway) and Kubernetes (Nginx frontend). |

The deployer hands the operator a Git URL + ESC environments; the operator
clones `workload/`, runs `pulumi up` for `k8s-hybrid-app/dev` inside the
cluster, and reconciles it every 60s.

- **IaC tool:** Pulumi (`nodejs` runtime, npm package manager)
- **Language:** TypeScript (strict mode)
- **Deployer entry point:** `index.ts`
- **Workload entry point:** `workload/index.ts`
- **Stacks:** `dev` for both projects
- **ESC environments:** `cloud-access/aws-access` (AWS creds + `Owner` default
  tag) and `lotctl/k8s-cluster-aks` (kubeconfig). Both are imported by the
  deployer `dev` stack (`Pulumi.dev.yaml`) and attached to the Stack CR.

See `README.md` for the full architecture diagram, ESC definitions, and
end-to-end usage.

## Layout

| Path | Purpose |
| --- | --- |
| `index.ts` | Deployer: PKO Stack CR + Secrets (`pulumi-api-secret`, `git-auth-secret`) + `system:auth-delegator` RBAC. |
| `Pulumi.yaml` / `Pulumi.dev.yaml` | Deployer project + `dev` stack config (imports both ESC envs). |
| `workload/index.ts` | Workload: AWS backend (DynamoDB/Lambda/API Gateway) + K8s frontend (ConfigMap/Deployment/Service). |
| `workload/Pulumi.yaml` | Workload project definition (`k8s-hybrid-app`). |
| `workload/frontend/index.html` | Bootstrap 5 responsive sign-up page with inlined Pulumipus SVG; contains the `__API_ENDPOINT__` placeholder. |
| `workload/frontend/nginx.conf` | Nginx server config (`/healthz`, no-cache on `index.html`). |
| `docs/frontend-screenshot.png` | Screenshot of the rendered frontend (referenced by the README). |
| `package.json` / `workload/package.json` | Node dependencies for each program. |
| `tsconfig.json` / `workload/tsconfig.json` | TypeScript compiler settings (strict mode on). |
| `.kiro/skills/` | Installed Pulumi agent skills (auto-loaded by Kiro). |
| `skills-lock.json` | Lockfile for installed agent skills. |

## Environment setup

This project requires **Node.js 22+** (the agent-skills tooling requires
`>=22.20.0`). Use nvm to select it:

```bash
nvm use 22   # or: nvm install 22
```

Install dependencies **in both programs**:

```bash
npm install
( cd workload && npm install )
```

## Common commands

```bash
# --- Deployer (repo root) ---
pulumi stack select dev
pulumi preview            # Show planned changes (Stack CR + Secrets + RBAC)
pulumi up                 # Apply; the operator then takes over the workload
pulumi config             # View deployer stack configuration
pulumi destroy            # Delete the Stack CR (destroyOnFinalize tears down the workload)

# --- Type-check BOTH programs before deploying ---
npx tsc --noEmit
( cd workload && npx tsc --noEmit )
```

The workload stack (`k8s-hybrid-app/dev`) is normally driven **by the
operator**, not by hand. To run it directly (e.g. to reproduce an in-cluster
failure locally) both ESC envs must be nested so AWS creds *and* kubeconfig are
present:

```bash
cd workload
pulumi env run cloud-access/aws-access -- \
  pulumi env run lotctl/k8s-cluster-aks -- \
    pulumi up --stack pequod/k8s-hybrid-app/dev
```

## Introspection (ESC + kubectl / AWS CLI)

No local kubeconfig or AWS profile is needed — source them from ESC. See the
README's "Introspecting the cluster" and "Validating records in DynamoDB"
sections for the full set. The core pattern:

```bash
# Cluster (kubeconfig from ESC)
pulumi env run lotctl/k8s-cluster-aks -- kubectl get stack k8s-hybrid-workload -n pulumi-kubernetes-operator -o wide

# AWS (creds from ESC)
pulumi env run cloud-access/aws-access -- aws dynamodb scan --region us-west-2 --table-name <table>
```

The workload's Kubernetes resources land in the **`pulumi-kubernetes-operator`**
namespace (the operator's namespace), not `default`.

## Conventions

- TypeScript runs in **strict mode**; keep both builds free of type errors.
- Prefer Pulumi `Output<T>` composition (`.apply()`, `pulumi.interpolate`,
  `pulumi.all`) over resolving values eagerly.
- Do not hardcode secrets. Use Pulumi config secrets (`pulumiAccessToken`,
  `githubToken`) or the ESC environments already wired into the `dev` stack.
- Follow existing resource naming (`appName` prefix in the workload) and
  structure.

### Workload-specific gotchas (learned the hard way)

- **Do NOT set a lowercase `owner` tag** in the workload. The AWS provider
  applies a stack-level `aws:defaultTags` `Owner` tag; AWS IAM rejects tag keys
  that collide case-insensitively (`Owner` vs `owner`) → `Duplicate tag keys
  found`. `baseTags` in `workload/index.ts` intentionally omits it.
- **`require` the AWS SDK INSIDE the Lambda callback**, not at module scope.
  Module-scope `import`s get captured by Pulumi's function serialization and
  fail (native code). The handler does
  `const { DynamoDBClient } = require("@aws-sdk/client-dynamodb")` at call time.
- The frontend HTML contains a `__API_ENDPOINT__` placeholder that the workload
  substitutes with the live API Gateway URL at deploy time. Preserve it when
  editing `workload/frontend/index.html`.
- The Deployment carries a `configmap/checksum` annotation derived from the
  rendered HTML so pods roll when the frontend content changes.

### Editing the frontend

`workload/frontend/index.html` uses **Bootstrap 5** (CDN) for the responsive
grid: a two-column card with the inlined Pulumipus SVG on the left and the
sign-up form on the right, stacking below the `md` breakpoint. Keep the SVG
inlined (no external asset dependency) and keep the `fetch()` POST logic and
the `__API_ENDPOINT__` placeholder intact.

## PKO reconciliation notes

- The operator polls the tracked branch every 60s; a **new commit** triggers an
  update. After **3 failures** the Stack CR stalls
  (`Stalled (UpdateFailed)`) — release with a new commit or bump
  `reconcileTrigger` on the deployer and `pulumi up`.
- A stuck Pulumi Cloud update lock (`409`) clears with
  `pulumi cancel --stack pequod/k8s-hybrid-app/dev --yes`.

## Safety

- **`pulumi up` and `pulumi destroy` change real cloud infrastructure.** Run
  `pulumi preview` first and confirm the diff before applying.
- Treat `pulumi destroy`, resource replacements, and stack/config deletions as
  high-risk. Confirm with a human before running them.
- Cluster-side infra fixes (the shared AKS cluster, operator install) should go
  through a **pull request for human review**, not be applied directly.
- Do not commit state, secrets, or the `bin/` build output. `.gitignore`
  already excludes `bin/`, `node_modules/`, and local agent files.

## Agent skills

Pulumi agent skills are installed under `.kiro/skills/` and are loaded
automatically by Kiro's default agent. They cover authoring best practices,
components, ESC, the Automation API, provider upgrades, migrations, debugging
failed operations, and Neo handoff. Reach for the relevant skill when a task
matches its domain. Refresh them with:

```bash
npx skills add pulumi/agent-skills/pulumi --skill '*' --agent kiro-cli
npx skills add pulumi/agent-skills/package-maintenance --skill '*' --agent kiro-cli
```
