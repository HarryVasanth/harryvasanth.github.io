---
layout: post
title: "DevOps - Kubernetes: Advanced Troubleshooting with Ephemeral Containers and Process Namespace Sharing"
date: 2023-01-15 10:25:31 +01
categories: devops kubernetes
tags: kubernetes k8s troubleshooting devops containers linux
---

## The Debugging Dilemma in Distroless Images

In a mature DevOps environment, we strive for minimal container images. "Distroless" images, which contain only your application and its immediate dependencies, are the gold standard for security. They don't have `shell`, `ls`, `curl`, or even `cat`. While this is excellent for reducing the attack surface, it makes traditional troubleshooting impossible. If a pod is failing to connect to a database, you can't `kubectl exec` into it to run a simple `nc` or `ping` command.

**Kubernetes Ephemeral Containers** (graduated to Stable in v1.25) provide a production ready solution to this paradox. They allow you to "inject" a temporary, tool-rich container into the same namespaces as your running (or failing) application pod.

## Why Ephemeral? The Architecture of Choice

Unlike a standard container, an ephemeral container:

- Does not have its own resource limits or probes.
- Is not automatically restarted.
- Is not defined in the original Pod specification.
- Shares the network and IPC namespaces of the target container.

It is a "sidecar" that is attached on-demand to a live process. This allows you to maintain a hardened production environment while still having the visibility needed to resolve complex, real-time incidents without a Pod restart.

## Implementation: The 'kubectl debug' Workflow

Imagine a Pod named `order-processor-xyz` is throwing connection errors. The image is distroless, so we can't exec into it.

### 1. Launching the Debugger

We use `kubectl debug` to attach a container based on a tool-heavy image like `nicolaka/netshoot` or a standard `busybox`.

```bash
kubectl debug -it order-processor-xyz --image=nicolaka/netshoot --target=order-processor
```

- `--image`: The container containing your tools.
- `--target`: This is the critical flag. It ensures the debug container can see the process namespace of the specific container within the Pod.

### 2. Performing the Diagnosis

Once inside the debug container, you have a full suite of networking tools.

```bash
# Check if the database port is reachable
nc -zv database-service 5432

# Look at the application's open network sockets
ss -ntlp

# View the application's environment variables (often the source of config errors)
cat /proc/1/environ | tr '\0' '\n'
```

## Advanced Strategy: Process Namespace Sharing (The 'gdb' Trick)

By default, containers in a Pod do not share a process namespace. To see the application's processes (e.g., to use `gdb`, `strace`, or `lsof`), you must have set `shareProcessNamespace: true` in your Pod spec.

If you didn't do this, you can still use `kubectl debug` with the `--copy-to` flag to create a new version of the Pod with this feature enabled:

```bash
kubectl debug order-processor-xyz --copy-to=debug-pod --share-processes --image=nicolaka/netshoot
```

This allows you to perform deep inspection:

```bash
# Find the PID of the main Java/Go process
ps -ef

# Attach strace to see syscalls
strace -p 1
```

## Debugging a 'CrashLoopBackOff'

If a Pod crashes immediately on startup, you can't "attach" a debug container because the Pod is not running.
**The Optimal Solution**: Use the `--copy-to` flag combined with a command override.

```bash
kubectl debug order-processor-xyz -it --copy-to=order-processor-debug \
    --container=order-processor --sh
```

This allows you to manually run the application's entrypoint script inside a shell to see exactly why it is crashing, including errors that might not be making it to the standard logs.

## Security Considerations for the Platform Team

Ephemeral containers are powerful, but they represent a potential security risk if mismanaged.

- **RBAC**: Ensure that only experienced administrators have the `pods/ephemeralcontainers` permission.
- **Audit Logging**: Every use of `kubectl debug` should be tracked in your Kubernetes audit logs, as it involves accessing potentially sensitive data inside a production pod.
- **Image Whitelisting**: Use an Admission Controller (like Kyverno or OPA) to ensure that only approved "debug" images (from your internal registry) can be used as ephemeral containers.

## Best Practice Summary

1.  **Always use a dedicated debug image**: Tools like `netshoot` are better than standard `alpine`.
2.  **Clean up**: Ephemeral containers stay in the Pod spec until the Pod is deleted.
3.  **Target specifically**: Use `--target` to ensure you are in the right container's namespace.

## Summary

The transition to minimal, distroless images is a mandatory step for container security, but it requires a corresponding shift in troubleshooting maturity. Ephemeral containers allow you to maintain a hardened production environment without sacrificing the visibility needed to resolve complex, real-time incidents. Mastering the `kubectl debug` workflow is a prerequisite for any administrator managing high-scale, security-sensitive Kubernetes clusters. It represents the perfect balance between the "Immutability" of containers and the "Observability" required for enterprise operations.
