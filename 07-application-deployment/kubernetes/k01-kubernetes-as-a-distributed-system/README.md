<!-- markdownlint-disable MD001 -->
<!-- markdownlint-disable MD024 -->
<!-- markdownlint-disable MD025 -->
<!-- markdownlint-disable MD026 -->
<!-- markdownlint-disable MD029 -->
<!-- markdownlint-disable MD033 -->
<!-- markdownlint-disable MD040 -->
<!-- markdownlint-disable MD056 -->
<!-- markdownlint-disable MD060 -->

# K01. Kubernetes as a Distributed System

## Contents

- 01. Running example
- 02. Glossary
- 03. Components and where they run
- 04. K01.01 The problem
- 05. K01.02 Declarative desired state
- 06. K01.03 Level-triggered vs edge-triggered control
- 07. K01.04 The control loop
- 08. K01.05 Eventual consistency inside the cluster
- 09. K01.06 Failure model
- 10. K01.07 CAP framing
- 11. K01.08 Where Kubernetes stops
- 12. Owner questions and answers
- 13. Interview questions and model answers
- 14. Exercises and model answers
- 15. One-page summary

---

## 01. Running example

Used throughout this module:

- A service called `orders`.
- It must have **3 copies** running at all times.
- The cluster has **4 machines**: m1, m2, m3, m4.

---

## 02. Glossary

### 02.01 The cluster

| Term          | Meaning in Kubernetes                                                                                     |
| ------------- | --------------------------------------------------------------------------------------------------------- |
| Cluster       | A set of machines managed as one pool by Kubernetes                                                       |
| Node          | One machine (VM or physical) in the cluster that runs workloads                                           |
| Control plane | The "brain" processes that store desired state and make decisions; separate from the nodes doing the work |
| Pod           | The smallest unit Kubernetes runs: one or more containers scheduled together on one node (K05)            |

### 02.02 Where state lives

| Term                 | Meaning                                                                                             |
| -------------------- | --------------------------------------------------------------------------------------------------- |
| Object               | A record of something in the cluster; every object has a kind, a name, a spec and a status          |
| Kind                 | The type of an object (Pod, Deployment, Service); each gets its own module                          |
| spec                 | The desired state of an object, written by the user                                                 |
| status               | The observed state of an object, written by the system                                              |
| API server           | The only front door; every read and write of every object goes through it                           |
| etcd                 | The database behind the API server; stores all objects, replicated across 3 or 5 members            |
| Raft                 | The consensus algorithm etcd uses so its members agree on every write, even when some fail          |
| Watch                | A long-lived connection to the API server: "notify me whenever objects of this kind change"         |
| Event (notification) | One change notification on a watch: added, modified or deleted                                      |
| Event (kind)         | A different thing with the same name: an object recording what happened, seen in `kubectl describe` |

### 02.03 How things get done

| Term          | Meaning                                                                                                      |
| ------------- | ------------------------------------------------------------------------------------------------------------ |
| Controller    | A program running a loop for one concern: compare spec with reality and act to close the gap                 |
| Reconcile     | One pass of that loop for one object: read desired, read actual, fix the difference                          |
| Informer      | A controller's local in-memory copy of the objects it cares about; filled by one list, kept fresh by a watch |
| List / relist | Fetching all objects of a kind in one call; done at startup and after a broken watch                         |
| Resync        | A periodic re-check of every cached object, even without a notification; a safety net                        |
| Work queue    | A queue of object keys waiting to be reconciled; duplicate keys collapse into one entry                      |
| Key           | An object's address, `namespace/name`, for example `prod/orders`                                             |
| Namespace     | A named group of objects inside a cluster, such as `prod` or `dev` (K21)                                     |
| Scheduler     | The control-plane component that picks a node for each new pod                                               |
| kubelet       | The agent on every node that starts and stops containers there and reports back                              |
| Lease         | A small record used as a lock with an expiry; used for leader election and node heartbeats                   |
| Idempotent    | Doing it twice has the same result as doing it once                                                          |

### 02.04 How they connect for one change

1. A user submits a spec; the API server validates it and stores it in etcd.
2. The watch sends a notification to the controllers' informers.
3. The informer updates its cache and pushes the object's key onto the work queue.
4. A controller reconciles that key; if action is needed it writes new or changed objects back through the API server (for example "create 2 pods").
5. The scheduler assigns those pods to nodes; each node's kubelet starts the containers.
6. Reality changes, status is updated, and the loop continues.

---

## 03. Components and where they run

### 03.01 Map

```text
                    kubectl, CI, GitOps (outside the cluster)
                                  |
                                  v  HTTPS
+------------------------ CONTROL-PLANE NODES (3 or 5) ------------------------+
|  kube-apiserver  <---->  etcd                                                 |
|        ^                                                                      |
|        |  watch / write                                                       |
|  kube-scheduler   kube-controller-manager   cloud-controller-manager          |
+-------------------------------------------------------------------------------+
                                  ^
                                  |  watch / status (nodes talk only to the API server)
+----------------------------- WORKER NODES (many) ----------------------------+
|  kubelet  --CRI-->  container runtime (containerd / CRI-O)  -->  containers   |
|  kube-proxy (Service network rules)     CNI plugin (pod networking)           |
|  Add-on pods: CoreDNS, metrics-server, ingress/gateway controller, CSI, ...   |
+-------------------------------------------------------------------------------+
```

