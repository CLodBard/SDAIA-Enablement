# Kubernetes Hands-On Labs

## Setup Instructions

### Prerequisites
- Kubernetes cluster running (MicroK8s, Charmed Kubernetes, Kind, or cloud-managed)
- `kubectl` configured to access your cluster
- A text editor (vim, nano, or IDE)
- Docker images available (nginx, alpine, etc.)

### Verify Setup
```bash
kubectl cluster-info
kubectl get nodes
kubectl version --client
```

---

# DAY 2: DEPLOYMENTS, REPLICASETS & YAML MANIFESTS

## Lab 2.1: Your First Deployment

### Objective
Create a simple Deployment with 3 replicas of nginx, understand how pods are managed.

### Steps

**1. Create a Deployment manifest** (`nginx-deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**2. Deploy it**:
```bash
kubectl apply -f nginx-deployment.yaml
```

**3. Observe what was created**:
```bash
# Check Deployment
kubectl get deployments

# Check ReplicaSet (created by Deployment)
kubectl get replicasets

# Check Pods (created by ReplicaSet)
kubectl get pods

# Get more details
kubectl describe deployment nginx-app
```

**4. What happened?**
Document:
- How many Pods were created? (should be 3)
- What's the ReplicaSet name? (auto-generated)
- What's the relationship: Deployment → ReplicaSet → Pods?

### Proof of Completion
```bash
# Run these commands and screenshot the output
kubectl get all -o wide
kubectl describe deployment nginx-app
```

**Show me**:
- All 3 pods running
- Deployment showing 3/3 ready

---

## Lab 2.2: Understanding Pod Labels & Selectors

### Objective
Understand how Deployments track pods using labels and selectors.

### Steps

**1. Examine labels on your pods**:
```bash
# Get pods with labels visible
kubectl get pods --show-labels

# Get pods with specific label
kubectl get pods -l app=nginx
```

**2. Check the Deployment selector**:
```bash
kubectl get deployment nginx-app -o yaml | grep -A 5 "selector:"
```

You should see:
```yaml
selector:
  matchLabels:
    app: nginx
```

**3. Manually label a new pod and see if ReplicaSet adopts it**:
```bash
# Create a standalone pod (not from deployment)
kubectl run manual-nginx --image=nginx:1.21 --labels=app=nginx

# Check ReplicaSet status
kubectl get replicasets
```

**Question**: What happened to the ReplicaSet?
- Was it already at 3 replicas?
- Did it delete one to maintain 3 total?

**4. Remove the manual pod's label**:
```bash
kubectl label pod manual-nginx app- --overwrite
```

**Question**: What happened to the ReplicaSet now?
- Did it create a new pod to maintain 3 replicas?

### Proof of Completion
Screenshots showing:
```bash
kubectl get pods --show-labels
kubectl get replicasets
```

**Show me**:
- All pods have label `app=nginx`
- Selector mechanism working (explain in 1-2 sentences what you observed)

---

## Lab 2.3: Scaling Deployments

### Objective
Understand how to scale applications up and down.

### Steps

**1. Current state**:
```bash
kubectl get deployment nginx-app
kubectl get pods
```

**2. Scale up to 5 replicas**:
```bash
kubectl scale deployment nginx-app --replicas=5
```

**3. Watch it happen in real-time**:
```bash
# Terminal 1: Watch pods being created
kubectl get pods -w

# Terminal 2 (in another tab): Scale it
kubectl scale deployment nginx-app --replicas=5
```

**4. Scale down to 2**:
```bash
kubectl scale deployment nginx-app --replicas=2
```

**5. Alternative: Edit manifest directly**:
```bash
# Edit and change replicas: 2 → 5
kubectl edit deployment nginx-app

# This opens an editor, change replicas field, save and exit
```

### Proof of Completion
Run these commands and screenshot:
```bash
kubectl get deployment nginx-app
kubectl get pods -o wide
```

**Show me**:
- Current replicas set to whatever you chose (document what number)
- All pods in Running state
- Screenshot showing the scaling happened

---

## Lab 2.4: Rolling Updates

### Objective
Update your app version with zero downtime via rolling update.

### Steps

**1. Current state**:
```bash
kubectl get pods
# All running nginx:1.21
```

**2. Update to new image version**:
```bash
# Method 1: Edit deployment
kubectl set image deployment/nginx-app nginx=nginx:1.23 --record

# OR Method 2: Edit manifest
kubectl edit deployment nginx-app
# Change: image: nginx:1.21 → image: nginx:1.23
# Save
```

**3. Watch the rolling update in real-time**:
```bash
# Terminal 1: Watch pods
kubectl get pods -w

# Terminal 2: Trigger update (if using edit)
# (if already done above, just watch)
```

What you should see:
```
Old pod terminating...
New pod starting...
Old pod terminating...
New pod starting...
```

**4. Verify update completed**:
```bash
kubectl get deployment nginx-app
kubectl get pods
kubectl describe pod <pod-name> | grep Image:
```

**5. Check update history**:
```bash
kubectl rollout history deployment/nginx-app
```

### Proof of Completion
Screenshots showing:
```bash
kubectl get pods
kubectl describe deployment nginx-app | grep -A 5 "Image:"
kubectl rollout history deployment/nginx-app
```

**Show me**:
- All pods running with new image version (1.23)
- Rollout history showing the update

---

## Lab 2.5: Rollback

### Objective
Learn how to quickly revert a bad update.

### Steps

**1. Simulate a bad update**:
```bash
# Update to non-existent image
kubectl set image deployment/nginx-app nginx=nginx:99.99 --record
```

**2. Watch pods fail to start**:
```bash
# Watch for 30 seconds
kubectl get pods -w

# Check what's wrong
kubectl describe pod <new-pod-name>
# You'll see ImagePullBackOff error
```

**3. Rollback to previous version**:
```bash
kubectl rollout undo deployment/nginx-app
```

**4. Watch pods recover**:
```bash
kubectl get pods -w
# Should return to running state
```

**5. Verify**:
```bash
kubectl get pods
kubectl describe pod <pod-name> | grep Image:
kubectl rollout history deployment/nginx-app
```

### Proof of Completion
Screenshots showing:
```bash
# Before rollback (showing ImagePullBackOff)
kubectl get pods
kubectl describe pod <pod-name>

# After rollback (showing Running with original image)
kubectl get pods
kubectl describe pod <pod-name> | grep Image:
```

**Show me**:
- Bad update causing pod failures
- Successful rollback returning to running state

---

## Lab 2.6: YAML Manifest Deep Dive

### Objective
Understand all fields in a Deployment manifest.

### Steps

**1. Export current deployment to YAML**:
```bash
kubectl get deployment nginx-app -o yaml > exported-deployment.yaml
cat exported-deployment.yaml
```

**2. Understand each section**:

Identify and document:
```
metadata:
  - What's the name? (nginx-app)
  - What's the namespace? (default)

spec:
  - replicas: How many pods?
  - selector.matchLabels: Which pods does this manage?
  
  template:
    - metadata.labels: What labels on pods?
    - spec.containers: What image? What port?
```

**3. Create a more realistic manifest** (`app-deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
  namespace: default
  labels:
    app: web
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        version: v1
    spec:
      containers:
      - name: app
        image: nginx:1.23
        ports:
        - containerPort: 80
          name: http
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
```

**4. Deploy it**:
```bash
kubectl apply -f app-deployment.yaml
```

**5. Understand the new fields**:
- `strategy.rollingUpdate.maxSurge`: Max additional pods during update
- `strategy.rollingUpdate.maxUnavailable`: Max pods down during update
- `resources.requests`: Guaranteed resources
- `resources.limits`: Max resources
- `livenessProbe`: Is container healthy?

### Proof of Completion
Create a document explaining:
1. What does `maxSurge: 1` mean?
2. What does `maxUnavailable: 1` mean?
3. What's the difference between requests and limits?
4. Why use livenessProbe?

Show output:
```bash
kubectl get deployment my-web-app
kubectl get pods
```

---

## Day 1 Completion Checklist

- [ ] Lab 2.1: Deployed nginx and saw 3 pods created
- [ ] Lab 2.2: Understood labels and selector relationship
- [ ] Lab 2.3: Scaled deployment up and down
- [ ] Lab 2.4: Performed rolling update to new image version
- [ ] Lab 2.5: Rolled back a bad update
- [ ] Lab 2.6: Understood manifest structure and resource limits

**Final Proof**: Run this and screenshot:
```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide
```

---

# REFERENCE

## Common kubectl Commands Used in Labs

```bash
# Deployments
kubectl create deployment <name> --image=<image>
kubectl get deployments
kubectl describe deployment <name>
kubectl edit deployment <name>
kubectl delete deployment <name>

# Scaling
kubectl scale deployment <name> --replicas=<number>

# Rolling updates
kubectl set image deployment/<name> <container>=<image>
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>

# Services
kubectl create service nodeport <name> --tcp=<port>:<targetPort>
kubectl get service
kubectl describe service <name>
kubectl delete service <name>

# ConfigMaps & Secrets
kubectl create configmap <name> --from-literal=key=value
kubectl create secret generic <name> --from-literal=key=value
kubectl get configmap / secret
kubectl describe configmap / secret <name>

# Pods
kubectl get pods
kubectl describe pod <name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- sh
kubectl port-forward <pod-name> <local-port>:<pod-port>

# General
kubectl get all
kubectl apply -f <file>
kubectl delete -f <file>
kubectl get <resource> -o yaml
kubectl edit <resource> <name>
```