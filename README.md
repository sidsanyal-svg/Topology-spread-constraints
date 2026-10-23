# Kubernetes Topology Spread Constraints: Spread Pods Evenly, Hands-On

A hands-on lab that teaches **how to spread pods evenly across nodes and zones** using topology spread constraints, on a local multi-node [kind](https://kind.sigs.k8s.io/) cluster. Everything runs on your laptop, no cloud account needed.

> 📚 This is the follow-up to my [Kubernetes Scheduling Lab](https://github.com/sidsanyal-svg/Kubernetes-scheduling-sid), which covers taints, tolerations, node affinity, and pod affinity/anti-affinity. You don't need to have done it first, but it helps.

We keep using the same analogy: **a city full of pubs**. 🍺

| Kubernetes concept | Pub analogy |
|---|---|
| Pod anti-affinity | "No two of us in the same pub. Ever." |
| **Topology spread constraint** | "Split our group across the pubs as evenly as you can." |
| `maxSkew` | How lopsided the crowd is allowed to get |
| `DoNotSchedule` | "If we can't split evenly, we wait outside." |
| `ScheduleAnyway` | "If we can't split evenly, we go in anyway." |
| Zone | A whole neighbourhood of pubs. If the power goes out there, every pub in it goes dark |

---

## Table of Contents

1. [Why this lab exists](#why-this-lab-exists)
2. [Prerequisites](#prerequisites)
3. [Repo layout](#repo-layout)
4. [The idea in 60 seconds](#the-idea-in-60-seconds)
5. [Setup: cluster and fake zones](#setup-cluster-and-fake-zones)
6. [Exercise 1: Spread across zones](#exercise-1-spread-across-zones)
7. [Exercise 2: The control-plane gotcha](#exercise-2-the-control-plane-gotcha)
8. [Exercise 3: Hard rule vs. soft rule](#exercise-3-hard-rule-vs-soft-rule)
9. [Exercise 4: Topology spread vs. pod anti-affinity](#exercise-4-topology-spread-vs-pod-anti-affinity)
10. [Exercise 5: The production pattern](#exercise-5-the-production-pattern)
11. [Key takeaways](#key-takeaways)
12. [Which scheduling tool for which problem?](#which-scheduling-tool-for-which-problem)
13. [Cleanup](#cleanup)
14. [What to read next](#what-to-read-next)

---

## Why this lab exists

Pod anti-affinity can keep copies of a service apart, but it has two weaknesses:

- It's **all or nothing**. With a hard rule and 4 replicas on 3 nodes, one pod sits in `Pending` forever.
- You have to write the rule **for every app**.

Topology spread constraints fix the first problem. Instead of "never sit together", a pod says:

> *"Keep my copies as **evenly spread** as possible across nodes (or zones), and tell me how much imbalance you'll tolerate."*

This matters in real life. If all your replicas land in one availability zone and that zone has an outage, your service goes down with it. Spreading is how you survive that.

In this lab you'll break things on purpose across **5 exercises**, including one gotcha that leaves pods `Pending` on a perfectly healthy cluster.

---

## Prerequisites

- **Docker** running on your machine
- **[kind](https://kind.sigs.k8s.io/)** installed
- **kubectl** installed

> Each kind "node" is just a Docker container on your laptop, but Kubernetes treats it like a real machine.

---

## Repo layout

All manifests used in this lab are in this repo. Apply them as described in each exercise.

| File | Used in |
|---|---|
| `kind-config.yaml` | Setup: creates the cluster (1 control plane + 3 workers) |
| `zone-spread.yaml` | Exercises 1 and 3: spread by **zone**, hard rule (6 replicas) |
| `node-spread-gotcha.yaml` | Exercise 2: spread by **node**, hard rule (3 replicas) |
| `node-spread-honor.yaml` | Exercises 2 and 4: same, plus `nodeTaintsPolicy: Honor` |
| `zone-spread-soft.yaml` | Exercise 3: spread by zone, **soft** rule (6 replicas) |
| `combined-spread.yaml` | Exercise 5: zones **hard** + nodes **soft** (8 replicas) |

Every Deployment is named `spread-demo` and has the label `app=spread-demo`. We delete it between exercises so each one starts clean.

### Handy commands used throughout the lab

Count pods per node:

```bash
kubectl get pods -l app=spread-demo -o json | ConvertFrom-Json | ForEach-Object { $_.items } | ForEach-Object { $pod = $_; $node = kubectl get node $pod.spec.nodeName -o json | ConvertFrom-Json; [PSCustomObject]@{ Pod = $pod.metadata.name; Node = $pod.spec.nodeName; Zone = $node.metadata.labels.'topology.kubernetes.io/zone' } } | Format-Table -AutoSize
```

See each pod, its status, and its node:

```bash
kubectl get pods -l app=spread-demo -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName --sort-by=.spec.nodeName
```

---

## The idea in 60 seconds

A topology spread constraint has four main fields:

| Field | Meaning |
|---|---|
| `topologyKey` | What counts as a "domain". `kubernetes.io/hostname` means each **node** is a domain. `topology.kubernetes.io/zone` means each **availability zone** is a domain |
| `maxSkew` | The maximum allowed difference in pod count between domains |
| `whenUnsatisfiable` | `DoNotSchedule` = **hard rule**, the pod waits in Pending. `ScheduleAnyway` = **soft rule**, the scheduler tries its best |
| `labelSelector` | Which pods to count. Usually the pod's own label |

**Skew** = (pods in this domain) minus (pods in the domain with the fewest pods).

Here's `maxSkew: 1` on three nodes:

```
Node A: 2 pods   Node B: 1 pod   Node C: 1 pod   → skew is 1 (2 - 1). Allowed ✅
Node A: 3 pods   Node B: 1 pod   Node C: 1 pod   → skew is 2 (3 - 1). Blocked ❌
```

The scheduler checks this **only at the moment a pod is scheduled**. It won't move running pods later.

---

## Setup: cluster and fake zones

Create the cluster:

```bash
kind create cluster --name placement-demo --config kind-config.yaml
kubectl get nodes
```

Expected output (versions will differ):

```
NAME                           STATUS   ROLES           AGE   VERSION
placement-demo-control-plane   Ready    control-plane   1m    v1.xx.x
placement-demo-worker          Ready    <none>          1m    v1.xx.x
placement-demo-worker2         Ready    <none>          1m    v1.xx.x
placement-demo-worker3         Ready    <none>          1m    v1.xx.x
```

> Already have the `placement-demo` cluster from the scheduling lab? Reuse it, but first clean up: `kubectl delete deployment order payment catalog --ignore-not-found` and `kubectl uncordon placement-demo-worker placement-demo-worker2 placement-demo-worker3`. Also remove any taints or labels you added earlier.

Kind has no real availability zones, so we make some up. We'll put **two nodes in `zone-a`** and **one node in `zone-b`**. The uneven split is deliberate, it makes the results far more interesting.

```bash
kubectl label nodes placement-demo-worker  topology.kubernetes.io/zone=zone-a
kubectl label nodes placement-demo-worker2 topology.kubernetes.io/zone=zone-a
kubectl label nodes placement-demo-worker3 topology.kubernetes.io/zone=zone-b
kubectl get nodes -L topology.kubernetes.io/zone
```

✅ **Expected:** the last column shows `zone-a` for `worker` and `worker2`, `zone-b` for `worker3`, and nothing for the control plane. That's fine.

---

## Exercise 1: Spread across zones

**Goal:** see that topology spread balances **zones**, not nodes.

Apply `zone-spread.yaml`. It runs 6 pods with `maxSkew: 1` on the **zone** key:

```bash
kubectl apply -f zone-spread.yaml
kubectl rollout status deployment spread-demo
```

Run the [count command](#handy-commands-used-throughout-the-lab).

✅ **Expected:**

- All 6 pods are Running.
- `worker3` (the only node in `zone-b`) has **3 pods**.
- `worker` and `worker2` (both in `zone-a`) share the other **3 pods**, for example 2 and 1.

**What just happened?** The scheduler balances **zones**, not nodes. `zone-a` has 2 nodes and `zone-b` has 1, so a perfect 3/3 zone split means `worker3` carries as much as the other two workers combined.

> **Real-world takeaway:** zone spread protects you from a zone outage, but it doesn't stop one node from being busier than another. Exercise 5 fixes that.

---

## Exercise 2: The control-plane gotcha

**Goal:** see a hard spread rule leave pods `Pending` on a healthy cluster, then fix it.

### Part A: reproduce the problem

Spread by **node** this time. Apply `node-spread-gotcha.yaml` (3 replicas, `hostname` key, hard rule):

```bash
kubectl delete deployment spread-demo
kubectl apply -f node-spread-gotcha.yaml
kubectl rollout status deployment spread-demo
```

Run the count command. ✅ You get **1 pod on each worker**. Looks perfect.

Now scale up:

```bash
kubectl scale deployment spread-demo --replicas=6
kubectl get pods -l app=spread-demo -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName --sort-by=.spec.nodeName
```

🤔 **Expected:** only about **3 pods Running** (one per worker) and about **3 stuck in `Pending`**. We have 3 workers with plenty of room, so 6 pods should fit as 2 per node!

Describe a pending pod and read the **Events**:

```bash
kubectl describe pod <pending-pod-name>
```

You'll see a message saying nodes *"didn't match pod topology spread constraints"*.

**Why?** By default, the scheduler counts **every node that has the topology label as a domain**, including tainted nodes your pod can't use. The control plane has a `hostname` label, so it counts as a fourth domain with **0 pods**. The smallest domain is always 0, so a worker that already has 1 pod has a skew of 1, and a second pod would make it 2. Blocked.

### Part B: the fix

Apply `node-spread-honor.yaml`. The only difference is one line: `nodeTaintsPolicy: Honor`. It tells the scheduler to respect taints when counting domains.

```bash
kubectl delete deployment spread-demo
kubectl apply -f node-spread-honor.yaml
kubectl rollout status deployment spread-demo
kubectl scale deployment spread-demo --replicas=9
kubectl rollout status deployment spread-demo
```

Run the count command. ✅ **Expected:** **3 / 3 / 3**, perfectly even, nothing Pending.

> **Real-world takeaway:** if a spread constraint on the `hostname` key behaves strangely on a cluster with tainted nodes (control plane, GPU nodes, dedicated nodes), check `nodeTaintsPolicy`. This only shows up when you break it on purpose. (`nodeTaintsPolicy` needs Kubernetes v1.26 or newer.)

---

## Exercise 3: Hard rule vs. soft rule

**Goal:** the same lesson as the scheduling lab, now with topology spread. What happens when a whole zone is unavailable?

### Part A: hard rule (`DoNotSchedule`)

Close `zone-b` by cordoning its only node, then apply `zone-spread.yaml`:

```bash
kubectl delete deployment spread-demo
kubectl cordon placement-demo-worker3
kubectl apply -f zone-spread.yaml
kubectl get pods -l app=spread-demo -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName --sort-by=.spec.nodeName
```

❌ **Expected:** roughly **1 pod Running** in `zone-a` and the other **5 `Pending`**.

`zone-b` is closed, but it still counts as a domain with 0 pods, so `zone-a` can't get more than 1 pod ahead of it. The pods would rather wait than break the spread rule.

### Part B: soft rule (`ScheduleAnyway`)

Apply `zone-spread-soft.yaml`. It's the same deployment, but with `ScheduleAnyway`:

```bash
kubectl delete deployment spread-demo
kubectl apply -f zone-spread-soft.yaml
kubectl rollout status deployment spread-demo
```

Run the count command. ✅ **Expected:** **all 6 pods Running**, all in `zone-a`. The scheduler *prefers* an even spread, but it won't leave pods waiting when it can't have one.

Re-open the node:

```bash
kubectl uncordon placement-demo-worker3
```

### Hard vs. soft at a glance

| | `DoNotSchedule` (hard) | `ScheduleAnyway` (soft) |
|---|---|---|
| Behavior | Spread rule is **strict**, pods wait in Pending | Spread rule is a **preference** |
| Protects | **Balance** | **Availability** |
| Use for | Zone spread of critical services where imbalance is unacceptable | Node-level spread, or anywhere you'd rather run than wait |

---

## Exercise 4: Topology spread vs. pod anti-affinity

**Goal:** the head-to-head. With hard pod anti-affinity, 4 replicas on 3 nodes left one pod `Pending`. What does topology spread do?

Apply `node-spread-honor.yaml` and scale up in steps:

```bash
kubectl delete deployment spread-demo
kubectl apply -f node-spread-honor.yaml
kubectl rollout status deployment spread-demo

kubectl scale deployment spread-demo --replicas=4
kubectl rollout status deployment spread-demo
```

Run the count command. ✅ **Expected:** **all 4 pods Running**, as **2 / 1 / 1**. Nobody is left Pending.

Keep going:

```bash
kubectl scale deployment spread-demo --replicas=7
kubectl rollout status deployment spread-demo

kubectl scale deployment spread-demo --replicas=12
kubectl rollout status deployment spread-demo
```

Count after each scale:

| Replicas | Expected split | Pending? |
|---|---|---|
| 4 | 2 / 1 / 1 | No |
| 7 | 3 / 2 / 2 | No |
| 12 | 4 / 4 / 4 | No |

### Head-to-head

| | Pod anti-affinity (hard) | Topology spread (`maxSkew: 1`) |
|---|---|---|
| 4 replicas, 3 nodes | 3 Running, **1 Pending** | **4 Running** (2 / 1 / 1) |
| Idea | "Never share a node" | "Stay as even as possible" |
| Scales past the node count? | No | Yes |

> **Rule of thumb:** use anti-affinity when two copies must **never** share a node. Use topology spread when you want the copies **evenly distributed** and want to keep scaling.

---

## Exercise 5: The production pattern

**Goal:** guarantee zone spread, and balance nodes when possible, without ever blocking a pod because of it.

A very common real-world setup:

- **Zones: strict** (`DoNotSchedule`). Surviving a zone outage is non-negotiable.
- **Nodes: soft** (`ScheduleAnyway`). Spread across nodes when you can, but never leave a pod waiting.

`combined-spread.yaml` has both constraints on one pod (8 replicas):

```bash
kubectl delete deployment spread-demo
kubectl apply -f combined-spread.yaml
kubectl rollout status deployment spread-demo
```

Run the count command. ✅ **Expected:**

- All 8 pods Running.
- `worker3` (all of `zone-b`) has **4 pods**.
- The other 4 are split roughly **2 and 2** between `worker` and `worker2` (`zone-a`).

The strict zone rule guaranteed the 4/4 zone split, and the soft node rule balanced the pods inside `zone-a`. Compare this with Exercise 1, where that inner split could be uneven like 2/1. The second constraint is what evens it out.

---

## Key takeaways

1. **Topology spread says "keep my copies evenly distributed". Anti-affinity says "never put two together".** They solve different problems.
2. **Skew** is the difference between a domain and the emptiest domain. `maxSkew` is the most imbalance you'll tolerate.
3. **`topologyKey` defines the domain.** Use `hostname` to spread across nodes and `zone` to spread across availability zones.
4. **Zone spread is not node spread.** Balancing zones doesn't mean every node carries the same load (Exercise 1).
5. **Tainted nodes can distort the math.** Control plane, GPU, and dedicated nodes count as domains unless you set `nodeTaintsPolicy: Honor` (Exercise 2).
6. **`DoNotSchedule` protects balance. `ScheduleAnyway` protects availability.** A hard rule can leave pods Pending when a domain is unavailable (Exercise 3).
7. **Topology spread keeps scaling past the node count.** Anti-affinity doesn't (Exercise 4).
8. **Combine constraints.** Strict on zones, soft on nodes is a battle-tested production pattern (Exercise 5).
9. **Spread is checked only at scheduling time.** If pods are deleted later, nothing rebalances the running ones. A tool like the **Descheduler** is built for that.

---

## Which scheduling tool for which problem?

| Tool | Sits on | What it does |
|---|---|---|
| **Taint** | Node | Keeps pods away unless they're allowed |
| **Toleration** | Pod | Allows a pod onto a tainted node |
| **Node affinity** | Pod | Attracts a pod to nodes with certain labels |
| **Pod affinity** | Pod | Places a pod near other pods |
| **Pod anti-affinity** | Pod | Keeps a pod away from other pods |
| **Topology spread constraint** | Pod | Spreads copies of a pod evenly across nodes or zones |

In a real project you'll combine these. A GPU node pool uses taint + toleration + node affinity. A highly available microservice uses topology spread across zones. You design the rules once and leave them alone. **You don't need to memorize the syntax. What matters is knowing which tool solves which problem.**

---

## 🧪 Practice it yourself

1. **Play with `maxSkew`.** Copy `node-spread-honor.yaml`, change `maxSkew` to `2`, scale to 6, and count pods per node. How uneven can it get?
2. **Taint plus spread.** Re-apply a taint on `worker3`:
   ```bash
   kubectl taint nodes placement-demo-worker3 dedicated=payment:NoSchedule
   ```
   Then apply `node-spread-honor.yaml` and scale to 6. The pods have no toleration, so `worker3` is excluded. Where do they land? Remove the taint with the trailing `-` when you're done.

---

## Cleanup

Remove the demo app and the fake zone labels:

```bash
kubectl delete deployment spread-demo
kubectl label nodes placement-demo-worker  topology.kubernetes.io/zone-
kubectl label nodes placement-demo-worker2 topology.kubernetes.io/zone-
kubectl label nodes placement-demo-worker3 topology.kubernetes.io/zone-
```

Or delete the whole cluster:

```bash
kind delete cluster --name placement-demo
```

---

## What to read next

- **Cluster-level default spread constraints.** Instead of adding constraints to every Deployment, a cluster admin can set defaults in the kube-scheduler.
- **`minDomains`.** Forces pods to spread across at least N domains, which pairs well with cluster autoscalers like Karpenter.
- **`matchLabelKeys`.** Keeps spread calculations correct during rolling updates, so old and new versions of a Deployment aren't counted together.
- **Descheduler.** Rebalances pods that were already running when the spread drifts.
