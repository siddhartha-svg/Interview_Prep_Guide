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

If you want, I can also give a **real crash scenario and solution you can tell in interviews**.
