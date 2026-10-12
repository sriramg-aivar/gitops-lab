=# Lab Workbook: CGOA + CAPA (local kind cluster)

How to use this file:

- **Part A** is the one-time setup. Do it once, top to bottom.
- **Part B** has CGOA labs (G1 to G10). **Part C** has CAPA labs (C1 to C12). Each lab says which exam topic it practises.
- Read the matching topic in `README-study-content.md` first, then do the lab here.
- Every lab has: **Goal**, **Do**, **Expect** (what you should see) and **Say it** (one question to answer in your own words).

Everything runs on your laptop and costs nothing.

---

# PART A: One-time setup

## A0. What you will have at the end

| Tool | Namespace | How you open it |
| --- | --- | --- |
| Argo CD | `argocd` | https://localhost:8080 |
| Argo Workflows | `argo` | https://localhost:2746 |
| Argo Rollouts | `argo-rollouts` | `kubectl argo rollouts dashboard`, then http://localhost:3100 |
| Argo Events | `argo-events` | no UI, use `kubectl` |
| Flux (CGOA lab G8 only) | `flux-system` | no UI, use the `flux` CLI |

## A1. Requirements

- Docker running (Docker Desktop, or Docker Engine on Linux). Give it at least 4 GB RAM, 8 GB is better.
- A free GitHub account.
- Linux or macOS terminal. On Windows use WSL2 with Ubuntu and run everything inside it.
- Internet access, because manifests and images are downloaded.

## A2. Install the tools

### macOS (Homebrew)

```bash
brew install kubectl kind argocd argo git
brew install argoproj/tap/kubectl-argo-rollouts
brew install fluxcd/tap/flux
```

### Linux / WSL2

```bash
# kubectl and kind: follow the official pages, they are one-time installs
#   https://kubernetes.io/docs/tasks/tools/
#   https://kind.sigs.k8s.io/docs/user/quick-start/

# Argo CD CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd /usr/local/bin/argocd && rm argocd

# Argo Workflows CLI
ARGO_WF_VERSION=$(curl -s https://api.github.com/repos/argoproj/argo-workflows/releases/latest | grep -m1 '"tag_name"' | cut -d'"' -f4)
echo $ARGO_WF_VERSION
curl -sLO https://github.com/argoproj/argo-workflows/releases/download/$ARGO_WF_VERSION/argo-linux-amd64.gz
gunzip argo-linux-amd64.gz && chmod +x argo-linux-amd64
sudo mv argo-linux-amd64 /usr/local/bin/argo

# Argo Rollouts kubectl plugin
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

# Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash
```

### Check everything is installed

```bash
docker version --format '{{.Server.Version}}'
kubectl version --client
kind version
argocd version --client
argo version
kubectl argo rollouts version
flux --version
```

If one command fails, fix it before moving on.

## A3. Working folder and cluster

```bash
mkdir -p ~/gitops-labs && cd ~/gitops-labs
mkdir -p argocd workflows rollouts events notifications
export GH_USER=your-github-username     # <- change this, and run it again in every new terminal

cat > kind-config.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

kind create cluster --name gitops --config kind-config.yaml
kubectl cluster-info --context kind-gitops
kubectl get nodes
```

You should see 3 nodes `Ready`. If your laptop is low on memory, use one node: `kind create cluster --name gitops`.

## A4. Git repositories (the GitOps "state store")

You need two public repos on GitHub (public, so Argo CD needs no credentials):

1. **Fork** https://github.com/argoproj/argocd-example-apps to your account. It provides the `guestbook` app.
2. **Create a new empty public repo** called `gitops-lab`.

Clone it and add all the demo content in one go:

```bash
cd ~/gitops-labs
git clone https://github.com/$GH_USER/gitops-lab.git
cd gitops-lab
mkdir -p waves helm-app/templates kustomize-app/base kustomize-app/overlays/dev kustomize-app/overlays/prod apps
```

