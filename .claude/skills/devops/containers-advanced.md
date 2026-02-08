# Skill: Containers Advanced

## Purpose

Orquestación de contenedores con Kubernetes para desplegar y escalar aplicaciones Rails en producción.

## Kubernetes Basics

### Arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│                      Control Plane                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│  │ API      │  │ etcd     │  │Scheduler │  │ Controller │  │
│  │ Server   │  │          │  │          │  │ Manager    │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────────┘  │
└─────────────────────────────────────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
    ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
    │   Node 1    │ │   Node 2    │ │   Node 3    │
    │ ┌────────┐  │ │ ┌────────┐  │ │ ┌────────┐  │
    │ │ kubelet│  │ │ │ kubelet│  │ │ │ kubelet│  │
    │ └────────┘  │ │ └────────┘  │ │ └────────┘  │
    │ ┌────────┐  │ │ ┌────────┐  │ │ ┌────────┐  │
    │ │ Pod    │  │ │ │ Pod    │  │ │ │ Pod    │  │
    │ │ Pod    │  │ │ │ Pod    │  │ │ │ Pod    │  │
    │ └────────┘  │ │ └────────┘  │ │ └────────┘  │
    └─────────────┘ └─────────────┘ └─────────────┘
```

### Conceptos principales

| Concepto | Descripción |
|----------|-------------|
| **Pod** | Unidad mínima, uno o más contenedores |
| **Deployment** | Gestiona réplicas de Pods |
| **Service** | Expone Pods como servicio de red |
| **Ingress** | Gestiona acceso HTTP externo |
| **ConfigMap** | Configuración no sensible |
| **Secret** | Datos sensibles encriptados |
| **PersistentVolume** | Almacenamiento persistente |
| **Namespace** | Aislamiento lógico de recursos |

## kubectl Comandos Esenciales

### Información del cluster

```bash
# Ver info del cluster
kubectl cluster-info
kubectl version

# Ver nodos
kubectl get nodes
kubectl describe node <node-name>

# Ver namespaces
kubectl get namespaces
kubectl get ns
```

### Gestión de Pods

```bash
# Listar pods
kubectl get pods
kubectl get pods -n production
kubectl get pods --all-namespaces
kubectl get pods -o wide  # Más info

# Describir pod
kubectl describe pod <pod-name>

# Logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>  # Multi-container
kubectl logs -f <pod-name>  # Follow
kubectl logs --tail=100 <pod-name>  # Últimas 100 líneas

# Ejecutar comando en pod
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec <pod-name> -- rails console

# Copiar archivos
kubectl cp <pod-name>:/app/log/production.log ./production.log
kubectl cp ./file.txt <pod-name>:/app/

# Port forward
kubectl port-forward <pod-name> 3000:3000
kubectl port-forward service/rails-app 3000:80
```

### Gestión de Deployments

```bash
# Listar deployments
kubectl get deployments
kubectl get deploy

# Crear deployment
kubectl create deployment rails-app --image=myapp:latest

# Escalar
kubectl scale deployment rails-app --replicas=5

# Actualizar imagen
kubectl set image deployment/rails-app rails=myapp:v2

# Ver historial
kubectl rollout history deployment/rails-app

# Rollback
kubectl rollout undo deployment/rails-app
kubectl rollout undo deployment/rails-app --to-revision=2

# Ver status del rollout
kubectl rollout status deployment/rails-app

# Pausar/reanudar rollout
kubectl rollout pause deployment/rails-app
kubectl rollout resume deployment/rails-app
```

### Gestión de Services

```bash
# Listar services
kubectl get services
kubectl get svc

# Exponer deployment
kubectl expose deployment rails-app --port=80 --target-port=3000

# Describir service
kubectl describe service rails-app
```

### Aplicar manifests

```bash
# Aplicar archivo
kubectl apply -f deployment.yaml

# Aplicar directorio
kubectl apply -f ./k8s/

# Aplicar con namespace
kubectl apply -f deployment.yaml -n production

# Eliminar
kubectl delete -f deployment.yaml
kubectl delete deployment rails-app
kubectl delete pod <pod-name>
```

### Debug

```bash
# Ver eventos
kubectl get events
kubectl get events --sort-by='.lastTimestamp'

# Ver recursos
kubectl top nodes
kubectl top pods

# Dry run
kubectl apply -f deployment.yaml --dry-run=client

