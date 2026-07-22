# Kubernetes Self-Signed TLS Certificates

## Complete Learning Guide with Hands-on Labs

> A comprehensive guide for Kubernetes beginners, DevOps engineers, and
> CKA candidates.

------------------------------------------------------------------------

# Table of Contents

1.  Introduction
2.  TLS Basics
3.  PKI Components
4.  Certificate Lifecycle
5.  Self-Signed vs CA Signed
6.  Kubernetes TLS Secrets
7.  End-to-End Workflow
8.  Hands-on Lab 1 -- Generate Keys
9.  Hands-on Lab 2 -- Create TLS Secret
10. Hands-on Lab 3 -- Mount Secret into Pod
11. Hands-on Lab 4 -- Secure NGINX with HTTPS
12. Hands-on Lab 5 -- Secure Ingress
13. Troubleshooting
14. Best Practices
15. CKA Tips
16. Interview Questions
17. Cheat Sheet

------------------------------------------------------------------------

# 1. Introduction

TLS (Transport Layer Security) encrypts communication between clients
and servers.

Common Kubernetes use cases:

-   HTTPS Ingress
-   Admission Webhooks
-   Internal Services
-   API Gateway
-   Service Mesh
-   Secure Applications

------------------------------------------------------------------------

# 2. TLS Components

``` text
Client
   │
   │ HTTPS
   ▼
Server
   │
   ├── server.key
   ├── server.crt
   └── TLS Secret
```

  File         Purpose
  ------------ -----------------------------
  server.key   Private key
  server.csr   Certificate Signing Request
  server.crt   Certificate
  Secret       Kubernetes TLS Secret

------------------------------------------------------------------------

# 3. PKI Workflow

``` text
Private Key
     │
     ▼
CSR
     │
     ▼
Certificate
     │
     ▼
TLS Secret
     │
     ▼
Ingress / Pod
```

------------------------------------------------------------------------

# 4. Generate Private Key

``` bash
openssl genrsa -out server.key 2048
```

Verify:

``` bash
openssl rsa -in server.key -check
```

------------------------------------------------------------------------

# 5. Generate CSR

``` bash
openssl req -new \
-key server.key \
-out server.csr
```

Example

``` text
Country: IN
State: Andhra Pradesh
Organization: Demo
Common Name: myapp.local
```

------------------------------------------------------------------------

# 6. Create Self-Signed Certificate

``` bash
openssl x509 \
-req \
-days 365 \
-signkey server.key \
-in server.csr \
-out server.crt
```

Inspect

``` bash
openssl x509 -text -noout -in server.crt
```

------------------------------------------------------------------------

# 7. Create Kubernetes TLS Secret

``` bash
kubectl create secret tls my-tls-secret \
--cert=server.crt \
--key=server.key
```

Verify

``` bash
kubectl get secret
kubectl describe secret my-tls-secret
```

------------------------------------------------------------------------

# 8. Lab 1 -- Generate Certificates

Objective

-   Install OpenSSL
-   Generate key
-   Generate CSR
-   Generate certificate

Validation

``` bash
ls
openssl x509 -text -noout -in server.crt
```

------------------------------------------------------------------------

# 9. Lab 2 -- Create TLS Secret

``` bash
kubectl create namespace tls-demo

kubectl create secret tls my-tls-secret \
--cert=server.crt \
--key=server.key \
-n tls-demo
```

Verify

``` bash
kubectl get secret -n tls-demo
```

------------------------------------------------------------------------

# 10. Lab 3 -- Mount Secret into Pod

Deployment

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: tls
          mountPath: /etc/nginx/tls
          readOnly: true
      volumes:
      - name: tls
        secret:
          secretName: my-tls-secret
```

Verify

``` bash
kubectl exec -it <pod> -- ls /etc/nginx/tls
```

Expected

``` text
tls.crt
tls.key
```

------------------------------------------------------------------------

# 11. Lab 4 -- HTTPS NGINX

Configure nginx.conf

``` nginx
server {
    listen 443 ssl;

    ssl_certificate /etc/nginx/tls/tls.crt;
    ssl_certificate_key /etc/nginx/tls/tls.key;

    location / {
        return 200 "HTTPS works!";
    }
}
```

Test

``` bash
curl -k https://<service-ip>
```

------------------------------------------------------------------------

# 12. Lab 5 -- Secure Ingress

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo
spec:
  tls:
  - hosts:
    - myapp.local
    secretName: my-tls-secret
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```

Test

``` bash
curl -k https://myapp.local
```

------------------------------------------------------------------------

# Troubleshooting

  Problem                Resolution
  ---------------------- ---------------------------------------
  Secret missing         Verify namespace
  TLS handshake failed   Verify certificate CN/SAN
  Browser warning        Expected for self-signed certificates
  Certificate expired    Regenerate certificate

------------------------------------------------------------------------

# Best Practices

-   Never commit private keys.
-   Use namespaces.
-   Rotate certificates.
-   Limit secret access using RBAC.
-   Use cert-manager for production.

------------------------------------------------------------------------

# CKA Exam Tips

-   Know `kubectl create secret tls`.
-   Understand TLS Secret structure.
-   Know how to mount secrets.
-   Practice Ingress TLS.
-   Understand Base64 encoding.

------------------------------------------------------------------------

# Interview Questions

1.  What is a TLS Secret?
2.  Difference between CSR and CRT?
3.  What is a private key?
4.  Why use TLS?
5.  What is a CA?
6.  Self-signed vs CA-signed?
7.  How does Kubernetes store secrets?
8.  How do you mount a secret?
9.  How do you inspect certificates?
10. What happens when a certificate expires?

------------------------------------------------------------------------

# Cheat Sheet

``` bash
openssl genrsa -out server.key 2048

openssl req -new \
-key server.key \
-out server.csr

openssl x509 \
-req \
-days 365 \
-signkey server.key \
-in server.csr \
-out server.crt

kubectl create secret tls my-tls-secret \
--cert=server.crt \
--key=server.key

kubectl get secret

kubectl describe secret my-tls-secret

kubectl get secret my-tls-secret \
-o yaml
```

------------------------------------------------------------------------

# Repository Structure

``` text
kubernetes-self-signed-tls/
│
├── README.md
├── certs/
│   ├── server.key
│   ├── server.csr
│   └── server.crt
├── manifests/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
├── labs/
│   ├── Lab1.md
│   ├── Lab2.md
│   ├── Lab3.md
│   ├── Lab4.md
│   └── Lab5.md
└── images/
```

------------------------------------------------------------------------

Happy Learning!