### 03.02 Control-plane components (on control-plane nodes)

| Component                | What it does                                                                                    | Copies active                | Example                                   |
| ------------------------ | ----------------------------------------------------------------------------------------------- | ---------------------------- | ----------------------------------------- |
| kube-apiserver           | Front door: authentication, authorization, validation, policy checks, storage to etcd, watches  | All (behind a load balancer) | `kubectl apply` lands here                |
| etcd                     | Database of every object; Raft-replicated; majority needed for writes                           | All store; one leads writes  | "orders: 3 copies" is saved here          |
| kube-scheduler           | Picks a node for every pod that has none, using free resources and placement rules              | One leader                   | Places the new `orders` copy on m4        |
| kube-controller-manager  | One process hosting all built-in controllers (replicas, rollouts, nodes, jobs, namespaces, GC)  | One leader                   | Sees 2 of 3 copies and creates 1 more     |
| cloud-controller-manager | Cloud loops: cloud load balancers, removing nodes whose VM is gone, routes; cloud clusters only | One leader                   | Creates an AWS load balancer for `orders` |

etcd usually runs on the control-plane nodes ("stacked"); large clusters may give it dedicated machines ("external etcd") to isolate disk and CPU.

### 03.03 Node components (on every worker node)

| Component         | What it does                                                                                                                            | Example                                               |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| kubelet           | Node agent: watches pods assigned to its node, starts and stops them, runs health checks, mounts volumes, reports status and heartbeats | Starts `orders` on m4 and reports "running, ready"    |
| Container runtime | Pulls images and creates containers (containerd or CRI-O, using runc); called by the kubelet through CRI                                | Pulls `orders:v2` and starts the process              |
| kube-proxy        | Programs node network rules so a stable Service address reaches a healthy pod; optional if the CNI replaces it                          | Traffic to "orders" on m1 is forwarded to a pod on m3 |
| CNI plugin        | Gives each pod an IP and connects pods across nodes (Calico, Cilium, cloud CNIs)                                                        | The `orders` pod on m4 gets IP 10.0.4.7               |

### 03.04 Cluster add-ons (pods, usually in `kube-system`, on worker nodes)

| Add-on                         | What it does                                                    |
| ------------------------------ | --------------------------------------------------------------- |
| CoreDNS                        | Cluster DNS: `orders.prod.svc.cluster.local` to an address      |
| metrics-server                 | CPU and memory per pod; feeds `kubectl top` and autoscaling     |
| Ingress / Gateway controller   | Implements external HTTP entry (NGINX, Envoy Gateway, cloud LB) |
| CSI driver                     | Creates and attaches disks (central part plus per-node part)    |
| Cluster Autoscaler / Karpenter | Adds and removes nodes based on pods that cannot be scheduled   |
| Log and metrics agents         | Ship logs and metrics off each node (Fluent Bit, node-exporter) |

### 03.05 How each component is run

| Component                                        | Typically runs as                                                                                    |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------- |
| kubelet, container runtime                       | Plain OS service (systemd), not a pod; something must exist before pods can                          |
| API server, etcd, scheduler, controller-manager  | kubeadm: static pods started by the kubelet from files on disk; managed clusters: hidden by provider |
| kube-proxy, CNI agent, CSI node part, log agents | One pod on every node (DaemonSet, K08)                                                               |
| CoreDNS, metrics-server, installed controllers   | Ordinary replicated pods on worker nodes (Deployment, K06)                                           |

With kubeadm, control-plane nodes also run a kubelet and a container runtime; that is how they run the control-plane components as pods.

### 03.06 Who talks to whom

- Everyone (users, controllers, scheduler, kubelets, add-ons) talks only to the API server.
- Only the API server talks to etcd.
- The API server calls a kubelet directly only for `logs`, `exec` and `port-forward`.
- The kubelet talks to the container runtime over a local socket (CRI).

This hub-and-spoke shape is why components can fail, restart or move independently.

---

## 04. K01.01 The problem

### 04.01 Three forces

1. **Utilization:** machines cost money; keep them full.
2. **Failure:** machines, disks and networks fail constantly; work must move.
3. **Change:** teams deploy many times a day; placement cannot be a manual act.

### 04.02 Without a cluster manager (static assignment)

- A human decides: copy 1 on m1, copy 2 on m2, copy 3 on m3.
- m2 dies at 3 a.m.; someone is paged and starts a new copy on m4.
- A new version ships; someone updates m1, m2, m3 one by one.

Resulting problems:

| Problem                | Why                                                                                        |
| ---------------------- | ------------------------------------------------------------------------------------------ |
| Waste                  | Each service sized for its own peak; spare capacity cannot be shared                       |
| Failure needs a human  | At fleet scale failures are daily: 10,000 machines with a 3-year MTBF fail about 9 per day |
| Drift and slow deploys | Hand-tuned hosts diverge; "works on host 3, fails on host 7"                               |

### 04.03 With Kubernetes

- The user states only the need: "orders needs 3 copies, each 1 CPU and 1 GB". No machine is named.
- The scheduler bin-packs pods onto machines with free room. Multi-dimensional bin packing is NP-hard, so it uses heuristics: filter feasible nodes, score them, pick the best.
- Controllers detect failures and replace lost work automatically; a dead machine becomes a routine event.
- Containers make workloads portable and immutable, which makes moving them cheap.

**In one sentence:** Kubernetes turns many unreliable machines into one pool and automatically keeps the stated needs true inside that pool.

