---
title: "Hello World"
description: "My first blog post"
pubDate: "2026-05-14"
---

# Resolving Kubernetes Pod OOMKilled Errors for Java Spring Boot Applications

Conquering Out of Memory Issues with Configurations

We encountered an issue with our Java Spring boot application where the Kubernetes pods were getting killed or restarted with an OOMKilled error and an Exit Code of 137.

![Kubernetes Pod Memory Issue](/sats17.blogs/images/blog_first.png)

## Reason for pod being killed

---

## What is OOMKilled (Exit Code: 137)?

When a container is OOMKilled, it means that it has consumed more memory than its specified memory limit, causing a severe resource constraint. The Kubernetes control plane or container runtime, such as Docker, recognizes this condition and forcefully terminates the container to prevent it from consuming excessive resources and potentially impacting other containers or the overall system stability.

---

## Initial Configuration

So, I will walk through the configurations we had when we faced this issue.

Our pod memory configuration looked like this:

```yaml
replicaset: 2
resources:
  requests:
    memory: "400Mi"
    cpu: "200m"
  limits:
    memory: "800Mi"
    cpu: "600m"