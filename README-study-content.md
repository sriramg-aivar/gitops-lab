# Study Guide: CGOA + CAPA

How to use this file:

1. Read one topic at a time. The topics follow the official exam outlines exactly.
2. When a topic says **Lab G#** or **Lab C#**, open `README-lab-setup.md` and do that lab.
3. When you finish a certification, do its practice quiz, then check the answer key.
4. Use the checklist at the end of each certification to see what is still weak.

Practice questions here are my own, in the style of the exam, not real exam questions.

---

# Part 1: Exam facts

| | CGOA | CAPA |
| --- | --- | --- |
| Full name | Certified GitOps Associate | Certified Argo Project Associate |
| Format | 60 multiple-choice questions, online proctored | 60 multiple-choice questions, online proctored |
| Time | 90 minutes | 90 minutes |
| Fee | $250, one free retake | $250, one free retake |
| Pass mark | about 75% (third-party sources, confirm on the exam page) | about 75% (third-party sources, confirm on the exam page) |

| CGOA domain | Weight | CAPA domain | Weight |
| --- | --- | --- | --- |
| GitOps Principles | 30% | Argo Workflows | 36% |
| GitOps Terminology | 20% | Argo CD | 34% |
| GitOps Patterns | 20% | Argo Rollouts | 18% |
| Related Practices | 16% | Argo Events | 12% |
| Tooling | 14% | | |

Order I suggest: CGOA first (ideas), then CAPA (tools). Spend most time on the big domains.

---

# Part 2: CGOA

## Domain 1: GitOps Terminology (20%)

GitOps terms come from the OpenGitOps glossary. Learn each one so you can explain it in one sentence.

| Term | Meaning |
| --- | --- |
| **Declarative description** | A configuration that says what the system should look like, not how to get there. It is machine-readable and kept in a versioned form. A Kubernetes YAML is declarative; a script of commands is not |
| **Desired state** | The state the system should be in, written as a declarative description |
| **Current (actual) state** | What is really running right now |
| **State store** | The system that keeps versioned, immutable copies of the desired state. Git is the usual one |
| **State drift** | The actual state no longer matches the desired state (for example, someone changed a Deployment by hand) |
| **State reconciliation** | The process of making the actual state match the desired state again |
| **Continuous** | Reconciliation keeps happening, over and over. It can be periodic (poll every few minutes) or event-driven (a webhook). It does not mean "instant" and it is not a one-time deploy |
| **Feedback loop** | The cycle of observing the actual state, comparing it to the desired state, and acting on the difference. It is what makes reconciliation continuous |
| **Rollback** | Going back to an earlier desired state, normally by reverting the commit in Git |
| **GitOps managed software system** | A software system whose desired state is kept in a state store and is managed by software agents that keep it reconciled. It includes the runtime environment and the agent |
| **Software agent** | The tool that pulls the desired state and reconciles (Argo CD, Flux) |

**Lab G1** (declarative, desired state, state store), **Lab G2** (drift, reconciliation, feedback loop), **Lab G3** (rollback).

## Domain 2: GitOps Principles (30%)

The four OpenGitOps principles. This is the biggest domain, know them by heart.

1. **Declarative:** a system managed by GitOps must have its desired state expressed declaratively.
2. **Versioned and Immutable:** desired state is stored in a way that enforces immutability and versioning, and keeps a complete version history. Git gives history, audit trail and easy rollback.
3. **Pulled Automatically:** software agents automatically pull the desired state declarations from the source. Nobody pushes them into the cluster from outside.
4. **Continuously Reconciled:** software agents continuously observe the actual state and try to apply the desired state.

Exam trap: a CI job running `kubectl apply` against the cluster is the **push** model, and it breaks principle 3.

Exam trap: a manual `kubectl edit` on the live system is **drift**. A reconciling agent will revert it.

**Labs G1, G2, G3.**

## Domain 3: Related Practices (16%)

The exam asks how well-known practices relate to GitOps and how to map them to it.

| Practice | What it is | How it maps to GitOps |
| --- | --- | --- |
| **Configuration as Code (CaC)** | Application and system configuration kept as files in version control, not set by hand | GitOps stores configuration as declarative files in Git. It adds automatic pull and reconciliation of that configuration |
| **Infrastructure as Code (IaC)** | Infrastructure (networks, clusters, VMs) defined in code (Terraform, Pulumi, CloudFormation) | IaC defines the resources. GitOps adds the agent that continuously reconciles them from Git. Many IaC tools can be driven this way |
| **DevOps** | Culture and practices that join development and operations for fast, reliable delivery | GitOps is a way to do DevOps delivery: Git review, automation, one workflow for dev and ops |
| **DevSecOps** | Security built into every stage of delivery | Pull requests, code review, signed commits, policy checks and scanning before changes reach Git. Credentials stay in the cluster (pull model) |
| **CI (continuous integration)** | Build, test and package each change | In GitOps, CI builds and tests, then publishes the image and updates the desired state in Git. CI does not deploy |
| **CD (continuous delivery/deployment)** | Getting a change into the running system | In GitOps the agent does CD by pulling the change from Git |

Remember: **IaC and CaC are about how you describe things. GitOps is about how the described state is delivered and kept true.**

**Lab G10** (CI and CD roles).

## Domain 4: GitOps Patterns (20%)

### Deployment and release patterns

*Deploying* means putting a version on the infrastructure. *Releasing* means exposing it to users. Patterns:

