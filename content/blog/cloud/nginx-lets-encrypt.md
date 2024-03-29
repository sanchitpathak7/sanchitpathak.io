+++
title = "Deep Dive: Deploying Nginx Controller and Securing it with Let's Encrypt Certificates via Cert-Manager"
date = 2024-03-05T12:11:28-06:00
+++

In this blog post we are going to go over a few examples of how to create ingress resources that utilize Cert-Manager and the Nginx Ingress Controller..

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "yc-clusterissuer"
spec:
  tls:
    - hosts:
      - <your_domain_URL>
      secretName: domain-name-secret
  rules:
    - host: <your_domain_URL>
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: app
              port:
                number: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  selector:
    app: app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  labels:
    app: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: nginx:latest
        ports:
        - containerPort: 80
```