---

## 05. K01.02 Declarative desired state

### 05.01 Two styles

| Style       | You send                               | System does                                  |
| ----------- | -------------------------------------- | -------------------------------------------- |
| Imperative  | Steps: "start a copy on m1, m2, m3"    | Executes each step once, in order            |
| Declarative | End result: "there should be 3 copies" | Compares target with reality, closes the gap |

### 05.02 Why steps break under partial failure

A script starts copies on m1 and m2, then crashes before m3:

- Rerun it: duplicates or "already exists" errors.
- Do not rerun it: 2 copies, and no record of what was missed.
- Being correct requires encoding every partial-failure path; scripts never do.

### 05.03 Why the end result works

The goal "3 copies" is stored durably in etcd, not in the memory of a running script. Whoever acts:

1. Reads the goal (3).
2. Counts reality (2).
3. Does only the difference (start 1).

- Crash mid-way: the next run recomputes a smaller diff and finishes.
- Apply the same goal twice: the diff is empty; nothing happens. Declarative apply is idempotent by construction.
- Concurrent actors: both compute against the same goal and converge.

### 05.04 spec and status

- **spec:** what you want ("3 copies"); written by the user.
- **status:** what the system currently observes ("2 running"); written by controllers.
- The system's whole job is to make status match spec.
- Objects carry a `generation` (bumped on each spec change) and controllers report `observedGeneration`; equal values mean the controller has *seen* the latest change, not that it is done.

### 05.05 The trap: success means "intent recorded"

- `kubectl apply` returns after the API server validates, admits and stores the spec. Pods may not exist yet.
- Real failures (image not found, no room on any node, failing health checks) arrive asynchronously in status conditions and Events.
- To know it worked, wait on status (`kubectl rollout status`, `kubectl wait --for=condition=...`, GitOps health). A CI pipeline that stops at `apply` reports a false green.

### 05.06 Ordering and imperative commands

- Ordering is not guaranteed and does not need to be: apply a workload before the config it mounts and it errors until the config appears, then the next reconcile succeeds.  
  Eventual convergence replaces strict ordering.
- Imperative `kubectl` commands (for example `kubectl scale`) still just edit the stored spec.  
  "Imperative" describes how a change is authored, not how it is executed. An imperative edit not recorded in git is reverted by the next apply or GitOps sync (drift).

### 05.07 Distributed-systems angle

- spec is the source of truth: durable, Raft-replicated.
- status is a cached observation that may be stale.
- Reality (processes on nodes) is what matters and is always slightly ahead of or behind status.
- GitOps applies the same idea one level up: git holds the desired state of the cluster's desired state.

---

## 06. K01.03 Level-triggered vs edge-triggered control

### 06.01 The two models

| Model           | Reacts to                     | Controller logic                  |
| --------------- | ----------------------------- | --------------------------------- |
| Edge-triggered  | The news (a change event)     | "A copy died, so start one copy." |
| Level-triggered | The current situation (state) | "Goal 3, running 1, so start 2."  |

A thermostat is level-triggered: it reads the temperature and does not care how many times the door opened.

### 06.02 What a distributed system does to news

- **Lost:** controller restarts, watch drops, network partitions.
- **Duplicated:** retries, relists after reconnect.
- **Reordered:** different paths, different latency.

### 06.03 Scenario

The controller restarts; while it is down, the copies on m2 and m3 both die.

| Approach        | What the controller knows    | What it does | Result                  |
| --------------- | ---------------------------- | ------------ | ----------------------- |
| Edge-triggered  | It saw only 1 "died" message | Starts 1     | 2 copies, wrong forever |
| Level-triggered | Counts: goal 3, running 1    | Starts 2     | 3 copies, correct       |

A "died" message delivered twice:

| Approach        | What it does                      | Result            |
| --------------- | --------------------------------- | ----------------- |
| Edge-triggered  | Starts a copy per message         | One copy too many |
| Level-triggered | Recounts: already 3, does nothing | Correct           |

### 06.04 What Kubernetes does: a hybrid

1. **Edge-triggered delivery for speed:** controllers are notified via watches within milliseconds.
2. **Level-triggered processing for correctness:** a notification only says which object to look at. The work queue stores the key (`namespace/name`), never the event.
3. **Free deduplication:** many notifications for one object collapse into one reconcile.
4. **Safety nets:** a full list on startup and periodic resyncs re-check everything.

```
loop:
    key = queue.next()                 // "prod/orders": which object, not what happened
    desired = readSpec(key)            // from local cache
    actual  = observeWorld(key)        // from local cache
    diff    = compare(desired, actual)
    if diff: act(diff)                 // must be idempotent
    on error: queue.requeueWithBackoff(key)
```

**Rule:** news is only a doorbell; the decision always comes from comparing goal and reality.

### 06.05 Rules for every controller

- Never trust the event payload; read current state.
- Never count events.
- Make every action idempotent (create-if-absent, set-to-value rather than increment).
- On failure, requeue the key with backoff.

### 06.06 Cost and mitigation

- Full reads are expensive, so controllers read a local informer cache, not the API server.
- The cache can be stale: a controller may create pods, not yet see them, and create them again.
- Mitigations: **expectations** (the controller remembers "I just created 2, waiting to see them") and **optimistic concurrency** (writes based on stale data are rejected, K03).

---

## 07. K01.04 The control loop