# Diff antes de aplicar
kubectl diff -f deployment.yaml
```

## Helm

### Conceptos

```
Chart = Paquete de manifests de Kubernetes
Release = Instancia de un Chart deployada
Values = Configuración personalizada
Repository = Colección de Charts
```

### Comandos básicos

```bash
# Añadir repositorio
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Buscar charts
helm search repo postgresql
helm search hub rails

# Instalar chart
helm install my-release bitnami/postgresql
helm install my-release bitnami/postgresql -f values.yaml
helm install my-release bitnami/postgresql --set postgresqlPassword=secret

# Listar releases
helm list
helm list -A  # Todos los namespaces

# Ver valores
helm show values bitnami/postgresql
helm get values my-release

# Actualizar release
helm upgrade my-release bitnami/postgresql -f values.yaml

# Rollback
helm rollback my-release 1

# Desinstalar
helm uninstall my-release

# Ver historial
helm history my-release
```

### Crear Chart propio

```bash
# Crear estructura
helm create myapp-chart

# Estructura generada:
myapp-chart/
├── Chart.yaml          # Metadata
├── values.yaml         # Valores por defecto
├── charts/             # Dependencias
├── templates/          # Manifests con templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── _helpers.tpl    # Funciones helper
│   └── NOTES.txt       # Instrucciones post-install
└── .helmignore
```

### Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
description: Rails application Helm chart
type: application
version: 0.1.0
appVersion: "1.0.0"

dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

### values.yaml

```yaml
# values.yaml
replicaCount: 3

image:
  repository: myregistry.com/myapp
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.example.com

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

rails:
  env: production
  masterKey: ""  # Se provee en install

postgresql:
  enabled: true
  auth:
    username: myapp
    database: myapp_production

redis:
  enabled: true
  auth:
    enabled: false
```

### Template de deployment

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 3000
          env:
            - name: RAILS_ENV
              value: {{ .Values.rails.env }}
            - name: RAILS_MASTER_KEY
              valueFrom:
                secretKeyRef:
                  name: {{ include "myapp.fullname" . }}-secrets
                  key: rails-master-key
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "myapp.fullname" . }}-secrets
                  key: database-url
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: /up
              port: http
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /up
              port: http
            initialDelaySeconds: 5
```

## ConfigMaps y Secrets

### ConfigMap

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rails-config
data:
  RAILS_ENV: "production"
  RAILS_LOG_TO_STDOUT: "true"
  RAILS_SERVE_STATIC_FILES: "true"
  WEB_CONCURRENCY: "2"
  RAILS_MAX_THREADS: "5"

  # Archivo de configuración
  database.yml: |
    production:
      adapter: postgresql
      encoding: unicode
      pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
      url: <%= ENV['DATABASE_URL'] %>
```

```yaml
# Usar en deployment
spec:
  containers:
    - name: rails
      envFrom:
        - configMapRef:
            name: rails-config
      volumeMounts:
        - name: config-volume
          mountPath: /app/config/database.yml
          subPath: database.yml
  volumes:
    - name: config-volume
      configMap:
        name: rails-config
```

### Secrets

```bash
# Crear secret desde literal
kubectl create secret generic rails-secrets \
  --from-literal=rails-master-key=abc123 \
  --from-literal=database-url=postgres://user:pass@host/db

# Crear desde archivo
kubectl create secret generic rails-secrets \
  --from-file=./master.key

# Ver secret (base64)
kubectl get secret rails-secrets -o yaml

# Decodificar
kubectl get secret rails-secrets -o jsonpath='{.data.rails-master-key}' | base64 -d
```

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: rails-secrets
type: Opaque
stringData:  # Se codifica automáticamente
  rails-master-key: "abc123def456"
  database-url: "postgres://user:pass@host:5432/myapp"

# O con data (ya codificado en base64)
# data:
#   rails-master-key: YWJjMTIzZGVmNDU2
```

```yaml
# Usar en deployment
spec:
  containers:
    - name: rails
      env:
        - name: RAILS_MASTER_KEY
          valueFrom:
            secretKeyRef:
              name: rails-secrets
              key: rails-master-key
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: rails-secrets
              key: database-url
```

### Sealed Secrets (para Git)

```bash
# Instalar kubeseal
brew install kubeseal

# Crear sealed secret
kubectl create secret generic rails-secrets \
  --from-literal=rails-master-key=abc123 \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > sealed-secret.yaml

# El sealed secret se puede commitear
```

## Scaling

### Horizontal Pod Autoscaler (HPA)

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: rails-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: rails-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
```

```bash
# Ver HPA
kubectl get hpa
kubectl describe hpa rails-app-hpa

