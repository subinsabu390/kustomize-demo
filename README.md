# Kustomization Demo — Environment‑Aware Kubernetes Manifests

> A portfolio‑friendly, hands‑on demo of **Kustomize** showing how to manage reusable Kubernetes manifests with environment overlays (`dev`, `staging`, `prod`) — no templating, just pure YAML.

---

## 🎯 Goal of this Repo

- Explain **what Kustomize is** and **why to use it**.
- Provide a **clean folder layout** (base + overlays) you can copy‑paste.
- Offer **ready‑to‑run commands** to *see* the benefits: build, diff, switch environments, change images, generate config.
- Be **GitOps‑ready** (can be used later with Argo CD/Flux in a separate repo).

---

## 🧠 What is Kustomize (in a nutshell)?

Kustomize is a Kubernetes‑native configuration tool that lets you **customize YAML** without copying or templating it. You keep a **`base/`** with common manifests and layer **`overlays/`** (dev, staging, prod) that patch/extend the base. It’s built into `kubectl`, so you can apply an overlay with:

```bash
kubectl apply -k overlays/dev
```

**Key ideas**

- **No templates** — just YAML + overlays.
- **DRY** manifests — reuse the same base across environments.
- **Deterministic builds** — `kustomize build` produces final YAML.
- **Native in kubectl** — no extra runtime in your cluster.

---

## 📁 Repository Structure (suggested)

```text
kustomization-demo/
├─ base/
│  ├─ deployment.yaml
│  ├─ service.yaml
│  └─ kustomization.yaml
├─ overlays/
│  ├─ dev/
│  │  ├─ kustomization.yaml
│  │  └─ patch-replicas.yaml
│  ├─ staging/
│  │  ├─ kustomization.yaml
│  │  └─ patch-resources.yaml
│  └─ prod/
│     ├─ kustomization.yaml
│     ├─ hpa.yaml
│     └─ patch-image-tag.yaml
└─ README.md
```

### `base/` — shared manifests

**`base/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: demo-app
          image: nginx:1.25
          ports:
            - containerPort: 80
```

**`base/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-app
spec:
  type: ClusterIP
  selector:
    app: demo-app
  ports:
    - port: 80
      targetPort: 80
```

**`base/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  app.kubernetes.io/name: demo-app
  app.kubernetes.io/part-of: kustomization-demo
```

---

## 🔀 Overlays — environment differences

### 1) `overlays/dev/` — fast inner‑loop

**`overlays/dev/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: dev-
namespace: demo-dev

patches:
  - path: patch-replicas.yaml
    target:
      kind: Deployment
      name: demo-app

configMapGenerator:
  - name: app-config
    literals:
      - APP_ENV=dev
      - WELCOME_MESSAGE=hello-from-dev

generatorOptions:
  disableNameSuffixHash: false
```

**`overlays/dev/patch-replicas.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  replicas: 1
```

> **What this shows:** small replica count, separate namespace, name prefix, and a generated ConfigMap (with hash so Pods roll when config changes).

---

### 2) `overlays/staging/` — closer to prod, resource tuning

**`overlays/staging/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: stg-
namespace: demo-staging

patches:
  - path: patch-resources.yaml
    target:
      kind: Deployment
      name: demo-app

configMapGenerator:
  - name: app-config
    literals:
      - APP_ENV=staging
```

**`overlays/staging/patch-resources.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  template:
    spec:
      containers:
        - name: demo-app
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "300m"
              memory: "256Mi"
```

> **What this shows:** resource requests/limits and staged namespace/prefix. Kustomize updates references to generated names automatically.

---

### 3) `overlays/prod/` — scaling + HPA + image pinning

**`overlays/prod/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - hpa.yaml

namePrefix: prod-
namespace: demo-prod

# Pin or override images per environment
images:
  - name: nginx
    newTag: "1.27.2"

patches:
  - path: patch-image-tag.yaml
    target:
      kind: Deployment
      name: demo-app

configMapGenerator:
  - name: app-config
    literals:
      - APP_ENV=prod
```

**`overlays/prod/hpa.yaml`**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: demo-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: demo-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

**`overlays/prod/patch-image-tag.yaml`** (optional explicit patch)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  template:
    spec:
      containers:
        - name: demo-app
          image: nginx:1.27.2
```

> **What this shows:** prod namespace/prefix, **HPA**, and **image overrides**. You can control rollout strategy/replicas via patches as needed.

---

## 🧪 Commands That Show the Advantage of Kustomize

> You can run these from the repo root after adding the files as shown above.

### 0) Prereqs
- `kubectl` v1.14+ (Kustomize is built in). Optional: `kustomize` CLI for `kustomize build`.
- A Kubernetes cluster context (kind/minikube/AKS/etc.) if you want to actually apply.

### 1) See the rendered YAML (no cluster needed)
```bash
# Build the dev overlay into plain manifests
kubectl kustomize overlays/dev

