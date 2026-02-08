# Skill: Google Cloud Platform (GCP) para Rails

## Purpose

Configurar, desplegar y gestionar aplicaciones Rails en Google Cloud Platform, aprovechando los servicios cloud para escalabilidad, rendimiento y seguridad.

## Cloud Run

### Desplegar Rails en Cloud Run

```dockerfile
# Dockerfile
FROM ruby:3.3.0-slim

# Instalar dependencias
RUN apt-get update -qq && \
    apt-get install -y build-essential libpq-dev nodejs npm && \
    npm install -g yarn && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copiar Gemfile primero para cache de layers
COPY Gemfile Gemfile.lock ./
RUN bundle config set --local deployment 'true' && \
    bundle config set --local without 'development test' && \
    bundle install --jobs 4

# Copiar aplicación
COPY . .

# Precompilar assets
ARG RAILS_MASTER_KEY
ENV RAILS_ENV=production
ENV SECRET_KEY_BASE=dummy
RUN bundle exec rails assets:precompile

# Puerto (Cloud Run usa $PORT)
EXPOSE 8080

# Comando
CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

```ruby
# config/puma.rb
port ENV.fetch("PORT") { 8080 }
environment ENV.fetch("RAILS_ENV") { "production" }

# Cloud Run: single process, multiple threads
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }.to_i
threads threads_count, threads_count

preload_app!
```

```bash
# Build y push a Container Registry
gcloud builds submit --tag gcr.io/mi-proyecto/rails-app

# Desplegar en Cloud Run
gcloud run deploy rails-app \
  --image gcr.io/mi-proyecto/rails-app \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars "RAILS_ENV=production" \
  --set-secrets "DATABASE_URL=database-url:latest,SECRET_KEY_BASE=secret-key-base:latest" \
  --memory 512Mi \
  --cpu 1 \
  --min-instances 0 \
  --max-instances 10 \
  --concurrency 80
```

### Cloud Run con Cloud SQL

```bash
# Desplegar con conexión a Cloud SQL
gcloud run deploy rails-app \
  --image gcr.io/mi-proyecto/rails-app \
  --add-cloudsql-instances mi-proyecto:us-central1:rails-db \
  --set-env-vars "RAILS_ENV=production" \
  --set-secrets "DATABASE_URL=database-url:latest"
```

```yaml
# config/database.yml para Cloud Run + Cloud SQL
production:
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  # Cloud Run usa socket Unix para Cloud SQL
  host: /cloudsql/<%= ENV["CLOUD_SQL_CONNECTION_NAME"] %>
  database: <%= ENV["DB_NAME"] %>
  username: <%= ENV["DB_USER"] %>
  password: <%= ENV["DB_PASSWORD"] %>
```

### Autoscaling Configuration

```yaml
# service.yaml para Cloud Run
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: rails-app
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"
        autoscaling.knative.dev/maxScale: "100"
        autoscaling.knative.dev/target: "80"
        run.googleapis.com/cpu-throttling: "false"
    spec:
      containerConcurrency: 80
      timeoutSeconds: 300
      containers:
        - image: gcr.io/mi-proyecto/rails-app
          ports:
            - containerPort: 8080
          resources:
            limits:
              cpu: "2"
              memory: 1Gi
```

## Compute Engine

### Crear VM para Rails

```bash
# Crear instancia
gcloud compute instances create rails-server \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --tags=http-server,https-server

# Crear regla de firewall
gcloud compute firewall-rules create allow-rails \
  --allow tcp:3000 \
  --target-tags=http-server

# Conectar por SSH
gcloud compute ssh rails-server --zone=us-central1-a
```

### Startup Script para Rails

```bash
#!/bin/bash
# startup-script.sh

# Instalar dependencias
apt-get update
apt-get install -y git curl build-essential libssl-dev zlib1g-dev \
  libpq-dev nodejs npm

# Instalar rbenv
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build

# Instalar Ruby
rbenv install 3.3.0
rbenv global 3.3.0