| Pattern | How it works | Good | Watch out |
| --- | --- | --- | --- |
| **Recreate** | Stop all old pods, then start new ones | Simple, never two versions together | Downtime |
| **Rolling update** | Replace pods a few at a time (Kubernetes default) | No downtime, little extra cost | Old and new versions run together for a while |
| **Blue-green** | Run a full new copy (green) next to the old (blue), then switch traffic | Instant switch, instant rollback | Double the resources |
| **Canary** | Send a small share of traffic to the new version, increase if healthy | Limits the damage of a bad release | Needs good metrics and traffic control |
| **A/B testing** | Different users see different versions to compare a business result | Tests user behaviour | Needs routing by user attribute |
| **Shadow (mirroring)** | Copy real traffic to the new version, ignore its responses | Test with real load, no user impact | Extra load and care with side effects |
| **Feature flags** | Ship code switched off, turn it on later | Separates deploy from release | Flag clean-up |

### Progressive delivery patterns

Progressive delivery means releasing gradually and **checking metrics automatically** at each step, with automatic rollback if the checks fail. Canary, blue-green, A/B and feature flags are the building blocks. Tools: Argo Rollouts, Flagger.

### Pull vs event-driven

| | Pull (periodic) | Event-driven |
| --- | --- | --- |
| How | The agent polls the state store at an interval (Argo CD about every 3 minutes, Flux `interval`) | An event (a Git webhook, a new image) tells the agent to reconcile now |
| Speed | Up to one interval of delay | Almost immediate |
| Still pull? | Yes | Yes. The event only wakes the agent, the agent still fetches and applies by itself |

Both are different from **push**, where a CI system applies changes from outside. Most setups use both: webhook for speed and polling as a safety net.

### Architecture patterns

| Pattern | Meaning | Trade-off |
| --- | --- | --- |
| **In-cluster reconciler** | The agent runs inside the cluster it manages (for example Flux in each cluster) | Cluster credentials never leave the cluster. One agent to run per cluster |
| **External reconciler** | One central agent outside the target manages many clusters (for example one Argo CD managing several clusters) | Easy central view. The central agent holds credentials to every cluster, so it is a bigger target |
| **Monorepo** | All apps and environments in one repo | Simple. Harder to give different access |
| **Repo per app/environment** | Separate repos | Fine-grained access control. More repos to manage |
| **Folder per environment** | `dev/`, `staging/`, `prod/` folders on one branch | Usually recommended. Easy to compare and promote |
| **Branch per environment** | One long-lived branch for each environment | Can drift apart and cause painful merges |
| **Separate config repo** | App source code in one repo, deployment config in another | Cleaner history, separate permissions, CI does not trigger on config changes |

State store management also means: protect the main branch, require pull requests and reviews, use signed commits, and use CODEOWNERS for production folders.

Other patterns: **environment promotion** (a Git change to the next environment's folder, normally through a pull request), **secrets management** (never plain secrets in Git: use encrypted/sealed secrets, SOPS, or an external secret store synced by a tool), **multi-tenancy** (teams separated by namespaces, permissions and repo access).

**Labs G4** (pull vs event-driven), **G5** (rolling vs recreate), **G6** (repo structure and promotion), **G7** (secrets), **C9 and C10** (canary and blue-green in practice).

## Domain 5: Tooling (14%)

### Manifest format and packaging

| Format | What it is |
| --- | --- |
| **Plain YAML/JSON** | Raw Kubernetes manifests. Simple, repeats a lot |
| **Helm** | Charts with Go templates and a `values.yaml`. Good for packaging and reuse. Charts can live in Git or in a Helm/OCI repository |
| **Kustomize** | No templates. A base plus overlays with patches (names, replicas, images). Built into `kubectl` |
| **Jsonnet, CUE and others** | Generate manifests from code or data languages |
| **OCI artifacts** | Manifests or charts packaged and stored in a container registry |

### State store systems

| Store | Notes |
| --- | --- |
| **Git** | The default. Versioning, history, review and audit trail built in |
| **OCI registry** | Used as a state store by Flux (`OCIRepository`) and supported by Argo CD. Tags or digests give versions, and digests are immutable |
| **S3-style bucket** | Flux can read a bucket. Needs versioning turned on to meet the "versioned and immutable" principle |
| **Helm repository** | A source of packaged charts |

Whatever store you use, it must keep **versioned and immutable** history.

### Reconciliation engines

| Engine | Notes |
| --- | --- |
| **Argo CD** | Application CRD, UI, multi-cluster, sync waves and hooks, ApplicationSet |
| **Flux** | Set of controllers (source, kustomize, helm, notification, image automation). Everything is Kubernetes CRDs, no built-in UI |
| **Alternatives** | Other tools do the same job, for example Rancher Fleet. Any controller that pulls desired state and reconciles it is a reconciliation engine |

### Interoperability with notifications, observability and CI

| Area | How GitOps tools connect |
| --- | --- |
| **Notifications** | Argo CD Notifications and Flux's notification controller send events (sync succeeded, failed, degraded) to Slack, Teams, email, webhooks, or update commit status in Git |
| **Observability** | Argo CD and Flux expose Prometheus metrics. Grafana shows dashboards. Kubernetes events and health status give more signals. Rollout analysis can read the same metrics |
| **CI tools** | Jenkins, GitHub Actions, GitLab CI, Tekton build and test, then push the image and update the tag in the config repo. Tools like Argo CD Image Updater or Flux image automation can do the tag update from the registry. The CI job does not apply to the cluster |

**Labs G6** (Kustomize), **G8** (Flux), **G9** (notifications and metrics), **G10** (CI), **C7** (Helm and Kustomize).

---

# Part 3: CGOA practice quiz

Try each question first. The answer key follows the quiz.

**1.** Which statement best describes "declarative"?
A) The system is changed by running a list of commands in order B) The desired state is described by what the system should look like C) Only admins can change the system D) Changes are made directly in production

**2.** A CI pipeline runs `kubectl apply` on production with stored credentials. Which principle does this break?
A) Declarative B) Versioned and immutable C) Pulled automatically D) Continuously reconciled