### 07.01 Anatomy

1. **Observe:** read goal (spec) and reality from the local cache.
2. **Compare:** compute the difference.
3. **Act:** take a small step toward the goal by writing to the API server.
4. **Report:** update status.

Then wait for the next doorbell and repeat.

- **Small steps, not one big plan:** after a crash, the next loop recomputes from reality; nothing is lost.
- **Idempotent actions:** "ensure 3 exist" is safe to repeat; "add 1" is not.

### 07.02 Many small loops, no boss

There is no central workflow engine. A chain of independent controllers does the work; each watches its own input and writes its own output.

"orders, version 2, 3 copies":

| Step | Who acts                 | Watches for                      | Writes                                  |
| ---- | ------------------------ | -------------------------------- | --------------------------------------- |
| 01   | User                     | n/a                              | The goal: "orders v2, 3 copies"         |
| 02   | Rollout controller       | Goals for orders                 | A "group of 3 copies of v2" record      |
| 03   | Replica controller       | Groups whose count is short      | 3 pod records (no machine assigned yet) |
| 04   | Scheduler                | Pods with no machine             | A machine name on each pod              |
| 05   | kubelet on that machine  | Pods assigned to its own machine | Starts containers; reports "running"    |
| 06   | Each controller upstream | Status changes                   | Its own status: "3 of 3 ready"          |

Steps 02 and 03 are the Deployment and ReplicaSet controllers (K06). The workflow exists only as a chain of records in the API. This style is **choreography**.

### 07.03 Choreography vs orchestration

| Aspect                | Central orchestrator                  | Kubernetes control loops                |
| --------------------- | ------------------------------------- | --------------------------------------- |
| Who drives            | One engine runs the whole flow        | Each controller reacts to state         |
| Failure mid-way       | Engine must remember where it stopped | Next loop recomputes from current state |
| Adding behaviour      | Change the engine                     | Add a controller; others are untouched  |
| Complexity per piece  | High (knows the whole flow)           | Low (knows one job)                     |
| Seeing the whole flow | Easy, in one place                    | Hard; follow the chain of records       |

### 07.04 Costs

- **Spread-out debugging:** "orders is not running" means walking the chain: goal saved, group created, pods created, machines assigned, kubelet started them (K24).
- **Controllers fighting:** an autoscaler sets copies to 5, a deploy tool resets to 3, forever. Rule: each field has exactly one owner (K03).
- **Emergent behaviour:** many loops interacting can surprise, for example a loop that keeps recreating something meant to be deleted.

### 07.05 Rules for controllers

- Act only through the API server, never by calling other controllers.
- Own a clear set of records or fields; leave everything else alone.
- Make every action idempotent; report progress in status.

---

## 08. K01.05 Eventual consistency inside the cluster

### 08.01 Definitions

- **Strong consistency:** every reader sees the latest write immediately.
- **Eventual consistency:** readers may briefly see old data; if writes stop, everyone converges.

### 08.02 Kubernetes is both

| Layer                                 | Consistency | Why                                                     |
| ------------------------------------- | ----------- | ------------------------------------------------------- |
| etcd plus API server (stored records) | Strong      | Raft majority agreement on every write                  |
| Everything that copies those records  | Eventual    | Each copy is refreshed by watch notifications, with lag |

Copies include controller caches, each kubelet's view of its pods, each node's network rules (kube-proxy), DNS answers, and status fields.  
Each updates independently, so parts of the cluster briefly disagree.

### 08.03 Example: a dead copy of `orders`

| Time       | Event                                                                            |
| ---------- | -------------------------------------------------------------------------------- |
| t0         | Container on m2 dies                                                             |
| t0 + ~1 s  | m2's kubelet marks the pod not ready via the API server                          |
| shortly    | A controller removes it from the "healthy backends for orders" list              |
| t0 + 1–2 s | Network rules on m1 to m4 update, each at a slightly different moment            |
| meanwhile  | A request reaching m4 before its rules update is sent to the dead copy and fails |

This is the normal propagation window; designs must tolerate it.

### 08.04 Where it bites

- **Read right after write:** listing pods immediately after apply shows old ones; wait on conditions.
- **Nodes disagree briefly:** one routes to old backends, another to new; the root of rolling-update errors without graceful shutdown (K05).
- **Stale controller caches:** guarded by optimistic concurrency (K03) and expectations.
- **Config lands late:** mounted config files update in running pods after up to about a minute (K10).

### 08.05 Designing for it

1. Never assume instant; wait for status, not the API response.
2. Shut down gracefully: keep serving a few seconds until every node stops routing to you.
3. Clients retry idempotent requests.
4. Treat convergence time as a design number:

| Propagation                                  | Typical lag              |
| -------------------------------------------- | ------------------------ |
| Watch notification to controller caches      | milliseconds to ~1 s     |
| Network rules updated on every node          | ~1 s to a few seconds    |
| DNS answers                                  | seconds (cached)         |
| Mounted config file updates in a running pod | up to ~1 minute          |
| Dead machine detected and its pods replaced  | minutes by default (K04) |

These grow under heavy load and in very large clusters.

---

## 09. K01.06 Failure model

Kubernetes assumes **crash failures**: a component works, stops, or goes silent.  
It does not defend against components that lie (Byzantine faults); that is a security concern (K20).

### 09.01 What fails and what Kubernetes does