# Instalar bundler
gem install bundler

# Clonar y configurar app
git clone https://github.com/tu-usuario/rails-app.git /opt/rails-app
cd /opt/rails-app
bundle install --deployment --without development test
RAILS_ENV=production rails db:migrate
RAILS_ENV=production rails assets:precompile
```

### Managed Instance Groups

```bash
# Crear template de instancia
gcloud compute instance-templates create rails-template \
  --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --metadata-from-file startup-script=startup-script.sh \
  --tags=http-server,https-server

# Crear managed instance group
gcloud compute instance-groups managed create rails-mig \
  --base-instance-name=rails \
  --template=rails-template \
  --size=2 \
  --zone=us-central1-a

# Configurar autoscaling
gcloud compute instance-groups managed set-autoscaling rails-mig \
  --zone=us-central1-a \
  --max-num-replicas=10 \
  --min-num-replicas=2 \
  --target-cpu-utilization=0.7
```

## Cloud Storage

### Configurar Active Storage

```ruby
# Gemfile
gem "google-cloud-storage", require: false

# config/storage.yml
google:
  service: GCS
  project: <%= ENV["GCP_PROJECT_ID"] %>
  credentials: <%= Rails.root.join("config/gcs-credentials.json") %>
  bucket: <%= ENV.fetch("GCS_BUCKET") { "mi-app-#{Rails.env}" } %>

# Para producción con ADC (Application Default Credentials)
google_production:
  service: GCS
  project: <%= ENV["GCP_PROJECT_ID"] %>
  bucket: <%= ENV["GCS_BUCKET"] %>
  # Sin credentials - usa ADC

# config/environments/production.rb
config.active_storage.service = :google_production
```

### Signed URLs

```ruby
# app/services/gcs_signer.rb
class GcsSigner
  def initialize
    @storage = Google::Cloud::Storage.new(
      project_id: ENV["GCP_PROJECT_ID"]
    )
    @bucket = @storage.bucket(ENV["GCS_BUCKET"])
  end

  # URL para subir
  def signed_upload_url(filename, content_type:, expires_in: 15.minutes)
    @bucket.signed_url(
      filename,
      method: "PUT",
      content_type: content_type,
      expires: expires_in.to_i,
      version: :v4
    )
  end

  # URL para descargar
  def signed_download_url(filename, expires_in: 1.hour, disposition: nil)
    options = {
      method: "GET",
      expires: expires_in.to_i,
      version: :v4
    }

    if disposition
      options[:query] = {
        "response-content-disposition" => disposition
      }
    end

    @bucket.signed_url(filename, **options)
  end

  # URL para streaming de video
  def signed_streaming_url(filename, expires_in: 4.hours)
    @bucket.signed_url(
      filename,
      method: "GET",
      expires: expires_in.to_i,
      version: :v4,
      headers: { "Range" => "bytes=0-" }
    )
  end
end
```

### CORS Configuration

```json
[
  {
    "origin": ["https://miapp.com", "http://localhost:3000"],
    "method": ["GET", "PUT", "POST", "DELETE"],
    "responseHeader": ["Content-Type", "Content-Range"],
    "maxAgeSeconds": 3600
  }
]
```

```bash
# Aplicar configuración CORS
gsutil cors set cors.json gs://mi-bucket
```

## Cloud SQL

### Crear instancia PostgreSQL

```bash
# Crear instancia
gcloud sql instances create rails-db \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=us-central1 \
  --storage-type=SSD \
  --storage-size=10GB \
  --backup-start-time=03:00

# Crear base de datos
gcloud sql databases create rails_production \
  --instance=rails-db

# Crear usuario
gcloud sql users create railsuser \
  --instance=rails-db \
  --password=SecurePassword123
```

### Configurar Rails

```yaml
# config/database.yml
production:
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  database: <%= ENV["DB_NAME"] %>
  username: <%= ENV["DB_USER"] %>
  password: <%= ENV["DB_PASSWORD"] %>

  # Conexión directa (con Cloud SQL Proxy)
  host: 127.0.0.1
  port: 5432

  # O con socket (Cloud Run)
  # host: /cloudsql/proyecto:region:instancia