**3.** Someone scales a Deployment by hand and it no longer matches Git. What is this called?
A) Rollback B) Drift C) Promotion D) Immutability

**4.** What does a GitOps agent do when it detects drift?
A) Sends an email and stops B) Deletes the repository C) Reconciles the actual state back to the desired state D) Waits for a CI job

**5.** What is the usual way to roll back a bad change in GitOps?
A) SSH into the node and edit files B) Revert the commit in Git C) Delete the cluster D) Re-run the CI build only

**6.** Why is Git a good state store?
A) It is the fastest database B) It keeps version history and an audit trail C) It encrypts all data by default D) It runs the deployments

**7.** Where should plain-text secrets live in a GitOps setup?
A) In the repo as normal Secret manifests B) In CI logs C) Nowhere in plain text; use encrypted secrets or an external store D) In a ConfigMap

**8.** How is a change promoted from staging to production?
A) A script on the prod node B) Change the prod environment's desired state in Git, often via pull request C) Restart the agent D) Edit the live cluster

**9.** Which are two well-known GitOps agents for Kubernetes?
A) Prometheus and Grafana B) Argo CD and Flux C) Helm and Jenkins D) Istio and Linkerd

**10.** In a GitOps flow, who does what?
A) CI builds and tests and updates the repo; the agent delivers by pulling B) CI reconciles the cluster; the agent builds images C) Both only build images D) The agent runs unit tests

**11.** Which is NOT one of the four OpenGitOps principles?
A) Declarative B) Versioned and immutable C) Continuously reconciled D) Manually approved by an administrator

**12.** Which repo pattern helps most to stop developers editing production config?
A) One repo with open write access B) Separate repos or restricted paths for app code and environment config C) Config on laptops D) No repo

**13.** In GitOps, what does "continuous" mean?
A) A change reaches the cluster in under one second B) Reconciliation keeps happening repeatedly, by polling or events C) CI runs on every commit D) The cluster never restarts

**14.** Which is NOT needed for a "declarative description"?
A) Describes the desired state B) Machine-readable C) Versioned D) Lists the exact steps to reach the state

**15.** What is a "GitOps managed software system"?
A) Any system that uses Git for source code B) A system whose desired state is in a state store and kept reconciled by software agents C) A CI server D) A Git hosting service

**16.** Which deployment pattern causes downtime by design?
A) Rolling update B) Blue-green C) Recreate D) Canary

**17.** Which pattern runs two full environments and switches traffic between them?
A) Canary B) Blue-green C) Recreate D) Shadow

**18.** What is the main idea of a canary release?
A) Send a small share of traffic to the new version first, then increase it B) Replace everything at once C) Test only in staging D) Keep two clusters

**19.** What does progressive delivery add on top of simple rollouts?
A) Automatic metric checks at each step and automatic rollback B) More YAML C) A bigger cluster D) Manual approvals only

**20.** What do feature flags do?
A) Encrypt secrets B) Separate deploying code from releasing a feature to users C) Speed up builds D) Replace Git

**21.** What is event-driven reconciliation?
A) The agent never runs B) An event such as a Git webhook makes the agent reconcile right away instead of waiting for the poll interval C) CI applies manifests D) A person runs `kubectl`

**22.** What is an advantage of an in-cluster reconciler?
A) One central credential store for all clusters B) Cluster credentials stay inside the cluster C) No agent is needed D) It works without Git

**23.** What is the main trade-off of an external (central) reconciler managing many clusters?
A) It cannot use Git B) It holds credentials for every cluster, which makes it a bigger target C) It needs no network D) It cannot reconcile

**24.** Which layout is commonly recommended for environments?
A) A long-lived branch per environment B) A folder per environment on one branch, changed through pull requests C) Editing the cluster directly D) A new repo for every commit

**25.** What is the key feature of Kustomize compared with Helm?
A) Go templates and values files B) Overlays and patches on plain YAML without templates C) It needs a registry D) It only works with Flux

**26.** What does a Helm chart provide?
A) Templated manifests with a values file, packaged for reuse B) Cluster monitoring C) A Git server D) A secrets vault

**27.** Besides Git, which can serve as a state store if it keeps versioned, immutable content?
A) A shared network drive with overwritable files B) An OCI registry (with tags or digests) C) A chat channel D) A local laptop folder

**28.** Which of these is a reconciliation engine?
A) Argo CD B) Prometheus C) Helm only D) Grafana

**29.** A team wants a Slack message when an app fails to sync. Which feature fits?
A) Argo CD Notifications or Flux's notification controller B) Kustomize overlays C) A bigger cluster D) Sealed secrets

**30.** Which description of CI/CD in GitOps is correct?
A) CI builds, tests and updates the config repo; the GitOps agent pulls the change (CD) B) CI applies manifests to the cluster; the agent builds images C) CI replaces Git D) CD is not needed

**31.** How do IaC and GitOps relate?
A) They are the same thing B) IaC describes the infrastructure as code; GitOps adds automatic pull and continuous reconciliation of it from the state store C) GitOps replaces IaC D) IaC needs no files

**32.** How does DevSecOps show up in GitOps?
A) Pull requests, reviews, signed commits and policy checks before changes reach Git; credentials stay in the cluster B) Security is done only after release C) Secrets are stored in plain text D) Nothing changes

## CGOA answer key