| Failure                                       | Who notices                          | What Kubernetes does                                           | What stays your job                                       |
| --------------------------------------------- | ------------------------------------ | -------------------------------------------------------------- | --------------------------------------------------------- |
| App process crashes                           | kubelet                              | Restarts in place with growing back-off                        | Fix the bug; a crash loop is still an outage of that copy |
| App hangs (alive, not working)                | Nobody, unless a health check exists | With a liveness check: restarts it                             | Define health checks correctly (K05)                      |
| App alive but not ready                       | Readiness check                      | Stops sending traffic; does not restart                        | Define readiness correctly                                |
| Pod removed (deleted, evicted)                | Replica controller (count is short)  | Creates a replacement, often elsewhere                         | Accept new IP and lost local disk; externalize state      |
| Machine dies                                  | Node controller (heartbeats stop)    | Marks unhealthy, waits, replaces its pods (minutes by default) | Keep spare capacity; spread copies                        |
| Network cut between machine and control plane | Node controller (identical to dead)  | Same as a dead machine                                         | Handle "old copy may still run" (09.03)                   |
| Whole zone down                               | Node controller (many machines)      | Replaces pods in surviving zones if there is room              | Spread across zones; headroom (K17, K18)                  |
| One control-plane node dies                   | Other control-plane nodes            | API keeps serving; standby leader takes over                   | Run 3 or 5 across zones                                   |
| Whole control plane or etcd majority lost     | Everyone (API stops answering)       | Nothing new happens (09.02)                                    | Restore it; back up etcd (K02, K25)                       |

### 09.02 Control plane down: frozen, not dead

The data plane (worker nodes and running apps) does not need the control plane minute to minute.

- **Keeps working:** running pods serve traffic; the kubelet restarts crashed containers locally from its last instructions; network rules stay as last set.
- **Stops:** deploys, `kubectl`, scaling; replacement of pods on dead machines; routing updates (traffic may keep going to copies that died during the outage).

Every extra minute of outage makes the cluster more fragile because nothing heals.

### 09.03 Slow and dead look the same

Death is detected by timeout (no heartbeat for N seconds). A crashed machine, a partitioned machine and a very slow machine are indistinguishable.

| Timeout | Benefit          | Cost                                               |
| ------- | ---------------- | -------------------------------------------------- |
| Short   | Fast replacement | False alarms: healthy copies replaced after a blip |
| Long    | Few false alarms | Slow recovery when the machine really is dead      |

- Stateless `orders`: m2 is cut off, a replacement starts on m4 while m2's copy may still run; briefly 4 copies; harmless.
- Single-writer database: the same thing gives two writers (split-brain) and corrupts data.  
  Kubernetes deliberately refuses to replace that kind of workload until the old machine is confirmed dead (K07).

**Rule:** when a machine looks dead, Kubernetes favours replacing stateless work and protecting stateful work.

---

## 10. K01.07 CAP framing

### 10.01 CAP

During a network partition each side must choose:

- **Consistency (C):** never return stale or conflicting data, even if that means refusing.
- **Availability (A):** always answer, even if stale or conflicting.

Partitions will happen, so the real question is which side each component takes.

### 10.02 CP brain, AP body

| Part                                  | Choice | During a partition                                                            |
| ------------------------------------- | ------ | ----------------------------------------------------------------------------- |
| etcd plus API server                  | CP     | Only the majority side accepts writes; the minority refuses                   |
| Controllers and scheduler             | CP     | Act only while holding a valid leader lease; a leader that cannot renew stops |
| Kubelets, network rules, running apps | AP     | Keep running and serving on last known instructions, even if stale            |

The brain must never make conflicting decisions (two leaders, double scheduling, two writers), so it stops rather than guesses.  
The body keeps serving users on what it last knew.

### 10.03 Zone C cut off (3 control-plane nodes, one per zone)

| Component in zone C     | What happens                                                                                     |
| ----------------------- | ------------------------------------------------------------------------------------------------ |
| etcd member             | Minority (1 of 3): rejects writes; A and B keep a majority and continue                          |
| API server              | Cannot complete writes or up-to-date reads; `kubectl` against it fails                           |
| Leaders (if in C)       | Cannot renew leases, so they stop; new leaders are elected in A or B within seconds              |
| Worker nodes            | Heartbeats do not reach the majority; A/B mark them unhealthy and recreate stateless pods in A/B |
| Copies of `orders` in C | Keep running and serving any clients that can still reach them                                   |

After healing: zone C's kubelets reconnect, see their pods were replaced, stop them, and the cluster converges to exactly 3 copies.  
The brief "more than 3" window is fine for stateless work and unacceptable for single-writer state (K07).

### 10.04 PACELC: latency vs consistency without partitions

Every etcd write waits for a majority to persist it to disk. Therefore:

- Keep etcd members close (one region, low-latency zones).
- Do not stretch one cluster across regions; run one cluster per region and coordinate above them (K25).

### 10.05 What it does not decide

Kubernetes being CP-brain/AP-body says nothing about the application.  
Whether `orders` is consistent during a partition depends on its own database and design.

---

## 11. K01.08 Where Kubernetes stops

Kubernetes provides primitives, not a finished platform: "a platform for building platforms".

### 11.01 What it gives vs what you add

