# 2-Day Kubernetes Hands-On Labs

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

# DAY 1: DEPLOYMENTS, REPLICASETS & YAML MANIFESTS

## Lab 1.1: Your First Deployment (30 minutes)

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

## Lab 1.2: Understanding Pod Labels & Selectors (20 minutes)

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

## Lab 1.3: Scaling Deployments (15 minutes)

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

## Lab 1.4: Rolling Updates (25 minutes)

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

## Lab 1.5: Rollback (15 minutes)

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

## Lab 1.6: YAML Manifest Deep Dive (20 minutes)

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

- [ ] Lab 1.1: Deployed nginx and saw 3 pods created
- [ ] Lab 1.2: Understood labels and selector relationship
- [ ] Lab 1.3: Scaled deployment up and down
- [ ] Lab 1.4: Performed rolling update to new image version
- [ ] Lab 1.5: Rolled back a bad update
- [ ] Lab 1.6: Understood manifest structure and resource limits

**Final Proof**: Run this and screenshot:
```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide
```

---

# DAY 2: SERVICES & NETWORKING

## Lab 2.1: ClusterIP Service (20 minutes)

### Objective
Create internal service for pod-to-pod communication.

### Prerequisites
- Keep your Deployments from Day 1 running

### Steps

**1. Create a ClusterIP Service** (`service-clusterip.yaml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

**2. Deploy it**:
```bash
kubectl apply -f service-clusterip.yaml
```

**3. Verify service creation**:
```bash
kubectl get service
kubectl describe service web-service
```

Look for:
- Cluster IP (internal IP address)
- Endpoints (should show IP addresses of matching pods)
- Port mapping (80→80)

**4. Test service from inside cluster** (only way to access ClusterIP):
```bash
# Run a temporary debug pod
kubectl run debug --image=alpine --rm -it -- sh

# Inside the pod:
wget http://web-service  # Access service by DNS name
# OR
wget http://web-service.default.svc.cluster.local  # Full DNS name

# Should get HTML response from nginx
cat index.html
```

**5. Observe DNS resolution**:
```bash
# From debug pod:
nslookup web-service
# Should resolve to Cluster IP
```

### Proof of Completion
Screenshots showing:
```bash
kubectl get service web-service
kubectl describe service web-service
```

**Show me**:
- Service has Cluster IP assigned
- Endpoints showing the 3 pods
- Successfully accessed service from debug pod

**Question to answer**: Why would you use ClusterIP instead of NodePort?

---

## Lab 2.2: NodePort Service (20 minutes)

### Objective
Create external service accessible from outside cluster.

### Steps

**1. Create a NodePort Service** (`service-nodeport.yaml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80           # Service internal port
    targetPort: 80     # Pod container port
    nodePort: 30001    # External port (30000-32767)
    protocol: TCP
```

**2. Deploy it**:
```bash
kubectl apply -f service-nodeport.yaml
```

**3. Verify creation**:
```bash
kubectl get service web-nodeport
kubectl describe service web-nodeport
```

Look for:
- Type: NodePort
- Port: 80
- NodePort: 30001
- Endpoints: IP addresses of pods

**4. Find node IP**:
```bash
kubectl get nodes -o wide
# Get the INTERNAL-IP of a node
```

**5. Access the service externally**:
```bash
# From your machine (outside cluster):
curl http://<NODE-IP>:30001

# Should get nginx HTML response
```

**6. Test load balancing** (optional):
```bash
# Add a header to see which pod you hit:
# First, add a label to pods to identify them:
kubectl get pods -o wide

# Note the pod names, add to different pods if you want to identify them
# Then access service multiple times and check pod logs
```

**7. Verify all replicas are in endpoints**:
```bash
kubectl describe service web-nodeport | grep "Endpoints:"
# Should show 3 different IPs
```

### Proof of Completion
Screenshots showing:
```bash
kubectl get service web-nodeport
kubectl describe service web-nodeport
curl http://<NODE-IP>:30001
```

**Show me**:
- Service created with NodePort: 30001
- Successfully accessed from external machine
- All 3 pods in Endpoints

**Question to answer**: What happens if a pod crashes while you're accessing the service?

---

## Lab 2.3: ConfigMap (25 minutes)

### Objective
Externalize configuration so same image works in dev/prod.

### Prerequisites
- Modify your deployment to use ConfigMap for configuration

### Steps

**1. Create a ConfigMap** (`app-config.yaml`):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  ENVIRONMENT: production
  LOG_LEVEL: INFO
  CACHE_TTL: "3600"
  DATABASE_HOST: "postgres.default.svc.cluster.local"
```

**2. Deploy it**:
```bash
kubectl apply -f app-config.yaml
```

**3. Verify**:
```bash
kubectl get configmap
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```

**4. Create a pod that uses the ConfigMap** (`pod-with-config.yaml`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-config-pod
spec:
  containers:
  - name: test
    image: alpine
    command: ['sh']
    args: ['-c', 'echo LOG_LEVEL=$LOG_LEVEL; echo DATABASE=$DATABASE_HOST; sleep 3600']
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    - name: ENVIRONMENT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: ENVIRONMENT
  restartPolicy: Never
```

**5. Deploy and verify**:
```bash
kubectl apply -f pod-with-config.yaml
kubectl logs test-config-pod

# Should output:
# LOG_LEVEL=INFO
# DATABASE=postgres.default.svc.cluster.local
# (plus environment variable)
```

**6. Modify ConfigMap and redeploy pod**:
```bash
# Edit configmap
kubectl edit configmap app-config
# Change LOG_LEVEL: INFO → LOG_LEVEL: DEBUG

# Delete and recreate pod
kubectl delete pod test-config-pod
kubectl apply -f pod-with-config.yaml
kubectl logs test-config-pod
# Should show LOG_LEVEL=DEBUG
```

**7. Bonus: Using ConfigMap as volume** (`pod-configmap-volume.yaml`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-pod
spec:
  containers:
  - name: test
    image: alpine
    command: ['sh']
    args: ['-c', 'cat /etc/config/LOG_LEVEL; sleep 3600']
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  restartPolicy: Never
```

**8. Deploy and check**:
```bash
kubectl apply -f pod-configmap-volume.yaml
kubectl exec configmap-volume-pod -- cat /etc/config/LOG_LEVEL
# Should output: INFO
```

### Proof of Completion
Screenshots showing:
```bash
kubectl get configmap
kubectl describe configmap app-config
kubectl logs test-config-pod
kubectl exec configmap-volume-pod -- cat /etc/config/LOG_LEVEL
```

**Show me**:
- ConfigMap created with values
- Pod able to access values via environment variables
- Pod able to read ConfigMap files from volume

---

## Lab 2.4: Secrets (20 minutes)

### Objective
Store sensitive data (passwords, API keys) separately.

### Steps

**1. Create a Secret** (two methods):

**Method 1: From literal values** (`secret-literal.yaml`):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  database-password: cGFzc3dvcmQxMjM=  # base64 encoded 'password123'
  api-key: c2VjcmV0YXBpa2V5              # base64 encoded 'secretapikey'
```

(Or create via CLI):
```bash
kubectl create secret generic app-secrets \
  --from-literal=database-password=password123 \
  --from-literal=api-key=secretapikey
```

**2. Verify creation**:
```bash
kubectl get secret
kubectl describe secret app-secrets
kubectl get secret app-secrets -o yaml
# Notice: values are base64 encoded, not encrypted by default!
```

**3. Decode to verify** (educational only):
```bash
# Get the encoded value
kubectl get secret app-secrets -o yaml | grep database-password

# Decode it (example, not secure):
echo "cGFzc3dvcmQxMjM=" | base64 -d
```

**4. Use Secret in Pod** (`pod-with-secret.yaml`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: alpine
    command: ['sh']
    args: ['-c', 'echo PASSWORD=$DB_PASSWORD; echo KEY=$API_KEY; sleep 3600']
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: database-password
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: api-key
  restartPolicy: Never
```

**5. Deploy and verify**:
```bash
kubectl apply -f pod-with-secret.yaml
kubectl logs secret-pod
# Should show:
# PASSWORD=password123
# KEY=secretapikey
```

**6. Security Note** (important):
```bash
# Check pod spec - secrets NOT visible
kubectl get pod secret-pod -o yaml | grep -i password
# Not shown in pod spec (good!)

# But they ARE in memory of running container
kubectl exec secret-pod -- env | grep PASSWORD
# Shows the actual value (pod can read it)
```

**7. Secret as files** (better practice):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-files-pod
spec:
  containers:
  - name: app
    image: alpine
    command: ['sh']
    args: ['-c', 'cat /etc/secrets/password; sleep 3600']
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secrets
  restartPolicy: Never
```

**8. Deploy and verify**:
```bash
kubectl apply -f secret-files-pod.yaml
kubectl exec secret-files-pod -- cat /etc/secrets/database-password
# Should show: password123
```

### Proof of Completion
Screenshots showing:
```bash
kubectl get secret
kubectl describe secret app-secrets
kubectl logs secret-pod
kubectl exec secret-files-pod -- cat /etc/secrets/database-password
```

**Show me**:
- Secret created
- Pod able to access secret values
- Understand difference: ConfigMap vs Secret (use cases)

**Question to answer**: Why use Secrets instead of just ConfigMaps? When would you use each?

---

## Lab 2.5: Putting It All Together (30 minutes)

### Objective
Create complete application stack with Deployment, Service, ConfigMap, and Secret.

### Steps

**1. Create namespace** (optional, but good practice):
```bash
kubectl create namespace demo
```

**2. Create ConfigMap**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo
data:
  ENVIRONMENT: production
  LOG_LEVEL: INFO
  APP_NAME: MyWebApp
```

**3. Create Secret**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: demo
type: Opaque
stringData:
  database-password: super-secret-password
  api-key: my-api-key-123
```

**4. Create Deployment**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: complete-app
  namespace: demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: complete-app
  template:
    metadata:
      labels:
        app: complete-app
    spec:
      containers:
      - name: app
        image: nginx:1.23
        ports:
        - containerPort: 80
        env:
        - name: ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: ENVIRONMENT
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-password
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

**5. Create Service**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: complete-app-service
  namespace: demo
spec:
  type: NodePort
  selector:
    app: complete-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30002
```

**6. Deploy everything**:
```bash
kubectl apply -f app-config.yaml
kubectl apply -f app-secrets.yaml
kubectl apply -f app-deployment.yaml
kubectl apply -f app-service.yaml

# Or single command:
kubectl apply -f .  # Applies all YAML files in current directory
```

**7. Verify all components**:
```bash
kubectl get all -n demo
kubectl get configmap -n demo
kubectl get secret -n demo

kubectl describe deployment complete-app -n demo
kubectl describe service complete-app-service -n demo

kubectl logs -n demo <pod-name>
```

**8. Test the service**:
```bash
# Get node IP
kubectl get nodes -o wide

# Access from outside
curl http://<NODE-IP>:30002
```

**9. Scale and observe**:
```bash
kubectl scale deployment complete-app --replicas=5 -n demo
kubectl get pods -n demo -w
```

**10. Update config and restart pods**:
```bash
# Update configmap
kubectl edit configmap app-config -n demo
# Change LOG_LEVEL: INFO → DEBUG

# Restart pods to pick up new config
kubectl rollout restart deployment/complete-app -n demo

# Watch pods restart
kubectl get pods -n demo -w
```

### Proof of Completion
Screenshots showing:
```bash
kubectl get all -n demo -o wide
kubectl describe deployment complete-app -n demo
kubectl describe service complete-app-service -n demo
curl http://<NODE-IP>:30002
```

**Show me**:
- All 3 components working together (Deployment, Service, ConfigMap, Secret)
- Service accessible externally
- Successfully scaled deployment

---

## Lab 2.6: Cleanup (5 minutes)

### Steps
```bash
# Delete all resources
kubectl delete namespace demo

# Or individual deletion:
kubectl delete deployment complete-app
kubectl delete service complete-app-service
kubectl delete configmap app-config
kubectl delete secret app-secrets

# Verify cleanup:
kubectl get all
```

---

## Day 2 Completion Checklist

- [ ] Lab 2.1: Created ClusterIP Service and accessed from debug pod
- [ ] Lab 2.2: Created NodePort Service and accessed externally
- [ ] Lab 2.3: Created ConfigMap and used in pod
- [ ] Lab 2.4: Created Secret and used in pod
- [ ] Lab 2.5: Deployed complete application with all components
- [ ] Lab 2.6: Cleaned up resources

---

# FINAL PROJECT: Complete Application Deployment

## Challenge
Deploy a complete application stack that demonstrates all concepts:

**Requirements**:
1. ✓ Deployment with 3+ replicas
2. ✓ Service for external access (NodePort)
3. ✓ ConfigMap for non-sensitive config
4. ✓ Secret for sensitive data
5. ✓ Health checks / livenessProbe
6. ✓ Resource limits

**Success Criteria**:
- [ ] Application accessible from external machine
- [ ] Configuration can be updated without redeploy
- [ ] Secrets are properly separated from config
- [ ] Demonstrate scaling up/down
- [ ] Demonstrate rolling update to new image
- [ ] All resources properly labeled and organized

**Deliverables**:
- All YAML manifest files (deployment.yaml, service.yaml, etc.)
- Screenshots showing:
  - kubectl get all
  - Service details with all endpoints
  - Successful external access
  - Scaling demonstration
  - Rolling update demonstration
- Brief document explaining your setup

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

