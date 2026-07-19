Create deployment api (image nginx , 2 replicas) where each
container requests 100m CPU / 128Mi memory and is limited to 250m CPU / 256Mi memory. Then scale it to 4 replicas

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

