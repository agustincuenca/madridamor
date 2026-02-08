# Skill: Amazon Web Services (AWS) para Rails

## Purpose

Configurar, desplegar y gestionar aplicaciones Rails en Amazon Web Services, aprovechando los servicios cloud para escalabilidad, rendimiento y seguridad.

## EC2 (Elastic Compute Cloud)

### Configurar instancia para Rails

```bash
# Conectar a la instancia
ssh -i "mi-key.pem" ec2-user@ec2-xx-xx-xx-xx.compute-1.amazonaws.com

# Instalar dependencias (Amazon Linux 2023)
sudo dnf update -y
sudo dnf install -y git gcc make openssl-devel readline-devel zlib-devel

# Instalar rbenv y Ruby
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
source ~/.bashrc

git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
rbenv install 3.3.0
rbenv global 3.3.0

# Instalar Rails
gem install bundler rails
```

### Security Groups

```ruby
# Terraform o CloudFormation para Security Groups
# security_group.tf

resource "aws_security_group" "rails_app" {
  name        = "rails-app-sg"
  description = "Security group for Rails application"
  vpc_id      = aws_vpc.main.id

  # SSH
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["TU_IP/32"]  # Solo tu IP
  }

  # HTTP
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTPS
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Rails en desarrollo
  ingress {
    from_port   = 3000
    to_port     = 3000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### AMI personalizada para Rails

```bash
# Después de configurar la instancia, crear AMI
aws ec2 create-image \
  --instance-id i-1234567890abcdef0 \
  --name "Rails-App-AMI-$(date +%Y%m%d)" \
  --description "Rails app with Ruby 3.3, dependencies"
```

## S3 (Simple Storage Service)

### Configurar Active Storage con S3

```ruby
# Gemfile
gem "aws-sdk-s3", require: false

# config/storage.yml
amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: us-east-1
  bucket: <%= ENV.fetch("S3_BUCKET") { "mi-app-#{Rails.env}" } %>

# Para uploads públicos
amazon_public:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: us-east-1
  bucket: mi-app-public
  public: true

# config/environments/production.rb
config.active_storage.service = :amazon
```

### Bucket Policy para Active Storage

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowRailsApp",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/rails-app-user"
      },
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::mi-app-production",
        "arn:aws:s3:::mi-app-production/*"
      ]
    }
  ]
}
```

### Presigned URLs

```ruby
# app/services/s3_presigner.rb
class S3Presigner
  def initialize
    @client = Aws::S3::Client.new(
      region: ENV["AWS_REGION"],
      credentials: Aws::Credentials.new(
        Rails.application.credentials.dig(:aws, :access_key_id),
        Rails.application.credentials.dig(:aws, :secret_access_key)
      )
    )
    @signer = Aws::S3::Presigner.new(client: @client)
  end

  # URL para subir archivo
  def presigned_put(key, content_type:, expires_in: 900)
    @signer.presigned_url(:put_object,
      bucket: ENV["S3_BUCKET"],
      key: key,
      content_type: content_type,
      expires_in: expires_in
    )
  end

  # URL para descargar archivo (privado)
  def presigned_get(key, expires_in: 3600, filename: nil)
    options = {
      bucket: ENV["S3_BUCKET"],
      key: key,
      expires_in: expires_in
    }

    if filename
      options[:response_content_disposition] = "attachment; filename=\"#{filename}\""
    end

    @signer.presigned_url(:get_object, options)
  end
end

# Uso en controller
class UploadsController < ApplicationController
  def presigned_url
    presigner = S3Presigner.new
    key = "uploads/#{SecureRandom.uuid}/#{params[:filename]}"

    render json: {
      url: presigner.presigned_put(key, content_type: params[:content_type]),
      key: key
    }
  end
end
```

### Configurar CORS en S3

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST", "DELETE"],
    "AllowedOrigins": ["https://miapp.com", "http://localhost:3000"],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3000
  }
]
```

## RDS (Relational Database Service)

### Configurar Rails con RDS PostgreSQL

```yaml
# config/database.yml
production:
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  database: <%= ENV["RDS_DB_NAME"] %>
  username: <%= ENV["RDS_USERNAME"] %>
  password: <%= ENV["RDS_PASSWORD"] %>
  host: <%= ENV["RDS_HOSTNAME"] %>
  port: <%= ENV["RDS_PORT"] || 5432 %>

  # Configuraciones de conexión
  connect_timeout: 5
  checkout_timeout: 5
  reaping_frequency: 10
  dead_connection_timeout: 5

  # SSL en producción
  sslmode: require
