# Skill: Microsoft Azure para Rails

## Purpose

Configurar, desplegar y gestionar aplicaciones Rails en Microsoft Azure, aprovechando los servicios cloud para escalabilidad, rendimiento y seguridad.

## App Service

### Configurar Web App para Rails

```bash
# Crear grupo de recursos
az group create --name rails-app-rg --location eastus

# Crear App Service Plan
az appservice plan create \
  --name rails-app-plan \
  --resource-group rails-app-rg \
  --sku B1 \
  --is-linux

# Crear Web App con Ruby
az webapp create \
  --resource-group rails-app-rg \
  --plan rails-app-plan \
  --name mi-rails-app \
  --runtime "RUBY:3.1"

# Configurar variables de entorno
az webapp config appsettings set \
  --resource-group rails-app-rg \
  --name mi-rails-app \
  --settings \
    RAILS_ENV=production \
    RAILS_SERVE_STATIC_FILES=true \
    RAILS_LOG_TO_STDOUT=true
```

### Deployment desde GitHub

```yaml
# .github/workflows/azure-deploy.yml
name: Deploy to Azure

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true

      - name: Precompile assets
        run: |
          bundle exec rails assets:precompile
        env:
          RAILS_ENV: production
          SECRET_KEY_BASE: ${{ secrets.SECRET_KEY_BASE }}

      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v2
        with:
          app-name: mi-rails-app
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
```

### Deployment Slots

```bash
# Crear slot de staging
az webapp deployment slot create \
  --name mi-rails-app \
  --resource-group rails-app-rg \
  --slot staging

# Configurar settings del slot
az webapp config appsettings set \
  --resource-group rails-app-rg \
  --name mi-rails-app \
  --slot staging \
  --settings RAILS_ENV=staging

# Swap slots (zero-downtime deployment)
az webapp deployment slot swap \
  --resource-group rails-app-rg \
  --name mi-rails-app \
  --slot staging \
  --target-slot production
```

### Auto-scaling

```bash
# Configurar autoscaling
az monitor autoscale create \
  --resource-group rails-app-rg \
  --resource mi-rails-app \
  --resource-type Microsoft.Web/serverfarms \
  --name rails-autoscale \
  --min-count 1 \
  --max-count 5 \
  --count 2

# Regla: escalar si CPU > 70%
az monitor autoscale rule create \
  --resource-group rails-app-rg \
  --autoscale-name rails-autoscale \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 1
```

## Blob Storage

### Configurar Active Storage

```ruby
# Gemfile
gem "azure-storage-blob", require: false

# config/storage.yml
azure:
  service: AzureStorage
  storage_account_name: <%= ENV["AZURE_STORAGE_ACCOUNT"] %>
  storage_access_key: <%= ENV["AZURE_STORAGE_ACCESS_KEY"] %>
  container: <%= ENV.fetch("AZURE_STORAGE_CONTAINER") { "uploads-#{Rails.env}" } %>

# Para uploads públicos
azure_public:
  service: AzureStorage
  storage_account_name: <%= ENV["AZURE_STORAGE_ACCOUNT"] %>
  storage_access_key: <%= ENV["AZURE_STORAGE_ACCESS_KEY"] %>
  container: public-uploads
  public: true

# config/environments/production.rb
config.active_storage.service = :azure
```

### SAS Tokens (Shared Access Signatures)

```ruby
# app/services/azure_blob_service.rb
class AzureBlobService
  def initialize
    @account_name = ENV["AZURE_STORAGE_ACCOUNT"]
    @account_key = ENV["AZURE_STORAGE_ACCESS_KEY"]
    @container = ENV["AZURE_STORAGE_CONTAINER"]

    @client = Azure::Storage::Blob::BlobService.create(
      storage_account_name: @account_name,
      storage_access_key: @account_key
    )
  end

  # Generar URL con SAS token para subir
  def generate_upload_url(blob_name, content_type:, expires_in: 15.minutes)
    sas_token = generate_sas_token(
      blob_name,
      permissions: "w",
      expires_in: expires_in
    )

    "https://#{@account_name}.blob.core.windows.net/#{@container}/#{blob_name}?#{sas_token}"
  end

  # Generar URL con SAS token para descargar
  def generate_download_url(blob_name, expires_in: 1.hour, filename: nil)
    content_disposition = filename ? "attachment; filename=\"#{filename}\"" : nil

    sas_token = generate_sas_token(
      blob_name,
      permissions: "r",
      expires_in: expires_in,
      content_disposition: content_disposition
    )

    "https://#{@account_name}.blob.core.windows.net/#{@container}/#{blob_name}?#{sas_token}"
  end

  private

  def generate_sas_token(blob_name, permissions:, expires_in:, content_disposition: nil)
    generator = Azure::Storage::Common::Core::Auth::SharedAccessSignature.new(
      @account_name,
      @account_key
    )

    options = {
      permissions: permissions,
      start: (Time.current - 5.minutes).utc.iso8601,
      expiry: (Time.current + expires_in).utc.iso8601,
      resource: "b"
    }

    options[:content_disposition] = content_disposition if content_disposition

    generator.generate_service_sas_token(
      "#{@container}/#{blob_name}",
      options
    )
  end
end
```