| Concern                              | Kubernetes provides                          | You add                                                             |
| ------------------------------------ | -------------------------------------------- | ------------------------------------------------------------------- |
| Building images                      | Nothing                                      | CI pipeline: build, test, scan, push                                |
| Deploy pipeline and promotion        | A basic rolling-update mechanism             | GitOps (Argo CD, Flux), canary tooling (Argo Rollouts), approvals   |
| Packaging and per-env config         | Plain records only                           | Helm or Kustomize                                                   |
| Pod network                          | Rules only ("every pod gets an IP")          | A CNI plugin (Calico, Cilium, cloud) (K12)                          |
| External HTTP entry                  | API for routing rules only                   | A controller implementing it (NGINX, Envoy Gateway, cloud LB) (K14) |
| Storage                              | API for requesting disks only                | A CSI driver (K16)                                                  |
| Service-to-service security, retries | Plain connections, no encryption, no retries | Service mesh or app libraries (Resilience4j) (K15)                  |
| Secrets                              | Base64-encoded storage by default            | Encryption at rest, external vault or secret manager (K10)          |
| Observability                        | `kubectl logs`, basic CPU/memory numbers     | Prometheus, log aggregation, tracing, dashboards, alerts (K24)      |
| Node autoscaling                     | Pod autoscaling only                         | Cluster Autoscaler or Karpenter (K19)                               |
| Running databases well               | Primitives for stable identity and disks     | An operator or a managed database (K07, K22)                        |
| Backup and disaster recovery         | Nothing for app data                         | Velero, database backups, tested restores (K25)                     |
| Multi-cluster, multi-region          | One cluster only                             | Fleet tooling, global load balancing, data replication (K25)        |
| Policy and governance                | Basic RBAC                                   | Policy engines (Kyverno, Gatekeeper) (K20)                          |

### 11.02 The application contract

Kubernetes can only automate an application that cooperates. `orders` must:

1. Stop gracefully on SIGTERM: finish in-flight requests, then exit.
2. Expose honest health endpoints: "am I alive?" and "can I take traffic?".
3. Be disposable: no important state on local disk; tolerate being killed and moved at any time.
4. Tolerate duplicates and retries.
5. Read config from environment or files, not values baked into the image.
6. Log to stdout/stderr.
7. Declare CPU and memory needs honestly (K11).

An application that breaks this contract stays fragile on Kubernetes; Kubernetes just restarts it faster.

### 11.03 When Kubernetes is the wrong choice

- Small team, few services: managed container platforms (Cloud Run, ECS Fargate, Azure Container Apps) give most of the benefit for far less operational load.
- One heavy stateful system with no mature operator: a managed service is usually safer.
- Very spiky or rare workloads: serverless functions may fit better.

Kubernetes earns its cost with many services, many teams, and a need for one consistent platform.

---

## 12. Owner questions and answers

### 12.01 Where does the "controller" live? Is there a dedicated machine for it?

A controller is code running as a process, not a machine.

- **Built-in controllers** are compiled into one program, `kube-controller-manager`, which runs on the control-plane nodes. There is no machine per controller; each controller is a separate loop inside that one process.
- **Only one copy is active.** The copies hold an election through a Lease lock; the winner runs the loops, the others wait. If the leader's machine dies, a standby takes over within seconds. Two active copies would double-act.
- **User applications normally do not run on control-plane nodes**; those nodes are marked so the scheduler keeps ordinary pods away.
- **Add-on controllers** (ingress controller, cert-manager, database operators, custom operators) run as ordinary pods on worker nodes.
- **Managed Kubernetes** (EKS, GKE, AKS) hides the control-plane machines; only the API server address is exposed.
- **Dev clusters** (kind, minikube, k3s) run everything on one machine.
- **Location does not matter to the design:** a controller talks only to the API server, so it can run anywhere that can reach it.
- **The kubelet is a controller too,** one per node, reconciling only the pods assigned to its own node.

### 12.02 Are control-plane nodes usually more than one, in case one dies?

Yes, and the count is almost always odd (3 or 5) because etcd needs a majority to accept writes.

| Control-plane nodes | Majority needed | Failures survived |
| ------------------- | --------------- | ----------------- |
| 1                   | 1               | 0                 |
| 2                   | 2               | 0                 |
| 3                   | 2               | 1                 |
| 4                   | 3               | 1                 |
| 5                   | 3               | 2                 |

An even count adds cost without adding fault tolerance. 3 is standard; 5 survives two simultaneous failures.

- API server: all copies active behind a load balancer; traffic shifts to live ones.
- Scheduler and controller-manager: one leader; a standby takes over.
- etcd: works while a majority is alive; below that it refuses writes and the cluster cannot change state.

Place the nodes in 3 different zones so one zone outage removes only one. Even with the whole control plane down, running apps keep serving (09.02).

### 12.03 Is only one of them the "real manager" at any time?

No. There is no single manager node; leadership is per component, not per machine.

| Component          | Active at once                   | Leader chosen by                          |
| ------------------ | -------------------------------- | ----------------------------------------- |
| API server         | All copies                       | No leader; load balancer spreads requests |
| controller-manager | One                              | A Lease lock stored through the API       |
| scheduler          | One                              | Its own separate Lease lock               |
| etcd               | All store data; one leads writes | Raft election among etcd members          |

At one moment the controller-manager leader can be on node 1, the scheduler leader on node 2, and the etcd leader on node 3, with all three API servers serving.

Lease-based election:

