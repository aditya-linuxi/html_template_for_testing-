# Tekton CI Configuration on VKS
*Technical Reference & Backup Documentation — CI (Build) Scope Only*

---

## 1. Architecture of Tekton on VKS

Tekton is deployed as the Pipeline Orchestrator inside a dedicated **CI-VKS cluster**, separate from the **Runtime-VKS cluster** that runs production application workloads. This isolation limits the blast radius of build activity and allows the CI and runtime environments to scale independently.

```
Developer ──push code──▶ App Source Repo (Git)
                              │
                        webhook trigger
                              ▼
        ┌───────────────────────────────────────────────┐
        │              CI-VKS Cluster                    │
        │  (dedicated build cluster)                      │
        │                                                  │
        │   Tekton Controller + Webhook   EventListener    │
        │        (tekton-pipelines ns)    (Tekton Triggers)│
        │              │  creates PipelineRun              │
        │              ▼                                   │
        │   TaskRun Pod:        TaskRun Pod:      Shared    │
        │   git-clone   ──────▶ buildpacks/  ───▶ Workspace │
        │                       buildkit           (PVC)    │
        │                                                    │
        │   CRDs: Pipeline, Task, PipelineRun, TaskRun,       │
        │   EventListener, TriggerBinding, TriggerTemplate    │
        │   (stored natively in etcd)                          │
        └───────────────────────────────────────────────┘
                              │  push OCI image
                              ▼
                     Harbor Registry
                - Vulnerability scan
                - Image signing
                              │
                              ▼
                  Runtime-VKS Cluster
        (ArgoCD-managed application workloads —
         separate from CI, out of this document's scope)
```

A developer pushes code to the App Source repository. A webhook notifies the EventListener, which creates a PipelineRun. Tekton then runs each step of the Pipeline (`git-clone`, then `buildpacks`/`buildkit`) as an isolated Pod, sharing source code through a Workspace backed by a PersistentVolumeClaim (PVC). The resulting OCI image is pushed to Harbor for scanning and signing. Everything past this point (GitOps update, ArgoCD, ingress) belongs to the CD track and is out of scope for this document.

---

## 2. Executive Summary: What is Tekton?

Tekton is a **Cloud Native Computing Foundation (CNCF)** project that provides a Kubernetes-native framework for building CI/CD pipelines. Rather than running as an external application, Tekton extends the Kubernetes API itself: pipelines, tasks, and their executions are all native Kubernetes objects.

It executes every pipeline step inside an isolated container, which standardizes how builds are defined and lets the pipeline scale elastically with normal Kubernetes workload demand — no separate build-farm infrastructure is required.

---

## 3. Why We Are Using Tekton in This Project

- **Kubernetes-native**: runs directly on the VKS cluster we already operate — no separate CI server, VM pool, or agent fleet to patch and maintain.
- **Fits the architecture**: the wider CI/CD design (Buildpacks/BuildKit → Harbor → ArgoCD → Istio Gateway) is built around CNCF, Kubernetes-first tooling, and Tekton is the CNCF standard for pipeline orchestration.
- **RBAC reuse**: because Pipelines and Tasks are just Kubernetes objects, existing namespace-level RBAC policies control who can create or trigger builds — no separate permission system.
- **Isolation and reproducibility**: every build step runs in its own Pod, so one team's build cannot leak state into another's, and a build behaves the same way every time it runs.
- **Elastic scaling**: build load is scheduled and scaled by the same Kubernetes scheduler used for everything else on the cluster — no manual capacity planning for a separate CI farm.

---

## 4. Why This Technology Over Legacy Tools (Jenkins)

Jenkins remains a valid tool, but for a Kubernetes-first platform like VKS, Tekton has structural advantages:

| Aspect | Jenkins (legacy) | Tekton (this project) |
|---|---|---|
| Execution model | Long-lived controller + static/VM agents or plugin-based Kubernetes agents bolted on afterward | Native Kubernetes Pods created and destroyed per build step — no separate agent fleet |
| Configuration | Groovy-based Jenkinsfile / plugin ecosystem, often UI-managed | Declarative YAML CRDs (Pipeline, Task) managed with kubectl, versioned in Git like any other manifest |
| Security / RBAC | Separate Jenkins user/role system layered on top of infrastructure | Reuses native Kubernetes RBAC and namespace boundaries directly |
| Infrastructure footprint | Requires dedicated controller node(s), plugin maintenance, and agent capacity management | No heavy external runner; scales using existing cluster nodes |
| Build isolation | Depends on agent/executor configuration; contamination risk if misconfigured | Every step runs in its own isolated Pod by design (DEC-CI-02) |
| Extensibility | Plugin marketplace (large but adds maintenance/security overhead) | Reusable Tasks pulled from the Tekton Catalog/Hub, composed like building blocks |