1. **B.** Declarative describes the end result, not the steps.
2. **C.** Pushing from outside is the push model, GitOps agents pull.
3. **B.** Drift is actual state differing from desired state.
4. **C.** Continuous reconciliation is the agent's job. (In Argo CD this needs self-heal on, otherwise the app only shows OutOfSync.)
5. **B.** Git holds the desired state, so reverting the commit restores it and the agent applies it.
6. **B.** Version history and review make Git a fit. The agent deploys, not Git.
7. **C.** A Kubernetes Secret is only base64, so committing it exposes it.
8. **B.** Promotion is a Git change to the next environment's config.
9. **B.**
10. **A.** Keep CI (build, test, publish) separate from CD (pull and reconcile).
11. **D.**
12. **B.** Separate repos or paths allow separate permissions.
13. **B.** Continuous means ongoing, either polling or event-driven. It does not mean instant.
14. **D.** A declarative description says what, not how.
15. **B.** The system, its desired state in a state store, and the agents that reconcile it.
16. **C.** Recreate stops old pods before starting new ones.
17. **B.**
18. **A.**
19. **A.** Progressive delivery adds automated analysis and rollback.
20. **B.** Deploy code switched off, release later.
21. **B.** The event wakes the agent, which still pulls and applies by itself.
22. **B.**
23. **B.**
24. **B.** Folders per environment are easy to compare and promote. Long-lived branches drift apart.
25. **B.**
26. **A.**
27. **B.** The store must be versioned and immutable. Overwritable files are not.
28. **A.**
29. **A.**
30. **A.**
31. **B.**
32. **A.**

## CGOA checklist

- [ ] I can define every term in Domain 1, including continuous, declarative description and GitOps managed software system
- [ ] I can say the four principles from memory and give an example of each
- [ ] I can explain how CaC, IaC, DevOps, DevSecOps and CI/CD relate to GitOps
- [ ] I can compare recreate, rolling, blue-green, canary, A/B, shadow and feature flags
- [ ] I can explain pull vs event-driven vs push
- [ ] I can compare in-cluster and external reconcilers, and folder vs branch per environment
- [ ] I can explain Helm vs Kustomize and name state stores other than Git
- [ ] I can name two reconciliation engines and how tools connect to notifications, observability and CI
- [ ] I did labs G1 to G10
- [ ] I scored 85% or more on a timed run of the quiz (60 questions in 90 minutes is the real exam)

---

# Part 4: CAPA

Argo is four tools. The exam asks which tool fits which job. **Workflows** runs jobs, **CD** deploys from Git, **Rollouts** releases safely, **Events** reacts to outside events.

## Domain 1: Argo Workflows (36%)

### Understand Argo Workflow fundamentals

Argo Workflows is a container-native workflow engine for Kubernetes. A Workflow is a custom resource, and each step runs as a pod. Typical uses: CI jobs, data and ML pipelines, batch jobs.

| Part | Role |
| --- | --- |
| **Workflow controller** | Watches Workflow resources and creates the pods |
| **Argo Server** | UI and API |
| **`argo` CLI** | `submit`, `list`, `get`, `logs`, `watch`, `delete`, `retry`, `stop` |
| **Executor** | Runs inside each pod, handles inputs, outputs and artifacts (the default is "emissary") |
| **Artifact repository** | Where artifacts are stored (S3, MinIO, GCS, Azure Blob and others) |

A workflow goes through phases: `Pending`, `Running`, `Succeeded`, `Failed`, `Error`.

**Labs C1 and C4.**

### Generating and consuming artifacts

Artifacts are **files** moved between steps (parameters carry small text values). A step *generates* an output artifact, the engine saves it to the artifact repository, and a later step *consumes* it as an input artifact.

- Output: `outputs.artifacts` with `name` and `path` (a file or folder in the container).
- Input: `inputs.artifacts` with `name` and `path`, where the file will appear.
- Passing between steps: `from: "{{steps.STEP.outputs.artifacts.NAME}}"` (steps) or `{{tasks.TASK.outputs.artifacts.NAME}}` (DAG).
- Input artifacts can also come directly from `git`, `http`, `s3`, or `raw` inline text.
- The default repository is set in the controller's configmap (`artifactRepository`). A Workflow can point to another with `artifactRepositoryRef`.

Parameters vs artifacts: small values and control flow use parameters; files and bigger data use artifacts.

**Lab C2.**

### Understand Argo Workflow templates

A template is a unit of work. The main types:

| Type | What it does |
| --- | --- |
| `container` | Runs one container |
| `script` | Runs inline source code. Its stdout is available as `outputs.result` |
| `resource` | Creates, applies, patches or deletes a Kubernetes resource |
| `suspend` | Pauses until resumed or a duration passes |
| `steps` | A list of lists. Outer list runs in order, each inner list runs in parallel |
| `dag` | Tasks with dependencies |
| `containerSet` | Several containers in one pod, run in a defined order |
| `http` | Makes an HTTP request |
| `data` | Transforms artifact data |

Reuse:

- **`WorkflowTemplate`**: a namespaced template stored in the cluster. Use it with `templateRef: {name, template}` or with `workflowTemplateRef` for the whole spec. Run it with `argo submit --from workflowtemplate/NAME`.
- **`ClusterWorkflowTemplate`**: same but cluster-wide. Reference it with `clusterScope: true`.
- **`CronWorkflow`**: runs a workflow on a cron schedule.

Inputs and outputs: `inputs.parameters`, `outputs.parameters`, referenced as `{{inputs.parameters.NAME}}`. Global values: `{{workflow.parameters.NAME}}`, `{{workflow.name}}`, `{{workflow.status}}`.

**Lab C3.**

### Understand the Argo Workflow spec

Top-level fields of `spec` you should recognise:

| Field | Purpose |
| --- | --- |
| `entrypoint` | The template that starts the workflow |
| `templates` | The list of templates |
| `arguments` | Input parameters and artifacts for the whole workflow |
| `serviceAccountName` | The service account the pods run as |
| `volumes` / `volumeClaimTemplates` | Volumes to share between steps |
| `parallelism` | Maximum pods running at once |
| `activeDeadlineSeconds` | Maximum run time for the workflow |
| `ttlStrategy` | How long finished workflows are kept |
| `podGC` | When to delete finished pods |
| `onExit` | Template to run when the workflow ends, whatever the result |
| `retryStrategy` | Retry failed steps (also at template level) |
| `nodeSelector`, `tolerations`, `affinity` | Scheduling |
| `synchronization` | Semaphore or mutex to limit concurrency |
| `suspend` | Start paused |