```

### Read Replicas

```ruby
# config/database.yml con read replica
production:
  primary:
    adapter: postgresql
    database: <%= ENV["RDS_DB_NAME"] %>
    username: <%= ENV["RDS_USERNAME"] %>
    password: <%= ENV["RDS_PASSWORD"] %>
    host: <%= ENV["RDS_PRIMARY_HOST"] %>

  primary_replica:
    adapter: postgresql
    database: <%= ENV["RDS_DB_NAME"] %>
    username: <%= ENV["RDS_USERNAME"] %>
    password: <%= ENV["RDS_PASSWORD"] %>
    host: <%= ENV["RDS_REPLICA_HOST"] %>
    replica: true

# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  primary_abstract_class

  # Usar replica para lecturas pesadas
  connects_to database: { writing: :primary, reading: :primary_replica }
end

# Uso en controladores
class ReportsController < ApplicationController
  def index
    ActiveRecord::Base.connected_to(role: :reading) do
      @reports = Report.complex_query.to_a
    end
  end
end
```

### Parameter Groups

```ruby
# Configuración recomendada para Rails (via AWS CLI o Console)
# Crear parameter group personalizado

# Parámetros importantes:
# - shared_buffers: 25% de la memoria
# - effective_cache_size: 75% de la memoria
# - work_mem: 64MB
# - maintenance_work_mem: 512MB
# - checkpoint_completion_target: 0.9
# - wal_buffers: 16MB
# - random_page_cost: 1.1 (para SSD)
# - log_statement: 'all' (desarrollo)
# - log_min_duration_statement: 1000 (queries > 1s)
```

## Lambda

### Rails con Lambda via Jets o Lamby

```ruby
# Gemfile
gem "lamby"

# config/lamby.rb
Lamby.config do |config|
  config.rack_app = Rails.application
end

# handler.rb
require_relative "config/boot"
require "lamby"
require_relative "config/application"
require_relative "config/environment"

$app = Rack::Builder.new { run Rails.application }.to_app

def handler(event:, context:)
  Lamby.handler($app, event, context)
end
```

### API Gateway + Lambda para Webhooks

```ruby
# Función Lambda para procesar webhooks
# handler.rb
require "json"
require "aws-sdk-sqs"

def handler(event:, context:)
  body = JSON.parse(event["body"])

  # Enviar a SQS para procesamiento async
  sqs = Aws::SQS::Client.new
  sqs.send_message(
    queue_url: ENV["WEBHOOK_QUEUE_URL"],
    message_body: body.to_json
  )

  {
    statusCode: 200,
    body: JSON.generate({ received: true })
  }
rescue JSON::ParserError
  {
    statusCode: 400,
    body: JSON.generate({ error: "Invalid JSON" })
  }
end
```

## ECS/Fargate

### Dockerfile para Rails

```dockerfile
# Dockerfile
FROM ruby:3.3.0-slim

# Instalar dependencias
RUN apt-get update -qq && \
    apt-get install -y build-essential libpq-dev nodejs npm && \
    npm install -g yarn && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Instalar gems
COPY Gemfile Gemfile.lock ./
RUN bundle config set --local deployment 'true' && \
    bundle config set --local without 'development test' && \
    bundle install

# Copiar aplicación
COPY . .

# Precompilar assets
RUN SECRET_KEY_BASE=dummy bundle exec rails assets:precompile

# Puerto
EXPOSE 3000

# Comando
CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

### Task Definition

```json
{
  "family": "rails-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "rails-app",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/rails-app:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        { "name": "RAILS_ENV", "value": "production" },
        { "name": "RAILS_LOG_TO_STDOUT", "value": "true" }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:rails-app/database-url"
        },
        {
          "name": "SECRET_KEY_BASE",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:rails-app/secret-key-base"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/rails-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

## CloudFront (CDN)

### Configurar con Rails Assets

```ruby
# config/environments/production.rb
config.asset_host = ENV["CLOUDFRONT_DOMAIN"]
# Ejemplo: "https://d1234567890.cloudfront.net"