### CORS Configuration

```bash
# Configurar CORS para el storage account
az storage cors add \
  --account-name mistorageaccount \
  --services b \
  --methods GET PUT POST DELETE \
  --origins "https://miapp.com" "http://localhost:3000" \
  --allowed-headers "*" \
  --exposed-headers "*" \
  --max-age 3600
```

## Azure SQL / PostgreSQL

### Configurar Rails con Azure Database for PostgreSQL

```yaml
# config/database.yml
production:
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  database: <%= ENV["AZURE_PG_DATABASE"] %>
  username: <%= ENV["AZURE_PG_USERNAME"] %>
  password: <%= ENV["AZURE_PG_PASSWORD"] %>
  host: <%= ENV["AZURE_PG_HOST"] %>
  port: 5432
  sslmode: require
```

```bash
# Crear servidor PostgreSQL
az postgres flexible-server create \
  --resource-group rails-app-rg \
  --name rails-pg-server \
  --location eastus \
  --admin-user railsadmin \
  --admin-password 'SecurePassword123!' \
  --sku-name Standard_B1ms \
  --version 15 \
  --storage-size 32

# Crear base de datos
az postgres flexible-server db create \
  --resource-group rails-app-rg \
  --server-name rails-pg-server \
  --database-name rails_production

# Configurar firewall (permitir Azure services)
az postgres flexible-server firewall-rule create \
  --resource-group rails-app-rg \
  --name rails-pg-server \
  --rule-name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

### Backups y Restauración

```bash
# Los backups son automáticos en Azure PostgreSQL
# Configurar retención (7-35 días)
az postgres flexible-server update \
  --resource-group rails-app-rg \
  --name rails-pg-server \
  --backup-retention 14

# Restaurar a punto en el tiempo
az postgres flexible-server restore \
  --resource-group rails-app-rg \
  --name rails-pg-server-restored \
  --source-server rails-pg-server \
  --restore-time "2024-01-15T10:00:00Z"
```

## Azure Functions

### HTTP Trigger para Webhooks

```ruby
# function.json
{
  "bindings": [
    {
      "authLevel": "function",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["post"]
    },
    {
      "type": "http",
      "direction": "out",
      "name": "$return"
    },
    {
      "type": "queue",
      "direction": "out",
      "name": "webhookQueue",
      "queueName": "webhooks",
      "connection": "AzureWebJobsStorage"
    }
  ]
}

# handler.rb (con Ruby custom handler)
require "json"
require "sinatra"

post "/api/webhook" do
  content_type :json
  payload = JSON.parse(request.body.read)

  # Encolar para procesamiento
  {
    statusCode: 200,
    body: { received: true }.to_json,
    webhookQueue: payload.to_json
  }.to_json
end
```

### Timer Trigger para Tareas Programadas

```json
{
  "bindings": [
    {
      "name": "timer",
      "type": "timerTrigger",
      "direction": "in",
      "schedule": "0 0 * * * *"
    }
  ]
}
```

## Container Instances / AKS

### Azure Container Instances (Simple)

```bash
# Crear container instance
az container create \
  --resource-group rails-app-rg \
  --name rails-app \
  --image miregistry.azurecr.io/rails-app:latest \
  --cpu 1 \
  --memory 1.5 \
  --ports 3000 \
  --environment-variables \
    RAILS_ENV=production \
    RAILS_LOG_TO_STDOUT=true \
  --secure-environment-variables \
    DATABASE_URL='postgresql://...' \
    SECRET_KEY_BASE='...'