**Lab C1.**

### Work with DAGs

A `dag` template has `tasks`. Each task names a `template` and can declare what it waits for.

- `dependencies: [A, B]`: waits for tasks A and B to succeed.
- `depends: "A.Succeeded || B.Failed"`: a logic expression that can react to results (`Succeeded`, `Failed`, `Skipped`, `Errored`).
- Tasks with no unmet dependency run at the same time.
- Cycles are not allowed (it is acyclic).
- By default one failed task stops new tasks (`failFast`).

**Steps vs DAG:** use `steps` for a simple ordered flow. Use `dag` when tasks form a graph and you want maximum parallelism.

**Lab C4.**

### Run data processing jobs with Argo Workflows

Typical pattern is **fan-out and fan-in** (map-reduce): split work into many parallel steps, then combine the results.

| Feature | What it does |
| --- | --- |
| `withItems` | Loop over a fixed list |
| `withParam` | Loop over a list produced by an earlier step (JSON) |
| `withSequence` | Loop over a number range |
| `{{item}}` | The current loop value |
| `outputs.result` | Stdout of a script. For a loop, it is a JSON list of all the results |
| `parallelism` | Limit how many loop pods run at once |
| Artifacts or volumes | Share large data between steps |
| `retryStrategy`, `resources`, `nodeSelector` | Make big jobs reliable and well placed |
| `CronWorkflow` | Run the processing on a schedule |
| `memoize` | Reuse the result of a step when the inputs have not changed |

**Lab C5.**

## Domain 2: Argo CD (34%)

### Understand Argo CD fundamentals

Argo CD is a declarative GitOps continuous delivery tool for Kubernetes. It compares the Git desired state with the live state and syncs.

| Component | Role |
| --- | --- |
| **API server** | UI, CLI and API endpoint, handles auth and RBAC |
| **Repo server** | Clones the Git repo and renders manifests (plain YAML, Helm, Kustomize) |
| **Application controller** | Compares live state and desired state and performs syncs |
| **Redis** | Cache |
| **Dex** | Optional SSO |
| **ApplicationSet controller** | Generates Applications from templates |
| **Notifications controller** | Sends messages on app events |

Statuses (two separate ones):

- **Sync status:** `Synced` or `OutOfSync` (live vs Git).
- **Health status:** `Healthy`, `Progressing`, `Degraded`, `Suspended`, `Missing`, `Unknown`.

Exam trap: **Synced does not mean Healthy.**

Git is polled about every 3 minutes by default. A webhook or `argocd app get NAME --refresh` makes it immediate.

**Lab C6.**

### Synchronise applications using Argo CD

- **Manual sync:** you click Sync or run `argocd app sync`.
- **Automated sync:** `syncPolicy.automated`, with `prune: true` (delete what was removed from Git) and `selfHeal: true` (revert manual changes in the cluster).
- **Sync options:** `CreateNamespace=true`, `PruneLast=true`, `ApplyOutOfSyncOnly=true`, `ServerSideApply=true`, `Replace=true`.
- **Retry:** `syncPolicy.retry` with limit and backoff.
- **Sync phases and hooks:** `PreSync`, `Sync`, `PostSync`, `SyncFail` (and `PostDelete`), set with the annotation `argocd.argoproj.io/hook`. Delete policy: `argocd.argoproj.io/hook-delete-policy` (for example `HookSucceeded`).
- **Sync waves:** `argocd.argoproj.io/sync-wave: "N"`. Lower numbers sync first, and Argo CD waits for a wave to be healthy before the next.
- **Diff and dry run:** `argocd app diff`, `argocd app sync --dry-run`.
- **History and rollback:** `argocd app history`, `argocd app rollback NAME ID`. Rollback needs auto-sync turned off.

**Labs C6 and C8.**

### Use an Argo CD Application

`Application` is the custom resource that links a source to a destination.

```yaml
spec:
  project: default            # which AppProject it belongs to
  source:
    repoURL: https://github.com/ORG/REPO.git
    targetRevision: HEAD       # branch, tag or commit
    path: some/folder          # folder in the repo (or "chart" for a Helm repo)
  destination:
    server: https://kubernetes.default.svc   # or "name:" of a registered cluster
    namespace: my-app
  syncPolicy: {}
```

- **`AppProject`** groups apps and restricts allowed source repos, destination clusters/namespaces and which resource kinds are allowed. It also holds RBAC roles.
- Cluster registration: `argocd cluster add`.
- Repo credentials: `argocd repo add`.
- Deleting an app with the finalizer `resources-finalizer.argocd.argoproj.io` (a cascade delete) also removes the resources it created.
- An Application can have several `sources`.

**Labs C6 and C8.**

### Configure Argo CD with Helm and Kustomize

**Helm:**

- The source can be a chart in a Git folder (`path`) or in a Helm repository (`chart` and `targetRevision` as the chart version).
- Configure with `source.helm`: `valueFiles`, `values` (inline), `parameters` (override single values), `releaseName`, `skipCrds`.
- Argo CD runs `helm template` and applies the result. It does **not** create a normal Helm release, so `helm ls` does not show the app.

**Kustomize:**

- The folder has a `kustomization.yaml`. Argo CD runs `kustomize build`.
- Configure with `source.kustomize`: `namePrefix`, `nameSuffix`, `images`, `replicas`, `commonLabels`, `commonAnnotations`.
- Typical layout is `base/` plus `overlays/dev` and `overlays/prod`, with one Application per overlay.

