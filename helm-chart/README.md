# 🚀 Jewelry Store - Helm Chart

Helm Chart מלא לפריסת אפליקציית Jewelry Store על Kubernetes (Frontend, Backend, Auth Service).

---

## 📋 תוכן עניינים

1. [סקירה כללית](#סקירה-כללית)
2. [מבנה ה-Chart](#מבנה-ה-chart)
3. [דרישות מוקדמות](#דרישות-מוקדמות)
4. [התקנה](#התקנה)
5. [שימוש עם Jenkins](#שימוש-עם-jenkins)
6. [הגדרות (values.yaml)](#הגדרות-valuesyaml)
7. [פקודות שימושיות](#פקודות-שימושיות)

---

## 🎯 סקירה כללית

ה-Helm Chart מפרס 3 שירותים:

### **Frontend (jewelry-store)**
- React + NGINX
- פורט: 80
- משתני סביבה: REACT_APP_*
- **אין JWT Secret!**
- כולל: Ingress, PVC (persistent volume), HPA

### **Backend (jewelry-backend)**
- FastAPI
- פורט: 8000
- משתני סביבה: AUTH_SERVICE_URL
- **יש JWT Secret!**
- כולל: HPA

### **Auth Service (jewelry-auth)**
- FastAPI
- פורט: 8001
- **יש JWT Secret!**
- כולל: HPA

---

## 📁 מבנה ה-Chart

```
helm-chart/
├── Chart.yaml                           # מטא-דאטה
├── values.yaml                          # הגדרות ברירת מחדל
├── values-dev.yaml                      # הגדרות dev
├── values-prod.yaml                     # הגדרות production
├── Jenkinsfile.example                  # דוגמת Jenkinsfile
├── README.md                            # המדריך הזה
└── templates/
    ├── jwt-secret.yaml                  # JWT Secret (משותף)
    ├── frontend/
    │   ├── configmap.yaml               # משתני סביבה
    │   ├── deployment.yaml              # Deployment + PVC
    │   ├── service.yaml                 # ClusterIP Service
    │   ├── ingress.yaml                 # Ingress
    │   ├── pvc.yaml                     # Persistent Volume Claim
    │   └── hpa.yaml                     # Horizontal Pod Autoscaler
    ├── backend/
    │   ├── configmap.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── hpa.yaml
    └── auth-service/
        ├── configmap.yaml
        ├── deployment.yaml
        ├── service.yaml
        └── hpa.yaml
```

---

## 🔧 דרישות מוקדמות

### 1. כלים:
```bash
# Helm 3
helm version

# kubectl
kubectl version --client

# Docker (לבניית images)
docker --version
```

### 2. Kubernetes Cluster:
```bash
# Minikube (לפיתוח מקומי)
minikube start

# הפעל addons
minikube addons enable ingress
minikube addons enable metrics-server
```

### 3. Docker Registry:
```bash
# Registry זמין (בדוגמה: localhost:8082)
# או שנה ב-values.yaml ל-registry שלך
```

---

## 🚀 התקנה

### אופציה 1: התקנה בסיסית

```bash
# 1. Build images
docker build -t localhost:8082/jewelry-store:v1.0.0 ./jewelry-store
docker push localhost:8082/jewelry-store:v1.0.0

docker build -t localhost:8082/jewelry-backend:v1.0.0 ./backend
docker push localhost:8082/jewelry-backend:v1.0.0

docker build -t localhost:8082/jewelry-auth:v1.0.0 ./auth-service
docker push localhost:8082/jewelry-auth:v1.0.0

# 2. הפעלה עם Helm
helm install jewelry-store ./helm-chart \
  --set image.frontendTag=v1.0.0 \
  --set image.backendTag=v1.0.0 \
  --set image.authTag=v1.0.0 \
  --set jwtSecret.key="your-super-secret-jwt-key-here" \
  --namespace default
```

### אופציה 2: עם values file (Dev)

```bash
helm install jewelry-store ./helm-chart \
  -f helm-chart/values-dev.yaml \
  --set image.frontendTag=dev-123 \
  --set image.backendTag=dev-123 \
  --set image.authTag=dev-123 \
  --set jwtSecret.key="dev-secret-key" \
  --namespace default
```

### אופציה 3: עם values file (Production)

```bash
helm install jewelry-store ./helm-chart \
  -f helm-chart/values-prod.yaml \
  --set image.registry=my-prod-registry.com \
  --set image.frontendTag=v1.0.0 \
  --set image.backendTag=v1.0.0 \
  --set image.authTag=v1.0.0 \
  --set jwtSecret.key="${JWT_SECRET_FROM_VAULT}" \
  --namespace production \
  --create-namespace
```

---

## 🔐 שימוש עם Jenkins

### 1. הוסף JWT Secret ל-Jenkins Credentials

```
1. Jenkins Dashboard → Manage Jenkins → Credentials
2. Add Credentials
3. Kind: Secret text
4. ID: JWT_SECRET_KEY
5. Secret: your-actual-jwt-secret
```

### 2. העתק את Jenkinsfile

```bash
cp helm-chart/Jenkinsfile.example Jenkinsfile-helm
```

### 3. עדכן את הפרמטרים:

```groovy
environment {
    REGISTRY_URL = "your-registry.com"  // שנה לregistry שלך
    NAMESPACE = "default"                // או namespace אחר
    HELM_RELEASE_NAME = "jewelry-store"
    IMAGE_TAG = "${env.BUILD_NUMBER}"
}
```

### 4. הרץ Pipeline!

```bash
# Jenkins ידאג ל:
# 1. Build של 3 ה-images (parallel)
# 2. Push לregistry
# 3. Deploy עם Helm + injection של JWT + Image tags
# 4. אימות הפריסה
```

---

## ⚙️ הגדרות (values.yaml)

### משתני סביבה חשובים:

#### Frontend:
```yaml
frontend:
  env:
    REACT_APP_API_BASE_URL: "http://backend-service:8000/api"
    REACT_APP_AUTH_BASE_URL: "http://auth-service:8001"
    REACT_APP_BACKEND_HOST: "backend-service"
    REACT_APP_BACKEND_PORT: "8000"
```

#### Backend:
```yaml
backend:
  env:
    AUTH_SERVICE_URL: "http://auth-service:8001"
```

#### Auth:
```yaml
authService:
  env: {}  # רק JWT_SECRET_KEY (אוטומטי)
```

### Image Tags:

```yaml
image:
  registry: localhost:8082
  frontendTag: latest  # ישונה ב-Jenkins
  backendTag: latest
  authTag: latest
```

### JWT Secret:

```yaml
jwtSecret:
  key: ""  # ⚠️ יוזרק דרך --set מ-Jenkins!
```

### Ingress (Frontend):

```yaml
frontend:
  ingress:
    enabled: true
    className: nginx
    host: jewelry-store.local
    path: /
```

### PVC (Frontend):

```yaml
frontend:
  persistence:
    enabled: true
    storageClass: standard
    size: 1Gi
```

### HPA (Autoscaling):

```yaml
frontend:
  hpa:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 50
```

---

## 📊 פקודות שימושיות

### התקנה ועדכון:

```bash
# התקנה ראשונית
helm install jewelry-store ./helm-chart --set jwtSecret.key="secret"

# עדכון (upgrade)
helm upgrade jewelry-store ./helm-chart \
  --set image.frontendTag=v1.0.1 \
  --set jwtSecret.key="secret"

# התקנה/עדכון ביחד
helm upgrade --install jewelry-store ./helm-chart \
  --set jwtSecret.key="secret"

# Dry-run (בדיקה ללא פריסה)
helm install jewelry-store ./helm-chart \
  --dry-run --debug \
  --set jwtSecret.key="secret"
```

### בדיקות:

```bash
# בדוק מה יפורס
helm template jewelry-store ./helm-chart

# רשימת releases
helm list

# סטטוס
helm status jewelry-store

# היסטוריה
helm history jewelry-store
```

### Rollback:

```bash
# חזרה לגירסה קודמת
helm rollback jewelry-store

# חזרה לגירסה ספציפית
helm rollback jewelry-store 3
```

### מחיקה:

```bash
# מחיקת ה-release
helm uninstall jewelry-store

# מחיקה כולל namespace
helm uninstall jewelry-store -n production
kubectl delete namespace production
```

### Kubernetes Commands:

```bash
# Pods
kubectl get pods -l app.kubernetes.io/instance=jewelry-store

# Services
kubectl get services

# Ingress
kubectl get ingress

# HPA
kubectl get hpa

# Logs
kubectl logs -l app=frontend --tail=100 -f

# Describe pod
kubectl describe pod <pod-name>

# Events
kubectl get events --sort-by='.lastTimestamp'
```

---

## 🔍 Debugging

### בדיקת templates:

```bash
# ראה מה Helm יוצר
helm template jewelry-store ./helm-chart \
  --set jwtSecret.key="test" \
  --set image.frontendTag=v1.0.0

# שמור ב-file
helm template jewelry-store ./helm-chart \
  --set jwtSecret.key="test" \
  > rendered-templates.yaml
```

### בדיקת values:

```bash
# ראה את כל ה-values (merged)
helm get values jewelry-store

# ראה את ה-values שנשלחו בפריסה
helm get values jewelry-store --all
```

### בעיות נפוצות:

#### 1. Pod לא עולה:
```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

#### 2. Image pull error:
```bash
# בדוק registry
kubectl describe pod <pod-name> | grep -i image

# בדוק שה-image קיים
docker pull localhost:8082/jewelry-store:v1.0.0
```

#### 3. JWT Secret חסר:
```bash
# בדוק שה-secret נוצר
kubectl get secret jwt-secret

# ראה את התוכן (base64)
kubectl get secret jwt-secret -o yaml
```

#### 4. Ingress לא עובד:
```bash
# בדוק ingress controller
kubectl get pods -n ingress-nginx

# בדוק ingress
kubectl describe ingress frontend-ingress

# For minikube
minikube addons enable ingress
```

---

## 🎯 דוגמאות שימוש

### 1. פריסה מקומית (Dev):

```bash
# Build images
docker build -t localhost:8082/jewelry-store:dev ./jewelry-store
docker build -t localhost:8082/jewelry-backend:dev ./backend
docker build -t localhost:8082/jewelry-auth:dev ./auth-service

# Push
docker push localhost:8082/jewelry-store:dev
docker push localhost:8082/jewelry-backend:dev
docker push localhost:8082/jewelry-auth:dev

# Deploy
helm upgrade --install jewelry-store ./helm-chart \
  -f helm-chart/values-dev.yaml \
  --set image.frontendTag=dev \
  --set image.backendTag=dev \
  --set image.authTag=dev \
  --set jwtSecret.key="dev-secret-123"

# Access
echo "$(minikube ip) jewelry-store-dev.local" | sudo tee -a /etc/hosts
open http://jewelry-store-dev.local
```

### 2. פריסה ב-Production:

```bash
# With CI/CD (Jenkins)
# See Jenkinsfile.example

# Or manually:
helm upgrade --install jewelry-store ./helm-chart \
  -f helm-chart/values-prod.yaml \
  --set image.registry=my-registry.com \
  --set image.frontendTag=v1.2.3 \
  --set image.backendTag=v1.2.3 \
  --set image.authTag=v1.2.3 \
  --set jwtSecret.key="${PROD_JWT_SECRET}" \
  --namespace production \
  --create-namespace \
  --atomic \
  --timeout 10m
```

### 3. עדכון רק של image tag:

```bash
# עדכון לגירסה חדשה
helm upgrade jewelry-store ./helm-chart \
  --reuse-values \
  --set image.frontendTag=v1.0.2 \
  --set image.backendTag=v1.0.2 \
  --set image.authTag=v1.0.2
```

---

## ✅ Checklist לפני Production

- [ ] JWT Secret מאובטח (לא hardcoded!)
- [ ] Image tags ספציפיים (לא :latest)
- [ ] Resource limits מוגדרים נכון
- [ ] HPA enabled ומוגדר
- [ ] Ingress עם TLS (HTTPS)
- [ ] Monitoring פועל
- [ ] Backups של PVC
- [ ] Rollback plan מוכן

---

## 📚 מקורות נוספים

- [Helm Documentation](https://helm.sh/docs/)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Jenkins with Helm](https://plugins.jenkins.io/kubernetes-helm/)

---

## 🆘 תמיכה

אם יש בעיות:
1. בדוק logs: `kubectl logs -l app=<service-name>`
2. בדוק events: `kubectl get events`
3. בדוק helm: `helm status jewelry-store`
4. בדוק templates: `helm template ./helm-chart`

---

**בהצלחה! 🚀**
