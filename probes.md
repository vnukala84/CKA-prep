# 🚀 Kubernetes Probes – Complete Guide

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.34+-326CE5?logo=kubernetes&logoColor=white)
![Health Checks](https://img.shields.io/badge/Health-Checks-green)
![CKA](https://img.shields.io/badge/CKA-Exam-blue)
![License](https://img.shields.io/badge/License-MIT-orange)

A comprehensive guide to **Kubernetes Probes** with architecture diagrams, YAML examples, troubleshooting, and hands-on labs.

This repository helps you understand:

- ✅ Liveness Probe
- ✅ Readiness Probe
- ✅ Startup Probe
- ✅ HTTP, TCP and Exec Probes
- ✅ Probe Lifecycle
- ✅ Best Practices
- ✅ Common Mistakes
- ✅ CKA Exam Tips

---

# Table of Contents

- Introduction
- Why Probes?
- Types of Probes
- Probe Lifecycle
- Probe Handlers
- Probe Configuration Fields
- Liveness Probe
- Readiness Probe
- Startup Probe
- HTTP Probe
- TCP Probe
- Exec Probe
- Complete YAML Example
- Hands-on Labs
- Troubleshooting
- Best Practices
- CKA Tips
- References

---

# Introduction

Kubernetes continuously monitors application health.

Instead of assuming a container is healthy because it is running, Kubernetes periodically checks the application using **Probes**.

Without probes:

```
Container Running
        │
        ▼
Application Hung ❌
        │
Traffic still reaches Pod
```

With probes:

```
Application
      │
      ▼
 Kubernetes Probe
      │
      ▼
Healthy? ─────► Yes
      │
      ▼
No
      │
Restart / Remove from Service
```

---

# Why Probes?

Containers may:

- Hang
- Deadlock
- Lose database connectivity
- Fail to initialize
- Become overloaded

Simply checking whether the process exists is **not enough**.

Probes allow Kubernetes to detect these failures automatically.

---

# Types of Probes

| Probe | Purpose | Failure Action |
|--------|----------|---------------|
| Liveness | Is application alive? | Restart container |
| Readiness | Ready for traffic? | Remove Pod from Service |
| Startup | Has application started? | Disable other probes until startup completes |

---

# Probe Lifecycle

```
                Pod Created
                     │
                     ▼
             Startup Probe
                     │
         Success     │    Failure
             │       │
             ▼       ▼
       Liveness Probe   Restart
             │
             ▼
      Readiness Probe
             │
             ▼
        Service Traffic
```

---

# Probe Handlers

Kubernetes supports three probe mechanisms.

## HTTP GET

```
HTTP GET

GET /health

200 OK
```

Example

```yaml
httpGet:
  path: /health
  port: 8080
```

---

## TCP Socket

Checks whether a TCP connection can be established.

```
TCP

Connect

Success
```

```yaml
tcpSocket:
  port: 3306
```

---

## Exec

Runs a command inside the container.

```yaml
exec:
  command:
  - cat
  - /tmp/healthy
```

Success

```
Exit Code = 0
```

Failure

```
Exit Code ≠ 0
```

---

# Probe Configuration Fields

| Field | Description |
|---------|-------------|
| initialDelaySeconds | Delay before first probe |
| periodSeconds | Probe interval |
| timeoutSeconds | Probe timeout |
| successThreshold | Required consecutive successes |
| failureThreshold | Required failures before action |
| terminationGracePeriodSeconds | Time allowed before forcefully terminating |

Example

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080

  initialDelaySeconds: 15
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
```

---

# Liveness Probe

## Purpose

Determines whether the application is still running correctly.

If it fails,

Kubernetes

```
Kills Container
        │
        ▼
Restarts Container
```

Typical use cases

- Deadlock
- Infinite loop
- Hung application
- Memory corruption

Example

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080

  initialDelaySeconds: 10
  periodSeconds: 5
```

---

# Readiness Probe

Determines whether the Pod can receive traffic.

If it fails,

```
Container Running

BUT

Removed from Service
```

The container continues running.

Typical use cases

- Waiting for database
- Waiting for cache
- Loading configuration
- External dependency unavailable

Example

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080

  periodSeconds: 5
```

---

# Startup Probe

Designed for slow-starting applications.

While Startup Probe is running

```
Liveness Disabled

Readiness Disabled
```

Only after Startup succeeds

```
Startup Complete

↓

Liveness Starts

↓

Readiness Starts
```

Example

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080

  failureThreshold: 30
  periodSeconds: 10
```

This gives the application

```
30 × 10

=

300 seconds
```

to start.

---

# Liveness vs Readiness vs Startup

| Feature | Liveness | Readiness | Startup |
|----------|-----------|-----------|----------|
| Restarts container | ✅ | ❌ | ❌ |
| Removes Pod from Service | ❌ | ✅ | ❌ |
| Used during startup | ❌ | ❌ | ✅ |
| Checks app health | ✅ | ✅ | Startup only |

---

# Complete Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: nginx
    image: nginx

    ports:
    - containerPort: 80

    startupProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 30
      periodSeconds: 5

    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10

    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```

---

# Hands-on Lab 1

Deploy nginx

```bash
kubectl apply -f probe.yaml
```

Watch Pod

```bash
kubectl get pods -w
```

Describe Pod

```bash
kubectl describe pod probe-demo
```

Observe Events

```bash
kubectl get events
```

---

# Hands-on Lab 2

Break the application

```
Delete index.html

↓

Readiness fails

↓

Pod removed from Service
```

Observe

```bash
kubectl describe pod
```

---

# Hands-on Lab 3

Kill application process

```
kill 1
```

Observe

```
Liveness Failure

↓

Restart
```

---

# Troubleshooting

Check probe events

```bash
kubectl describe pod POD_NAME
```

View logs

```bash
kubectl logs POD_NAME
```

Previous container logs

```bash
kubectl logs POD_NAME --previous
```

Check endpoints

```bash
kubectl get endpoints
```

Test manually

```bash
kubectl exec POD_NAME -- curl localhost:8080/health
```

---

# Common Mistakes

❌ Using Liveness instead of Readiness

❌ Timeout too small

❌ Wrong port

❌ Wrong path

❌ Forgetting Startup Probe

❌ Heavy database queries in health endpoint

❌ Health endpoint depends on third-party APIs

---

# Best Practices

✅ Keep health endpoints lightweight

✅ Use Startup Probe for JVM applications

✅ Separate `/health` and `/ready`

✅ Avoid expensive SQL queries

✅ Keep probe timeout realistic

✅ Tune failureThreshold carefully

✅ Monitor probe failures using Prometheus

---

# CKA Exam Tips

✔ Know all three probes

✔ Remember HTTP, TCP and Exec handlers

✔ Practice troubleshooting

✔ Know probe lifecycle

✔ Read `kubectl describe pod`

✔ Understand probe timing fields

---

# Useful Commands

```bash
kubectl get pods

kubectl describe pod

kubectl logs

kubectl logs --previous

kubectl exec

kubectl get events

kubectl get endpoints

kubectl get endpointslices
```

---

# Official Documentation

- https://kubernetes.io/docs/concepts/workloads/pods/probes/

- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

---

# Repository Structure

```
kubernetes-probes/
│
├── README.md
├── diagrams/
│   ├── lifecycle.png
│   ├── liveness.png
│   ├── readiness.png
│   ├── startup.png
│   └── infographic.png
│
├── labs/
│   ├── Lab01-Liveness
│   ├── Lab02-Readiness
│   ├── Lab03-Startup
│   ├── Lab04-Exec-Probe
│   ├── Lab05-TCP-Probe
│   ├── Lab06-HTTP-Probe
│   └── Lab07-Troubleshooting
│
└── yaml/
    ├── liveness.yaml
    ├── readiness.yaml
    ├── startup.yaml
    └── combined.yaml
```

---

# License

MIT License

---

## ⭐ If you found this repository helpful, please consider giving it a star!