**waves/** (sync waves and hooks):

```bash
cat > waves/config.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  annotations:
    argocd.argoproj.io/sync-wave: "0"
data:
  mode: demo
EOF

cat > waves/migrate.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: pre-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: alpine:3.19
          command: [sh, -c, "echo migrating"]
EOF

cat > waves/web.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27
EOF
```

**helm-app/** (a tiny Helm chart, no Helm CLI needed because Argo CD renders it):

```bash
cat > helm-app/Chart.yaml <<'EOF'
apiVersion: v2
name: helm-app
description: Demo chart
type: application
version: 0.1.0
appVersion: "1.0"
EOF

cat > helm-app/values.yaml <<'EOF'
replicaCount: 1
image:
  repository: nginx
  tag: "1.27"
message: hello
EOF

cat > helm-app/values-prod.yaml <<'EOF'
replicaCount: 2
message: prod
EOF

cat > helm-app/templates/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-web
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-web
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-web
    spec:
      containers:
        - name: web
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          env:
            - name: MESSAGE
              value: {{ .Values.message | quote }}
EOF
```

**kustomize-app/** (base plus dev and prod overlays):

```bash
cat > kustomize-app/base/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kweb
  template:
    metadata:
      labels:
        app: kweb
    spec:
      containers:
        - name: web
          image: nginx
EOF

cat > kustomize-app/base/kustomization.yaml <<'EOF'
resources:
  - deployment.yaml
EOF

cat > kustomize-app/overlays/dev/kustomization.yaml <<'EOF'
resources:
  - ../../base
namePrefix: dev-
images:
  - name: nginx
    newTag: "1.27"
EOF

cat > kustomize-app/overlays/prod/kustomization.yaml <<'EOF'
resources:
  - ../../base
namePrefix: prod-
replicas:
  - name: web
    count: 2
images:
  - name: nginx
    newTag: "1.26"
EOF
```

**apps/** (child Applications for the app-of-apps lab; unquoted `EOF` so `$GH_USER` is filled in):

```bash
cat > apps/child-helm.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: child-helm
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/$GH_USER/gitops-lab.git
    targetRevision: HEAD
    path: helm-app
  destination:
    server: https://kubernetes.default.svc
    namespace: child-helm
  syncPolicy:
    automated: {}
    syncOptions:
      - CreateNamespace=true
EOF

cat > apps/child-waves.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: child-waves
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/$GH_USER/gitops-lab.git
    targetRevision: HEAD
    path: waves
  destination:
    server: https://kubernetes.default.svc
    namespace: child-waves
  syncPolicy:
    automated: {}
    syncOptions:
      - CreateNamespace=true
EOF
```

Push everything:

```bash
git add . && git commit -m "lab content" && git push
cd ~/gitops-labs
```

If `git push` asks for a password, use a GitHub personal access token as the password.

## A5. Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server --timeout=300s
kubectl -n argocd get pods
```

All pods should be `Running`. `--server-side` is required because the CRDs are very large.

Get the admin password and open the UI (leave this terminal running):

```bash
export ARGOCD_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo $ARGOCD_PWD
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Open https://localhost:8080 (accept the certificate warning). User is `admin`. In a **new terminal** log in with the CLI:

```bash
cd ~/gitops-labs && export GH_USER=your-github-username
export ARGOCD_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
argocd login localhost:8080 --username admin --password "$ARGOCD_PWD" --insecure
argocd cluster list
```

Only if a repo is private: `argocd repo add https://github.com/$GH_USER/gitops-lab.git --username $GH_USER --password <token>`.

## A6. Install Argo Workflows

```bash
ARGO_WF_VERSION=$(curl -s https://api.github.com/repos/argoproj/argo-workflows/releases/latest | grep -m1 '"tag_name"' | cut -d'"' -f4)
echo $ARGO_WF_VERSION
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/$ARGO_WF_VERSION/quick-start-minimal.yaml
kubectl -n argo rollout status deploy/workflow-controller --timeout=300s
kubectl -n argo get pods
```

The quick-start also installs MinIO (the artifact store) and the Argo Server. UI, in a **new terminal**:

```bash
kubectl -n argo port-forward deployment/argo-server 2746:2746
```

Open https://localhost:2746. The quick-start runs without a login.

## A7. Install Argo Rollouts

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
kubectl -n argo-rollouts rollout status deploy/argo-rollouts --timeout=300s
kubectl argo rollouts version
```

## A8. Install Argo Events

```bash
kubectl create namespace argo-events
kubectl apply -f https://github.com/argoproj/argo-events/releases/latest/download/install.yaml
kubectl -n argo-events rollout status deploy/controller-manager --timeout=300s
kubectl -n argo-events get pods
```

Create the EventBus (the message bus) from our own file, using NATS JetStream:

```bash
cat > events/eventbus.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: EventBus
metadata:
  name: default
  namespace: argo-events
spec:
  jetstream:
    version: latest
EOF

kubectl apply -f events/eventbus.yaml
kubectl -n argo-events get eventbus
kubectl -n argo-events get pods      # wait for the eventbus-default-js-* pods to be Running
```

If the controller deployment name differs, run `kubectl -n argo-events get deploy` and wait for the one that is there.

## A9. Quick health check

```bash
kubectl get pods -A | grep -v kube-system
```

Everything in `argocd`, `argo`, `argo-rollouts` and `argo-events` should be `Running`. Setup is done. Now pick a lab.

---

# PART B: CGOA labs

## G1. First GitOps deployment

**Topics:** Declarative description, desired state, state store, GitOps principles, managed software system.

**Do:**

```bash
cat > argocd/guestbook-app.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/$GH_USER/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f argocd/guestbook-app.yaml
argocd app get guestbook          # OutOfSync, nothing applied yet
argocd app sync guestbook
argocd app get guestbook
kubectl -n guestbook get pods
```

**Expect:** the app goes from `OutOfSync` to `Synced` and `Healthy`. You never ran `kubectl apply` for the guestbook itself, the agent pulled it from Git.

**Say it:** where is the desired state stored, which part is the state store, and which part is the software agent?

## G2. Drift and reconciliation (feedback loop)

**Topics:** State drift, state reconciliation, continuously reconciled, feedback loop.

**Do:**

```bash
cat > argocd/guestbook-app-selfheal.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/$GH_USER/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

# Part 1: without self-heal (the app from G1)
kubectl -n guestbook scale deploy guestbook-ui --replicas=5
argocd app get guestbook --refresh        # OutOfSync, replicas stay at 5

# Part 2: with self-heal
kubectl apply -f argocd/guestbook-app-selfheal.yaml
kubectl -n guestbook scale deploy guestbook-ui --replicas=5
kubectl -n guestbook get deploy guestbook-ui -w      # goes back to 1 (Ctrl+C to stop)
```

**Expect:** without self-heal you see drift and nothing fixes it. With self-heal the agent reverts it within seconds.

**Say it:** which words describe step 1 (drift) and step 2 (reconciliation)? What are the three parts of the feedback loop?

## G3. Rollback and versioned history

**Topics:** Rollback, versioned and immutable, Git as audit trail.

**Do:**

```bash
cd ~/gitops-labs
git clone https://github.com/$GH_USER/argocd-example-apps.git
cd argocd-example-apps
# edit guestbook/guestbook-ui-deployment.yaml: change replicas to 3
git commit -am "scale to 3" && git push
argocd app get guestbook --refresh
kubectl -n guestbook get deploy guestbook-ui          # 3 replicas

git revert HEAD --no-edit && git push                 # this is the rollback
argocd app get guestbook --refresh
kubectl -n guestbook get deploy guestbook-ui          # back to 1
git log --oneline | head -5
cd ~/gitops-labs
```

**Expect:** the change and the rollback are both commits, and the cluster follows each one.

**Say it:** why is `git revert` a better rollback than editing the live cluster?

## G4. "Continuous": polling vs event-driven

**Topics:** Continuous, pull vs event-driven.

**Do:** (1) Make a Git change (for example, replicas to 2 in your fork), push, and time how long until Argo CD notices without touching anything. Argo CD polls Git about every 3 minutes. (2) Make another change, push, then trigger immediately: `argocd app get guestbook --refresh`. (3) Optional, shorten the poll interval:

```bash
kubectl -n argocd patch configmap argocd-cm --type merge -p '{"data":{"timeout.reconciliation":"30s"}}'
kubectl -n argocd rollout restart deployment argocd-repo-server
kubectl -n argocd rollout restart statefulset argocd-application-controller
```

**Expect:** polling is periodic, the manual refresh is the same as a webhook "event" telling the agent to check now. In production a Git webhook to Argo CD (`/api/webhook`) or a Flux Receiver does this automatically. It needs a public URL, so you only read about it here.

**Say it:** is an event-triggered sync still "pull"? (Yes: the agent still fetches from Git itself.)

## G5. Deployment and release patterns (rolling vs recreate)

**Topics:** Deployment and release patterns. Canary and blue-green are in labs C9 and C10.

**Do:**

```bash
kubectl create namespace patterns
kubectl -n patterns create deployment rolling --image=nginx:1.26 --replicas=4
kubectl -n patterns rollout status deploy/rolling

# Rolling update: watch pods in a second terminal with: kubectl -n patterns get pods -w
kubectl -n patterns set image deployment/rolling nginx=nginx:1.27
kubectl -n patterns rollout status deploy/rolling

# Switch the same Deployment to Recreate and update again
kubectl -n patterns patch deployment rolling -p '{"spec":{"strategy":{"type":"Recreate","rollingUpdate":null}}}'
kubectl -n patterns set image deployment/rolling nginx=nginx:1.26
kubectl -n patterns get pods -w
```

**Expect:** rolling replaces pods a few at a time, old and new run together. Recreate kills all old pods first, so there is a moment with no pods (downtime).

**Say it:** which strategy has downtime, and which one runs two versions at once?

## G6. Repo structure and environment promotion

**Topics:** State store management, environment promotion, packaging with Kustomize.

**Do:**

```bash
cat > argocd/kustomize-dev.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kustomize-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/$GH_USER/gitops-lab.git
    targetRevision: HEAD
    path: kustomize-app/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: kz-dev
  syncPolicy:
    automated: {}
    syncOptions:
      - CreateNamespace=true
EOF

cat > argocd/kustomize-prod.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kustomize-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/$GH_USER/gitops-lab.git
    targetRevision: HEAD
    path: kustomize-app/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: kz-prod
  syncPolicy:
    automated: {}
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f argocd/kustomize-dev.yaml -f argocd/kustomize-prod.yaml
kubectl -n kz-dev get deploy -o wide
kubectl -n kz-prod get deploy -o wide      # prod has 2 replicas and nginx 1.26
```

Promote the new version to prod through Git: in `gitops-lab`, edit `kustomize-app/overlays/prod/kustomization.yaml` and change `newTag: "1.26"` to `"1.27"`, commit and push (in real life this is a pull request).

```bash
cd ~/gitops-labs/gitops-lab
sed -i.bak 's/newTag: "1.26"/newTag: "1.27"/' kustomize-app/overlays/prod/kustomization.yaml && rm kustomize-app/overlays/prod/kustomization.yaml.bak
git commit -am "promote 1.27 to prod" && git push
cd ~/gitops-labs
argocd app get kustomize-prod --refresh
kubectl -n kz-prod get deploy -o wide
```

**Expect:** the same base is used with two overlays. Promotion is just a Git change to the next environment's folder.

**Say it:** why folders per environment on one branch instead of one long-lived branch per environment?

## G7. Secrets (why not plain Secrets in Git)

**Topics:** Secrets management pattern.

**Do:**

```bash
kubectl create secret generic db --from-literal=password=SuperSecret123 --dry-run=client -o yaml > /tmp/secret.yaml
cat /tmp/secret.yaml
grep password: /tmp/secret.yaml | awk '{print $2}' | base64 -d; echo
rm /tmp/secret.yaml
```

**Expect:** anyone who sees the YAML can decode the password. So plain Secret manifests must never be committed.

**Say it:** name three ways to keep secrets out of Git. (Sealed/encrypted secrets, an external secret store with a sync tool, encrypted files with a tool like SOPS.) You only need the idea for the exam.

## G8. Flux, a second reconciliation engine

**Topics:** Reconciliation engines, pull vs event-driven.

**Do:**

```bash
flux check --pre
flux install
flux create source git podinfo --url=https://github.com/stefanprodan/podinfo --branch=master --interval=1m
flux create kustomization podinfo --source=GitRepository/podinfo --path="./kustomize" --prune=true --interval=5m --target-namespace=default
flux get sources git
flux get kustomizations
kubectl -n default get deploy

# trigger now instead of waiting for the interval
flux reconcile source git podinfo
flux reconcile kustomization podinfo --with-source
```

**Expect:** Flux splits the work into a **source** (where to pull) and a **Kustomization** (what to apply). Argo CD uses one **Application** for both. `flux reconcile` is the manual "event".

**Say it:** two differences between Flux and Argo CD in how you describe an app.

## G9. Interoperability: notifications and observability

**Topics:** Interoperability with notifications and observability tools.

**Do, notifications:** create a small receiver inside the cluster that prints what it receives, then make Argo CD send it a webhook when an app syncs.

```bash
kubectl -n argocd create deployment echo --image=mendhak/http-https-echo
kubectl -n argocd expose deployment echo --port=8080
kubectl -n argocd rollout status deploy/echo

cat > notifications/patch.yaml <<'EOF'
data:
  service.webhook.local: |
    url: http://echo:8080
    headers:
    - name: Content-Type
      value: application/json
  template.app-synced: |
    webhook:
      local:
        method: POST
        body: |
          {"app":"{{.app.metadata.name}}","sync":"{{.app.status.sync.status}}","health":"{{.app.status.health.status}}"}
  trigger.on-synced: |
    - when: app.status.sync.status == 'Synced'
      send: [app-synced]
EOF

kubectl -n argocd patch configmap argocd-notifications-cm --type merge --patch-file notifications/patch.yaml
kubectl -n argocd annotate app guestbook notifications.argoproj.io/subscribe.on-synced.local="" --overwrite

# in one terminal, watch the receiver
kubectl -n argocd logs deploy/echo -f
# in another terminal, cause OutOfSync then Synced again (self-heal must be on, from G2)
kubectl -n guestbook scale deploy guestbook-ui --replicas=5
```

**Expect:** a JSON request with the app name and `Synced` appears in the echo logs. This is the same idea as sending the message to Slack or Teams. If nothing appears, check `kubectl -n argocd logs deploy/argocd-notifications-controller`.

**Do, observability:** Argo CD exposes Prometheus metrics.

```bash
kubectl -n argocd port-forward svc/argocd-metrics 8082:8082 &
curl -s localhost:8082/metrics | grep argocd_app_info | head
kill %1
```

**Expect:** a line per app with its sync and health status. Prometheus scrapes this, and Grafana draws dashboards from it.

**Say it:** which tools connect GitOps to notifications and to observability?

## G10. Interoperability: CI tools

**Topics:** CI vs CD, interoperability with CI tools, CI updates Git, agent deploys.

**Do:** act as the CI pipeline. CI builds an image, then updates the image tag in the config repo. You do the "update" step by hand:

```bash
cd ~/gitops-labs/gitops-lab
sed -i.bak 's/tag: "1.27"/tag: "1.26"/' helm-app/values.yaml && rm helm-app/values.yaml.bak
git commit -am "ci: bump image tag" && git push
cd ~/gitops-labs
```

If you applied the `child-helm` app (lab C8) or the Helm app (lab C7), watch it pick up the new tag with `argocd app get <name> --refresh` and `kubectl get deploy -A -o wide | grep web`.

A real pipeline step looks like this (read it, do not run it):

```yaml
# .github/workflows/ci.yaml (example only)
name: ci
on: { push: { branches: [main] } }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "build and test the app, push the image to a registry"
      - run: |
          git clone https://github.com/ORG/config-repo.git
          cd config-repo
          sed -i "s/tag: .*/tag: ${GITHUB_SHA}/" helm-app/values.yaml
          git commit -am "bump image" && git push
