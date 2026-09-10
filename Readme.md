# Full Stack Runtime

A runtime environment for the complete application stack — frontend and
API/backend — with PostgreSQL assumed to be provided externally. It's
designed to be dropped into any Kubernetes cluster reachable only via
`kubectl` — no container registry, no image build step, and no CI/CD
pipeline required for ordinary code changes.

**The runtime is immutable; the application is not.** `helm upgrade`
provisions and configures the runtime itself (Deployments running stock,
unmodified public images, Services, Ingress, PVCs, ConfigMaps) and is only
needed again when *that* changes. The application running inside it —
frontend build output, backend source — lives on a PVC and is pushed
straight onto the running pod with `kubectl cp`, independently of the chart,
with no image to rebuild or publish. Immutable runtime, mutable app.

If you're an AI coding agent reading this to figure out how to deploy or
iterate on an app, read **"For AI agents: agentic development with no
pipeline"** below first — it's the point of this chart.

## What this chart deploys

- **`frontend`** — a Deployment running `nginx-unprivileged` (stock image,
  unmodified) that serves static files from a PersistentVolumeClaim. nginx
  config (caching, security headers, SPA/SSG fallback) is templated from
  `files/default.conf` / `files/security-headers.conf` into a ConfigMap and
  mounted in.
- **`api-backend`** (optional, `backend.enabled`) — a Deployment running the
  **stock `node:*-alpine` image** (again unmodified — no custom build) that
  runs app code from a second PVC under `nodemon`. There is no image for this
  service at all; the PVC *is* the deployment artifact.
- **Ingress** — two Traefik `Ingress` objects (plain-HTTP → HTTPS redirect,
  and the TLS-terminating one) plus a Traefik `Middleware` CRD (`force-https`)
  that the redirect Ingress references. The TLS Ingress requests a
  certificate via cert-manager. `/api` (if the backend is enabled) and `/`
  are both routed off the same host.
- **ConfigMaps** — the nginx-config ConfigMap (`files/default.conf` +
  `files/security-headers.conf`, templated and mounted into the frontend
  pod), plus an optional `frontend-extra-env` ConfigMap when
  `frontend.env` is set.
- **PVCs** — one per Deployment (`frontend`, `<backend.name>-code`), both
  `ReadWriteOnce`, both mounted by a `strategy: Recreate` Deployment (a
  rolling update would deadlock trying to attach an RWO volume to a second
  pod before the first releases it).

Nothing in `templates/` needs editing for a new site or app — every
site-specific value (hostname, image tags, resource limits, storage class,
replica counts, whether the backend exists at all, secret names) lives in
`values.yaml`. See the comments there for every key.

## Requirements

- A Kubernetes cluster with the **Traefik** ingress controller (the chart's
  `Ingress` annotations and `Middleware` CRDs are Traefik-specific).
- **cert-manager** with a `ClusterIssuer` already configured, if
  `ingress.tls.enabled` (the default).
- A **StorageClass** that can satisfy `ReadWriteOnce` PVCs (defaults assume
  one named `retain-local`; change `frontend.persistence.storageClassName`
  and `backend.persistence.storageClassName` to match your cluster).
- `helm` 3.x and `kubectl` pointed at the target cluster.

## Install

```bash
helm upgrade --install <release-name> ./full-stack-runtime -n <namespace> -f my-values.yaml
```

Copy `values.yaml`, change the hostname/image/resource/storage values for
your site, and point `-f` at your copy — or use `--set` for one-off
overrides. Set `backend.enabled: false` to deploy a static-only site with no
API (the backend Deployment/Service/PVC and the `/api` Ingress path are
omitted entirely).

## For AI agents: agentic development with no pipeline