```

### Azure Kubernetes Service (AKS)

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
      containers:
        - name: rails-app
          image: miregistry.azurecr.io/rails-app:latest
          ports:
            - containerPort: 3000
          env:
            - name: RAILS_ENV
              value: production
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: rails-secrets
                  key: database-url
            - name: SECRET_KEY_BASE
              valueFrom:
                secretKeyRef:
                  name: rails-secrets
                  key: secret-key-base
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
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
---
apiVersion: v1
kind: Service
metadata:
  name: rails-app-service
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 3000
  selector:
    app: rails-app
```

```bash
# Crear cluster AKS
az aks create \
  --resource-group rails-app-rg \
  --name rails-aks \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --generate-ssh-keys

# Obtener credenciales
az aks get-credentials \
  --resource-group rails-app-rg \
  --name rails-aks

# Desplegar
kubectl apply -f kubernetes/
```

## Azure CDN

### Configurar CDN para Assets

```bash
# Crear perfil CDN
az cdn profile create \
  --resource-group rails-app-rg \
  --name rails-cdn-profile \
  --sku Standard_Microsoft

# Crear endpoint
az cdn endpoint create \
  --resource-group rails-app-rg \
  --profile-name rails-cdn-profile \
  --name rails-cdn \
  --origin mi-rails-app.azurewebsites.net \
  --origin-host-header mi-rails-app.azurewebsites.net
```

```ruby
# config/environments/production.rb
config.asset_host = ENV["AZURE_CDN_ENDPOINT"]
# Ejemplo: "https://rails-cdn.azureedge.net"
```

### Purge Cache

```bash
# Purgar todo el contenido
az cdn endpoint purge \
  --resource-group rails-app-rg \
  --profile-name rails-cdn-profile \
  --name rails-cdn \
  --content-paths "/*"

# Purgar assets específicos
az cdn endpoint purge \
  --resource-group rails-app-rg \
  --profile-name rails-cdn-profile \
  --name rails-cdn \
  --content-paths "/assets/*"
```

## DNS Zones

### Configurar DNS

```bash
# Crear zona DNS
az network dns zone create \
  --resource-group rails-app-rg \
  --name midominio.com

# Agregar registro A
az network dns record-set a add-record \
  --resource-group rails-app-rg \
  --zone-name midominio.com \
  --record-set-name app \
  --ipv4-address 20.xx.xx.xx

# Agregar CNAME para CDN
az network dns record-set cname set-record \
  --resource-group rails-app-rg \
  --zone-name midominio.com \
  --record-set-name cdn \
  --cname rails-cdn.azureedge.net
```

### Traffic Manager (Load Balancing Global)

```bash
# Crear perfil Traffic Manager
az network traffic-manager profile create \
  --resource-group rails-app-rg \
  --name rails-traffic-manager \
  --routing-method Performance \
  --unique-dns-name rails-app-tm

# Agregar endpoints
az network traffic-manager endpoint create \
  --resource-group rails-app-rg \
  --profile-name rails-traffic-manager \
  --name eastus-endpoint \
  --type azureEndpoints \
  --target-resource-id /subscriptions/.../sites/rails-app-eastus \
  --endpoint-status Enabled

az network traffic-manager endpoint create \
  --resource-group rails-app-rg \
  --profile-name rails-traffic-manager \
  --name westus-endpoint \
  --type azureEndpoints \
  --target-resource-id /subscriptions/.../sites/rails-app-westus \
  --endpoint-status Enabled
```

## Azure AD

### Configurar OAuth con Azure AD

```ruby
# Gemfile
gem "omniauth-azure-activedirectory-v2"

# config/initializers/omniauth.rb
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :azure_activedirectory_v2,
    client_id: ENV["AZURE_AD_CLIENT_ID"],
    client_secret: ENV["AZURE_AD_CLIENT_SECRET"],
    tenant_id: ENV["AZURE_AD_TENANT_ID"]
end

# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  def azure_callback
    auth = request.env["omniauth.auth"]

    user = User.find_or_create_by(azure_id: auth.uid) do |u|
      u.email = auth.info.email
      u.name = auth.info.name
    end

    session[:user_id] = user.id
    redirect_to root_path, notice: "Sesion iniciada"
  end

  def failure
    redirect_to root_path, alert: "Error de autenticacion"
  end
end

# config/routes.rb
get "/auth/azure_activedirectory_v2/callback", to: "sessions#azure_callback"
get "/auth/failure", to: "sessions#failure"
```