```

**Say it:** which part is CI, which part is CD, and does the CI job ever touch the cluster?

---

# PART C: CAPA labs

All Workflow files are applied in the `argo` namespace.

## C1. Workflow fundamentals and the Workflow spec

**Topics:** Argo Workflows fundamentals, the Workflow spec.

**Do:**

```bash
cat > workflows/spec-tour.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: spec-tour-
spec:
  entrypoint: main
  arguments:
    parameters:
      - name: who
        value: world
  parallelism: 2
  activeDeadlineSeconds: 300
  ttlStrategy:
    secondsAfterCompletion: 600
  podGC:
    strategy: OnWorkflowCompletion
  onExit: cleanup
  templates:
    - name: main
      steps:
        - - name: hello
            template: say
            arguments:
              parameters: [{name: msg, value: "hello {{workflow.parameters.who}}"}]
        - - name: flaky
            template: flaky
    - name: say
      inputs:
        parameters:
          - name: msg
      container:
        image: alpine:3.19
        command: [echo, "{{inputs.parameters.msg}}"]
    - name: flaky
      retryStrategy:
        limit: 3
      container:
        image: alpine:3.19
        command: [sh, -c, "if [ $((RANDOM % 2)) -eq 0 ]; then echo ok; else echo fail; exit 1; fi"]
    - name: cleanup
      container:
        image: alpine:3.19
        command: [echo, "exit handler ran, status {{workflow.status}}"]