1. Each copy tries to write its name into a shared Lease using the API's conflict check; only one write wins.
2. The winner renews periodically (defaults: retry every 2 s; give up if not renewed within 10 s).
3. If renewals stop, the lease expires (default 15 s) and a standby takes it.
4. A leader that cannot renew must stop acting immediately, to avoid two leaders.

Consequence: if the controller-manager leader dies, built-in controllers pause for roughly 15 s; running apps are unaffected.

---

## 13. Interview questions and model answers

### 13.01 Why Kubernetes? What problem does it actually solve?

It is a cluster manager.  
It turns a fleet of failure-prone machines into one schedulable pool: workloads declare resource needs, a scheduler bin-packs them for utilization, and control loops continuously replace failed work.  
That turns machine failure and deploys from manual operations into routine automated events.

Weak answer to avoid: "it orchestrates containers" (true, but says nothing about the problem).

### 13.02 `kubectl apply` succeeded. Is the app deployed?

No. Success means the intent was validated, admitted and persisted to etcd.  
Convergence is asynchronous; failures such as bad images or no capacity appear later in status conditions and Events.  
Verify with rollout status, conditions, readiness and Events.

### 13.03 Why is declarative better for distributed systems?

Stored intent plus diff-based actions make every operation idempotent and safely retryable after partial failures, crashes and concurrent actors.  
Recovery is "recompute the diff", not "remember where the script stopped".

### 13.04 What happens if a controller misses an event?

Nothing permanent. Processing is level-triggered, so the next reconcile (triggered by any later event, a relist or a resync) sees the real gap between desired and actual state and closes it.

### 13.05 Why not just process events reliably?

Exactly-once, in-order delivery across crashes and partitions is expensive and fragile.  
Level-triggered reconciliation only needs "eventually wake up and read the latest state", which is cheap and robust.

### 13.06 Who is responsible for getting a pod running after a deploy?

No single component. A chain of independent control loops moves the state one step each: goal, replica group, pods, scheduler placement, kubelet starting containers.  
Each watches the API and writes back to it. This choreography makes the system resilient and extensible, at the cost of spread-out debugging.

### 13.07 Is Kubernetes strongly or eventually consistent?

Both. The store (etcd through the API server) is strongly consistent.  
Every component acts on its own watch-refreshed copy, so the cluster as a whole is eventually consistent.

### 13.08 I deleted a pod but requests still hit it for a second. Why?

Removal from routing propagates to every node asynchronously, and the application may already be shutting down during that window.  
Fix with readiness, a short pre-stop delay, graceful shutdown, and client retries.

### 13.09 The control plane is down. Does traffic stop?

No. Running apps keep serving and kubelets keep restarting crashed containers locally. Deploys, scaling, recovery from node failures, and routing updates stop.  
Risk grows with the length of the outage.

### 13.10 A node loses network to the control plane. What happens to its pods?

After a timeout the node is treated as dead; stateless pods are recreated elsewhere while the old ones may still run.  
Stateful identities are not duplicated until the node is confirmed gone, to avoid split-brain. After the partition heals, stale copies are stopped and the cluster converges.

### 13.11 Why does recovery from a node failure take minutes, not seconds?

Detection is timeout-based and cannot distinguish slow from dead. Defaults trade recovery speed for fewer false alarms; they can be tuned per workload (K04).

### 13.12 Is Kubernetes CP or AP?

Both, by layer. The control plane is CP: etcd needs a majority and leaders stop without a lease.  
The data plane is AP: running pods keep serving on the last known state.

### 13.13 Why not run one cluster across three regions?

etcd needs majority agreement on every write; cross-region latency slows every control-plane write, and a region partition can leave the minority side unable to change anything.  
Run one cluster per region and coordinate above them.

### 13.14 We moved to Kubernetes. Are we production-ready?

Not by itself. You still need CI, a delivery pipeline, a CNI and an ingress or gateway controller, a storage driver, secret management, observability, node autoscaling, backup and DR, and policy.  
Applications must also follow the contract: graceful shutdown, health checks, statelessness, idempotency.

### 13.15 When would you not use Kubernetes?

When operational cost outweighs the benefit: few services, a small team, or workloads that a managed container platform or serverless serves well.

### 13.16 Name the Kubernetes components and where they run.

- Control plane, on control-plane nodes: API server, etcd, scheduler, controller-manager, cloud-controller-manager.
- Every node: kubelet, container runtime, kube-proxy, CNI.
- Add-ons as pods: CoreDNS, metrics-server, ingress or gateway controller, CSI.
- Everything talks through the API server; only the API server touches etcd.

---

## 14. Exercises and model answers

### 14.01 Exercise 1: etcd has 3 members and 2 die. What still works and what stops?

**etcd essentials needed:**

- etcd stores every object: specs, statuses, leases.
- Each of the 3 members holds a full copy.
- One leader accepts writes; a write commits only after a majority (2 of 3) persists it.
- Electing a leader needs a majority of votes.
- By default the API server reads only majority-confirmed, up-to-date data.
- Almost everything in Kubernetes is a write to etcd, including leader lease renewals and node heartbeats.

**Inside etcd:**

- 1 survivor is not a majority: no leader can be elected, so no writes and no up-to-date reads.
- The survivor keeps its data on disk; nothing is deleted.
- A write committed only on the two dead members may be missing on the survivor ("committed" means on a majority, not on every member).

**Control plane:**