# Para Active Storage con CloudFront
config.active_storage.resolve_model_to_route = :rails_storage_proxy
```

### Invalidar cache después de deploy

```ruby
# lib/tasks/cloudfront.rake
namespace :cloudfront do
  desc "Invalidar cache de CloudFront"
  task invalidate: :environment do
    require "aws-sdk-cloudfront"

    client = Aws::CloudFront::Client.new(region: "us-east-1")

    client.create_invalidation(
      distribution_id: ENV["CLOUDFRONT_DISTRIBUTION_ID"],
      invalidation_batch: {
        paths: {
          quantity: 1,
          items: ["/assets/*"]
        },
        caller_reference: "deploy-#{Time.current.to_i}"
      }
    )

    puts "Invalidation created"
  end
end
```

## Route53

### Configurar DNS para la app

```ruby
# Ejemplo con Terraform
resource "aws_route53_record" "app" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.midominio.com"
  type    = "A"

  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}

# Health check
resource "aws_route53_health_check" "app" {
  fqdn              = "app.midominio.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/up"
  failure_threshold = "3"
  request_interval  = "30"
}
```

## IAM (Identity and Access Management)

### Política mínima para Rails App

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3Access",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::mi-app-production/*"
    },
    {
      "Sid": "S3ListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::mi-app-production"
    },
    {
      "Sid": "SESAccess",
      "Effect": "Allow",
      "Action": [
        "ses:SendEmail",
        "ses:SendRawEmail"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ses:FromAddress": "noreply@midominio.com"
        }
      }
    },
    {
      "Sid": "SecretsManagerAccess",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:rails-app/*"
    },
    {
      "Sid": "SQSAccess",
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:rails-app-*"
    }
  ]
}
```

## ElastiCache (Redis)

### Configurar con Rails

```ruby
# Gemfile
gem "redis"
gem "hiredis"

# config/initializers/redis.rb
REDIS = Redis.new(
  url: ENV.fetch("ELASTICACHE_URL") { "redis://localhost:6379/0" },
  driver: :hiredis
)

# config/environments/production.rb
# Cache store
config.cache_store = :redis_cache_store, {
  url: ENV["ELASTICACHE_URL"],
  pool_size: ENV.fetch("RAILS_MAX_THREADS") { 5 }.to_i,
  pool_timeout: 5
}

# Session store
config.session_store :redis_store,
  servers: [ENV["ELASTICACHE_URL"]],
  expire_after: 1.day,
  key: "_myapp_session"

# Action Cable
config.action_cable.adapter = :redis
config.action_cable.url = ENV["ELASTICACHE_URL"]
```

## SES (Simple Email Service)

### Configurar Action Mailer

```ruby
# config/environments/production.rb
config.action_mailer.delivery_method = :ses

# Opción 1: Con gem aws-sdk-ses
config.action_mailer.ses_settings = {
  region: "us-east-1",
  credentials: Aws::Credentials.new(
    Rails.application.credentials.dig(:aws, :access_key_id),
    Rails.application.credentials.dig(:aws, :secret_access_key)
  )
}

# Opción 2: Con SMTP
config.action_mailer.smtp_settings = {
  address: "email-smtp.us-east-1.amazonaws.com",
  port: 587,
  user_name: ENV["SES_SMTP_USERNAME"],
  password: ENV["SES_SMTP_PASSWORD"],
  authentication: :login,
  enable_starttls_auto: true
}
```

## Secrets Manager / Parameter Store

### Cargar secrets al iniciar Rails

```ruby
# config/initializers/aws_secrets.rb
if Rails.env.production?
  require "aws-sdk-secretsmanager"

  client = Aws::SecretsManager::Client.new(region: "us-east-1")

  begin
    secret = client.get_secret_value(secret_id: "rails-app/#{Rails.env}")
    secrets = JSON.parse(secret.secret_string)

    # Setear variables de entorno
    secrets.each do |key, value|
      ENV[key] = value unless ENV[key].present?
    end
  rescue Aws::SecretsManager::Errors::ServiceError => e
    Rails.logger.error "Error loading secrets: #{e.message}"
  end
end
```

### Rotar secrets automáticamente

```ruby
# Lambda function para rotación
def handler(event:, context:)
  require "aws-sdk-secretsmanager"
  require "securerandom"

  client = Aws::SecretsManager::Client.new
  secret_id = event["SecretId"]
  step = event["Step"]

  case step
  when "createSecret"
    # Generar nuevo secret
    new_secret = SecureRandom.hex(64)
    client.put_secret_value(
      secret_id: secret_id,
      client_request_token: event["ClientRequestToken"],
      secret_string: new_secret,
      version_stages: ["AWSPENDING"]
    )
  when "setSecret"
    # Aplicar nuevo secret (actualizar en la app)
  when "testSecret"
    # Verificar que funciona
  when "finishSecret"
    # Marcar como current
    client.update_secret_version_stage(
      secret_id: secret_id,
      version_stage: "AWSCURRENT",
      move_to_version_id: event["ClientRequestToken"],
      remove_from_version_id: get_current_version(client, secret_id)
    )
  end
end
```