### Managed Identity

```ruby
# Para acceder a recursos Azure sin credenciales hardcodeadas
# app/services/azure_managed_identity.rb
class AzureManagedIdentity
  def initialize
    @token_endpoint = "http://169.254.169.254/metadata/identity/oauth2/token"
  end

  def get_token(resource:)
    response = HTTParty.get(
      @token_endpoint,
      query: {
        "api-version" => "2019-08-01",
        resource: resource
      },
      headers: { "Metadata" => "true" }
    )

    JSON.parse(response.body)["access_token"]
  end

  def get_secret(vault_name:, secret_name:)
    token = get_token(resource: "https://vault.azure.net")

    response = HTTParty.get(
      "https://#{vault_name}.vault.azure.net/secrets/#{secret_name}?api-version=7.4",
      headers: { "Authorization" => "Bearer #{token}" }
    )

    JSON.parse(response.body)["value"]
  end
end
```

## Redis Cache

### Configurar con Rails

```ruby
# Gemfile
gem "redis"
gem "hiredis"

# config/initializers/redis.rb
REDIS = Redis.new(
  url: ENV.fetch("AZURE_REDIS_URL") { "redis://localhost:6379/0" },
  ssl: Rails.env.production?,
  ssl_params: { verify_mode: OpenSSL::SSL::VERIFY_NONE }
)

# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url: ENV["AZURE_REDIS_URL"],
  ssl_params: { verify_mode: OpenSSL::SSL::VERIFY_NONE }
}

# Session store
config.session_store :redis_store,
  servers: [{
    url: ENV["AZURE_REDIS_URL"],
    ssl: true,
    ssl_params: { verify_mode: OpenSSL::SSL::VERIFY_NONE }
  }],
  expire_after: 1.day
```

```bash
# Crear Azure Cache for Redis
az redis create \
  --resource-group rails-app-rg \
  --name rails-redis-cache \
  --location eastus \
  --sku Basic \
  --vm-size c0

# Obtener connection string
az redis list-keys \
  --resource-group rails-app-rg \
  --name rails-redis-cache
```

## SendGrid / Communication Services

### Configurar Email con SendGrid (via Azure)

```ruby
# Gemfile
gem "sendgrid-ruby"

# config/environments/production.rb
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings = {
  address: "smtp.sendgrid.net",
  port: 587,
  domain: "midominio.com",
  user_name: "apikey",
  password: ENV["SENDGRID_API_KEY"],
  authentication: :plain,
  enable_starttls_auto: true
}
```

### Azure Communication Services (Email)

```ruby
# app/services/azure_email_service.rb
class AzureEmailService
  def initialize
    @endpoint = ENV["AZURE_COMMUNICATION_ENDPOINT"]
    @access_key = ENV["AZURE_COMMUNICATION_KEY"]
  end

  def send_email(to:, subject:, body:, from: "noreply@midominio.com")
    uri = URI("#{@endpoint}/emails:send?api-version=2023-03-31")

    request = Net::HTTP::Post.new(uri)
    request["Content-Type"] = "application/json"
    request["Authorization"] = generate_auth_header("POST", uri.path)

    request.body = {
      senderAddress: from,
      recipients: {
        to: [{ address: to }]
      },
      content: {
        subject: subject,
        plainText: body
      }
    }.to_json

    http = Net::HTTP.new(uri.host, uri.port)
    http.use_ssl = true
    http.request(request)
  end

  private

  def generate_auth_header(method, path)
    # Implementar HMAC-SHA256 auth
    # ...
  end
end
```

## Key Vault

### Cargar Secrets al Iniciar

```ruby
# config/initializers/azure_key_vault.rb
if Rails.env.production?
  require "azure/key_vault"

  client = Azure::KeyVault::KeyVaultClient.new

  # Con Managed Identity
  vault_url = "https://rails-app-vault.vault.azure.net"

  secrets = %w[DATABASE_URL SECRET_KEY_BASE REDIS_URL]

  secrets.each do |secret_name|
    begin
      secret = client.get_secret(vault_url, secret_name.downcase.tr("_", "-"), "")
      ENV[secret_name] = secret.value unless ENV[secret_name].present?
    rescue StandardError => e
      Rails.logger.warn "Could not load secret #{secret_name}: #{e.message}"
    end
  end
end
```

