# Creating a Custom Helm Chart for AI-BankApp
>Yesterday we deployed MySQL with a community Helm chart. Today we build a custom Helm chart for the AI-BankApp itself - converting the 12 raw YAML files from the `k8s/` directory into a templated, configurable, reusable Helm chart.

The AI-BankApp (https://github.com/TrainWithShubham/AI-BankApp-DevOps, branch `feat/gitops`) has three services: the Spring Boot banking app, a MySQL database, and an Ollama AI chatbot.  By the end, all three will be deployable - with full configuration control - via `helm install`.

## Table of Contents 
1. [Scaffold the Chart and Study the Raw Manifests](#1-scaffold-the-chart-and-study-the-raw-manifests)
2. [Define Chart.yaml and values.yaml](#2-define-chartyaml-and-valuesyaml)
3. [Write the Core Templates](#3-write-the-core-templates)
4. [Write the Deployment Templates](#4-write-the-deployment-templates)
5. [Write the Services and HPA Templates](#5-write-the-services-and-hpa-templates)
6. [Validate and Deploy](#6-validate-and-deploy)

## 1. Scaffold the Chart and Study the Raw Manifests
### Clone and Enter the AI-BankApp Repo:
```bash
git clone -b feat/gitops https://github.com/TrainWithShubham/AI-BankApp-DevOps.git
cd AI-BankApp-DevOps
```
- This ensures we’re working with the correct branch (`feat/gitops`) where the raw Kubernetes manifests live.

### Inspect the Raw Manifests
Study the raw manifests we are converting:
```bash
ls k8s/
```
Map each file to what it does:
| File | Resource | Purpose |
|---|---|---|
| `namespace.yml` | Namespace | Creates the `bankapp` namespace |
| `configmap.yml` | ConfigMap | MySQL host, port, database name, Ollama URL |
| `secrets.yml` | Secret | MySQL credentials (hardcoded base64) |
| `pv.yml` | StorageClass | gp3 via the EBS CSI driver |
| `pvc.yml` | PersistentVolumeClaim | MySQL (5Gi) and Ollama (10Gi) volumes |
| `bankapp-deployment.yml` | Deployment | Spring Boot app with init containers and probes |
| `mysql-deployment.yml` | Deployment | MySQL with EBS volume mount and probes |
| `ollama-deployment.yml` | Deployment | Ollama with `postStart` model pull and probes |
| `service.yml` | Service | ClusterIP services for all three components |
| `hpa.yml` | HorizontalPodAutoscaler | BankApp autoscaling (2–4 replicas, 70% CPU) |
| `gateway.yml` | Gateway + HTTPRoute | Envoy Gateway with TLS termination |
| `cert-manager.yml` | ClusterIssuer | Let's Encrypt certificate provisioning |
- **Verify**: Open one file (e.g., `cat k8s/bankapp-deployment.yml`) and skim to confirm it matches the description above.

### Scaffold the Helm Chart
Now create a Helm chart skeleton:
```bash
mkdir helm-chart && cd helm-chart
helm create bankapp
```
- This generates a `bankapp/` directory with default templates, helpers, and values.
- **Verify**: Run `tree bankapp/` - we should see `Chart.yaml`, `values.yaml`, `templates/`, `_helpers.tpl`, `NOTES.txt`.

### Clean Up Default Templates
We don’t want the boilerplate templates because we’ll replace them with our own (based on the raw manifests).
```bash
rm -rf bankapp/templates/*.yaml bankapp/templates/tests/
```
- Keep `_helpers.tpl` (naming helpers, labels) and `NOTES.txt` (we’ll customize later).
- **Verify**: Run `ls bankapp/templates/` - only `_helpers.tpl` and `NOTES.txt` should remain.

### At this point, we’ve:
- Cloned the repo and confirmed the raw manifests.
- Mapped each manifest to its purpose.
- Scaffolded a Helm chart.
- Cleaned out boilerplate templates, ready to start templating.

## 2. Define Chart.yaml and values.yaml
### Edit `bankapp/Chart.yaml`
Replace the default content with:
```yaml
apiVersion: v2
name: bankapp
description: AI-BankApp -- Spring Boot banking application with MySQL and Ollama AI chatbot
type: application
version: 0.1.0
appVersion: "1.0.0"
maintainers:
  - name: TrainWithShubham
    url: https://github.com/TrainWithShubham
keywords:
  - bankapp
  - spring-boot
  - mysql
  - ollama
  - ai
```
- This metadata describes our chart. It’s what `helm search` and `helm show chart` will display.

### Create `bankapp/values.yaml`
Extract every hardcoded value from the raw manifests into configurable values:
```yaml
# BankApp configuration
bankapp:
  replicaCount: 4
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "latest"
    pullPolicy: Always
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  service:
    type: ClusterIP
    port: 8080
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 4
    targetCPUUtilization: 70

# MySQL configuration
mysql:
  enabled: true
  image:
    repository: mysql
    tag: "8.0"
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  persistence:
    size: 5Gi
    storageClass: gp3

# Ollama AI configuration
ollama:
  enabled: true
  image:
    repository: ollama/ollama
    tag: "latest"
  model: tinyllama
  resources:
    requests:
      memory: "2Gi"
      cpu: "900m"
    limits:
      memory: "2.5Gi"
      cpu: "1500m"
  persistence:
    size: 10Gi
    storageClass: gp3

# Shared configuration
config:
  mysqlDatabase: bankappdb
  ollamaUrl: ""  # Auto-generated from service name if empty

# Secrets
secrets:
  mysqlRootPassword: Test@123
  mysqlUser: root
  mysqlPassword: Test@123

# Storage
storageClass:
  create: true
  name: gp3
  provisioner: ebs.csi.aws.com

# Gateway (optional -- for EKS with Envoy Gateway)
gateway:
  enabled: false
  hostname: ""
  tls:
    enabled: false
```

**Key improvement over raw manifests:** `k8s/secrets.yml` contains hardcoded base64-encoded credentials. The Helm chart stores plaintext values here and encodes them at render time via the `b64enc` template function — no manual encoding, and credentials can be overridden per environment without editing any YAML.

### Verify
```bash
helm lint bankapp/
```
- Should pass with no errors.
- If we see warnings, they’re usually about unused values (we’ll fix them once templates reference these values).

### At this point, we’ve:
- Defined chart metadata in `Chart.yaml`.
- Extracted all hardcoded values into `values.yaml`.
- Prepared the foundation for templating.

## 3. Write the Core Templates
Convert the raw manifests into Helm templates. Each template uses `{{ .Values }}` instead of hardcoded values.

### Create the ConfigMap Template
`bankapp/templates/configmap.yaml` (from `k8s/configmap.yml`):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "bankapp.fullname" . }}-config
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
data:
  MYSQL_HOST: {{ include "bankapp.fullname" . }}-mysql
  MYSQL_PORT: "3306"
  MYSQL_DATABASE: {{ .Values.config.mysqlDatabase | quote }}
  OLLAMA_URL: {{ default (printf "http://%s-ollama:11434" (include "bankapp.fullname" .)) .Values.config.ollamaUrl | quote }}
  SERVER_FORWARD_HEADERS_STRATEGY: "native"
```
- This replaces hardcoded DB and Ollama URLs with dynamic values.

Verify
```bash
helm template my-bankapp bankapp/ | grep ConfigMap
```
- We should see the rendered config with our values.

### Create the Secrets Template
`bankapp/templates/secrets.yaml` (from `k8s/secrets.yml`):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "bankapp.fullname" . }}-secret
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: {{ .Values.secrets.mysqlRootPassword | b64enc | quote }}
  MYSQL_USER: {{ .Values.secrets.mysqlUser | b64enc | quote }}
  MYSQL_PASSWORD: {{ .Values.secrets.mysqlPassword | b64enc | quote }}
```
- Helm’s `b64enc` automatically encodes values, so you don’t need to manually base64 them.

Verify
```bash
helm template my-bankapp bankapp/ | grep MYSQL_ROOT_PASSWORD
```
- We should see an encoded string.

### Create the Storage Template
`bankapp/templates/storage.yaml` (from `k8s/pv.yml` + `k8s/pvc.yml`):
```yaml
{{- if .Values.storageClass.create }}
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: {{ .Values.storageClass.name }}
provisioner: {{ .Values.storageClass.provisioner }}
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
{{- end }}
---
{{- if .Values.mysql.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "bankapp.fullname" . }}-mysql-pvc
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
spec:
  storageClassName: {{ .Values.mysql.persistence.storageClass }}
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: {{ .Values.mysql.persistence.size }}
{{- end }}
---
{{- if .Values.ollama.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "bankapp.fullname" . }}-ollama-pvc
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
spec:
  storageClassName: {{ .Values.ollama.persistence.storageClass }}
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: {{ .Values.ollama.persistence.size }}
{{- end }}
```
- This makes PVC creation conditional and configurable. We can disable Ollama or MySQL entirely with a boolean.

Verify
```bash
helm template my-bankapp bankapp/ | grep PersistentVolumeClaim
```
- We should see PVCs for MySQL and Ollama.

### Validate
```bash
helm lint bankapp/
helm template my-bankapp bankapp/
```
- Check that all `{{ .Values }}` references resolve correctly.
- Ensure no YAML syntax errors.

### At this point, we’ve:
- Converted **ConfigMap**, **Secrets**, and **Storage** manifests into Helm templates.
- Made them configurable via `values.yaml`.
- Verified with `helm template`.

## 4. Write the Deployment Templates
### 1. BankApp Deployment
`bankapp/templates/bankapp-deployment.yaml` (from `k8s/bankapp-deployment.yml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "bankapp.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.bankapp.autoscaling.enabled }}
  replicas: {{ .Values.bankapp.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      app: {{ include "bankapp.fullname" . }}
  template:
    metadata:
      labels:
        app: {{ include "bankapp.fullname" . }}
    spec:
      initContainers:
        - name: wait-for-mysql
          image: busybox:1.36
          command: ["/bin/sh", "-c", "until nc -z {{ include "bankapp.fullname" . }}-mysql 3306; do sleep 2; done"]
          resources:
            requests: { memory: "32Mi", cpu: "50m" }
            limits: { memory: "64Mi", cpu: "100m" }
        {{- if .Values.ollama.enabled }}
        - name: wait-for-ollama
          image: busybox:1.36
          command: ["/bin/sh", "-c", "until nc -z {{ include "bankapp.fullname" . }}-ollama 11434; do sleep 2; done"]
          resources:
            requests: { memory: "32Mi", cpu: "50m" }
            limits: { memory: "64Mi", cpu: "100m" }
        {{- end }}
      containers:
        - name: bankapp
          image: "{{ .Values.bankapp.image.repository }}:{{ .Values.bankapp.image.tag }}"
          imagePullPolicy: {{ .Values.bankapp.image.pullPolicy }}
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: {{ include "bankapp.fullname" . }}-config
            - secretRef:
                name: {{ include "bankapp.fullname" . }}-secret
          {{- with .Values.bankapp.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 30
            failureThreshold: 15
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 5
```
**Notable template decisions:**
| Decision | Reason |
|---|---|
| `replicas` omitted when autoscaling enabled | HPA manages replica count — static `replicas` would conflict |
| Init container names reference `bankapp.fullname` | Service names are release-scoped, not hardcoded |
| Ollama init container wrapped in `{{- if }}` | Removing Ollama via values also removes its startup dependency |
| `/actuator/health` for probes | Spring Boot Actuator's built-in health endpoint |

Verify
```bash
helm template my-bankapp bankapp/ | grep Deployment
```
Check that replicas appear only if autoscaling is disabled.

### 2. MySQL Deployment
**`bankapp/templates/mysql-deployment.yaml`** (from `k8s/mysql-deployment.yml`):
```yaml
{{- if .Values.mysql.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "bankapp.fullname" . }}-mysql
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      app: {{ include "bankapp.fullname" . }}-mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: {{ include "bankapp.fullname" . }}-mysql
    spec:
      containers:
        - name: mysql
          image: "{{ .Values.mysql.image.repository }}:{{ .Values.mysql.image.tag }}"
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "bankapp.fullname" . }}-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: {{ include "bankapp.fullname" . }}-config
                  key: MYSQL_DATABASE
          {{- with .Values.mysql.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
          readinessProbe:
            exec:
              command: ["mysqladmin", "ping", "-h", "localhost"]
            initialDelaySeconds: 15
            failureThreshold: 10
          livenessProbe:
            exec:
              command: ["mysqladmin", "ping", "-h", "localhost"]
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 5
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: {{ include "bankapp.fullname" . }}-mysql-pvc
{{- end }}
```
- Uses secrets for root password.
- PVC mounts /var/lib/mysql.
- `strategy: Recreate` ensures MySQL shuts down completely before a new pod starts — necessary because MySQL uses a single-writer PVC that cannot be mounted by two pods simultaneously.

Verify
```bash
helm template my-bankapp bankapp/ | grep mysql
```
Check that env vars reference ConfigMap and Secret.

### 3. Ollama Deployment
`bankapp/templates/ollama-deployment.yaml` (from `k8s/ollama-deployment.yml`):
```yaml
{{- if .Values.ollama.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "bankapp.fullname" . }}-ollama
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      app: {{ include "bankapp.fullname" . }}-ollama
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: {{ include "bankapp.fullname" . }}-ollama
    spec:
      containers:
        - name: ollama
          image: "{{ .Values.ollama.image.repository }}:{{ .Values.ollama.image.tag }}"
          ports:
            - containerPort: 11434
          {{- with .Values.ollama.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          volumeMounts:
            - name: ollama-storage
              mountPath: /root/.ollama
          lifecycle:
            postStart:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - |
                    until ollama list > /dev/null 2>&1; do sleep 2; done
                    ollama pull {{ .Values.ollama.model }}
          readinessProbe:
            exec:
              command: ["/bin/sh", "-c", "ollama list | grep -q {{ .Values.ollama.model }}"]
            initialDelaySeconds: 30
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /
              port: 11434
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 5
      volumes:
        - name: ollama-storage
          persistentVolumeClaim:
            claimName: {{ include "bankapp.fullname" . }}-ollama-pvc
{{- end }}
```
>Notice: the Ollama model name (`tinyllama`) is now a value (`{{ .Values.ollama.model }}`). You can switch models without editing YAML.

Verify
```bash
helm template my-bankapp bankapp/ | grep ollama
```
Check that the model name comes from `values.yaml`.

### Validate
```bash
helm lint bankapp/
helm template my-bankapp bankapp/
```
- Ensure all templates render correctly.
- Confirm conditional blocks (`mysql.enabled`, `ollama.enabled`) work by toggling values in `values.yaml`.

### At this point, we’ve:
- Written **BankApp**, **MySQL**, and **Ollama** deployments.
- Preserved init containers, lifecycle hooks, probes, and PVCs.
- Made everything configurable via `values.yaml`.

## 5. Write the Services and HPA Templates
### 1. Services Template
`bankapp/templates/services.yaml` (from `k8s/service.yml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "bankapp.fullname" . }}-mysql
  namespace: {{ .Release.Namespace }}
spec:
  selector:
    app: {{ include "bankapp.fullname" . }}-mysql
  ports:
    - port: 3306
---
{{- if .Values.ollama.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "bankapp.fullname" . }}-ollama
  namespace: {{ .Release.Namespace }}
spec:
  selector:
    app: {{ include "bankapp.fullname" . }}-ollama
  ports:
    - port: 11434
{{- end }}
---
apiVersion: v1
kind: Service
metadata:
  name: {{ include "bankapp.fullname" . }}-service
  namespace: {{ .Release.Namespace }}
spec:
  type: {{ .Values.bankapp.service.type }}
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
  selector:
    app: {{ include "bankapp.fullname" . }}
  ports:
    - port: {{ .Values.bankapp.service.port }}
      targetPort: 8080
```
- MySQL and Ollama get internal ClusterIP services.
- BankApp gets a configurable service (`ClusterIP`, `NodePort`, or `LoadBalancer`).
- Session affinity ensures sticky sessions for clients.

Verify
```bash
helm template my-bankapp bankapp/ | grep Service
```
Check that all three services render, and Ollama disappears if `ollama.enabled=false`.

### 2. HorizontalPodAutoscaler (HPA) Template
`bankapp/templates/hpa.yaml` (from `k8s/hpa.yml`):
```yaml
{{- if .Values.bankapp.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "bankapp.fullname" . }}-hpa
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "bankapp.fullname" . }}
  minReplicas: {{ .Values.bankapp.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.bankapp.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.bankapp.autoscaling.targetCPUUtilization }}
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
{{- end }}
```
- Autoscaling is conditional - only created if enabled.
- CPU utilization target is configurable.
- Scale-up and scale-down behavior windows prevent flapping.

Verify
```bash
helm template my-bankapp bankapp/ | grep HorizontalPodAutoscaler
```
- Check that the HPA appears only when `bankapp.autoscaling.enabled=true`.

### At this point, we’ve:
- Written **Services** for MySQL, Ollama, and BankApp.
- Added a conditional **HPA** for BankApp.
- Verified that toggling values controls which resources are created.

## 6. Validate and Deploy
### Lint the Chart
```bash
helm lint bankapp/
```
- Catches common issues (missing fields, bad YAML, template errors).

### Render Templates Locally
```bash
helm template my-bankapp bankapp/
```
- Shows the final YAML without deploying.
- Every `{{ }}` should be resolved to actual values. We should see full manifests for ConfigMap, Secrets, Deployments, Services, PVCs, etc.

### Render with Overrides
Try:
```bash
helm template my-bankapp bankapp/ \
  --set bankapp.image.tag=abc1234 \
  --set bankapp.replicaCount=2 \
  --set ollama.enabled=false
```
-  Confirms that values can be overridden at install time.
- BankApp image tag should be `abc1234`.
- Replicas should be `2` (if autoscaling disabled).

>Notice: setting `ollama.enabled=false` removes the Ollama Deployment, Service, PVC, and the init container from the BankApp. One boolean controls an entire component.

### Dry Run Against Cluster
```bash
helm install my-bankapp bankapp/ --dry-run --debug -n bankapp --create-namespace
```
- Simulates install, checks cluster compatibility.
- Output shows rendered manifests with no errors.

### Deploy for Real 
On Kind (skip StorageClass creation since Kind uses its own):
```bash
helm install my-bankapp bankapp/ \
  -n bankapp --create-namespace \
  --set storageClass.create=false \
  --set mysql.persistence.storageClass=standard \
  --set ollama.persistence.storageClass=standard
```
- Installs chart into `bankapp` namespace.

Verify
```bash
helm list -n bankapp
```
- We should see `my-bankapp` release.

### Verify Resources
```bash
kubectl get all -n bankapp
kubectl get pvc -n bankapp
kubectl get configmap,secret -n bankapp
```
- Confirms Deployments, Pods, Services, PVCs, ConfigMaps, and Secrets are created.
- Pods should be in `Running` state. Ollama may take time to pull the model.

Wait for all pods to be ready (Ollama takes time to pull the model):
```bash
kubectl get pods -n bankapp -w
```

### Access the App
Forward the service:
```bash
kubectl port-forward svc/my-bankapp-bankapp-service -n bankapp 8080:8080
```
Open:
```Code
http://localhost:8080
```
- We should see the AI-BankApp login page.

>**Compare: 12 raw YAML files vs 1 Helm command.** Same result, but now configurable, versionable, and rollback-safe.

### Clean Up
```bash
helm uninstall my-bankapp -n bankapp
```
- Removes all resources created by the chart.

Verify
```bash
helm list -n bankapp
```
- Release should be gone.

### At this point, we’ve:
- Validated your chart with lint and template.
- Tested overrides and conditional logic.
- Deployed the full stack with one Helm command.
- Verified resources and accessed the app.
- Cleaned up safely.

## Summary - Raw Manifests vs. Helm Chart
| Dimension | 12 Raw YAML Files | Custom Helm Chart |
|---|---|---|
| **Deploy command** | `kubectl apply -f k8s/` (all 12 files) | `helm install my-bankapp bankapp/` |
| **Image tag change** | Edit `bankapp-deployment.yml` | `--set bankapp.image.tag=<tag>` |
| **Disable a component** | Delete or comment out 3–4 files | `--set ollama.enabled=false` |
| **Environment config** | Duplicate files per environment | One `values-<env>.yaml` per environment |
| **Secret management** | Hardcoded base64 in `secrets.yml` | Plaintext in values, encoded at render time |
| **Rollback** | `git revert` + `kubectl apply` | `helm rollback my-bankapp <revision>` |