```

### Cloud SQL Proxy

```bash
# Descargar proxy
curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.8.0/cloud-sql-proxy.linux.amd64
chmod +x cloud-sql-proxy

# Iniciar proxy
./cloud-sql-proxy mi-proyecto:us-central1:rails-db &

# Para desarrollo local con credenciales
./cloud-sql-proxy --credentials-file=service-account.json mi-proyecto:us-central1:rails-db
```

### Backups y Restauración

```bash
# Crear backup on-demand
gcloud sql backups create \
  --instance=rails-db \
  --description="Pre-deploy backup"

# Listar backups
gcloud sql backups list --instance=rails-db

# Restaurar backup
gcloud sql backups restore BACKUP_ID \
  --restore-instance=rails-db \
  --backup-instance=rails-db

# Clonar instancia
gcloud sql instances clone rails-db rails-db-clone
```

### Read Replicas

```bash
# Crear réplica
gcloud sql instances create rails-db-replica \
  --master-instance-name=rails-db \
  --region=us-central1
```

```ruby
# config/database.yml con réplica
production:
  primary:
    adapter: postgresql
    database: rails_production
    host: /cloudsql/mi-proyecto:us-central1:rails-db

  primary_replica:
    adapter: postgresql
    database: rails_production
    host: /cloudsql/mi-proyecto:us-central1:rails-db-replica
    replica: true
```

## Cloud Functions

### HTTP Function para Webhooks

```ruby
# app.rb (para Cloud Functions con Ruby)
require "functions_framework"
require "json"

FunctionsFramework.http "webhook_handler" do |request|
  payload = JSON.parse(request.body.read)

  # Procesar webhook
  case payload["event"]
  when "payment.completed"
    # Publicar a Pub/Sub para procesamiento async
    pubsub = Google::Cloud::Pubsub.new
    topic = pubsub.topic("payment-events")
    topic.publish(payload.to_json)
  end

  { status: "received" }.to_json
rescue JSON::ParserError
  [400, {}, ["Invalid JSON"]]
end
```

```bash
# Desplegar función
gcloud functions deploy webhook-handler \
  --runtime ruby32 \
  --trigger-http \
  --allow-unauthenticated \
  --entry-point webhook_handler
```

### Pub/Sub Trigger

```ruby
# background_function.rb
require "functions_framework"
require "json"
require "base64"

FunctionsFramework.cloud_event "process_payment" do |event|
  data = JSON.parse(Base64.decode64(event.data["message"]["data"]))

  # Procesar el pago
  PaymentProcessor.new(data).process

  puts "Processed payment: #{data['id']}"
end
```

## GKE (Google Kubernetes Engine)

### Crear Cluster

```bash
# Crear cluster autopilot
gcloud container clusters create-auto rails-cluster \
  --region=us-central1

# O cluster standard
gcloud container clusters create rails-cluster \
  --zone=us-central1-a \
  --num-nodes=3 \
  --machine-type=e2-medium

# Obtener credenciales
gcloud container clusters get-credentials rails-cluster \
  --zone=us-central1-a
```

### Deployment

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rails-app
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
      serviceAccountName: rails-app-sa
      containers:
        - name: rails-app
          image: gcr.io/mi-proyecto/rails-app:latest
          ports:
            - containerPort: 8080
          env:
            - name: RAILS_ENV
              value: production
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: rails-secrets
                  key: database-url
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /up
              port: 8080
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /up
              port: 8080
            initialDelaySeconds: 5
        - name: cloud-sql-proxy
          image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.8.0
          args:
            - "--structured-logs"
            - "mi-proyecto:us-central1:rails-db"
          securityContext:
            runAsNonRoot: true
---
apiVersion: v1
kind: Service
metadata:
  name: rails-app-service
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: rails-app
```

## Cloud CDN

### Configurar con Load Balancer

