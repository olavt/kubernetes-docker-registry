# How to setup a Docker Container Registry on a Kubernetes cluster

## Create a file for basic authnitication (local file on Kubernetes master node)

```
$ mkdir auth
$ docker run --entrypoint htpasswd httpd -Bbn testuser testpassword > auth/htpasswd
```

## Create a Kubernetes secret from the htpasswd file

```
$ kubectl create namespace docker-registry

$ kubectl create secret generic docker-registry-auth-secret --from-file=auth/htpasswd --namespace docker-registry
```

## Create a Kuberbetes deployment file for running the Docker Container Registry

```
apiVersion: v1
kind: Namespace
metadata:
  name: docker-registry

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: docker-registry
  namespace: docker-registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: docker-registry
  template:
    metadata:
      labels:
        app: docker-registry
    spec:
      containers:
        - name: docker-registry
          image: registry:latest
          env:
            - name: REGISTRY_AUTH
              value: "htpasswd"
            - name: REGISTRY_AUTH_HTPASSWD_REALM
              value: "Registry Realm"
            - name: REGISTRY_AUTH_HTPASSWD_PATH
              value: "/auth/htpasswd"
            - name: REGISTRY_HTTP_ADDR
              value: ":5000"
            - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
              value: "/var/lib/registry"
          ports:
          - name: http
            containerPort: 5000
          volumeMounts:
          - name: image-store
            mountPath: "/var/lib/registry"
          - name: auth-vol
            mountPath: "/auth"
            readOnly: true
      volumes:
        - name: image-store
          nfs:
            server: <your-nfs-server>
            path: /mnt/nfs/docker-registry
        - name: auth-vol
          secret:
            secretName: docker-registry-auth-secret

---

kind: Service
apiVersion: v1
metadata:
  name: docker-registry
  namespace: docker-registry
  labels:
    app: docker-registry
spec:
  type: NodePort
  selector:
    app: docker-registry
  ports:
  - name: http
    port: 5000
    targetPort: 5000
```

## Create a NGINX ingress

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
  name: docker-registry-ingress
  namespace: docker-registry
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - registry.<your domain>
    secretName: docker-registry-tls
  rules:
  - host: registry.<your domain>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: docker-registry
            port:
              number: 5000

## List repositories

https://<registry.your domain>/v2/_catalog
