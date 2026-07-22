# Kubernetes Init Containers

> **CKA Topic:** Pods | Init Containers | Pod Lifecycle

---

# What is an Init Container?

An **Init Container** is a special container that executes **before the main application container**.

Its job is to perform initialization tasks such as:

- Waiting for dependencies
- Downloading configuration
- Running database migrations
- Preparing data
- Setting file permissions
- Generating certificates

The application container starts **only after all init containers complete successfully.**

---

# Pod Startup Flow

```

Pod Created
|
v
Init Container 1
|
v
Init Container 2
|
v
Init Container 3
|
v
Application Container
|
v
Pod Ready

```

---

# Key Characteristics

| Feature | Init Container |
|----------|---------------|
| Runs before app | ✅ |
| Runs only once | ✅ |
| Runs sequentially | ✅ |
| Must finish successfully | ✅ |
| Serves traffic | ❌ |
| Runs forever | ❌ |

---

# Why Use Init Containers?

Instead of adding startup scripts inside your application image, Kubernetes lets you perform initialization separately.

Benefits

- Cleaner application image
- Better security
- Easier debugging
- Separate responsibilities
- Predictable startup sequence

---

# Common Use Cases

## Wait for Database

```

Init Container
|
v
Database Available?
|
Yes
|
v
Application Starts

```

---

## Download Configuration

```

Init Container
|
v
Downloads config.json
|
v
Shared Volume
|
v
Application Reads File

```

---

## Database Migration

```

Init Container
|
v
Run Migration
|
v
Success
|
v
Application Starts

```

---

## Fix Permissions

```

Init Container

chmod
chown

↓

Application

```

---

# Lab 1 - Wait for a Service

## pod-init-container.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-init-demo

spec:
  initContainers:
  - name: wait-for-service
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      echo "Waiting for service..."
      until nslookup kubernetes.default.svc.cluster.local
      do
        echo "Still waiting..."
        sleep 2
      done
      echo "Service Found!"

  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
```

Deploy

```bash
kubectl apply -f pod-init-container.yaml
```

Check Pod

```bash
kubectl get pods
```

Describe Pod

```bash
kubectl describe pod nginx-init-demo
```

Logs of Init Container

```bash
kubectl logs nginx-init-demo -c wait-for-service
```

---

# Lab 2 - Multiple Init Containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-init-demo

spec:
  initContainers:

  - name: init-one
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Running Init Container 1"
      sleep 5

  - name: init-two
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Running Init Container 2"
      sleep 5

  containers:
  - name: nginx
    image: nginx
```

Execution Order

```

Pod

↓

Init 1

↓

Init 2

↓

Nginx

```

Notice that **Init 2 never starts until Init 1 completes.**

---

# Lab 3 - Shared Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-volume-demo

spec:

  volumes:
  - name: shared-data
    emptyDir: {}

  initContainers:
  - name: create-file
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Welcome to Kubernetes!" > /work-dir/index.html

    volumeMounts:
    - name: shared-data
      mountPath: /work-dir

  containers:
  - name: nginx
    image: nginx

    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
```

Deploy

```bash
kubectl apply -f pod-init-shared-volume.yaml
```

Verify

```bash
kubectl exec -it init-volume-demo -- cat /usr/share/nginx/html/index.html
```

Expected

```
Welcome to Kubernetes!
```

---

# Resource Management

Each init container can define:

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "128Mi"

  limits:
    cpu: "500m"
    memory: "256Mi"
```

Resources are only needed while the init container is running.

---

# Init Container vs App Container

| Feature | Init | App |
|----------|------|-----|
| Runs First | ✅ | ❌ |
| Runs Once | ✅ | ❌ |
| Long Running | ❌ | ✅ |
| Handles Traffic | ❌ | ✅ |
| Sequential | ✅ | ❌ |

---

# Init Container vs Sidecar

| Feature | Init | Sidecar |
|----------|------|----------|
| Starts Before App | ✅ | ✅ |
| Stops After Completion | ✅ | ❌ |
| Runs Entire Pod Life | ❌ | ✅ |
| Logging | ❌ | ✅ |
| Monitoring | ❌ | ✅ |

---

# Troubleshooting

## Pod Stuck in Init

```bash
kubectl get pods
```

Output

```
Init:0/1
```

Describe

```bash
kubectl describe pod POD_NAME
```

Logs

```bash
kubectl logs POD_NAME -c INIT_CONTAINER_NAME
```

---

## Restarting Init Container

```
Init:CrashLoopBackOff
```

Possible reasons

- Wrong command
- DNS failure
- Dependency unavailable
- Permission issues

---

# Useful Commands

Create Pod

```bash
kubectl apply -f pod-init-container.yaml
```

Describe

```bash
kubectl describe pod nginx-init-demo
```

Logs

```bash
kubectl logs nginx-init-demo -c wait-for-service
```

Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

Delete

```bash
kubectl delete pod nginx-init-demo
```

---

# CKA Exam Tips

✅ Init containers execute sequentially.

✅ Application waits until all init containers finish.

✅ Init containers execute only once.

✅ They are ideal for:

- Waiting for Services
- Database Migration
- Downloading Files
- Permission Fixes
- Bootstrapping

---

# Quick Revision

```

INIT = PREPARE

I → Initialize

N → No Application Yet

I → In Sequence

T → Terminate

```

Remember:

> **No Init Container Success → No Application Container**

---

# References

- Kubernetes Documentation:
  https://kubernetes.io/docs/concepts/workloads/pods/init-containers/

- CKA Curriculum:
  https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/