## CloudWatch

### Configurar logging

```ruby
# config/environments/production.rb
if ENV["AWS_EXECUTION_ENV"].present? || ENV["ECS_CONTAINER_METADATA_URI"].present?
  config.logger = ActiveSupport::Logger.new(STDOUT)
  config.logger.formatter = proc do |severity, datetime, progname, msg|
    {
      timestamp: datetime.iso8601,
      level: severity,
      message: msg,
      progname: progname
    }.to_json + "\n"
  end
end
```

### Custom Metrics

```ruby
# app/services/cloudwatch_metrics.rb
class CloudwatchMetrics
  def initialize
    @client = Aws::CloudWatch::Client.new(region: "us-east-1")
    @namespace = "RailsApp/#{Rails.env}"
  end

  def record(metric_name, value, unit: "Count", dimensions: {})
    @client.put_metric_data(
      namespace: @namespace,
      metric_data: [
        {
          metric_name: metric_name,
          value: value,
          unit: unit,
          timestamp: Time.current,
          dimensions: dimensions.map { |k, v| { name: k.to_s, value: v.to_s } }
        }
      ]
    )
  end

  def record_request_time(duration_ms, controller:, action:)
    record(
      "RequestDuration",
      duration_ms,
      unit: "Milliseconds",
      dimensions: { Controller: controller, Action: action }
    )
  end
end

# Uso en ApplicationController
class ApplicationController < ActionController::Base
  around_action :record_metrics

  private

  def record_metrics
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    yield
  ensure
    duration = ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - start) * 1000).round
    CloudwatchMetrics.new.record_request_time(
      duration,
      controller: controller_name,
      action: action_name
    )
  end
end
```

### Alarms

```ruby
# Crear alarma para errores 5xx
resource "aws_cloudwatch_metric_alarm" "high_5xx_errors" {
  alarm_name          = "rails-app-high-5xx-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = "300"
  statistic           = "Sum"
  threshold           = "10"
  alarm_description   = "Alerta cuando hay mas de 10 errores 5xx en 5 minutos"

  dimensions = {
    LoadBalancer = aws_lb.main.arn_suffix
    TargetGroup  = aws_lb_target_group.rails.arn_suffix
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]
}
```

## Costos y Optimización

### Instancias Reservadas vs Spot

```ruby
# Para workloads predecibles: Reserved Instances
# Ahorro: 30-60% vs On-Demand

# Para workers/jobs: Spot Instances
# Ahorro: hasta 90%

# Estrategia mixta para ECS:
# - Web servers: On-Demand o Reserved
# - Background workers: Spot con fallback a On-Demand
```

### Calculadora de costos típicos

```markdown
## Estimación mensual para app Rails pequeña-mediana

| Servicio | Configuración | Costo/mes |
|----------|--------------|-----------|
| EC2/ECS | t3.medium (2 instancias) | ~$60 |
| RDS | db.t3.small PostgreSQL | ~$25 |
| ElastiCache | cache.t3.micro | ~$12 |
| S3 | 50GB + transferencia | ~$5 |
| CloudFront | 100GB transferencia | ~$10 |
| Route53 | 1 zona + queries | ~$1 |
| SES | 10,000 emails | ~$1 |
| **Total** | | **~$114/mes** |

## Optimizaciones:
- Usar Savings Plans (hasta 72% ahorro)
- Auto Scaling para reducir en horas bajas
- S3 Intelligent-Tiering para archivos
- Reserved Capacity para RDS
```

## Checklist de Deployment

- [ ] VPC con subnets públicas y privadas
- [ ] Security Groups configurados (mínimo privilegio)
- [ ] RDS en subnet privada con backups automáticos
- [ ] S3 bucket con versionado y lifecycle policies
- [ ] IAM roles con políticas mínimas
- [ ] Secrets en Secrets Manager o Parameter Store
- [ ] CloudWatch logs y métricas configuradas
- [ ] Alarmas para métricas críticas
- [ ] CloudFront para assets estáticos
- [ ] Route53 con health checks
- [ ] SSL/TLS con ACM (Certificate Manager)
- [ ] Backups automáticos probados