EOF

argo submit -n argo --watch workflows/spec-tour.yaml
argo submit -n argo --watch workflows/spec-tour.yaml -p who=Sriram
argo list -n argo
argo get -n argo @latest
argo logs -n argo @latest
```

**Expect:** the second run prints "hello Sriram". The `flaky` step may retry. The exit handler runs at the end whatever the result. In the UI at https://localhost:2746 you can see the graph.

**Say it:** which fields did I just use: entrypoint, arguments, parallelism, activeDeadlineSeconds, ttlStrategy, podGC, onExit, retryStrategy? What does each do?

## C2. Generating and consuming artifacts

**Topics:** Artifacts: output artifacts, input artifacts, artifact repository.

**Do:**

```bash
cat > workflows/artifacts.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: artifacts-
spec:
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: generate
            template: gen-file
        - - name: consume
            template: print-file
            arguments:
              artifacts:
                - name: msg
                  from: "{{steps.generate.outputs.artifacts.out}}"
    - name: gen-file
      container:
        image: alpine:3.19
        command: [sh, -c]
        args: ["echo hello artifact > /tmp/hello.txt"]
      outputs:
        artifacts:
          - name: out
            path: /tmp/hello.txt
    - name: print-file
      inputs:
        artifacts:
          - name: msg
            path: /tmp/in.txt
      container:
        image: alpine:3.19
        command: [sh, -c]
        args: ["cat /tmp/in.txt"]
EOF