```bash
# Crear Key Vault
az keyvault create \
  --resource-group rails-app-rg \
  --name rails-app-vault \
  --location eastus

# Agregar secreto
az keyvault secret set \
  --vault-name rails-app-vault \
  --name "database-url" \
  --value "postgresql://..."

# Dar acceso a la App Service
az keyvault set-policy \
  --name rails-app-vault \
  --object-id $(az webapp identity show --name mi-rails-app --resource-group rails-app-rg --query principalId -o tsv) \
  --secret-permissions get list
```

## Application Insights

### Configurar Monitoring

```ruby
# Gemfile
gem "application_insights"

# config/initializers/application_insights.rb
if Rails.env.production?
  require "application_insights"

  ApplicationInsights::TelemetryClient.new(ENV["AZURE_APP_INSIGHTS_KEY"])

  # Middleware para tracking automático
  Rails.application.config.middleware.use(
    ApplicationInsights::Rack::TrackRequest,
    ENV["AZURE_APP_INSIGHTS_KEY"]
  )
end

# app/services/azure_insights.rb
class AzureInsights
  def initialize
    @client = ApplicationInsights::TelemetryClient.new(
      ENV["AZURE_APP_INSIGHTS_KEY"]
    )
  end

  def track_event(name, properties = {}, measurements = {})
    @client.track_event(name, properties, measurements)
    @client.flush
  end

  def track_exception(exception, properties = {})
    @client.track_exception(exception, properties)
    @client.flush
  end

  def track_metric(name, value, properties = {})
    @client.track_metric(name, value, properties: properties)
    @client.flush
  end

  def track_request(name, url, success, start_time, duration)
    @client.track_request(name, url, start_time, duration, "200", success)
    @client.flush
  end
end
```

### Custom Logging

```ruby
# app/middleware/azure_logging.rb
class AzureLogging
  def initialize(app)
    @app = app
    @insights = AzureInsights.new
  end

  def call(env)
    start_time = Time.current

    begin
      status, headers, response = @app.call(env)

      duration = ((Time.current - start_time) * 1000).round
      @insights.track_request(
        env["PATH_INFO"],
        env["REQUEST_URI"] || env["PATH_INFO"],
        status < 400,
        start_time,
        duration
      )

      [status, headers, response]
    rescue StandardError => e
      @insights.track_exception(e)
      raise
    end
  end
end
```

## Costos y Optimización

### Calculadora de Costos

```markdown
## Estimación mensual para app Rails pequeña-mediana

| Servicio | Configuración | Costo/mes |
|----------|--------------|-----------|
| App Service | B1 (1 core, 1.75GB) | ~$55 |
| PostgreSQL | B1ms (1 vCore, 2GB) | ~$25 |
| Redis Cache | C0 (250MB) | ~$16 |
| Blob Storage | 50GB + transacciones | ~$5 |
| CDN | 100GB transferencia | ~$8 |
| DNS Zone | 1 zona | ~$0.50 |
| **Total** | | **~$110/mes** |

## Optimizaciones:
- Reserved Instances (1-3 años): hasta 72% ahorro
- Azure Hybrid Benefit: si tienes licencias Windows
- Dev/Test pricing: para ambientes no-producción
- Autoscaling: reducir en horas bajas
```

### Reserved Instances

```bash
# Ver recomendaciones de reservas
az consumption reservation recommendation list \
  --scope "subscriptions/xxx-xxx-xxx"

# Comprar reserva
az reservations reservation-order purchase \
  --sku Standard_B2s \
  --term P1Y \
  --billing-plan Monthly \
  --quantity 2
```

## Checklist de Deployment

- [ ] Resource Group creado
- [ ] App Service Plan configurado
- [ ] Web App con Ruby runtime
- [ ] PostgreSQL/MySQL provisionado
- [ ] Blob Storage para archivos
- [ ] Redis Cache para sessions/cache
- [ ] Key Vault para secrets
- [ ] Managed Identity habilitado
- [ ] Application Insights configurado
- [ ] CDN para assets estáticos
- [ ] DNS configurado
- [ ] SSL/TLS con App Service Managed Certificate
- [ ] Deployment slots para zero-downtime
- [ ] Backups automáticos verificados
- [ ] Alertas configuradas
