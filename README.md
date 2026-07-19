Create deployment api (image nginx , 2 replicas) where each
container requests 100m CPU / 128Mi memory and is limited to 250m CPU / 256Mi memory. Then scale it to 4 replicas

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: api
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: api
    spec:
      containers:
      - image: nginx
        name: nginx
        resources:
          requests: { cpu: "100m",memory: "128Mi" }
          limits: { cpu: "250m",memory: "256Mi" }
```
```bash
vagrant@master-node:~\$ k scale deploy api --replicas=4
deployment.apps/api scaled
vagrant@master-node:~\$
```

DaemonSet (4 pts). Create a DaemonSet node-agent in kube-system running image busybox with command
sh -c "while true; do sleep 30; done" on every node (tolerate control-plane taints so it also lands on the control plane).

```yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
  namespace: kube-system
  labels:
    k8s-app: node-agent
spec:
  selector:
    matchLabels:
      name: node-agent
  template:
    metadata:
      labels:
        name: node-agent
    spec:
      tolerations:
      # these tolerations are to have the daemonset runnable on control plane nodes
      # remove them if your control plane nodes should not run pods
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: agent
        image: busybox
        command: ['sh', '-c', 'echo "Hello, Kubernetes!" && sleep 3600']
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
```