# Crear HPA rápido
kubectl autoscale deployment rails-app --min=2 --max=10 --cpu-percent=70
```

### Vertical Pod Autoscaler (VPA)

```yaml
# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: rails-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: rails-app
  updatePolicy:
    updateMode: "Auto"  # Off, Initial, Recreate, Auto
  resourcePolicy:
    containerPolicies:
      - containerName: rails
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 2
          memory: 2Gi
```

### Cluster Autoscaler

```yaml
# Para EKS
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: us-west-2

nodeGroups:
  - name: ng-1
    instanceType: t3.medium
    desiredCapacity: 3
    minSize: 2
    maxSize: 10
    iam:
      withAddonPolicies:
        autoScaler: true

# El cluster autoscaler se instala como addon
```

## Networking

### Service Types

```yaml
# ClusterIP (interno)
apiVersion: v1
kind: Service
metadata:
  name: rails-app
spec:
  type: ClusterIP  # Default
  selector:
    app: rails-app
  ports:
    - port: 80
      targetPort: 3000

---
# NodePort (expone en puerto del nodo)
apiVersion: v1
kind: Service
metadata:
  name: rails-app-nodeport
spec:
  type: NodePort
  selector:
    app: rails-app
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30080  # 30000-32767

---
# LoadBalancer (cloud provider)
apiVersion: v1
kind: Service
metadata:
  name: rails-app-lb
spec:
  type: LoadBalancer
  selector:
    app: rails-app
  ports:
    - port: 80
      targetPort: 3000
```

### Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rails-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rails-app
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: rails-api
                port:
                  number: 80
```

### Network Policies

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: rails-app-policy
spec:
  podSelector:
    matchLabels:
      app: rails-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: nginx-ingress
      ports:
        - port: 3000
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgresql
      ports:
        - port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - port: 6379
```

## Storage

### PersistentVolume y PersistentVolumeClaim

```yaml
# pv.yaml (admin crea)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: rails-uploads-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  # AWS EBS
  awsElasticBlockStore:
    volumeID: vol-xxx
    fsType: ext4

---
# pvc.yaml (dev solicita)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rails-uploads-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

```yaml
# Usar en deployment
spec:
  containers:
    - name: rails
      volumeMounts:
        - name: uploads
          mountPath: /app/storage
  volumes:
    - name: uploads
      persistentVolumeClaim:
        claimName: rails-uploads-pvc
```

### StorageClass

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

## Monitoring

### Prometheus + Grafana

```yaml
# prometheus-values.yaml para Helm
alertmanager:
  enabled: true

server:
  retention: "15d"
  persistentVolume:
    size: 50Gi

# Instalar
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack -f prometheus-values.yaml
```

### Métricas de Rails

```ruby
# Gemfile
gem "prometheus_exporter"

# config/initializers/prometheus.rb
if Rails.env.production?
  require "prometheus_exporter/middleware"

  Rails.application.middleware.unshift PrometheusExporter::Middleware
end

# Procfile
worker: bundle exec prometheus_exporter -b 0.0.0.0
```

### ServiceMonitor

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rails-app
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: rails-app
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

## Logging

### ELK Stack

```yaml
# Instalar con Helm
helm repo add elastic https://helm.elastic.co
helm install elasticsearch elastic/elasticsearch
helm install kibana elastic/kibana
helm install filebeat elastic/filebeat -f filebeat-values.yaml
```

```yaml
# filebeat-values.yaml
daemonset:
  filebeatConfig:
    filebeat.yml: |
      filebeat.inputs:
      - type: container
        paths:
          - /var/log/containers/*.log
        processors:
          - add_kubernetes_metadata:
              host: ${NODE_NAME}
              matchers:
              - logs_path:
                  logs_path: "/var/log/containers/"

      output.elasticsearch:
        hosts: ['${ELASTICSEARCH_HOST:elasticsearch-master}:${ELASTICSEARCH_PORT:9200}']
```

### Fluentd

```yaml
# fluent.conf
<source>
  @type tail
  path /var/log/containers/*rails*.log
  pos_file /var/log/fluentd-containers.log.pos
  tag kubernetes.*
  read_from_head true
  <parse>
    @type json
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
</source>

<match kubernetes.**>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix k8s-rails
</match>
```

## CI/CD con Kubernetes

### ArgoCD