Net effect: Tekton avoids the heavy external runner infrastructure Jenkins requires and simplifies RBAC integration within VKS namespaces, while giving us build reproducibility through per-step Pod isolation.

---

## 5. What Are CRDs, and Why Is Tekton Built on Them?

A **Custom Resource Definition (CRD)** extends the standard Kubernetes API so the cluster understands new object types beyond the built-ins (Pod, Deployment, Service, etc.). Once a CRD is installed, Kubernetes treats the new object exactly like a native one — it is stored in etcd, managed through kubectl, and governed by the same RBAC rules.

Tekton defines its core concepts entirely as CRDs:

- **Pipeline** — the CRD describing an ordered set of Tasks.
- **Task** — the CRD describing one unit of build work (a set of Steps run in one Pod).
- **PipelineRun / TaskRun** — the CRDs representing one actual execution of a Pipeline or Task.
- **EventListener / TriggerBinding / TriggerTemplate** — the CRDs (from Tekton Triggers) that turn a webhook event into a new PipelineRun.

### Why this matters

- **No external dependency**: Tekton needs no separate database or control plane of its own — it reuses the Kubernetes control plane and etcd that already exist.
- **Standard tooling**: pipelines are managed with the same kubectl commands, GitOps flows, and RBAC policies used for every other Kubernetes resource.
- **Declarative and versionable**: Pipeline/Task definitions are plain YAML, so they can be code-reviewed and stored in Git like any other manifest.

---

## 6. Sample Infrastructure Manifest (release.yaml)

This is the structural deployment manifest that installs the Tekton control plane — the dedicated namespace, the controller's ServiceAccount, and the CRDs the engine needs to operate safely on VKS. In practice this file is applied from the official Tekton release URL; the excerpt below shows the key object types it contains.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tekton-pipelines
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-pipelines-controller
  namespace: tekton-pipelines
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: pipelines.tekton.dev
spec:
  group: tekton.dev
  names:
    kind: Pipeline
    plural: pipelines
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
```

**Applied with:**

```bash
kubectl apply --filename \
  https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl get pods -n tekton-pipelines
```

---

## 7. Sample Pipeline Workflow (pipeline.yaml)

This declarative Pipeline standardizes the code-to-artifact workflow. It clones the source, then attempts a Dockerfile-less build via Buildpacks (falling back to BuildKit only when a Dockerfile is explicitly present), and the resulting OCI image is pushed toward Harbor.

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: vks-modern-app-pipeline
spec:
  params:
    - name: repo-url
      type: string
    - name: image-reference
      type: string
  workspaces:
    - name: shared-workspace
  tasks:
    - name: fetch-source-code
      taskRef:
        name: git-clone
      workspaces:
        - name: output
          workspace: shared-workspace
      params:
        - name: url
          value: $(params.repo-url)

    - name: build-and-push-artifact
      taskRef:
        name: buildpacks
      runAfter:
        - fetch-source-code
      workspaces:
        - name: source
          workspace: shared-workspace
      params:
        - name: APP_IMAGE
          value: $(params.image-reference)
```

### Pipeline Workflow Parameters Explained

| Field | Rationale & Function |
|---|---|
| `params` | Injects variables at runtime (e.g. the Git repo URL, the destination image tag in Harbor) without hardcoding values into the Pipeline definition. |
| `workspaces` | Binds a PVC-backed shared volume to the Pipeline so the `git-clone` Task can write source code to disk that the `buildpacks` Task then reads. |
| `taskRef` | References a prebuilt, standardized Task (e.g. from the Tekton Catalog) instead of redefining build logic from scratch, cutting developer overhead. |
| `runAfter` | Enforces execution order — guarantees the source code is fully cloned before the artifact builder Task starts. |
| `tasks[].name` | A unique label for that step within the Pipeline; referenced by other steps' `runAfter` and by logs/dashboards. |
| `workspaces` (per task) | Maps the Pipeline-level `shared-workspace` to the specific workspace name each Task expects internally (e.g. `output` for git-clone, `source` for buildpacks). |

### Starting a Run Manually (Local Test)

```bash
tkn pipeline start vks-modern-app-pipeline \
  -w name=shared-workspace,claimName=my-pvc \
  -p repo-url=https://github.com/org/sample-app.git \
  -p image-reference=harbor.local/team/sample-app:test

tkn pipelinerun logs -f
```

---

## 8. Official Release Manifests, Commands & References

All commands below are copied directly from the official Tekton project documentation (tekton.dev) and the tektoncd GitHub organization. These are the exact manifests to apply for a clean install — no third-party mirrors.

