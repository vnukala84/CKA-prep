# Kubernetes Sidecar Containers

> **CKA Topic:** Pods | Sidecar Containers | Native Sidecars | Pod Lifecycle

---

# What is a Sidecar Container?

A **Sidecar Container** is a container that runs **alongside the main application container** inside the same Pod.

It provides supporting functionality without modifying the application itself.

Typical responsibilities include:

- Log collection
- Monitoring
- Metrics exporting
- Reverse proxy
- Service Mesh (Envoy/Istio)
- Configuration synchronization
- Secret synchronization

Unlike Init Containers, Sidecars **continue running throughout the Pod's lifetime**.

---

# Pod Architecture

```

+------------------------------------------------------+
|                      POD                             |
|                                                      |
|   +----------------------+      +----------------+   |
|   | Application          |      | Sidecar       |   |
|   |                      |<---->| Shared Volume |   |
|   | Spring Boot / Nginx  |      | Fluent Bit    |   |
|   +----------------------+      +----------------+   |
|                                                      |
+------------------------------------------------------+

```

---

# Why Use Sidecars?

Instead of embedding logging, monitoring or proxy code inside your application:

Application

↓

Business Logic Only

↓

Sidecar

↓

Logging / Metrics / Proxy / Monitoring

Benefits

- Separation of concerns
- Reusable components
- Easier maintenance
- Independent updates
- Improved observability

---

# Native Sidecar Containers (Kubernetes v1.33+)

Kubernetes now supports **Native Sidecar Containers**.

A Native Sidecar is implemented as a **restartable Init Container**.

Characteristics

- Starts before application
- Continues running
- Stops after application exits
- Can restart independently
- Better lifecycle management

---

# Pod Startup Flow

```

Pod Created

↓

Native Sidecar Starts

↓

Regular Init Containers

↓

Application Starts

↓

Application + Sidecar Running

↓

Application Stops

↓

Sidecar Stops Last

```

---

# Native Sidecar YAML

**pod-native-sidecar.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: native-sidecar-demo

spec:

  initContainers:
  - name: log-agent
    image: alpine:3.20
    restartPolicy: Always

    command:
    - sh
    - -c
    - |
      while true
      do
        echo "Collecting logs..."
        sleep 5
      done

  containers:

  - name: nginx
    image: nginx
```

Notice

```yaml
restartPolicy: Always
```

This makes the Init Container behave as a **Native Sidecar**.

---

# Legacy Sidecar Pattern

Before Native Sidecars, a Sidecar was simply another container.

**pod-legacy-sidecar.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: legacy-sidecar-demo

spec:
  containers:

  - name: nginx
    image: nginx

  - name: logger
    image: alpine

    command:
    - sh
    - -c
    - |
      while true
      do
        echo "Logging..."
        sleep 5
      done
```

Both containers run simultaneously.

---

# Shared Volume Example

Sidecars commonly share data using **emptyDir**.

**pod-sidecar-shared-volume.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-demo

spec:

  volumes:
  - name: logs
    emptyDir: {}

  containers:

  - name: app
    image: busybox

    command:
    - sh
    - -c
    - |
      while true
      do
        date >> /logs/app.log
        sleep 5
      done

    volumeMounts:
    - name: logs
      mountPath: /logs

  - name: log-reader
    image: busybox

    command:
    - sh
    - -c
    - |
      tail -F /logs/app.log

    volumeMounts:
    - name: logs
      mountPath: /logs
```

---

# Execution Flow

```

Application

writes logs

↓

Shared Volume

↓

Sidecar

reads logs

↓

ElasticSearch / Loki / Splunk

```

---

# Sidecar Lifecycle

```

Pod Created

↓

Sidecar Starts

↓

Application Starts

↓

Both Run Together

↓

Application Ends

↓

Sidecar Ends

↓

Pod Deleted

```

---

# Common Use Cases

| Use Case | Example |
|----------|---------|
| Log Collection | Fluent Bit |
| Metrics | Prometheus Exporter |
| Reverse Proxy | Nginx |
| Service Mesh | Envoy |
| Secret Sync | Vault Agent |
| Configuration Sync | Git Sync |
| File Synchronization | rsync |

---

# Init Container vs Sidecar

| Feature | Init Container | Sidecar |
|----------|---------------|----------|
| Purpose | Prepare application | Support application |
| Runs before app | Yes | Yes |
| Runs with app | No | Yes |
| Stops after setup | Yes | No |
| Continuous execution | No | Yes |
| Handles traffic | No | Sometimes |

---

# Native Sidecar vs Legacy Sidecar

| Feature | Native | Legacy |
|----------|---------|---------|
| Defined in initContainers | ✅ | ❌ |
| Defined in containers | ❌ | ✅ |
| restartPolicy: Always | ✅ | ❌ |
| Startup order guaranteed | ✅ | ❌ |
| Shutdown order guaranteed | ✅ | ❌ |
| Better for Jobs | ✅ | ❌ |

---

# Sidecar for Kubernetes Job

Native Sidecars allow Jobs to complete properly.

**pod-sidecar-job.yaml**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job

spec:
  template:
    spec:

      restartPolicy: Never

      initContainers:
      - name: uploader
        image: alpine
        restartPolicy: Always
        command:
        - sh
        - -c
        - |
          while true
          do
            sleep 5
          done

      containers:
      - name: backup
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Backup Completed"
```

The Job completes when the main container finishes. Kubernetes then terminates the native sidecar gracefully.

---

# Useful Commands

Deploy

```bash
kubectl apply -f pod-native-sidecar.yaml
```

List Pods

```bash
kubectl get pods
```

Describe Pod

```bash
kubectl describe pod native-sidecar-demo
```

View Application Logs

```bash
kubectl logs native-sidecar-demo -c nginx
```

View Sidecar Logs

```bash
kubectl logs native-sidecar-demo -c log-agent
```

Open Shell

```bash
kubectl exec -it native-sidecar-demo -c nginx -- sh
```

Delete

```bash
kubectl delete pod native-sidecar-demo
```

---

# Troubleshooting

## Sidecar Keeps Restarting

```text
CrashLoopBackOff
```

Possible causes

- Incorrect command
- Missing mounted volume
- Configuration errors
- Application dependency failures

---

## Check Events

```bash
kubectl describe pod native-sidecar-demo
```

---

## Check Logs

```bash
kubectl logs native-sidecar-demo -c log-agent
```

---

# Best Practices

- Keep sidecars focused on a single responsibility.
- Use shared volumes instead of network communication when exchanging files.
- Define CPU and memory requests/limits for both the application and sidecar.
- Avoid putting business logic in sidecars.
- Prefer native sidecars on Kubernetes versions that support them.

---

# CKA Exam Tips

✅ Sidecars run alongside the application.

✅ Native Sidecars use:

```yaml
initContainers:
  - restartPolicy: Always
```

✅ Legacy Sidecars use:

```yaml
containers:
```

✅ Sidecars are commonly used for:

- Logging
- Monitoring
- Proxying
- Metrics
- Service Mesh
- Secret Synchronization

---

# Quick Revision

```

SIDECAR

S → Supports Application

I → Inside Same Pod

D → During Entire Lifecycle

E → Extra Functionality

CAR → Runs Beside the Application

```

Remember:

> **Init Container prepares the application. Sidecar supports the application while it is running.**

---

# References

- Kubernetes Documentation:
  https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/

- Kubernetes Native Sidecars:
  https://kubernetes.io/docs/tutorials/configuration/pod-sidecar-containers/

- CKA Curriculum:
  https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/