**Lab C7.**

### Identify common reconciliation patterns

| Pattern | Use |
| --- | --- |
| **Auto-sync + self-heal + prune** | Git is always the truth. Manual changes are reverted and removed resources are deleted |
| **Manual sync with diff** | Review before applying, common for production |
| **App of apps** | One root Application points to a folder of other Application manifests, to bootstrap many apps |
| **ApplicationSet** | One template and generators create many Applications. Generators: `list`, `cluster`, `git` (directories or files), `matrix`, `merge`, `pullRequest`, `scmProvider` |
| **Environment promotion** | Overlay folders or values files per environment, changed through Git |
| **Sync waves and hooks** | Control order (database migration before the app) |
| **`ignoreDifferences`** | Ignore fields changed by other controllers (for example `/spec/replicas` managed by an autoscaler) |
| **Webhook refresh** | Reconcile immediately on push instead of waiting for polling |
| **Rollback** | Revert in Git (preferred), or `argocd app rollback` |

**Lab C8.**

## Domain 3: Argo Rollouts (18%)

### Understand Argo Rollouts fundamentals

Argo Rollouts adds progressive delivery to Kubernetes. A **`Rollout`** resource replaces a `Deployment` and has the same pod template, with a richer `strategy`.

- The controller manages ReplicaSets (stable and new) and can shift traffic gradually.
- You can keep an existing Deployment and point a Rollout to it with `workloadRef`.
- Rollout statuses include `Healthy`, `Progressing`, `Paused`, `Degraded`.
- Commands (`kubectl argo rollouts ...`): `get rollout NAME --watch`, `set image`, `promote` (`--full` skips all steps), `abort`, `retry`, `undo`, `dashboard`.
- Rollouts does **not** pull from Git. Use it together with Argo CD.

**Lab C9.**

### Use common progressive rollout strategies

**Canary** (`strategy.canary`): move traffic to the new version in steps.

- `steps`: `setWeight` (percent), `pause` (for a duration or `{}` until promoted), `analysis`, `experiment`, `setCanaryScale`.
- `maxSurge` and `maxUnavailable` control the replica change.
- With a traffic router (`trafficRouting` for NGINX, Istio, ALB, SMI and others) the weight is exact. Without one, the weight is approximated with pod counts.
- `stableService` and `canaryService` name the two Services.

**Blue-green** (`strategy.blueGreen`): run the new version fully next to the old, then switch.

- `activeService`: serves production users.
- `previewService`: serves the new version for testing.
- `autoPromotionEnabled: false` waits for manual promotion. `autoPromotionSeconds` promotes after a delay.
- `scaleDownDelaySeconds`: how long the old version stays after the switch (fast rollback window).
- `prePromotionAnalysis` and `postPromotionAnalysis`: run checks before or after the switch.

**Experiment:** runs one or more versions side by side for a set time and analyses them, without taking production traffic.

**Labs C9 and C10.**

### Describe AnalysisTemplate and AnalysisRun

- **`AnalysisTemplate`** (namespaced) and **`ClusterAnalysisTemplate`** (cluster-wide) define *what to check*: a list of `metrics`, each with a `provider`, an `interval`, a `count`, a `successCondition` and/or `failureCondition`, and a `failureLimit`. Templates can take `args`.
- **`AnalysisRun`** is one execution of an analysis, created by the Rollout (from a step or from background analysis) or by an Experiment.
- AnalysisRun results: `Successful`, `Failed`, `Inconclusive`, `Error`, `Running`, `Pending`.
- If the run **fails**, the Rollout **aborts and rolls back** automatically. If it is inconclusive, the Rollout pauses.
- Analysis can run as a **step** (inline, in the canary steps) or in the **background** during the rollout (`startingStep`).
- Metric providers: Prometheus, Datadog, New Relic, CloudWatch, Wavefront, Kayenta, Web (HTTP and JSON path), Kubernetes Job, and others.

**Lab C10.**

## Domain 4: Argo Events (12%)

### Understand Argo Events fundamentals

Argo Events is an event-driven automation framework. It listens for events from many sources and then triggers actions, such as starting an Argo Workflow.

- Event sources include webhook, GitHub, GitLab, S3/MinIO, Kafka, NATS, SNS/SQS, calendar (cron), Kubernetes resources, files, Slack and more.
- Triggers include Argo Workflows, any Kubernetes object, HTTP requests, Slack, AWS Lambda, Kafka, NATS and others.
- The flow: **EventSource → EventBus → Sensor → Trigger.**

**Lab C11.**

### Understand Argo Event components and architecture

| Component | Role |
| --- | --- |
| **EventSource** | A custom resource that listens to an outside source and publishes events. It runs as its own pod (a webhook EventSource also gets a Service) |
| **EventBus** | The messaging layer between EventSources and Sensors. Implementations: **NATS JetStream** (the current native option) and **Kafka**. Sensors and EventSources connect to it |
| **Sensor** | Defines `dependencies` (which events to wait for) and `triggers` (what to do). Runs as its own pod |
| **Trigger** | The action. Can use `parameters` to copy data from the event into the resource it creates |
| **Controllers** | The controller manager watches the EventSource, EventBus and Sensor resources and creates their pods |

Sensor details to remember:

- A dependency has `eventSourceName` and `eventName`.
- **Filters** drop events you do not want: `data` (on payload fields), `time`, `context`, `expr`, `script`.
- **Conditions** combine dependencies with logic like `A && B`.
- **Parameters** map event data into the trigger with `src` (`dependencyName`, `dataKey`) and `dest` (a path in the resource).
- The Sensor needs a service account with permission to create what it triggers.

**Labs C11 and C12.**

---

# Part 5: CAPA practice quiz

