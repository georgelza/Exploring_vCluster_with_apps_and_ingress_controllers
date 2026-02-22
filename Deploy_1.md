## Multi Node Kubernetes cluster on vCluster, 2 Web Apps sharing single Traefik Application Proxy and shared Ingress Controller

See `vcluster.yaml`

```bash
# ============================================
# Step 1. Create a vCluster in Docker (automatically connects)
# ============================================

sudo vcluster create my-vc1 --values vcluster.yaml

# ============================================
# Step 2. Verify it's working
# ============================================
kubectl get nodes
kubectl get namespaces
```

```bash
# ============================================
# STEP 3: Install Traefik open-source HTTP reverse proxy 
# https://traefik.io/traefik
# ============================================
echo "Installing Traefik with ClusterIP..."

helm upgrade --install traefik1 traefik \
  --repo https://helm.traefik.io/traefik \
  --namespace ingress-traefik1 --create-namespace \
  --set service.type=ClusterIP \
  --set ingressClass.name=traefik1
```

```bash
# ============================================
# STEP 4: Create PV for nginx PVC's to be created
# ============================================
echo "Creating PVs..."
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: vc-pv-nginx1
spec:
  capacity:
    storage: 200Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/nginx1
    type: DirectoryOrCreate
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: vc-pv-nginx2
spec:
  capacity:
    storage: 200Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/nginx2
    type: DirectoryOrCreate
EOF
```

```bash
# ============================================
# STEP 5: Create Project Namespaces, first time we'll be using them now.
# ============================================
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: webstack1
EOF
```

```bash
# ============================================
# STEP 6: Create PVC for nginx deployments, hosted on the #3 PV
# ============================================
echo "Creating PVCs..."
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx1-data
  namespace: webstack1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  volumeName: vc-pv-nginx1
  storageClassName: ""
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx2-data
  namespace: webstack1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  volumeName: vc-pv-nginx2
  storageClassName: ""
EOF
```

```bash
# ============================================
# STEP 7: Deploy NGINX 1 (via Traefik1 on /app1/)
# ============================================
echo "Deploying NGINX 1..."
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx1
  namespace: webstack1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx1
  template:
    metadata:
      labels:
        app: nginx1
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: nginx1-data
---
apiVersion: v1
kind: Service
metadata:
  name: nginx1-svc
  namespace: webstack1
spec:
  selector:
    app: nginx1
  ports:
    - port: 80
      targetPort: 80
EOF
```

```bash
# ============================================
# STEP 8: Deploy NGINX 2 (via Traefik1 on /app2/)
# ============================================
echo "Deploying NGINX 2..."
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx2
  namespace: webstack1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx2
  template:
    metadata:
      labels:
        app: nginx2
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: nginx2-data
---
apiVersion: v1
kind: Service
metadata:
  name: nginx2-svc
  namespace: webstack1
spec:
  selector:
    app: nginx2
  ports:
    - port: 80
      targetPort: 80
EOF
```

```bash
# ============================================
# STEP 9: Deploy NGINX 3 (via Traefik1 on /app3/)
# ============================================
echo "Deploying NGINX 3..."
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx3
  namespace: webstack2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx3
  template:
    metadata:
      labels:
        app: nginx3
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: nginx3-data
---
apiVersion: v1
kind: Service
metadata:
  name: nginx3-svc
  namespace: webstack2
spec:
  selector:
    app: nginx3
  ports:
    - port: 80
      targetPort: 80
EOF
```

```bash
# ============================================
# STEP 10: Create Ingress & Middleware for NGINX 1, 2 serviced by Traefik1 
# ============================================
echo "Creating Ingress for NGINX 1, 2 via Traefik1..."
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx1-2-ingress
  namespace: webstack1
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: webstack1-stripprefix1@kubernetescrd
spec:
  ingressClassName: traefik1
  rules:
    - host: apps.local
      http:
        paths:
          - path: /app1
            pathType: Prefix
            backend:
              service:
                name: nginx1-svc
                port:
                  number: 80
          - path: /app2
            pathType: Prefix
            backend:
              service:
                name: nginx2-svc
                port:
                  number: 80
EOF
```

```bash
# ============================================
# The middleware name must match what's referenced in the ingress annotation: 
# webstack1-stripprefix1 means namespace webstack1, middleware named stripprefix1.
# ============================================
echo "Creating Middleware for app1, 2"
cat <<EOF | kubectl apply -f -
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: stripprefix1
  namespace: webstack1
spec:
  stripPrefix:
    prefixes:
      - /app1
      - /app2
EOF
```

```bash
# ============================================
# STEP 11: Wait for pods to be ready
# ============================================
echo "Waiting for all nginx pods to be ready..."
kubectl wait --for=condition=ready pod -l app=nginx1 -n webstack1 --timeout=60s 
kubectl wait --for=condition=ready pod -l app=nginx2 -n webstack1 --timeout=60s 
```

```bash
# ============================================
# STEP 12: Add test content to each nginx
# ============================================
echo "Adding test content to NGINX 1..."

kubectl exec -it -n webstack1 $(kubectl get pod -l app=nginx1 -o jsonpath='{.items[0].metadata.name}' -n webstack1) -- sh -c 'echo "<h1>NGINX 1 - Traefik1 Path: /app1/</h1><p>App 1</p>" > /usr/share/nginx/html/index.html'

echo "Adding test content to NGINX 2..."

kubectl exec -it -n webstack1 $(kubectl get pod -l app=nginx2 -o jsonpath='{.items[0].metadata.name}' -n webstack1) -- sh -c 'echo "<h1>NGINX 2 - Traefik1 Path: /app2/</h1><p>App 2</p>" > /usr/share/nginx/html/index.html'

```