### 8.1 Tekton Pipelines (Core) — Required

Current official CDN (tekton.dev points here as of 2026):

```bash
kubectl apply --filename \
  https://infra.tekton.dev/tekton-releases/pipeline/latest/release.yaml

kubectl get pods --namespace tekton-pipelines --watch
```

Legacy mirror (still valid, used in most existing docs/tutorials):

```bash
kubectl apply --filename \
  https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

Install a specific version instead of `latest` (recommended for reproducible documentation — see note below):

```bash
kubectl apply --filename \
  https://infra.tekton.dev/tekton-releases/pipeline/previous/v1.15.1/release.yaml
```

Untagged variant (only if the container runtime doesn't support `image:tag@digest`, e.g. older CRI-O/OpenShift):

```bash
kubectl apply --filename \
  https://infra.tekton.dev/tekton-releases/pipeline/latest/release.notags.yaml
```

> **Version-pinning note:** `latest` always points to whatever the newest Tekton release is *at the moment the command runs*, so the same documented command can silently install a different version months apart. For anything meant to be reproduced later (like this document), pin to a specific version (e.g. `v1.15.1` LTS) instead of `latest`. Use `latest` only for quick local experimentation.

### 8.2 Tekton Triggers — Required only for webhook-based auto-trigger

```bash
kubectl apply --filename \
  https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply --filename \
  https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml

kubectl get pods --namespace tekton-pipelines --watch
```

### 8.3 Tekton Dashboard — Optional (visual PipelineRun monitoring)

Read-only mode (default, recommended for shared/production clusters):

```bash
kubectl apply --filename \
  https://infra.tekton.dev/tekton-releases/dashboard/latest/release.yaml
```

Read/write mode (lets you trigger runs from the UI — fine for local testing):

```bash
kubectl apply --filename \
  https://infra.tekton.dev/tekton-releases/dashboard/latest/release-full.yaml
```

Access locally via port-forward:

```bash
kubectl --namespace tekton-pipelines port-forward \
  svc/tekton-dashboard 9097:9097
# then open http://localhost:9097
```

### 8.4 Tekton CLI (tkn) — for running/inspecting Pipelines from a terminal

```bash
# macOS (Homebrew)
brew install tektoncd-cli

# Linux (x86_64)
curl -LO https://github.com/tektoncd/cli/releases/download/\
  v0.41.0/tkn_0.41.0_Linux_x86_64.tar.gz
tar xvzf tkn_0.41.0_Linux_x86_64.tar.gz -C /usr/local/bin tkn

tkn version
```

Check the CLI Releases page (link in the reference table below) for the current version number before running the Linux install command.

### 8.5 Verifying the Installation

```bash
kubectl get pods -n tekton-pipelines
kubectl get crd | grep tekton.dev
```

### 8.6 Uninstalling (rollback / cleanup)

```bash
kubectl delete --filename \
  https://infra.tekton.dev/tekton-releases/pipeline/latest/release.yaml
kubectl delete --filename \
  https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
```

### 8.7 Official References

| Resource | Official URL |
|---|---|
| Tekton Pipelines — Install docs | tekton.dev/docs/pipelines/install |
| Tekton Triggers — Install docs | tekton.dev/docs/triggers/install |
| Tekton Dashboard — Install docs | tekton.dev/docs/dashboard/install |
| Tekton Getting Started (Tasks) | tekton.dev/docs/getting-started/tasks |
| Tekton Catalog / Hub (reusable Tasks: git-clone, buildpacks, etc.) | hub.tekton.dev |
| Tekton Pipelines source & releases | github.com/tektoncd/pipeline |
| Tekton Triggers source & releases | github.com/tektoncd/triggers |
| Tekton Dashboard source & releases | github.com/tektoncd/dashboard |
| Tekton CLI (tkn) releases | github.com/tektoncd/cli/releases |
| Tekton Operator (managed install/upgrade of all components) | github.com/tektoncd/operator |
| CNCF project page | cncf.io/projects/tekton |

*Note: as of 2026 the Tekton project is migrating its canonical release CDN from `storage.googleapis.com` to `infra.tekton.dev`. Both currently resolve to valid release manifests; `infra.tekton.dev` is what the live tekton.dev documentation shows first, so it is used as primary in this document, with the older domain listed as a working fallback.*

---

## 9. Summary

Tekton gives this project a Kubernetes-native, CRD-based CI engine that removes the need for external build infrastructure, reuses existing RBAC boundaries, and guarantees reproducible, isolated builds per step. The Pipeline shown above (clone → build via Buildpacks/BuildKit → push) is the minimum working CI loop that can be validated locally before wiring in Harbor governance, GitOps updates, and ArgoCD delivery, which remain out of this document's scope.