**1.** Which Argo tool runs a multi-step data pipeline where each step is a container?
A) Argo CD B) Argo Workflows C) Argo Rollouts D) Argo Events

**2.** Which Argo Workflows component watches Workflow resources and creates the pods?
A) Argo Server B) Workflow controller C) Artifact repository D) `argo` CLI

**3.** How do you pass a file produced in one step to a later step?
A) A parameter B) An output artifact consumed as an input artifact C) A label D) An annotation

**4.** Where does the default artifact repository get configured?
A) In every pod spec B) In the workflow controller's configuration, with optional override per workflow C) In the Argo CD repo D) In the Sensor

**5.** Which template type runs inline source code and exposes its stdout as `outputs.result`?
A) `container` B) `script` C) `resource` D) `suspend`

**6.** Which resource is a reusable, namespaced template stored in the cluster?
A) `CronWorkflow` B) `WorkflowTemplate` C) `AnalysisTemplate` D) `Sensor`

**7.** How do you reference a `ClusterWorkflowTemplate` from a step?
A) `templateRef` with `clusterScope: true` B) `workflowRef` C) `include` D) You cannot

**8.** Which `spec` field runs a template when the workflow ends, whatever the result?
A) `entrypoint` B) `onExit` C) `podGC` D) `ttlStrategy`

**9.** Which `spec` field limits how many pods run at the same time?
A) `parallelism` B) `activeDeadlineSeconds` C) `retryStrategy` D) `serviceAccountName`

**10.** Which field removes finished workflows after a period?
A) `podGC` B) `ttlStrategy` C) `onExit` D) `synchronization`

**11.** Which template type lets tasks declare dependencies on each other?
A) `container` B) `script` C) `dag` D) `suspend`

**12.** In a `steps` template, what runs in parallel?
A) All steps of the outer list B) Steps in the same inner list C) Nothing D) Only steps with `when`

**13.** Which DAG field lets a task wait on a logic expression like `A.Succeeded || B.Failed`?
A) `dependencies` B) `depends` C) `when` D) `withItems`

**14.** Which feature loops over a list produced by an earlier step?
A) `withItems` B) `withParam` C) `withSequence` D) `onExit`

**15.** After a loop step finishes, what is `steps.NAME.outputs.result`?
A) The last item B) A JSON list of the results of every iteration C) Always empty D) A file

**16.** Which resource runs a workflow on a schedule?
A) `WorkflowTemplate` B) `CronWorkflow` C) `Sensor` D) `AnalysisRun`

**17.** An Argo CD app shows `Synced` and `Degraded`. What does it mean?
A) Live state matches Git, but resources are not healthy B) Git and cluster differ C) Repo unreachable D) Sync cancelled

**18.** Which Argo CD component clones the repo and renders Helm or Kustomize?
A) API server B) Repo server C) Redis D) Dex

**19.** Which `automated` option reverts manual changes made in the cluster?
A) `prune` B) `selfHeal` C) `allowEmpty` D) `retry`

**20.** Which `automated` option deletes resources that were removed from Git?
A) `prune` B) `selfHeal` C) `retry` D) `ServerSideApply`

**21.** Resource A has `sync-wave: "0"` and resource B has `sync-wave: "2"`. What happens?
A) B first B) Together C) A first, and Argo CD waits for it to be healthy before B D) The sync fails

**22.** Which annotation makes a Job run before the main sync?
A) `argocd.argoproj.io/hook: PreSync` B) `argocd.argoproj.io/hook: PostSync` C) `argocd.argoproj.io/hook: SyncFail` D) `argocd.argoproj.io/sync-options`

**23.** What does an `AppProject` do?
A) Runs workflows B) Restricts which repos, clusters, namespaces and resources its apps may use C) Builds images D) Creates events

**24.** How does Argo CD deploy a Helm chart?
A) It runs `helm install` and tracks a Helm release B) It runs `helm template` and applies the rendered manifests C) It cannot use Helm D) It only works with Helm repos

**25.** Which `source.helm` field overrides a single value without a new file?
A) `valueFiles` B) `parameters` C) `releaseName` D) `skipCrds`

**26.** In an Argo CD Application using Kustomize, which field changes image tags?
A) `kustomize.images` B) `helm.parameters` C) `syncOptions` D) `ignoreDifferences`

**27.** You need one Application per cluster for 30 clusters from one template. What do you use?
A) 30 hand-written Applications B) An `ApplicationSet` with a cluster generator C) An `AppProject` D) An `EventSource`

**28.** What is the "app of apps" pattern?
A) A root Application whose source folder contains other Application manifests B) An app with 2 containers C) A Helm dependency D) A Workflow of workflows

**29.** An autoscaler changes `spec.replicas` and the app keeps showing OutOfSync. What fixes this?
A) `ignoreDifferences` on `/spec/replicas` B) Turn off Git C) Delete the app D) Use a hook

**30.** Which command rolls an Argo CD app back to an earlier history ID?
A) `argocd app rollback NAME ID` (auto-sync must be off) B) `argocd app delete` C) `argocd repo add` D) `argocd login`

**31.** Which resource replaces a Deployment to give canary and blue-green releases?
A) `Workflow` B) `Rollout` C) `Sensor` D) `ApplicationSet`

**32.** In a canary strategy, what does `pause: {}` do?
A) Pauses for 1 second B) Pauses indefinitely until promoted C) Deletes the rollout D) Aborts the rollout

**33.** In a blue-green Rollout, which Service sends test traffic to the new version before promotion?
A) `activeService` B) `previewService` C) `canaryService` D) `stableService`

**34.** Which statement about Argo Rollouts is true?
A) It pulls manifests from Git B) It manages ReplicaSets and release strategy, and is often paired with Argo CD for the Git part C) It replaces Argo Events D) It runs only batch jobs