```bash
# ============================================
# STEP 13: Setup port forwarding for external access
# ============================================
# Option 1:
# Use sudo for low port 80
sudo kubectl port-forward -n ingress-traefik1 svc/traefik1 --address 10.0.0.28 --address 0.0.0.0 80:80 &
# or
sudo kubectl port-forward -n ingress-traefik1 svc/traefik1 --address 10.0.0.28 --address 0.0.0.0 80:80 > /tmp/traefik1-forward.log 2>&1 &

echo "Traefik1 port forward started on port 80 (PID: $!)"

# OR Option 2:
# Use high port 8080 (no sudo needed)
kubectl port-forward -n ingress-traefik1 svc/traefik1 --address 10.0.0.28 --address 0.0.0.0 8080:80 &
# or
kubectl port-forward -n ingress-traefik1 svc/traefik1 --address 10.0.0.28 --address 0.0.0.0 8080:80 > /tmp/traefik1-forward.log 2>&1 &

echo "Traefik1 port forward started on port 8080 (PID: $!)"

echo ""
echo "Port forward is running in background"
echo "Log: /tmp/traefik1-forward.log"
```

```bash
# Find the process IDs
ps aux | grep "port-forward"

# Kill them
pkill -f "kubectl port-forward"
```

```bash
# ============================================
# STEP 14: Add /etc/host entry
# ============================================
echo ""
echo "10.0.0.28 apps.local"
echo ""
```

```bash
# ============================================
# STEP 15: Display Status / Test Apps.
# ============================================
echo ""
echo "============================================"
echo "Deployment Complete!"
echo "============================================"
echo ""

echo "Access URLs (via port-forward on 10.0.0.28:8080):"
echo "============================================"
echo ""
echo "  - http://apps.local:8080/app1/    (NGINX 1 via Traefik1)"
echo "  - http://apps.local:8080/app2/    (NGINX 2 via Traefik1)"
echo ""

echo "Test commands (after port-forward is running):"
echo "============================================"
echo ""
echo "  curl -H "Host: apps.local" http://10.0.0.28:8080/app1/"
echo "  curl -H "Host: apps.local" http://10.0.0.28:8080/app2/"
echo ""
echo "Or with /etc/hosts entry (10.0.0.28 apps.local):"
echo ""
echo "  curl http://apps.local:8080/app1/"
echo "  curl http://apps.local:8080/app2/"
```

```bash
# ============================================
# Step 16: verify ingress and storage
# ============================================
kubectl get ingress -A
kubectl get pvc -A
kubectl get svc -A
kubectl get pods -A
```

```bash
# ============================================
# Notes/Hints regarding access and usage.
# ============================================
#
# Pod to Pod / Service to Service Access
#
# Internal Access Options:
#
# Option 1: 
# Direct to Service (Bypass Traefik) - Recommended
# ============================================

# From any pod in the same namespace (webstack1)
http://nginx1-svc/

# From any pod in a different namespace
http://nginx1-svc.webstack1.svc.cluster.local/

# **Full DNS format:**
# <service-name>.<namespace>.svc.cluster.local

# ============================================
# Option 2: 
# Through Traefik (Less Common Internally)
# Access via Traefik service
# ============================================

http://traefik1.ingress-traefik1.svc.cluster.local/app1/

# With Host header (mimics external access)
curl -H "Host: apps.local" http://traefik1.ingress-traefik1.svc.cluster.local/app1/
```

```bash
# ============================================
# EXAMPLES
#
# Create a test pod
# ============================================

kubectl run -it --rm debug --image=busybox --restart=Never -- sh

# ============================================
# Inside the pod, test direct service access:
# ============================================

wget -qO- http://nginx1-svc.webstack1.svc.cluster.local/
wget -qO- http://nginx2-svc.webstack1.svc.cluster.local/

# ============================================
# Or via Traefik (with path):
# ============================================

wget -qO- --header="Host: apps.local" http://traefik1.ingress-traefik1.svc.cluster.local/app1/
wget -qO- --header="Host: apps.local" http://traefik1.ingress-traefik1.svc.cluster.local/app2/

```

```bash
# From Another Pod's Application:
# If you had a backend service in namespace: webstack1 that needs to call the nginx1 API in same namespace: webstack1

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: webstack1
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: NGINX1_URL
          value: "http://nginx1-svc"                              # backend-api is in same namespace, short form works
EOF
```

```bash
# If you had a backend service in namespace: backend1 that needs to call the nginx1 API in namespace: webstack1

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: backend1
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: NGINX1_URL
          value: "http://nginx1-svc.webstack1.svc.cluster.local"  # backend-api is in different namespace, so we use FQDN
EOF
```

```bash
# ================================================================================================================================
# Summary Table:
# Scenario                Address                                                   Notes
# ================================================================================================================================
# Same namespace          http://nginx1-svc/                                        Short form works
# Different namespace     http://nginx1-svc.webstack1.svc.cluster.local/            Use FQDN
# Via Traefik (internal)  http://traefik1.ingress-traefik1.svc.cluster.local/app1/  Less common, adds overhead
# External                http://apps.local:8080/app1/                              Via port-forward
# ================================================================================================================================
```