This chart's defining property: **immutable runtime, mutable app.**
`helm upgrade` deploys infrastructure, not code. Neither container image is
custom-built or pushed anywhere — both
Deployments run stock public images (`nginxinc/nginx-unprivileged`,
`node:*-alpine`) and mount a PVC where the actual application content/code
lives. That split means an agent — or a human — can go from a local code edit
to a verifiable, live result on the cluster's real ingress-exposed URL using
nothing but `kubectl`, in seconds, with no image build, no registry
credentials, and no waiting on external CI infrastructure:

- **Frontend**: build the static site locally, then copy the build output
  straight onto the running pod:
  ```bash
  kubectl cp <build-output-dir>/. <namespace>/<frontend-pod>:/usr/share/nginx/html
  ```
  nginx serves the new files immediately — no restart, no rollout.
- **Backend**: copy source (plus a Linux-safe `node_modules`, if the backend
  has any dependencies) straight onto the running pod:
  ```bash
  kubectl cp <server-dir>/. <namespace>/<backend-pod>:/app
  ```
  The container's entrypoint runs the app under `nodemon`, which watches
  `/app` and restarts the `node` process — not the container — whenever new
  code lands. `kubectl get pods` shows `RESTARTS: 0` across ordinary code
  pushes; only `kubectl logs -f <pod>` shows nodemon cycling the process.

`helm upgrade` only re-enters the loop when *infrastructure* changes: a new
env var, a resource limit, a different hostname, flipping `backend.enabled`,
bumping a base image tag, editing the nginx config. Everything else — every
ordinary feature/bugfix iteration — is a single `kubectl cp`.

### Why this shape, specifically for a Softwaredam Kubernetes cluster

This pattern exists because of two real constraints on how these clusters
(Softwaredam's included) are typically reachable:

1. **No registry access, no SSH to the node.** The cluster is only reachable
   through `kubectl` against the kubeconfig you've been given (e.g.
   `KUBECONFIG=./kubeconfig.yaml kubectl ...`). There's nowhere to push a
   custom image to, and no way to place one on the node directly. Running a
   stock public image and mounting your own code over a PVC sidesteps both
   problems entirely.
2. **Pure-JS / no compiled dependencies.** This only works cleanly when the
   backend's dependencies have no native bindings — a `node_modules`
   installed on your dev machine (with `npm ci --omit=optional` to drop
   macOS-only optional natives like `fsevents`) runs unmodified inside the
   Linux pod. If your backend needs compiled/native modules, this approach
   breaks down and you'd need an actual image build for it instead — the
   kubeconfig-only, no-registry constraint stays the same, but the
   `kubectl cp` shortcut for that one service does not apply.

For an agent operating directly on a developer's machine — with shell access
and a working kubeconfig, but no CI system, no registry login, and no
pipeline definitions to write or trigger — this is the entire deploy
surface: `helm upgrade --install` once to stand up the infrastructure, then
`kubectl cp` on every iteration after that. There is no separate "deploy"
step to design, no image tag to bump and wait on, and no pipeline run to
poll — the change is live as soon as the copy finishes, so the agent can
copy, `curl` (or hit the Ingress host) and read the result, and re-edit in
the same loop it already uses for local files.

### Bootstrapping a first deploy

Both PVCs start empty. The frontend's `location /` will 404 until you
`kubectl cp` a build; the backend container's entrypoint waits (polling every
few seconds, visible in `kubectl logs`) for its entrypoint file to appear
before it will start `nodemon` — this is expected on a fresh install, not a
crash. Run the `kubectl cp` steps above once right after `helm upgrade
--install` to populate both PVCs for the first time.

## Configuration reference

All keys live in `values.yaml`, grouped by resource. Highlights:

