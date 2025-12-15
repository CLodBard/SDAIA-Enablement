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