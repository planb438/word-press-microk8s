Using Argo CD to deploy WordPress is an excellent GitOps approach. Here's how to do it:

Step 1: Create a Git Repository for Your WordPress Manifest
First, create a new GitHub repository (or any Git repo) for your WordPress configuration.

--

Step 2: Create WordPress Application Manifest
Create wordpress-app.yaml in your Git repository:

yaml
# wordpress-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: wordpress
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-username/your-wordpress-repo.git
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: wordpress
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
    - CreateNamespace=true
    
--
    
Step 3: Create WordPress Kubernetes Manifests
Create a manifests/ directory in your Git repo with these files:

manifests/namespace.yaml
yaml

--

apiVersion: v1
kind: Namespace
metadata:
  name: wordpress
manifests/mysql-secret.yaml
yaml

--

apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: wordpress
type: Opaque
data:
  password: cGFzc3dvcmQ=  # "password" base64 encoded
manifests/mysql.yaml
yaml

--

apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  namespace: wordpress
  labels:
    app: wordpress-mysql
spec:
  selector:
    matchLabels:
      app: wordpress-mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress-mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wordpress
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
          
---

apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  namespace: wordpress
spec:
  ports:
  - port: 3306
  selector:
    app: wordpress-mysql
  clusterIP: None

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  namespace: wordpress
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
manifests/wordpress.yaml
yaml

--

apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - image: wordpress:latest
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_USER
          value: wordpress
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: WORDPRESS_DB_NAME
          value: wordpress
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wordpress-pv-claim
        
---

apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress
spec:
  ports:
  - port: 80
  selector:
    app: wordpress
  type: LoadBalancer

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pv-claim
  namespace: wordpress
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
manifests/kustomization.yaml
yaml

--

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- namespace.yaml
- mysql-secret.yaml
- mysql.yaml
- wordpress.yaml

--

Step 4: Deploy Using Argo CD
bash
# Apply the Application manifest to Argo CD
microk8s kubectl apply -f wordpress-app.yaml

# Alternatively, use argocd CLI if installed
argocd app create wordpress \
  --repo https://github.com/your-username/your-wordpress-repo.git \
  --path manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace wordpress \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# Sync the application
argocd app sync wordpress

# Check application status
argocd app get wordpress
Step 5: Monitor the Deployment
bash
# Watch the application deployment
argocd app get wordpress

# Check pods
microk8s kubectl get pods -n wordpress --watch

# Check services
microk8s kubectl get svc -n wordpress

# Get the WordPress URL
WORDPRESS_IP=$(microk8s kubectl get svc -n wordpress wordpress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "WordPress URL: http://$WORDPRESS_IP"
Step 6: Set Up Automatic Syncing (Optional)
yaml
# Add this to your wordpress-app.yaml for automatic syncing
syncPolicy:
  automated:
    selfHeal: true
    prune: true
  syncOptions:
  - CreateNamespace=true
Step 7: Add Ingress for Better Access (Optional)
manifests/ingress.yaml
yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress-ingress
  namespace: wordpress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: wordpress.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wordpress
            port:
              number: 80
Add the ingress to your kustomization.yaml:

yaml
resources:
- namespace.yaml
- mysql-secret.yaml
- mysql.yaml
- wordpress.yaml
- ingress.yaml
Step 8: Update and Sync
Whenever you make changes to your Git repository:

bash
# Argo CD will automatically detect changes and sync
# Or manually sync if needed
argocd app sync wordpress

# Check sync status
argocd app get wordpress
Step 9: Access Argo CD UI to Manage
bash
# Get Argo CD admin password
argocd admin initial-password -n argocd

# Port-forward to access Argo CD UI
microk8s kubectl port-forward -n argocd svc/argocd-server 8080:80
Then access Argo CD at http://localhost:8080 and you'll see your WordPress application with sync status and health.

Benefits of This Approach:
GitOps: All configuration is version controlled in Git

Automation: Automatic syncing and self-healing

Audit Trail: All changes are tracked in Git history

Consistency: Same configuration across environments

Easy Rollbacks: Revert to previous Git commits if needed

This setup gives you a fully automated WordPress deployment managed by Argo CD!