**35.** What is an `AnalysisRun`?
A) A reusable definition of metrics B) One execution of an analysis, created by a Rollout or Experiment C) A type of Service D) A Git commit

**36.** During a canary rollout an `AnalysisRun` fails. What happens by default?
A) The rollout goes to 100% B) The rollout aborts and returns to the stable version C) Nothing happens D) The cluster is deleted

**37.** Which object defines the metrics, interval, `successCondition` and `failureLimit` for analysis?
A) `Rollout` B) `AnalysisTemplate` C) `EventSource` D) `AppProject`

**38.** Which of these is a valid metric provider for analysis?
A) Prometheus B) Dex C) Redis D) Kustomize

**39.** What is the correct flow of Argo Events for an event that starts a workflow?
A) Sensor → EventSource → EventBus → Trigger B) EventSource → EventBus → Sensor → Trigger C) Trigger → Sensor → EventBus → EventSource D) EventBus → EventSource → Trigger → Sensor

**40.** Which Argo Events resource defines what to do when an event arrives?
A) `EventSource` B) `EventBus` C) `Sensor` D) `CronWorkflow`

**41.** Which implementations can the EventBus use?
A) NATS JetStream and Kafka B) Redis and Dex C) MinIO only D) Helm and Kustomize

**42.** In a Sensor, what do `filters` do?
A) Format YAML B) Drop events whose data does not match conditions C) Delete the EventBus D) Create namespaces

**43.** How does a Sensor copy data from an event into the triggered Workflow?
A) Trigger `parameters` with `src` (`dataKey`) and `dest` B) It cannot C) By labels D) By annotations on the Sensor

**44.** A team wants Git-driven deployment plus safe canary releases. Which combination fits?
A) Argo Events only B) Argo CD for GitOps delivery with Argo Rollouts for the release strategy C) Argo Workflows only D) Argo Rollouts only, because it pulls from Git

## CAPA answer key

1. **B.**
2. **B.** The Argo Server is the UI/API.
3. **B.** Parameters carry small text, artifacts carry files.
4. **B.** Configured in the controller, with `artifactRepositoryRef` as the override.
5. **B.**
6. **B.**
7. **A.**
8. **B.**
9. **A.**
10. **B.** `podGC` deletes pods, `ttlStrategy` deletes workflows.
11. **C.**
12. **B.** The outer list is sequential, each inner list is a parallel group.
13. **B.** `dependencies` waits only for success, `depends` supports expressions.
14. **B.** `withItems` is a fixed list, `withParam` takes JSON from a previous step.
15. **B.**
16. **B.**
17. **A.** Sync status and health status are separate.
18. **B.** The application controller compares and syncs, Dex does SSO.
19. **B.**
20. **A.**
21. **C.** Lower waves go first.
22. **A.**
23. **B.**
24. **B.** Argo CD renders with `helm template`, so there is no Helm release.
25. **B.**
26. **A.**
27. **B.**
28. **A.**
29. **A.**
30. **A.**
31. **B.**
32. **B.** `{}` means wait until promoted. A duration like `{duration: 20s}` is timed.
33. **B.** The `activeService` serves production users.
34. **B.** Rollouts does not pull from Git.
35. **B.** The template is the definition, the run is one execution.
36. **B.**
37. **B.**
38. **A.** Prometheus is one of several (also Datadog, New Relic, Web, Job and others).
39. **B.**
40. **C.** The Sensor holds the dependencies and the triggers.
41. **A.**
42. **B.**
43. **A.**
44. **B.**

## CAPA checklist

Argo Workflows
- [ ] I can explain the controller, server, executor and artifact repository
- [ ] I can write a workflow that generates an artifact and consumes it in the next step
- [ ] I know every template type and when to use each
- [ ] I can use `WorkflowTemplate`, `ClusterWorkflowTemplate` and `templateRef`
- [ ] I know the key `spec` fields: `entrypoint`, `arguments`, `parallelism`, `onExit`, `retryStrategy`, `ttlStrategy`, `podGC`
- [ ] I can write DAG and steps versions of the same flow and explain `dependencies` vs `depends`
- [ ] I can build a fan-out and fan-in with `withParam` and `outputs.result`

Argo CD
- [ ] I know the components and the difference between sync status and health status
- [ ] I can explain automated sync, `prune`, `selfHeal`, hooks, waves and rollback
- [ ] I can write an `Application` and an `AppProject` from memory
- [ ] I can configure Helm (values, parameters) and Kustomize (images, prefix, overlays)
- [ ] I can explain app of apps, ApplicationSet generators and `ignoreDifferences`

Argo Rollouts
- [ ] I can explain what a Rollout replaces and what the controller does
- [ ] I can configure canary (`setWeight`, `pause`, `analysis`) and blue-green (`activeService`, `previewService`)
- [ ] I can explain AnalysisTemplate vs AnalysisRun and what happens when a run fails

Argo Events
- [ ] I can draw EventSource → EventBus → Sensor → Trigger and say what each does
- [ ] I can write a webhook EventSource and a Sensor with a filter and a parameterised trigger

Overall
- [ ] I did labs C1 to C12
- [ ] I scored 85% or more on a timed run of the quiz

---

# Part 6: Exam day tips (both exams)

- Both are 90 minutes for 60 questions, so you have about 90 seconds per question. Skip a hard one, flag it, and come back.
- Read the last line of the question first. Many are "which is NOT" or "which is BEST".
- For YAML questions, find the `kind` first, then the fields that decide the behaviour (`strategy`, `syncPolicy`, `dependencies`).
- Online proctored: clear your desk, have your ID, and test your camera and internet beforehand.
- One free retake comes with the $250 fee, so a first attempt is low risk. Check the validity period on each exam page when you book.
- As a Kubestronaut you have yearly 50% coupons. Check whether they apply before paying full price.