```yaml
# application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: rails-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp.git
    targetRevision: HEAD
    path: k8s/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### GitHub Actions + K8s

```yaml
# .github/workflows/deploy.yml
name: Deploy to Kubernetes

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        run: |
          docker build -t myregistry.com/myapp:${{ github.sha }} .
          docker push myregistry.com/myapp:${{ github.sha }}

      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG }}

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/rails-app \
            rails=myregistry.com/myapp:${{ github.sha }} \
            -n production

      - name: Wait for rollout
        run: kubectl rollout status deployment/rails-app -n production
```

## Rails en Kubernetes

### Dockerfile optimizado

```dockerfile
# Dockerfile
FROM ruby:3.3.0-slim AS builder

WORKDIR /app

# Install dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libpq-dev \
    nodejs \
    npm \
    git \
    && rm -rf /var/lib/apt/lists/*

# Install bundler
RUN gem install bundler:2.5.0

# Copy Gemfile first for caching
COPY Gemfile Gemfile.lock ./
RUN bundle config set --local deployment 'true' && \
    bundle config set --local without 'development test' && \
    bundle install

# Copy app
COPY . .

# Precompile assets
RUN SECRET_KEY_BASE=dummy RAILS_ENV=production bundle exec rails assets:precompile

# Production image
FROM ruby:3.3.0-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    libpq-dev \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy from builder
COPY --from=builder /app /app
COPY --from=builder /usr/local/bundle /usr/local/bundle

# Create non-root user
RUN useradd -m rails && chown -R rails:rails /app
USER rails

EXPOSE 3000

CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

### Kubernetes manifests completos

```yaml
# k8s/production/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rails-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rails-app
  template:
    metadata:
      labels:
        app: rails-app
    spec:
      containers:
        - name: rails
          image: myregistry.com/myapp:latest
          ports:
            - containerPort: 3000
          env:
            - name: RAILS_ENV
              value: production
            - name: RAILS_LOG_TO_STDOUT
              value: "true"
            - name: RAILS_SERVE_STATIC_FILES
              value: "true"
            - name: RAILS_MASTER_KEY
              valueFrom:
                secretKeyRef:
                  name: rails-secrets
                  key: master-key
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: rails-secrets
                  key: database-url
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          livenessProbe:
            httpGet:
              path: /up
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /up
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
          volumeMounts:
            - name: uploads
              mountPath: /app/storage
      volumes:
        - name: uploads
          persistentVolumeClaim:
            claimName: rails-uploads
      imagePullSecrets:
        - name: registry-credentials

---
# k8s/production/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: rails-app
  namespace: production
spec:
  selector:
    app: rails-app
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP

---
# k8s/production/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rails-app
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.com
      secretName: myapp-tls
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rails-app
                port:
                  number: 80
```

### Job para migraciones

```yaml
# k8s/production/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: rails-migrate-{{ .Release.Revision }}
  namespace: production
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-1"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myregistry.com/myapp:{{ .Values.image.tag }}
          command: ["bundle", "exec", "rails", "db:migrate"]
          env:
            - name: RAILS_ENV
              value: production
            - name: RAILS_MASTER_KEY
              valueFrom:
                secretKeyRef:
                  name: rails-secrets
                  key: master-key
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: rails-secrets
                  key: database-url
      restartPolicy: Never
  backoffLimit: 3
```

### Worker deployment (Solid Queue)

```yaml
# k8s/production/worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rails-worker
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: rails-worker
  template:
    metadata:
      labels:
        app: rails-worker
    spec:
      containers:
        - name: worker
          image: myregistry.com/myapp:latest
          command: ["bundle", "exec", "rake", "solid_queue:start"]
          env:
            - name: RAILS_ENV
              value: production
            - name: RAILS_MASTER_KEY
              valueFrom:
                secretKeyRef:
                  name: rails-secrets
                  key: master-key
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: rails-secrets
                  key: database-url
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

## Checklist de Kubernetes

### Antes de deploy

- [ ] Dockerfile optimizado (multi-stage)
- [ ] Health checks configurados
- [ ] Resource limits definidos
- [ ] Secrets en Secret/ConfigMap
- [ ] Probes (liveness/readiness)
- [ ] Network policies si es necesario

### Deploy

- [ ] Aplicar ConfigMaps y Secrets primero
- [ ] Ejecutar migraciones (Job)
- [ ] Deploy con rolling update
- [ ] Verificar rollout status
- [ ] Verificar logs

### Post-deploy

- [ ] Verificar pods healthy
- [ ] Verificar ingress/service
- [ ] Verificar métricas
- [ ] Verificar logs centralizados
- [ ] Test de carga si es necesario