| Still works                                           | Stops                                                                                                      |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| API server processes up, returning errors or timeouts | Every write: deploy, scale, delete, `kubectl apply`                                                        |
| Components keep their last cached view in memory      | Up-to-date reads: most `kubectl get` calls fail or hang                                                    |
|                                                       | Leader leases cannot be renewed, so controller-manager and scheduler leaders step down; no controller acts |
|                                                       | Scheduling of new pods                                                                                     |
|                                                       | Node heartbeats cannot be recorded                                                                         |

Side effect that helps: marking a node unhealthy is also a write, so the cluster cannot wrongly evict pods from nodes that merely missed heartbeats.  
Everything is frozen, mistakes included.

**Running apps:**

| Still works                                            | Stops                                                                           |
| ------------------------------------------------------ | ------------------------------------------------------------------------------- |
| Running pods keep serving traffic                      | Pods on a machine that dies are not replaced                                    |
| kubelet restarts crashed containers on its own machine | Routing does not update: a copy that dies keeps receiving traffic, which errors |
| Network rules stay as last set; traffic flows          | No autoscaling, no new pods, no config or secret updates                        |

**Recovery:**

- One member returns: 2 of 3 is a majority, a leader is elected, writes resume, and every controller reconciles the backlog because processing is level-triggered; no event replay is needed.
- Two members lost for good: restore all members from the latest etcd snapshot (losing changes since it), or rebuild from the survivor with etcd's force-new-cluster recovery (possibly losing its last missing writes). etcd backups are mandatory in production.

**Interview one-liner:** with etcd quorum lost, the control plane is read-dead and write-dead because even leases and heartbeats are writes; reconciliation stops but the data plane keeps serving on its last state; restoring quorum resumes everything automatically; permanent member loss means restoring from a snapshot or the survivor, with possible loss of recent writes.

### 14.02 Exercise 2: while the replica controller restarts, 2 of 3 `orders` copies crash and a duplicate "pod died" notification arrives. Why is the final count still 3?

1. **Restart means a full list.** On startup the controller lists all objects into its informer cache, then watches. It does not depend on notifications that were sent while it was down.
2. **Notifications only enqueue a key.** Both crash notifications and the duplicate all enqueue the same key for the `orders` replica group. Duplicate keys collapse into one work-queue entry.
3. **Reconcile reads state, not events.** The reconcile reads desired = 3 and counts the pods that actually exist and are not being deleted = 1. It creates exactly 2.
4. **A later duplicate is harmless.** If another reconcile runs for the same key, it counts 3 (or sees its own pending creations via expectations) and does nothing.
5. **Stale-cache protection.** If the cache has not yet shown the 2 new pods, expectations stop the controller from creating them again, and optimistic concurrency rejects writes based on stale versions.

The result is 3 because the decision is computed from the level (goal vs actual count), and creation is idempotent with respect to that goal; the number of notifications is irrelevant.

### 14.03 Exercise 3: two things in a typical Java service that break the application contract, and the fixes.

Representative findings for a Spring Boot service:

| Problem                                                                 | Contract rule broken          | Fix                                                                                                                                                                                   |
| ----------------------------------------------------------------------- | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| JVM exits immediately on SIGTERM, dropping in-flight requests           | 01 Graceful shutdown          | `server.shutdown=graceful` and `spring.lifecycle.timeout-per-shutdown-phase`; run the JVM as PID 1 with exec-form entrypoint so it receives SIGTERM; add a short pre-stop delay (K05) |
| One `/health` endpoint that also checks the database, used for liveness | 02 Honest health endpoints    | Separate Actuator probe groups: liveness checks only the JVM's own state; readiness includes critical dependencies; never restart on a downstream outage                              |
| Uploaded files or session state written to local disk                   | 03 Disposable                 | Store files in object storage (S3), sessions in Redis or a stateless token                                                                                                            |
| Non-idempotent POST handlers (payments, orders) retried by clients      | 04 Tolerate duplicates        | Client-generated idempotency keys stored with a unique constraint (see REST track R2 idempotency notes)                                                                               |
| `application-prod.yml` with secrets baked into the image                | 05 External config            | Config via env vars or mounted files from ConfigMaps/Secrets or an external secret manager (K10)                                                                                      |
| Logback writing to `/var/log/app.log`                                   | 06 Log to stdout              | Console appender with JSON layout; let the node agent ship logs                                                                                                                       |
| No container-aware memory sizing; `-Xmx` near the container limit       | 07 Declare resources honestly | Set requests and limits from measured usage; use `-XX:MaxRAMPercentage` and leave headroom for non-heap memory (K11)                                                                  |

---

## 15. One-page summary

- Kubernetes is a cluster manager: declared needs, scheduler bin-packing, control loops replacing failed work.
- Declarative desired state: intent is stored durably; actions compute and apply the diff; everything is retryable.
- `apply` success means intent recorded, not work done; verify through status.
- Level-triggered processing with edge-triggered delivery: events are doorbells; decisions compare goal and reality.
- Choreography of small, single-purpose controllers through the API server; one owner per field.
- Strongly consistent store, eventually consistent cluster; convergence time is a design parameter.
- Crash-failure model; slow and dead look the same; timeouts trade speed for false alarms.
- CP brain (etcd, leaders), AP body (kubelets, running pods); frozen, not dead, without the control plane.
- Odd control-plane counts (3 or 5); per-component leadership via Leases; etcd backups are mandatory.
- Kubernetes provides primitives; CI, delivery, networking, storage, security, observability and DR are added; applications must honour the contract.

---