argo submit -n argo --watch workflows/artifacts.yaml
argo logs -n argo @latest
```

**Expect:** the second step prints `hello artifact`. The file went from step 1 to the artifact store (MinIO in the quick-start) and was downloaded into step 2. To look at it: `kubectl -n argo port-forward svc/minio 9001:9001`, open http://localhost:9001, login `admin` / `password`, bucket `my-bucket`.

**Say it:** when do I use a parameter and when do I use an artifact?

## C3. Workflow templates (reusable)

**Topics:** WorkflowTemplate, ClusterWorkflowTemplate, templateRef.

**Do:**

```bash
cat > workflows/workflowtemplate.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: shared-steps
spec:
  entrypoint: greet
  arguments:
    parameters:
      - name: name
        value: team
  templates:
    - name: greet
      inputs:
        parameters:
          - name: name
      container:
        image: alpine:3.19
        command: [echo, "hello {{inputs.parameters.name}} from a shared template"]
EOF

cat > workflows/uses-template.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: uses-template-
spec:
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: call
            templateRef:
              name: shared-steps
              template: greet
            arguments:
              parameters: [{name: name, value: Sriram}]
EOF

cat > workflows/clusterworkflowtemplate.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: ClusterWorkflowTemplate
metadata:
  name: cluster-shared
spec:
  templates:
    - name: hello
      container:
        image: alpine:3.19
        command: [echo, "from a cluster-wide template"]
EOF

cat > workflows/uses-cluster-template.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: uses-cluster-template-
spec:
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: call
            templateRef:
              name: cluster-shared
              template: hello
              clusterScope: true
EOF

kubectl apply -n argo -f workflows/workflowtemplate.yaml -f workflows/clusterworkflowtemplate.yaml
argo template list -n argo
argo submit -n argo --watch workflows/uses-template.yaml
argo submit -n argo --watch workflows/uses-cluster-template.yaml
argo submit -n argo --watch --from workflowtemplate/shared-steps -p name=Sriram
```

**Expect:** all three runs succeed. The last one runs a stored template directly with `--from`.

**Say it:** what is the difference between a `WorkflowTemplate` and a `ClusterWorkflowTemplate`? What does `clusterScope: true` do?

## C4. DAG (and steps)

**Topics:** Work with DAGs.

**Do:**

```bash
cat > workflows/dag.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: dag-diamond-
spec:
  entrypoint: diamond
  templates:
    - name: echo
      inputs:
        parameters:
          - name: message
      container:
        image: alpine:3.19
        command: [echo, "{{inputs.parameters.message}}"]
    - name: diamond
      dag:
        tasks:
          - name: A
            template: echo
            arguments: {parameters: [{name: message, value: A}]}
          - name: B
            dependencies: [A]
            template: echo
            arguments: {parameters: [{name: message, value: B}]}
          - name: C
            dependencies: [A]
            template: echo
            arguments: {parameters: [{name: message, value: C}]}
          - name: D
            depends: "B && C"
            template: echo
            arguments: {parameters: [{name: message, value: D}]}
EOF

cat > workflows/steps.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: steps-demo-
spec:
  entrypoint: main
  templates:
    - name: echo
      inputs:
        parameters:
          - name: message
      container:
        image: alpine:3.19
        command: [echo, "{{inputs.parameters.message}}"]
    - name: main
      steps:
        - - name: A
            template: echo
            arguments: {parameters: [{name: message, value: A}]}
        - - name: B
            template: echo
            arguments: {parameters: [{name: message, value: B}]}
          - name: C
            template: echo
            arguments: {parameters: [{name: message, value: C}]}
        - - name: D
            template: echo
            arguments: {parameters: [{name: message, value: D}]}
EOF

argo submit -n argo --watch workflows/dag.yaml
argo submit -n argo --watch workflows/steps.yaml
```

**Expect:** in both, B and C run together after A, and D waits for both. Look at the graph in the UI. Then edit `dag.yaml`, make task B `exit 1` (use a failing command), and see that D does not run. Then try `depends: "B.Succeeded || C.Succeeded"` for D.

**Say it:** how does `dependencies` differ from `depends`? When would I choose DAG over steps?

## C5. Data processing jobs (fan-out and fan-in)

**Topics:** Run data processing jobs with Argo Workflows, loops, script templates.

**Do:**

```bash
cat > workflows/data-processing.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: data-processing-
spec:
  entrypoint: main
  parallelism: 3
  templates:
    - name: main
      steps:
        - - name: generate
            template: gen-numbers
        - - name: square
            template: square
            arguments:
              parameters: [{name: n, value: "{{item}}"}]
            withParam: "{{steps.generate.outputs.result}}"
        - - name: total
            template: total
            arguments:
              parameters: [{name: results, value: "{{steps.square.outputs.result}}"}]
    - name: gen-numbers
      script:
        image: python:3.12-alpine
        command: [python]
        source: |
          import json
          print(json.dumps([1, 2, 3, 4, 5]))
    - name: square
      inputs:
        parameters:
          - name: n
      script:
        image: python:3.12-alpine
        command: [python]
        source: |
          n = int("{{inputs.parameters.n}}")
          print(n * n)
    - name: total
      inputs:
        parameters:
          - name: results
      script:
        image: python:3.12-alpine
        command: [python]
        source: |
          import json
          values = json.loads('''{{inputs.parameters.results}}''')
          print("sum of squares =", sum(int(v) for v in values))
EOF

argo submit -n argo --watch workflows/data-processing.yaml
argo logs -n argo @latest
```

**Expect:** five `square` pods run in parallel (limited to 3 at a time by `parallelism`), then `total` prints `sum of squares = 55`. This is the fan-out and fan-in (map-reduce) pattern. Try `withItems: [1,2,3]` instead of `withParam`, and `withSequence`.

**Say it:** what do `withItems`, `withParam` and `outputs.result` do? When would I use artifacts or a volume instead of parameters for big data?

## C6. Argo CD Application and synchronisation

**Topics:** Argo CD fundamentals, synchronise applications, use an Application.

**Do:**

```bash
argocd app create guestbook-cli \
  --repo https://github.com/$GH_USER/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace guestbook-cli \
  --sync-option CreateNamespace=true

