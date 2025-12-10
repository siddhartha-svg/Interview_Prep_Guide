1. **“How you would troubleshoot CrashLoopBackOff in production”**, using **First → Next → Then → Finally** format:

---

### ⭐ **How I Troubleshoot CrashLoopBackOff in Production (150 Words)**

**First**, I start by checking **why the container is crashing**. I run:

```bash
kubectl describe pod <pod>
kubectl logs <pod>
```

This helps me identify whether the issue is related to missing environment variables, incorrect entrypoint/command, configuration errors, or application exceptions causing the container to exit.

**Next**, I check **health probes**. Many CrashLoopBackOff issues happen due to misconfigured liveness or readiness probes. If liveness probes fail repeatedly, the container restarts even if the application is healthy. I review probe paths, ports, and timeouts.

**Then**, I check for **resource issues** such as OOMKilled (Out of Memory), CPU throttling, or insufficient limits. I verify this using:

```bash
kubectl get events
kubectl top pod
```

I adjust resources if required.

**Finally**, I validate **dependencies and configurations**—databases, secrets, configmaps, network policies. Once the root cause is fixed, I redeploy the pod and monitor it through metrics and logs to confirm stability in production.

---

2. **DaemonSet vs StatefulSet vs Deployment** using your preferred **First → Next → Then → Finally** format **with real examples**:

---

### ⭐ **Difference Between DaemonSet, StatefulSet, Deployment (With Real Examples – 150 Words)**

**First**, a **Deployment** is used for **stateless applications** where pods do not need identity or persistent storage. It supports rolling updates and scaling easily.
**Real example:** Running multiple replicas of a **frontend service**, **REST API**, or **Node.js application** where any pod can serve traffic.

**Next**, a **StatefulSet** is used for **stateful applications** that need stable hostnames, ordered deployment, and persistent volumes. Each pod gets a fixed identity and its own storage.
**Real example:** **MongoDB, Cassandra, Kafka, Redis cluster**, or **MinIO**, where data consistency and stable pod names are required.

**Then**, a **DaemonSet** ensures **one pod runs on every node** in the cluster. When a new node joins, the DaemonSet automatically schedules a pod there.
**Real example:** **Fluentd/FluentBit log collectors**, **Node Exporter**, **Kube-proxy**, **network agents**, or **security scanners**.

**Finally**, the core difference is their purpose:

* Deployment → stateless apps
* StatefulSet → stateful apps
* DaemonSet → node-level agents

---