| Key | Purpose |
| --- | --- |
| `ingress.host` | Public hostname; also templates nginx's `server_name`. |
| `ingress.className` | IngressClass name (`traefik` by default). |
| `ingress.tls.enabled` / `.secretName` / `.clusterIssuer` | TLS via cert-manager; set `enabled: false` to skip TLS/cert-manager entirely. |
| `ingress.httpEntrypoint` / `.httpsEntrypoint` | Traefik entrypoint names, only if your cluster's Traefik uses non-default ones. |
| `frontend.name` | Base name for the frontend Deployment/PVC/Service/ConfigMap and its `app` label — change to run more than one instance of this chart's frontend in one namespace. |
| `frontend.image.*`, `.containerPort`, `.servicePort` | nginx image/tag and ports. |
| `frontend.persistence.size` / `.storageClassName` | Frontend content PVC. |
| `frontend.env` | Extra env vars for the frontend container, injected via a ConfigMap. |
| `backend.enabled` | Set `false` to omit the backend entirely (Deployment/Service/PVC/`/api` Ingress path). |
| `backend.name`, `.apiPathPrefix` | Backend base name and the Ingress path prefix routed to it. |
| `backend.image.*`, `.containerPort`, `.servicePort` | Base Node image/tag and ports — this is **not** an app-specific image. |
| `backend.persistence.size` / `.storageClassName` | Backend code PVC. |
| `backend.postgres.enabled` | Set `false` to skip wiring `PGHOST`/`PGPORT`/`PGDATABASE`/`PGUSER`/`PGPASSWORD` entirely — for a backend with no Postgres dependency, or one whose DB connection you'd rather wire yourself via `backend.env`. |
| `backend.postgres.secretName` | Existing Secret (provisioned elsewhere — e.g. a CloudNativePG cluster or a separate Postgres Helm release, not this chart) supplying the connection details. |
| `backend.postgres.secretKeys.*` | Key names inside that Secret (`host`/`port`/`dbname`/`username`/`password`) — default to CloudNativePG's own connection-Secret convention; override per key if yours differs. |
| `backend.adminSecretName` | Optional Secret (`username`/`password`) for HTTP Basic Auth on privileged endpoints; if absent, those endpoints just respond `503` and the rest of the API still works. |

## Resources created

| Template | Resources |
| --- | --- |
| `templates/pvc.yaml` | `frontend` PVC; `<backend.name>-code` PVC (if `backend.enabled`) |
| `templates/configmap.yaml` | `frontend-conf` (nginx config + security headers); `frontend-extra-env` (only if `frontend.env` set) |
| `templates/deployment.yaml` | `frontend` Deployment (nginx); `<backend.name>` Deployment (Node + nodemon, if `backend.enabled`) |
| `templates/service.yaml` | `<frontend.name>-service`; `<backend.name>` Service (if `backend.enabled`) |
| `templates/ingress.yaml` | `nginx-redirect-ingress-to-https`, `nginx-ingress` (with `/api` + `/` paths), `force-https` Traefik `Middleware` |

Path matching note: Traefik always routes by longest matching prefix
regardless of the order paths appear in the manifest, so `/api/*` reliably
wins over `/` for anything under the API prefix.

## Adopting existing resources into a release

If any of these objects already exist in the cluster from before this chart
managed them (e.g. created by hand with `kubectl apply`), `helm upgrade` will
refuse to touch them until you adopt them into the release:

```bash
kubectl annotate <kind> <name> meta.helm.sh/release-name=<release-name> meta.helm.sh/release-namespace=<namespace>
kubectl label <kind> <name> app.kubernetes.io/managed-by=Helm
```

## Notes when adapting this chart for a different app

- If your backend has compiled/native dependencies, the `kubectl cp`
  code-push pattern doesn't carry over safely — a `node_modules` built on
  your dev machine won't match the pod's OS/arch. You'd need to build and
  push a real image for that service instead.
- `frontend.persistence.storageClassName` / `backend.persistence.storageClassName`
  must name a StorageClass that exists on the target cluster — there's no
  default StorageClass assumed.
- The nginx config in `files/default.conf` is written for a static Angular
  SPA/prerendered build (immutable caching for hashed JS/CSS, SPA fallback to
  a CSR shell) — adjust it if you're serving something else.
