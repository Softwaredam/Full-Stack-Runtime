# Full Stack Runtime

A runtime environment for the complete application stack — frontend and
API/backend — with PostgreSQL assumed to be provided externally. It's
designed to be dropped into any Kubernetes cluster reachable only via
`kubectl` — no container registry, no image build step, and no CI/CD
pipeline required for ordinary code changes.

**The runtime is immutable; the application is not.** `helm install`/`helm
upgrade` provisions and configures the runtime itself — Deployments running
stock, unmodified public images, Services, Ingress, PersistentVolumeClaims,
ConfigMaps — and is only needed again when *that* changes. The application
running inside it — frontend build output, backend source — lives on a PVC
and is pushed straight onto the running pod with `kubectl cp`, independently
of the chart, with no image to build or publish. Immutable runtime, mutable
app.

This README is meant to be a complete, standalone operating manual — for a
human or an AI agent. Everything needed to stand up the runtime, deploy a
real application onto it, and know exactly what does and doesn't survive a
restart is below; nothing else needs to be read first. If you're an agent
asked to "spin up a demo full-stack app," **Quickstart** below is directly
executable.

## Architecture

| | Frontend | Backend (optional) |
| --- | --- | --- |
| Image | `nginxinc/nginx-unprivileged` (stock, unmodified) | `node:*-alpine` (stock, unmodified) |
| App code lives on | PVC named `frontend.name` | PVC named `<backend.name>-code` |
| Mounted at | `/usr/share/nginx/html` | `/app` |
| Process supervisor | nginx itself | `nodemon`, watching `/app` |
| Required entry point | any static files nginx can serve | `/app/index.js` — this exact filename |
| Port | `frontend.containerPort` (default `8080`) | `backend.containerPort` (default `3000`, read from `$PORT`) |

Also deployed: two Traefik `Ingress` objects (a plain-HTTP → HTTPS redirect,
and the TLS-terminating one — cert issued via cert-manager if
`ingress.tls.enabled`), one Traefik `Middleware` (`force-https`), a
`Service` per component, and a ConfigMap carrying the nginx config
(`files/default.conf` + `files/security-headers.conf`). `/api` (if the
backend is enabled) and `/` route off the same Ingress host. Full object
list in **Resources created** below.

Nothing in `templates/` needs editing to deploy a different app or site —
every site-specific value (hostname, image tags, resource limits, storage
class, replica counts, whether the backend exists at all, secret names)
lives in `values.yaml`.

## Requirements

- A Kubernetes cluster with the **Traefik** ingress controller (the chart's
  `Ingress` annotations and `Middleware` CRD are Traefik-specific).
- **cert-manager** with a `ClusterIssuer` already configured — only needed
  if `ingress.tls.enabled` (the default; set it `false` to skip both).