```bash
# Crear backend bucket para assets
gsutil mb gs://rails-assets-cdn

# Hacer público
gsutil iam ch allUsers:objectViewer gs://rails-assets-cdn

# Crear backend bucket
gcloud compute backend-buckets create rails-assets-backend \
  --gcs-bucket-name=rails-assets-cdn \
  --enable-cdn

# Crear URL map
gcloud compute url-maps create rails-lb \
  --default-service=rails-backend-service

# Agregar path para assets
gcloud compute url-maps add-path-matcher rails-lb \
  --path-matcher-name=assets \
  --default-service=rails-backend-service \
  --backend-bucket-path-rules="/assets/*=rails-assets-backend"
```

```ruby
# config/environments/production.rb
config.asset_host = ENV["CDN_HOST"]
# Ejemplo: "https://storage.googleapis.com/rails-assets-cdn"
```

### Cloud Armor (WAF)

```bash
# Crear política de seguridad
gcloud compute security-policies create rails-security-policy

# Regla para bloquear países
gcloud compute security-policies rules create 1000 \
  --security-policy=rails-security-policy \
  --expression="origin.region_code == 'XX'" \
  --action=deny-403

# Regla para rate limiting
gcloud compute security-policies rules create 2000 \
  --security-policy=rails-security-policy \
  --expression="true" \
  --action=rate-based-ban \
  --rate-limit-threshold-count=1000 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=600
```

## Cloud DNS

### Configurar Zona

```bash
# Crear zona
gcloud dns managed-zones create mi-zona \
  --dns-name=midominio.com. \
  --description="Zona DNS para mi app"

# Agregar registro A
gcloud dns record-sets create app.midominio.com. \
  --zone=mi-zona \
  --type=A \
  --ttl=300 \
  --rrdatas=34.xx.xx.xx

# Agregar CNAME
gcloud dns record-sets create www.midominio.com. \
  --zone=mi-zona \
  --type=CNAME \
  --ttl=300 \
  --rrdatas=app.midominio.com.
```

## IAM

### Service Account para Rails

```bash
# Crear service account
gcloud iam service-accounts create rails-app-sa \
  --display-name="Rails Application"

# Asignar roles mínimos
gcloud projects add-iam-policy-binding mi-proyecto \
  --member="serviceAccount:rails-app-sa@mi-proyecto.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"

gcloud projects add-iam-policy-binding mi-proyecto \
  --member="serviceAccount:rails-app-sa@mi-proyecto.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"

gcloud projects add-iam-policy-binding mi-proyecto \
  --member="serviceAccount:rails-app-sa@mi-proyecto.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Generar key (para desarrollo local)
gcloud iam service-accounts keys create key.json \
  --iam-account=rails-app-sa@mi-proyecto.iam.gserviceaccount.com
```

## Memorystore (Redis)

### Configurar

```bash
# Crear instancia Redis
gcloud redis instances create rails-redis \
  --size=1 \
  --region=us-central1 \
  --redis-version=redis_7_0

# Obtener host
gcloud redis instances describe rails-redis \
  --region=us-central1 \
  --format="value(host)"
```

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url: "redis://#{ENV['REDIS_HOST']}:6379/0"
}

config.session_store :redis_store,
  servers: ["redis://#{ENV['REDIS_HOST']}:6379/1"],
  expire_after: 1.day
```

## Secret Manager

### Usar Secrets

```bash
# Crear secret
echo -n "mi-valor-secreto" | gcloud secrets create my-secret \
  --data-file=-

# Agregar versión
echo -n "nuevo-valor" | gcloud secrets versions add my-secret \
  --data-file=-