# Or with the standalone CLI
kustomize build overlays/dev
```

### 2) Compare environments quickly
```bash
# Save builds to files
kubectl kustomize overlays/dev > /tmp/dev.yaml
kubectl kustomize overlays/staging > /tmp/stg.yaml
kubectl kustomize overlays/prod > /tmp/prod.yaml

# Diff them to see what changes across envs
diff -u /tmp/dev.yaml /tmp/stg.yaml | less
diff -u /tmp/stg.yaml /tmp/prod.yaml | less
```

### 3) Dry‑run / Diff against the cluster
```bash
# What would change if we applied dev?
kubectl diff -k overlays/dev

# Server‑side dry run (no changes are persisted)
kubectl apply --server-dry-run=server -k overlays/dev
```

### 4) Apply and switch environments
```bash
# Deploy dev
kubectl apply -k overlays/dev
kubectl get deploy,svc -n demo-dev

# Switch to staging
kubectl apply -k overlays/staging
kubectl get deploy,svc -n demo-staging

# Then prod
kubectl apply -k overlays/prod
kubectl get deploy,hpa -n demo-prod
```

### 5) Roll changes just by editing config
```bash
# Update a value in a ConfigMap generator (e.g., WELCOME_MESSAGE)
# Re-apply the overlay; Kustomize will create a new ConfigMap name with a hash,
# and update references so Pods roll automatically.
kubectl apply -k overlays/dev
```

### 6) Bump the container image without touching YAML
```bash
# From inside overlays/prod (or repo root using -k flag paths)
kustomize edit set image nginx=nginx:1.28.0

# Verify what would be applied
kustomize build . | grep image:

# Apply
kubectl apply -k overlays/prod
```

> **Why this is powerful:** You keep a clean base and declaratively express environment differences. No string templating, fewer merge conflicts, easy diffs, and Git‑friendly reviews.

---

## 🧩 Common Kustomize Features You Can Add

- **`commonLabels` / `commonAnnotations`**: Attach metadata across all resources.
- **`namePrefix` / `nameSuffix`**: Avoid name clashes per environment.
- **`namespace`**: Place resources into the right namespace per overlay.
- **`configMapGenerator` / `secretGenerator`**: Generate from literals or files.
  - ⚠️ Do **not** commit raw secrets. Prefer External Secrets / Sealed Secrets in real setups.
- **`images`**: Override tags/digests per environment.
- **`patches`** (strategic merge) and **`patchesJson6902`** (JSON 6902) for precise changes.
- **Components** (advanced): reusable “mix‑ins” like `observability/` or `ingress/` you can include across overlays.

**Example: JSON 6902 patch to change Service type**

_`overlays/staging/patch-service-type.yaml`_
```yaml
- op: replace
  path: /spec/type
  value: LoadBalancer
```
_`overlays/staging/kustomization.yaml`_
```yaml
patchesJson6902:
  - target:
      version: v1
      kind: Service
      name: demo-app
    path: patch-service-type.yaml
```

---

## 🛠️ CI Idea (Optional)

Validate that overlays always build successfully:

```yaml
# .github/workflows/kustomize-check.yml
name: Kustomize Build Check
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install kustomize
        run: |
          curl -sSLo kustomize.tar.gz https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.4.2/kustomize_v5.4.2_linux_amd64.tar.gz
          tar -xzf kustomize.tar.gz && sudo mv kustomize /usr/local/bin/
      - name: Build overlays
        run: |
          for d in overlays/*; do
            echo "Building $d"
            kustomize build "$d" >/dev/null
          done
```

This catches broken patches or schema mismatches early.

---

## 🔐 Security Notes

- Don’t commit real secrets. Use generators only for **non‑sensitive** defaults.
- Consider **External Secrets Operator** or **Sealed Secrets** in production.
- Pin images in prod to tags or digests to ensure repeatable deploys.

---

## 🧯 Troubleshooting

- **“patches failed to find target”** → Check `kind`, `name`, and `apiVersion` in the patch target.
- **Fields not patched** → Make sure patch path matches actual YAML structure and container name.
- **Generator not rolling pods** → Ensure you reference the generated ConfigMap/Secret by name in Deployment; Kustomize will update references when the name hash changes.
- **Different kubectl/kustomize versions** → If you use the standalone CLI, align versions with CI to avoid drift.

---

## 📚 Why this repo is useful in a portfolio

- Shows **clean structure** (base + overlays).
- Demonstrates **practical features** (patches, generators, images, HPA).
- Provides **runnable commands** to *see* the value of Kustomize quickly.
- Ready to plug into a **GitOps tool** later (Argo CD/Flux) without changing the layout.

---

## 🧹 Cleanup

```bash
kubectl delete -k overlays/dev
kubectl delete -k overlays/staging
kubectl delete -k overlays/prod
```

---

### License

MIT (or your choice)