argocd app get guestbook-cli
argocd app diff guestbook-cli                  # shows what a sync would change
argocd app sync guestbook-cli --dry-run
argocd app sync guestbook-cli
argocd app history guestbook-cli
argocd app set guestbook-cli --sync-policy automated --auto-prune --self-heal
argocd app set guestbook-cli --sync-policy none
argocd app rollback guestbook-cli 0            # roll back to history ID 0 (needs auto-sync off)
argocd app get guestbook-cli
argocd app delete guestbook-cli -y             # cascade delete: removes the resources too
```

In the UI open the app, click a Deployment, and look at the live manifest, the diff and the events.

**Expect:** you see the status (Synced, Healthy), the history of syncs, and rollback working only when auto-sync is off.

**Say it:** what is the difference between sync status and health status? What do `prune` and `selfHeal` each do?

## C7. Argo CD with Helm and Kustomize

**Topics:** Configure Argo CD with Helm and Kustomize.

**Do (Helm):**

```bash
cat > argocd/helm-app.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: helm-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/$GH_USER/gitops-lab.git
    targetRevision: HEAD
    path: helm-app
    helm:
      releaseName: demo
      valueFiles:
        - values-prod.yaml
      parameters:
        - name: message
          value: from-argocd
  destination:
    server: https://kubernetes.default.svc
    namespace: helm-demo
  syncPolicy:
    automated: {}
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f argocd/helm-app.yaml
kubectl -n helm-demo get deploy
kubectl -n helm-demo get deploy demo-web -o jsonpath='{.spec.replicas}{"  "}{.spec.template.spec.containers[0].env[0].value}{"\n"}'
```

**Expect:** 2 replicas (from `values-prod.yaml`) and the message `from-argocd` (the `parameters` entry overrides the values file). Argo CD runs `helm template` and applies the result; it does not make a Helm release, so `helm ls` would show nothing.

**Do (Kustomize):** you already have `kustomize-dev` and `kustomize-prod` from lab G6. Add overrides in the Application:

```bash
argocd app set kustomize-dev --kustomize-image nginx=nginx:1.25
argocd app get kustomize-dev --refresh
kubectl -n kz-dev get deploy -o wide
```

**Expect:** the image changes to 1.25 because Argo CD applied a Kustomize image override on top of the overlay (`argocd app unset` or a Git change brings it back).

**Say it:** how does Argo CD turn a Helm chart or a Kustomize folder into manifests?

## C8. Common reconciliation patterns

**Topics:** App of apps, ApplicationSet, sync waves and hooks, auto-sync, self-heal, prune, `ignoreDifferences`.

**Do, app of apps:**

```bash
cat > argocd/root-app.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/$GH_USER/gitops-lab.git
    targetRevision: HEAD
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: {}
EOF

