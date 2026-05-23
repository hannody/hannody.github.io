---
title: "On-Demand Namespace Error Log Watcher in a Locked-Down Kubernetes Environment"
date: 2026-05-23
draft: false
slug: "kubernetes-stern-error-watcher"
tags:
  - kubernetes
  - stern
  - logging
  - rbac
  - observability
aliases:
  - /posts/stern-ironlight-ai-error-watcher/
---

During a large-scale stress test, the last thing you want is to be hunting errors across 150 pods manually.

So I built an on-demand error log watcher for a complex Kubernetes namespace under real-world constraints, without touching anything outside the namespace, and without installing a single binary on the server.

**What we are building:** a Kubernetes Deployment, scaled to zero by default, that runs [Stern](https://github.com/stern/stern) as a container inside the cluster. It watches all pods in a target namespace, filters output to errors only, and can be started or stopped with a single `kubectl scale` command. The entire setup RBAC, ServiceAccount, and Deployment lives within the targeted namespace it monitors and leaves no trace when removed.

---

## The Environment

The namespace in question that I was stress-testing, `ironlight-ai`, runs on a private GPU cluster. The infrastructure is completely private cloud environment, Node-level access is restricted, and installing tools server-wide is not permitted — and frankly, not something I would do even if I could. Any solution I introduce must be contained, reversible, and have zero footprint beyond its own namespace.

The namespace itself is not small. It runs:

- On-Prem LLM inference services
- Vision-Language Model (VLM) pipelines
- Classical ML classification workers
- Deep learning document processors
- OCR pipelines with pre- and post-processing stages
- Backend orchestration services
- File conversion, image processing, and rule evaluation workers

At peak stress test load, this namespace runs around **150 pod replicas**. The goal of the stress tests is to burst large numbers of concurrent users through the full pipeline to expose weak points in the software stack and infrastructure before they become production incidents.

---

## The Observability Gap

I had already set up a full Grafana observability stack for this environment: Alloy for log collection, Prometheus for metrics, and Loki for log aggregation, all configured and functional. However, during high volume stress tests, log visibility for this namespace became inconsistent, but functional for other namespaces and services. Rather than interrupt the test to debug the collection pipeline, I needed a solution I could reach for immediately; something that would give me error visibility in minutes, not hours.

The observability stack will be revisited and fixed properly. Since the stress-test uncovered this gap in it. However, this stern tool exists for the time between now and then and as a backup for future incidents.

---

## Why Not Just Install Stern the Normal Way

[Stern](https://github.com/stern/stern) is the obvious tool for this. It tails logs across multiple pods and containers simultaneously, supports regex filtering, and produces clean, readable output.

The challenge is getting it running in this environment.

I tried the straightforward approaches first:

**Local install with SSH tunnel:** Installed Stern on my local machine and attempted to tunnel access through SSH and VPN to reach the cluster API. It did not work. The network topology between my machine and the cluster API server under VPN made this unreliable.

**Python helper script:** I wrote a helper to establish and maintain the connection programmatically. It did not work reliably either. At this point I had spent more time on the tooling problem than on the actual debugging I needed to do.

**Following the official Kubernetes and GitHub documentation:** I checked the standard installation paths. They assumed a level of access to the nodes and the cluster that I simply do not have them readily available in this environment.

I also considered the Kubernetes native alternatives before settling on this approach:

**`kubectl logs` with label selectors:** Works well for a handful of pods. At 150 replicas it becomes unwieldy, and it offers no log filtering — every line from every container comes through regardless of severity.

**Parallel `kubectl logs` with `xargs`:** A common pattern for watching many pods at once looks like this:

```bash
kubectl get pods -n ironlight-ai -o name \
  | xargs -n1 -P 10 kubectl logs -n ironlight-ai --all-containers --tail=100 -f
```

This is clever, but it has the same problems as the label selector approach: no filtering, pure log volume from 150 replicas. It also batches with `-P 10`, so at 150 pods you are waiting for processes to cycle rather than watching the full namespace simultaneously.

**Ephemeral debug pod:** A `kubectl debug` or temporary utility pod could run tools interactively, I have tried this approach, for another scenarios, e.g. use 'curl' commands for connectivity checks as most of my microservices images are minimal images optimized for production with no extra tooling, but it still requires either installing Stern inside the pod at runtime or building a custom image — both of which reintroduce the installation problem in a different form.

**Fix the Loki ingestion issue first:** The correct long-term answer. Not the right call during an active stress test.

I stopped pursuing these paths when I recognised a pattern: every approach either required access I did not have, depended on local connectivity I could not trust, or introduced more complexity than the problem warranted.

That is the wrong direction. The solution should make my day easier, not more complicated.

---

## The Design Criteria

Before writing a single line of YAML, I defined what an acceptable solution looked like:

1. **Namespace-contained.** No resources, no side effects, nothing outside `ironlight-ai`.
2. **No server-wide installation.** Nothing touches the nodes or the cluster outside my namespace.
3. **On-demand only.** It should run when I need it and consume zero cluster resources when I do not.
4. **Minimal RBAC surface.** Only the permissions strictly necessary — nothing broader.
5. **Readable output.** During a stress test I need to see errors fast. The output must be clean.
6. **Reversible.** I can remove the entire thing with one command and leave no trace.

The solution that met all of these criteria: **run Stern as a container inside the cluster itself**, deployed as a Kubernetes Deployment scaled to zero by default.

---

## Why a Deployment Scaled to Zero

A standalone Pod would work, but it has no scaling controller. Once created it runs until deleted.

A Deployment with `replicas: 0` solves this cleanly:

- The configuration lives in the cluster permanently
- No pod runs, no log streams are open, no API calls are made
- When I need it, I scale to 1 with a single command
- When I am done, I scale back to 0
- The Deployment itself stays, ready for next time

This is the operational model I wanted: **always available, never running unless explicitly started.**

---

## RBAC Design

Because Stern runs as a pod inside the cluster, it needs permission to read pod logs. I created a `ServiceAccount`, `Role`, and `RoleBinding` scoped exclusively to the `ironlight-ai` namespace.

Note the use of `Role`, not `ClusterRole`. A `ClusterRole` would grant permissions across the entire cluster. A namespace-scoped `Role` limits access to only the namespace it is defined in which is exactly the boundary I wanted.

This also means **the entire setup must be deployed into the same namespace you want to monitor**. The `ServiceAccount`, `Role`, `RoleBinding`, and `Deployment` all carry `namespace: ironlight-ai`. If you are adapting this for your own use, replace every occurrence of `ironlight-ai` with your target namespace. Stern will then watch only that namespace, with permissions that cannot reach anything outside it.

The Role grants only what Stern needs:

```yaml
resources:
- pods
- pods/log

verbs:
- get
- watch
- list
```

Nothing else. No cluster-wide permissions. No write access.

---

## Handling logging for 150 Replicas

Stern has a default limit of 50 concurrent log streams. With 150 pod replicas in `ironlight-ai`, it exits immediately:

```text
Error: stern reached the maximum number of log requests (50), use --max-log-requests to increase the limit
```

The fix is straightforward:

```bash
--max-log-requests 200 # HALT! Higher numbers could risk throttling or stressing the Kubernetes API server.
```

This gives enough headroom for the full namespace at peak replica count.

---

## Filtering for What Matters

During a stress test, I do not want to see every log line from 150 pods. I want errors, and only errors.

Different services in this namespace log at different severity levels and with different casing conventions:

```text
ERROR / Error / error
CRITICAL / Critical / critical
FATAL / Fatal / fatal
```

A case-insensitive regex covers all of them:

```regex
(?i)error|critical|fatal
```

I also exclude the Stern pod itself from its own log watch to avoid a feedback loop:

```bash
--exclude-pod stern.*
```

And I use `--tail 0` so Stern does not replay historical logs on startup — only new lines from the moment it starts.

---

## Output Template

The default Stern output includes full pod names and container metadata. That is useful in some contexts, but during a stress test it is noise due to replicas.

I used a minimal Go template:

```gotemplate
{{printf "%s %s\n" .ContainerName .Message}}
```

This produces:

```text
backend 2026-05-22 13:23:25,094 ERROR streams.document.views 1619 ...
ocr 2026-05-22 13:23:31,201 CRITICAL pipeline.processor 88 ...
```

Service name, timestamp, severity, location, message. Everything I need, nothing I do not.

---

## The Full YAML 🫡

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: stern
  namespace: ironlight-ai
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: stern
  namespace: ironlight-ai
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: stern
  namespace: ironlight-ai
subjects:
- kind: ServiceAccount
  name: stern
roleRef:
  kind: Role
  name: stern
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stern
  namespace: ironlight-ai
spec:
  replicas: 0
  selector:
    matchLabels:
      app: stern
  template:
    metadata:
      labels:
        app: stern
    spec:
      serviceAccountName: stern
      containers:
      - name: stern
        image: ghcr.io/stern/stern:v1.34.0
        args:
        - "."
        - "-n"
        - "ironlight-ai"
        - "--include"
        - "(?i)error|critical|fatal"
        - "--exclude-pod"
        - "stern.*"
        - "--tail"
        - "0"
        - "--max-log-requests"
        - "200"
        - "--only-log-lines"
        - "--template"
        - '{{printf "%s %s\n" .ContainerName .Message}}'
```

---

## Applying and Using It

> **Before applying:** make sure every `namespace:` field in the YAML matches the namespace you want to monitor. All resources, the ServiceAccount, Role, RoleBinding, and Deployment must live in the same namespace as the pods you are watching. The image `ghcr.io/stern/stern:v1.34.0` is publicly available; this solution was tested against this version at time of writing. In a restricted deployment environment, substitute your internal registry path.

Apply once:

```bash
kubectl apply -f stern-error-watcher.yaml
```

Start watching:

```bash
kubectl scale deployment -n ironlight-ai stern --replicas=1
kubectl logs -n ironlight-ai -f deployment/stern
```

Stop:

```bash
kubectl scale deployment -n ironlight-ai stern --replicas=0
```

Check status:

```bash
kubectl get deployment -n ironlight-ai stern
kubectl get pods -n ironlight-ai -l app=stern
```

---

## What This Is and What It Is Not

This is not a replacement for a properly functioning Grafana/Loki observability stack. That stack exists, it covers the full environment, and it will be fixed properly.

This is a deliberate tool built for a specific scenario: active stress testing, high replica counts, and a need for immediate error visibility scoped to one namespace.

It is:

- Entirely self-contained within the namespace
- Zero-cost when not running
- Removable without side effects
- Scoped to the minimum necessary RBAC permissions
- Fast to start and fast to stop

In a constrained environment, the right solution is not always the one the documentation recommends. Sometimes it is the one that fits cleanly inside the boundaries you are given 🙄 and leaves no mess behind.

---

## References

- [Tail Kubernetes with Stern](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/)
- [Stern — Kubernetes log tailing](https://github.com/stern/stern)
- [Kubernetes ServiceAccounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes Deployment Scaling](https://kubernetes.io/docs/tasks/run-application/scale-deployment/)
