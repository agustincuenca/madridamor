# Comando: /deploy

## Descripción

Despliega la aplicación a producción de forma guiada y segura.

## Uso

```
/deploy [entorno]
```

## Parámetros

- `entorno` (opcional): "staging" o "production" (default: production)

## Proceso de deploy

### 1. Pre-checks

Antes de desplegar, verificar:

```bash
# Estado del repositorio
git status
git log --oneline -5

# Tests pasando
bin/rails test
# o
bundle exec rspec

# Assets compilan
bin/rails assets:precompile

# Migraciones pendientes
bin/rails db:migrate:status
```

### 2. Checklist pre-deploy

- [ ] Todos los tests pasan
- [ ] No hay cambios sin commitear
- [ ] La rama está actualizada con main/master
- [ ] Las migraciones son reversibles
- [ ] Variables de entorno configuradas
- [ ] Backup de base de datos realizado

### 3. Opciones de hosting gratuito

#### Fly.io (Recomendado)

```bash
# Instalación
curl -L https://fly.io/install.sh | sh

# Login
fly auth login

# Primera vez: crear app
fly launch

# Deploy
fly deploy

# Ver logs
fly logs

# Consola Rails
fly ssh console -C "bin/rails console"

# Migraciones
fly ssh console -C "bin/rails db:migrate"
```

**fly.toml básico:**
```toml
app = "mi-app"
primary_region = "mad"

[build]
  [build.args]
    RUBY_VERSION = "3.3.0"

[env]
  RAILS_ENV = "production"
  RAILS_LOG_TO_STDOUT = "true"

[http_service]
  internal_port = 3000
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true
  min_machines_running = 0

[[services]]
  protocol = "tcp"
  internal_port = 3000

  [[services.ports]]
    port = 80
    handlers = ["http"]

  [[services.ports]]
    port = 443
    handlers = ["tls", "http"]

[[statics]]
  guest_path = "/rails/public"
  url_prefix = "/"
```

#### Render.com

```yaml
# render.yaml
services:
  - type: web
    name: mi-app
    env: ruby
    buildCommand: bundle install && bin/rails assets:precompile
    startCommand: bin/rails server
    envVars:
      - key: RAILS_MASTER_KEY
        sync: false
      - key: DATABASE_URL
        fromDatabase:
          name: mi-app-db
          property: connectionString

databases:
  - name: mi-app-db
    plan: free
```

#### Railway.app

```bash
# Instalación CLI
npm i -g @railway/cli

# Login
railway login

# Crear proyecto
railway init

# Deploy
railway up

# Variables de entorno
railway variables set RAILS_ENV=production
```

### 4. Deploy con Kamal (Docker)

```bash
# Instalación
gem install kamal

# Configurar
kamal init

# Deploy
kamal deploy
```

**config/deploy.yml:**
```yaml
service: mi-app
image: usuario/mi-app

servers:
  web:
    - 192.168.0.1
  job:
    hosts:
      - 192.168.0.1
    cmd: bin/jobs

registry:
  username: usuario
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  clear:
    RAILS_ENV: production
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_URL

accessories:
  db:
    image: sqlite3
    host: 192.168.0.1
    directories:
      - data:/rails/storage

traefik:
  options:
    publish:
      - "443:443"
    volume:
      - "/letsencrypt:/letsencrypt"
```

### 5. Scripts de deploy

```bash
#!/bin/bash
# bin/deploy

set -e

echo "🚀 Iniciando deploy..."

# Pre-checks
echo "📋 Verificando pre-requisitos..."

if ! git diff-index --quiet HEAD --; then
  echo "❌ Hay cambios sin commitear"
  exit 1
fi

echo "🧪 Ejecutando tests..."
bin/rails test || bundle exec rspec

echo "📦 Compilando assets..."
bin/rails assets:precompile

echo "🔄 Desplegando..."
# Descomentar según plataforma:
# fly deploy
# railway up
# kamal deploy
# git push heroku main

echo "🗃️ Ejecutando migraciones..."
# fly ssh console -C "bin/rails db:migrate"
# railway run bin/rails db:migrate
# kamal app exec 'bin/rails db:migrate'

echo "✅ Deploy completado!"
```

### 6. Post-deploy checks

```ruby
# lib/tasks/health.rake
namespace :health do
  desc "Verificar estado de la aplicación"
  task check: :environment do
    checks = []

    # Base de datos
    begin
      ActiveRecord::Base.connection.execute("SELECT 1")
      checks << "✅ Base de datos: OK"
    rescue => e
      checks << "❌ Base de datos: #{e.message}"
    end

    # Cache
    begin
      Rails.cache.write("health_check", "ok", expires_in: 1.minute)
      Rails.cache.read("health_check") == "ok" or raise "Cache read failed"
      checks << "✅ Cache: OK"
    rescue => e
      checks << "❌ Cache: #{e.message}"
    end

    # Storage
    begin
      ActiveStorage::Blob.service.exist?("health_check_#{Time.now.to_i}")
      checks << "✅ Storage: OK"
    rescue => e
      checks << "⚠️ Storage: #{e.message}"
    end

    # Jobs
    begin
      SolidQueue::Job.count
      checks << "✅ Jobs: OK"
    rescue => e
      checks << "⚠️ Jobs: #{e.message}"
    end

    puts checks.join("\n")
  end
end
```

### 7. Rollback

```bash
# Fly.io
fly releases
fly deploy --image registry.fly.io/mi-app:v123

# Kamal
kamal rollback

# Railway
railway rollback

# Git-based (Heroku, Render)
git revert HEAD
git push
```

## Variables de entorno necesarias

```bash
# Producción mínimo
RAILS_ENV=production
RAILS_MASTER_KEY=xxx
SECRET_KEY_BASE=xxx
DATABASE_URL=sqlite3:///rails/storage/production.sqlite3

# Opcionales
RAILS_LOG_TO_STDOUT=true
RAILS_SERVE_STATIC_FILES=true
WEB_CONCURRENCY=2
RAILS_MAX_THREADS=5
```

## Ejemplo de salida

```
🚀 Iniciando proceso de deploy...

📋 Pre-checks:
   ✅ Git: rama main, sin cambios pendientes
   ✅ Tests: 142 tests, 0 failures
   ✅ Assets: compilados correctamente
   ✅ Migraciones: 0 pendientes

🔐 Verificación de seguridad:
   ✅ RAILS_MASTER_KEY configurado
   ✅ Credenciales encriptadas
   ✅ No hay secretos expuestos

📦 Desplegando a Fly.io...
   → Building image...
   → Pushing to registry...
   → Deploying v24...
   → Health check passed

🗃️ Post-deploy:
   → Ejecutando migraciones... OK
   → Verificando salud... OK

✅ Deploy completado exitosamente!

🌐 Tu app está disponible en: https://mi-app.fly.dev
```

## Respuesta del agente

Al ejecutar `/deploy`, el agente:

1. **Verifica pre-requisitos**
   - Estado de git
   - Tests pasando
   - Assets compilando

2. **Detecta plataforma**
   - Busca fly.toml, render.yaml, etc.
   - Sugiere opciones si no hay configuración

3. **Ejecuta deploy**
   - Usa comandos de la plataforma detectada
   - Muestra progreso en tiempo real

4. **Verifica post-deploy**
   - Health checks
   - Migraciones
   - Logs de errores

5. **Reporta resultado**
   - URL de la aplicación
   - Cualquier warning o error