kubectl apply -f argocd/root-app.yaml
argocd app list      # root, child-helm and child-waves appear
```

**Do, ApplicationSet with a Git directory generator** (one app per overlay folder):

```bash
cat > argocd/appset.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: kz-envs
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/$GH_USER/gitops-lab.git
        revision: HEAD
        directories:
          - path: kustomize-app/overlays/*
  template:
    metadata:
      name: 'set-{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/$GH_USER/gitops-lab.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: 'set-{{path.basename}}'
      syncPolicy:
        automated: {}
        syncOptions:
          - CreateNamespace=true
EOF

kubectl apply -f argocd/appset.yaml
argocd app list      # set-dev and set-prod appear
```

Add a new folder `kustomize-app/overlays/staging` (copy `dev`, change the prefix to `staging-`), push, and a `set-staging` app appears by itself.

**Do, waves and hooks:** the `waves` folder has a PreSync Job, wave 0 and wave 1. Watch the order:

```bash
cat > argocd/waves-app.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: waves
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/$GH_USER/gitops-lab.git
    targetRevision: HEAD
    path: waves
  destination:
    server: https://kubernetes.default.svc
    namespace: waves
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f argocd/waves-app.yaml
argocd app sync waves
```

In the UI, watch the PreSync Job first, then the ConfigMap (wave 0), then the Deployment (wave 1). Then in `waves/web.yaml` change the image to `nginx:does-not-exist`, push, sync: the app becomes **Synced** but **Degraded**.

**Do, ignoreDifferences:** add this under `spec` of `waves-app.yaml`, apply it, then scale `web` by hand:

```yaml
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

```bash
kubectl -n waves scale deploy web --replicas=3
argocd app get waves --refresh      # still Synced: the replicas difference is ignored
```

**Do, AppProject:**

```bash
cat > argocd/project-restricted.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: restricted
  namespace: argocd
spec:
  description: Only one repo and one namespace
  sourceRepos:
    - https://github.com/$GH_USER/gitops-lab.git
  destinations:
    - server: https://kubernetes.default.svc
      namespace: restricted-ns
EOF

cat > argocd/restricted-app.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: restricted-test
  namespace: argocd
spec:
  project: restricted
  source:
    repoURL: https://github.com/$GH_USER/gitops-lab.git
    targetRevision: HEAD
    path: waves
  destination:
    server: https://kubernetes.default.svc
    namespace: waves
EOF

kubectl apply -f argocd/project-restricted.yaml -f argocd/restricted-app.yaml
argocd app get restricted-test      # error: destination not permitted in project
```

**Say it:** which pattern would I use for 30 clusters? For ordering a DB migration? For ignoring changes made by an autoscaler?

## C9. Argo Rollouts: canary and blue-green

**Topics:** Rollouts fundamentals, common progressive rollout strategies.

**Do, canary:**

```bash
cat > rollouts/analysis-template.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: check
spec:
  metrics:
    - name: smoke
      count: 1
      provider:
        job:
          spec:
            backoffLimit: 0
            template:
              spec:
                restartPolicy: Never
                containers:
                  - name: smoke
                    image: alpine:3.19
                    command: [sh, -c, "exit 0"]
EOF

cat > rollouts/rollout-canary.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: demo
spec:
  replicas: 4
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
        - name: demo
          image: argoproj/rollouts-demo:blue
          ports:
            - containerPort: 8080
  strategy:
    canary:
      steps:
        - setWeight: 25
        - pause: {}
        - analysis:
            templates:
              - templateName: check
        - setWeight: 75
        - pause: {duration: 20s}
EOF

kubectl apply -f rollouts/analysis-template.yaml -f rollouts/rollout-canary.yaml
kubectl argo rollouts get rollout demo --watch
```

Leave that terminal open. In a **second terminal** (`cd ~/gitops-labs`):

```bash
kubectl argo rollouts set image demo demo=argoproj/rollouts-demo:yellow
# the rollout pauses at 25%. Promote:
kubectl argo rollouts promote demo
# try these on later releases:
kubectl argo rollouts abort demo
kubectl argo rollouts retry rollout demo
kubectl argo rollouts undo demo
```

Dashboard (new terminal): `kubectl argo rollouts dashboard`, then open http://localhost:3100.

**Do, blue-green:**

```bash
cat > rollouts/rollout-bluegreen.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: bg
spec:
  replicas: 2
  selector:
    matchLabels:
      app: bg
  template:
    metadata:
      labels:
        app: bg
    spec:
      containers:
        - name: bg
          image: argoproj/rollouts-demo:blue
          ports:
            - containerPort: 8080
  strategy:
    blueGreen:
      activeService: bg-active
      previewService: bg-preview
      autoPromotionEnabled: false
---
apiVersion: v1
kind: Service
metadata:
  name: bg-active
spec:
  selector:
    app: bg
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: bg-preview
spec:
  selector:
    app: bg
  ports:
    - port: 80
      targetPort: 8080
EOF

kubectl apply -f rollouts/rollout-bluegreen.yaml
kubectl argo rollouts get rollout bg --watch
# second terminal:
kubectl argo rollouts set image bg bg=argoproj/rollouts-demo:yellow
# new version is on bg-preview, old on bg-active. Promote:
kubectl argo rollouts promote bg
```

**Expect:** canary moves in steps with a pause for a human, blue-green runs both versions fully and switches the active Service at promotion. Without a service mesh or ingress, canary weight is approximated with pod counts.

**Say it:** when would I pick canary and when blue-green? What does a Rollout replace?

## C10. AnalysisTemplate and AnalysisRun

**Topics:** Describe AnalysisTemplate and AnalysisRun, automatic abort.

**Do:**

```bash
cat > rollouts/analysis-args.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: check-args
spec:
  args:
    - name: code
  metrics:
    - name: smoke
      interval: 10s
      count: 2
      failureLimit: 0
      provider:
        job:
          spec:
            backoffLimit: 0
            template:
              spec:
                restartPolicy: Never
                containers:
                  - name: smoke
                    image: alpine:3.19
                    command: [sh, -c, "exit {{args.code}}"]
EOF

cat > rollouts/rollout-analysis.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: ana
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ana
  template:
    metadata:
      labels:
        app: ana
    spec:
      containers:
        - name: ana
          image: argoproj/rollouts-demo:blue
          ports:
            - containerPort: 8080
  strategy:
    canary:
      steps:
        - setWeight: 34
        - analysis:
            templates:
              - templateName: check-args
            args:
              - name: code
                value: "0"
        - setWeight: 100
EOF

kubectl apply -f rollouts/analysis-args.yaml -f rollouts/rollout-analysis.yaml
kubectl argo rollouts get rollout ana --watch
```

Second terminal:

```bash
kubectl argo rollouts set image ana ana=argoproj/rollouts-demo:yellow
kubectl get analysisrun
kubectl describe analysisrun $(kubectl get analysisrun -o name | head -1)
```

**Expect:** with `code: "0"` the AnalysisRun is `Successful` and the rollout finishes. Now change the value to `"1"` in `rollout-analysis.yaml`, apply, and release another image:

```bash
sed -i.bak 's/value: "0"/value: "1"/' rollouts/rollout-analysis.yaml && rm rollouts/rollout-analysis.yaml.bak
kubectl apply -f rollouts/rollout-analysis.yaml
kubectl argo rollouts set image ana ana=argoproj/rollouts-demo:green
```

The AnalysisRun becomes `Failed` and the rollout **aborts and goes back** to the stable version by itself.

Read-only examples of real providers (do not run):

```yaml
# Prometheus metric
provider:
  prometheus:
    address: http://prometheus.monitoring:9090
    query: sum(rate(http_requests_total{status!~"5.."}[2m])) / sum(rate(http_requests_total[2m]))
successCondition: result[0] >= 0.95
# Web metric
provider:
  web:
    url: http://my-app/health
    jsonPath: "{$.ok}"
successCondition: result == true
```

**Say it:** what is the relationship between AnalysisTemplate, AnalysisRun and the Rollout? What are the possible results of an AnalysisRun?

## C11. Argo Events: webhook to Pod

**Topics:** Argo Events fundamentals, components (EventSource, EventBus, Sensor, trigger).

**Do:**

```bash
kubectl -n argo-events create serviceaccount operate-pod-sa
kubectl -n argo-events create role operate-pod --verb=create,get,list --resource=pods
kubectl -n argo-events create rolebinding operate-pod --role=operate-pod --serviceaccount=argo-events:operate-pod-sa

cat > events/eventsource-webhook.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: webhook
  namespace: argo-events
spec:
  webhook:
    example:
      port: "12000"
      endpoint: /example
      method: POST
EOF

cat > events/sensor-webhook.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: webhook
  namespace: argo-events
spec:
  template:
    serviceAccountName: operate-pod-sa
  dependencies:
    - name: dep
      eventSourceName: webhook
      eventName: example
  triggers:
    - template:
        name: make-pod
        k8s:
          operation: create
          source:
            resource:
              apiVersion: v1
              kind: Pod
              metadata:
                generateName: event-pod-
              spec:
                restartPolicy: Never
                containers:
                  - name: hello
                    image: alpine:3.19
                    command: [echo, "event received"]
EOF

kubectl apply -f events/eventsource-webhook.yaml -f events/sensor-webhook.yaml
kubectl -n argo-events get eventsource,sensor,eventbus
kubectl -n argo-events get pods          # webhook-eventsource-* and webhook-sensor-* pods must be Running
kubectl -n argo-events get svc           # find the webhook service name
kubectl -n argo-events port-forward svc/webhook-eventsource-svc 12000:12000
```

In a second terminal:

```bash
curl -X POST -H "Content-Type: application/json" -d '{"msg":"hi"}' localhost:12000/example
kubectl -n argo-events get pods          # a new event-pod-xxxxx appears
```

**Expect:** a new pod is created each time you send a request. Trace it: EventSource → EventBus → Sensor → trigger.

**Say it:** what does each of the four parts do, and where does the EventBus run?

## C12. Argo Events: trigger a Workflow, with a filter and parameters

**Topics:** Sensor dependencies, filters, parameterised triggers, Argo Workflows trigger.

**Do:** the Sensor creates a Workflow in the `argo` namespace, so it needs permission there.

```bash
kubectl -n argo-events create serviceaccount operate-workflow-sa
kubectl -n argo create role operate-workflow --verb=create,get,list,watch --resource=workflows.argoproj.io
kubectl -n argo create rolebinding operate-workflow --role=operate-workflow --serviceaccount=argo-events:operate-workflow-sa

cat > events/sensor-workflow.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: webhook-wf
  namespace: argo-events
spec:
  template:
    serviceAccountName: operate-workflow-sa
  dependencies:
    - name: dep
      eventSourceName: webhook
      eventName: example
      filters:
        data:
          - path: body.msg
            type: string
            value:
              - "hi"
              - "hello"
  triggers:
    - template:
        name: run-workflow
        argoWorkflow:
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: from-event-
                namespace: argo
              spec:
                entrypoint: main
                arguments:
                  parameters:
                    - name: msg
                      value: placeholder
                templates:
                  - name: main
                    inputs:
                      parameters:
                        - name: msg
                    container:
                      image: alpine:3.19
                      command: [echo, "got event message: {{inputs.parameters.msg}}"]
          parameters:
            - src:
                dependencyName: dep
                dataKey: body.msg
              dest: spec.arguments.parameters.0.value
EOF

kubectl apply -f events/sensor-workflow.yaml
kubectl -n argo-events get sensor
```

Make sure the port-forward from C11 is running, then:

```bash
curl -X POST -H "Content-Type: application/json" -d '{"msg":"hello"}' localhost:12000/example    # accepted
curl -X POST -H "Content-Type: application/json" -d '{"msg":"nope"}'  localhost:12000/example    # filtered out
argo list -n argo
argo logs -n argo @latest
```

**Expect:** only the `hello` request starts a Workflow, and its log shows `got event message: hello`. The `nope` request is dropped by the filter. (The first sensor from C11 still creates a pod for every request, delete it with `kubectl -n argo-events delete sensor webhook` if you want a clean test.)

**Say it:** how does event data get into the triggered Workflow? What other trigger types exist?

---

# PART D: Checks, problems and cleanup

## D1. Quick check of the whole lab

```bash
kubectl get pods -A | grep -v kube-system
argocd app list
kubectl get workflows -n argo
kubectl get rollouts
kubectl get eventsources,sensors,eventbus -n argo-events
```

## D2. Troubleshooting

| Problem | Fix |
| --- | --- |
| `metadata.annotations: Too long` when installing Argo CD | Use `kubectl apply --server-side` as shown in A5 |
| Pods `Pending` or kind is slow | Docker is out of memory. Raise it to 6 to 8 GB, or use a single-node cluster |
| `ImagePullBackOff` | Check your internet, then `kubectl describe pod <name>` and retry later (Docker Hub rate limits) |
| Port-forward says "address already in use" | Another port-forward is running. Stop it with Ctrl+C or use another local port |
| Argo CD app shows a repo error or stays `Unknown` | The `repoURL` is wrong, the path does not exist in the repo, or the repo is private and you did not run `argocd repo add` |
| Git changes do not show up fast | Argo CD polls about every 3 minutes. Use `argocd app get NAME --refresh` |
| Helm app is `Unknown` | Check the folder has `Chart.yaml` and that `templates/deployment.yaml` was pushed to `gitops-lab` |
| Workflow pod fails with permission errors | You must run it in the `argo` namespace (`-n argo`) |
| Artifact step fails | MinIO pod in `argo` is not running. Check `kubectl -n argo get pods` |
| Sensor trigger "forbidden" | Redo the RBAC commands of that lab and recreate the sensor |
| Sensor or EventSource pod missing | The EventBus is not ready yet. Check `kubectl -n argo-events get eventbus,pods` |
| `curl` to the webhook gives connection refused | Port-forward is not running, or the service name is different. Check `kubectl -n argo-events get svc` |
| `kubectl argo rollouts` not found | The plugin file must be named `kubectl-argo-rollouts` and be on your PATH |
| No notification in the echo logs | Check `kubectl -n argocd logs deploy/argocd-notifications-controller`, and the annotation on the app |

## D3. Clean up

Remove only the demo apps and keep the cluster:

```bash
kubectl delete application root kustomize-dev kustomize-prod helm-demo waves restricted-test guestbook -n argocd --ignore-not-found
kubectl delete applicationset kz-envs -n argocd --ignore-not-found
kubectl delete -f rollouts/ --ignore-not-found
kubectl delete -f events/sensor-webhook.yaml -f events/sensor-workflow.yaml -f events/eventsource-webhook.yaml --ignore-not-found
kubectl delete namespace patterns --ignore-not-found
```

Delete everything, including the cluster:

```bash
kind delete cluster --name gitops
```

You can recreate it any time by repeating Part A.