- A **StorageClass** that can satisfy `ReadWriteOnce` PVCs (`values.yaml`
  defaults assume one named `retain-local` — override
  `frontend.persistence.storageClassName` / `backend.persistence.storageClassName`
  to match your cluster; `kubectl get storageclass` lists what's available).
- `helm` 3.x and `kubectl`, both pointed at the target cluster (check with
  `kubectl config current-context`).

## Quickstart: deploy a demo full-stack app from nothing

The complete loop — runtime up, a trivial app running on it, verified
live — assuming nothing beyond `kubectl`/`helm` pointed at a cluster that
meets the requirements above.

### 1. Install the runtime

```bash
NAMESPACE=demo
RELEASE=demo
HOST=demo.<your-cluster-domain>   # must resolve to this cluster's Ingress

helm install "$RELEASE" . -n "$NAMESPACE" --create-namespace \
  --set ingress.host="$HOST" \
  --set backend.postgres.enabled=false   # this demo backend has no database
```

If the cluster has no cert-manager/`ClusterIssuer`, also pass `--set
ingress.tls.enabled=false` or the Ingress will never get a certificate.

Both PVCs start **empty**: the frontend's `/` returns 404 and the backend
pod loops printing "waiting for code" until step 4. Expected, not a
failure — see **Persistence & lifecycle**.

### 2. Scaffold a minimal frontend

```bash
mkdir -p demo-app/frontend
cat > demo-app/frontend/index.html <<'EOF'
<!doctype html>
<html><body><h1>Hello from the Full Stack Runtime frontend</h1></body></html>
EOF
# nginx falls back to this exact filename for any path that isn't a real
# file on disk (see "SPA fallback filename" under Trade-offs) — for this
# single-page demo it's just a copy of index.html.
cp demo-app/frontend/index.html demo-app/frontend/index.csr.html
```

### 3. Scaffold a minimal backend

```bash
mkdir -p demo-app/backend
cat > demo-app/backend/package.json <<'EOF'
{
  "name": "demo-backend",
  "type": "module",
  "main": "index.js",
  "dependencies": {
    "express": "^4.19.2",
    "nodemon": "^3.1.4"
  }
}
EOF
cat > demo-app/backend/index.js <<'EOF'
import express from 'express';
const app = express();
// This exact path is hardcoded as the pod's readiness/liveness probe (see
// Trade-offs) — the backend never receives traffic without it.
app.get('/api/health', (_req, res) => res.sendStatus(200));
app.get('/api/hello', (_req, res) => res.json({ message: 'Hello from the backend' }));
const port = process.env.PORT || 3000;
app.listen(port, '0.0.0.0', () => console.log(`listening on ${port}`));
EOF
(cd demo-app/backend && npm install --omit=dev --omit=optional)
```

Two easy-to-miss requirements baked into that snippet:
- `nodemon` must be a regular `dependencies` entry, not `devDependencies` —
  a production install (`--omit=dev`) would otherwise strip the exact
  binary the pod's entrypoint runs (`/app/node_modules/.bin/nodemon`).
- The app must bind `0.0.0.0`, not `localhost` — the kubelet's readiness/
  liveness probes hit the pod's IP directly, not loopback.

### 4. Push both onto the running pods

```bash
FRONTEND_POD=$(kubectl get pod -n "$NAMESPACE" -l app=frontend -o jsonpath='{.items[0].metadata.name}')
BACKEND_POD=$(kubectl get pod -n "$NAMESPACE" -l app=api-backend -o jsonpath='{.items[0].metadata.name}')

kubectl cp demo-app/frontend/. "$NAMESPACE/$FRONTEND_POD:/usr/share/nginx/html"
kubectl cp demo-app/backend/.  "$NAMESPACE/$BACKEND_POD:/app"
```

nginx serves the new frontend files immediately. The backend takes a couple
of seconds: `nodemon` notices `/app/index.js` appear and starts the process
(first deploy) or restarts it (every deploy after) — watch it happen with
`kubectl logs -n "$NAMESPACE" "$BACKEND_POD" -f`.

### 5. Verify

```bash
curl -s https://"$HOST"/            # the frontend HTML
curl -s https://"$HOST"/api/hello   # {"message":"Hello from the backend"}
```

No public DNS for `$HOST` yet (e.g. a local kind/minikube cluster)? Skip
Ingress entirely:

```bash
kubectl port-forward -n "$NAMESPACE" svc/frontend-service 8080:8080 &
kubectl port-forward -n "$NAMESPACE" svc/api-backend 3000:3000 &
curl -s localhost:8080/
curl -s localhost:3000/api/hello
```

That's the whole loop. Everything past this point is the same two
`kubectl cp` commands, repeated as the app changes — see **Iterating on a
real app**.

## Persistence & lifecycle

**Survives a pod restart or reschedule:** everything on the PVCs. Both
`frontend` and `<backend.name>-code` are `ReadWriteOnce` PersistentVolumeClaims,
and both Deployments use `strategy: Recreate` (not `RollingUpdate`) —
required because an RWO volume can't attach to a second pod before the
first releases it. When a pod is deleted, rescheduled, or crashes, the new
pod mounts the *same* PVC and finds the *same* files — deployed frontend
build output, backend source + `node_modules` — no re-`kubectl cp` needed.
Only in-memory application state is lost, exactly like any other process
restart.

**Survives `helm uninstall` / `helm delete`:** the PVCs, and therefore
everything on them. Both carry a `helm.sh/resource-policy: keep` annotation
(`templates/pvc.yaml`), which tells Helm to skip deleting them when the
release goes away — `helm uninstall` prints "These resources were kept due
to the resource policy" as confirmation. Reinstalling under the *same*
release name and namespace re-adopts them automatically, since their
existing `meta.helm.sh/release-name`/`release-namespace` annotations already
match the new install — no manual `kubectl annotate` needed, a plain `helm
install` after the uninstall is enough. The backend resumes straight into
`nodemon` without ever re-entering the "waiting for code" bootstrap loop,
since `/app/index.js` was never actually gone.

**Does *not* survive:** deleting the PVC object directly (`kubectl delete
pvc ...`), or a StorageClass whose reclaim policy is `Delete` rather than
`Retain` — that's a property of the StorageClass on your cluster, not
something this chart controls (check with `kubectl get storageclass
<name> -o jsonpath='{.reclaimPolicy}'`). Either one actually destroys the
underlying volume and its data.

## Iterating on a real app

Same two commands as the quickstart, generalized:

- **Frontend**: build your app locally into a static output directory,
  then:
  ```bash
  kubectl cp <build-output-dir>/. "$NAMESPACE/$FRONTEND_POD:/usr/share/nginx/html"
  ```
- **Backend**: install a clean, Linux-safe `node_modules` locally —
  `npm ci --omit=dev --omit=optional` (the second flag drops
  platform-specific optional natives, e.g. `fsevents` on macOS, which have
  no business on the Linux pod) — then:
  ```bash
  kubectl cp <server-dir>/. "$NAMESPACE/$BACKEND_POD:/app"
  ```

`$FRONTEND_POD`/`$BACKEND_POD` change on every pod recreate — re-derive them
with the `kubectl get pod -l app=... -o jsonpath=...` one-liner from step 4
rather than hardcoding a pod name.

`helm upgrade` only re-enters the picture when *infrastructure* changes: a
new env var, a resource limit, a different hostname, flipping
`backend.enabled`, bumping a base image tag, editing the nginx config.
Every ordinary feature/bugfix iteration on the app itself is a `kubectl cp`,
live in seconds — no image build, no registry, no pipeline.

## Why this shape

Two real constraints this design is built around:

1. **No registry access, no SSH to the node.** On a cluster reachable only
   through `kubectl` (no SSH, no push access to a registry — true of many
   small/managed clusters), there's nowhere to push a custom image and no
   way to place one on the node directly. A stock public image with your
   own code mounted over a PVC sidesteps both problems.
2. **Pure-JS / no compiled dependencies.** The `kubectl cp`-a-`node_modules`
   trick only works when the backend's dependencies have no native
   bindings — a `node_modules` installed on your dev machine runs
   unmodified inside the Linux pod. A backend needing compiled/native
   modules would need a real image build instead for that one service; the
   runtime's immutable-infra/mutable-app split stays the same idea, it just
   isn't reachable through `kubectl cp` there.

For an agent with shell access and a working kubeconfig but no CI system, no
registry login, and no pipeline to write or trigger, this is the entire
deploy surface: `helm install` once, `kubectl cp` for every change after
that — no separate "deploy" step to design, no image tag to bump and wait
on, no pipeline run to poll. The change is live as soon as the copy
finishes.

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
| `backend.enabled` | Set `false` to omit the backend entirely (Deployment/Service/PVC/`/api` Ingress path) — a static-only site. |
| `backend.name`, `.apiPathPrefix` | Backend base name and the Ingress path prefix routed to it. |
| `backend.image.*`, `.containerPort`, `.servicePort` | Base Node image/tag and ports — not an app-specific image. |
| `backend.persistence.size` / `.storageClassName` | Backend code PVC. |
| `backend.postgres.enabled` | Set `false` to skip wiring `PGHOST`/`PGPORT`/`PGDATABASE`/`PGUSER`/`PGPASSWORD` entirely — for a backend with no Postgres dependency, or one whose DB connection you'd rather wire yourself via `backend.env`. |
| `backend.postgres.secretName` | Existing Secret (provisioned elsewhere — e.g. a CloudNativePG cluster or a separate Postgres Helm release, not this chart) supplying the connection details. |
| `backend.postgres.secretKeys.*` | Key names inside that Secret (`host`/`port`/`dbname`/`username`/`password`) — default to CloudNativePG's own connection-Secret convention; override per key if yours differs. |
| `backend.adminSecretName` | Optional Secret (`username`/`password`) for HTTP Basic Auth on privileged endpoints; if absent, those endpoints just respond `503` and the rest of the API still works. |

## Resources created

| Template | Resources |
| --- | --- |
| `templates/pvc.yaml` | `frontend` PVC; `<backend.name>-code` PVC (if `backend.enabled`) — both annotated `helm.sh/resource-policy: keep` |
| `templates/configmap.yaml` | `frontend-conf` (nginx config + security headers); `frontend-extra-env` (only if `frontend.env` set) |
| `templates/deployment.yaml` | `frontend` Deployment (nginx); `<backend.name>` Deployment (Node + nodemon, if `backend.enabled`) |
| `templates/service.yaml` | `<frontend.name>-service`; `<backend.name>` Service (if `backend.enabled`) |
| `templates/ingress.yaml` | `nginx-redirect-ingress-to-https`, `nginx-ingress` (with `/api` + `/` paths), `force-https` Traefik `Middleware` |

Path matching note: Traefik always routes by longest matching prefix
regardless of the order paths appear in the manifest, so `/api/*` reliably
wins over `/` for anything under the API prefix.

## Trade-offs & constraints

- **Traefik-specific.** `Ingress` annotations and the `Middleware` CRD won't
  work against nginx-ingress, ALB, or other controllers without rewriting
  `templates/ingress.yaml`.
- **No horizontal scaling past one replica per component.** Both PVCs are
  `ReadWriteOnce`; a second replica can't mount the same volume, so bumping
  `frontend.replicas`/`backend.replicas` above `1` will leave extra pods
  stuck unable to attach the volume, not silently load-balanced.
- **SPA fallback filename is fixed.** `files/default.conf`'s `try_files`
  falls back to `/index.csr.html` specifically (an Angular convention this
  chart was originally built around), not a generic `index.html`. A
  single-page app with client-side routing needs a file at exactly that
  path for deep links to resolve instead of 404ing.
- **The backend health-check path is hardcoded.** Both probes hit
  `/api/health` literally, regardless of `backend.apiPathPrefix` — your app
  must implement `GET /api/health` returning 2xx, or the pod never becomes
  Ready and the Service never sends it traffic (the container keeps
  running regardless; only routing is affected).
- **No compiled/native backend dependencies.** See **Why this shape** — a
  `node_modules` built on your dev machine must run unmodified on Linux.
- **First deploy (or a from-scratch PVC) needs a manual `kubectl cp`.**
  `helm install` alone leaves both PVCs empty; nothing serves real content
  until step 4 of the Quickstart runs once.

## Adopting existing resources into a release

If any of these objects already exist in the cluster from before this chart
managed them (e.g. created by hand with `kubectl apply`), `helm upgrade`
will refuse to touch them until you adopt them into the release:

```bash
kubectl annotate <kind> <name> meta.helm.sh/release-name=<release-name> meta.helm.sh/release-namespace=<namespace>
kubectl label <kind> <name> app.kubernetes.io/managed-by=Helm
```

(This is not needed for the PVCs after an `helm uninstall` — see
**Persistence & lifecycle**; their ownership annotations already survive
intact.)