# Acceder a secret
gcloud secrets versions access latest --secret=my-secret
```

```ruby
# config/initializers/gcp_secrets.rb
if Rails.env.production?
  require "google/cloud/secret_manager"

  client = Google::Cloud::SecretManager.secret_manager_service

  secrets = %w[DATABASE_URL SECRET_KEY_BASE REDIS_HOST]

  secrets.each do |secret_name|
    begin
      name = "projects/#{ENV['GCP_PROJECT_ID']}/secrets/#{secret_name.downcase.tr('_', '-')}/versions/latest"
      version = client.access_secret_version(name: name)
      ENV[secret_name] = version.payload.data unless ENV[secret_name].present?
    rescue Google::Cloud::NotFoundError
      Rails.logger.warn "Secret #{secret_name} not found"
    end
  end
end
```

## Cloud Logging / Monitoring

### Configurar Logging

```ruby
# config/environments/production.rb
if ENV["K_SERVICE"].present? # Cloud Run
  require "google/cloud/logging"

  logging = Google::Cloud::Logging.new
  resource = Google::Cloud::Logging::Middleware.build_monitored_resource

  config.logger = ActiveSupport::Logger.new(
    Google::Cloud::Logging::Logger.new(
      logging,
      "rails-app",
      resource
    )
  )
end
```

### Structured Logging

```ruby
# app/lib/json_logger.rb
class JsonLogger < ActiveSupport::Logger
  def format_message(severity, timestamp, progname, msg)
    {
      severity: severity,
      timestamp: timestamp.iso8601,
      message: msg,
      progname: progname
    }.to_json + "\n"
  end
end

# config/environments/production.rb
config.logger = JsonLogger.new(STDOUT)
```

### Custom Metrics

```ruby
# app/services/cloud_monitoring.rb
class CloudMonitoring
  def initialize
    @client = Google::Cloud::Monitoring.metric_service
    @project_name = "projects/#{ENV['GCP_PROJECT_ID']}"
  end

  def record_metric(metric_type, value, labels = {})
    series = Google::Cloud::Monitoring::V3::TimeSeries.new(
      metric: {
        type: "custom.googleapis.com/rails/#{metric_type}",
        labels: labels
      },
      resource: {
        type: "global",
        labels: { project_id: ENV["GCP_PROJECT_ID"] }
      },
      points: [{
        interval: { end_time: Google::Protobuf::Timestamp.new(seconds: Time.current.to_i) },
        value: { double_value: value }
      }]
    )

    @client.create_time_series(name: @project_name, time_series: [series])
  end
end
```

## Costos y Optimización

### Calculadora

```markdown
## Estimación mensual para app Rails pequeña-mediana

| Servicio | Configuración | Costo/mes |
|----------|--------------|-----------|
| Cloud Run | 1M requests, 512Mi | ~$20 |
| Cloud SQL | db-f1-micro | ~$10 |
| Memorystore | 1GB Redis | ~$35 |
| Cloud Storage | 50GB | ~$1 |
| Cloud CDN | 100GB | ~$8 |
| Secret Manager | 10 secrets | ~$0.06 |
| **Total** | | **~$75/mes** |

## Optimizaciones:
- Cloud Run: min-instances=0 para desarrollo
- Committed Use Discounts (1-3 años)
- Preemptible VMs para workers
- Cloud Storage Nearline para backups
```

### Committed Use Discounts

```bash
# Ver recomendaciones
gcloud recommender recommendations list \
  --project=mi-proyecto \
  --location=us-central1 \
  --recommender=google.compute.commitment.UsageCommitmentRecommender
```

## Checklist de Deployment

- [ ] Proyecto GCP creado
- [ ] APIs habilitadas (Cloud Run, SQL, Storage, etc.)
- [ ] Service Account con roles mínimos
- [ ] Cloud SQL instancia creada
- [ ] Cloud Storage bucket configurado
- [ ] Secret Manager con secrets
- [ ] Container Registry con imagen
- [ ] Cloud Run service desplegado
- [ ] Cloud SQL Proxy o conexión configurada
- [ ] Load Balancer + CDN (opcional)
- [ ] Cloud DNS configurado
- [ ] SSL/TLS con certificado managed
- [ ] Cloud Logging habilitado
- [ ] Alertas en Cloud Monitoring
- [ ] Backups automáticos verificados
