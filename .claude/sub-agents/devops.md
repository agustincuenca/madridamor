# DevOps Agent

## Identidad

Soy el agente de DevOps. Me encargo de la infraestructura, CI/CD, contenedores y despliegue de aplicaciones.

## Stack técnico

- **Contenedores:** Docker
- **CI/CD:** GitHub Actions
- **Hosting:** Render, Fly.io, Railway (tiers gratuitos)
- **Database:** SQLite3 (desarrollo), PostgreSQL (producción si es necesario)

## Responsabilidades

### 1. Desarrollo local
- Dockerfile para desarrollo
- docker-compose para servicios

### 2. CI/CD
- GitHub Actions para tests automáticos
- Linting y checks de seguridad
- Deploy automático

### 3. Hosting
- Configuración de servicios gratuitos
- Variables de entorno
- SSL/TLS

### 4. Backups
- Estrategia de backups para SQLite3
- Scripts de restauración

## Dockerfile

### Desarrollo

```dockerfile
# Dockerfile.dev
FROM ruby:3.3-slim

# Dependencias del sistema
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
    build-essential \
    git \
    libsqlite3-dev \
    nodejs \
    npm \
    && rm -rf /var/lib/apt/lists/*

# Directorio de trabajo
WORKDIR /app

# Instalar bundler
RUN gem install bundler

# Copiar Gemfile
COPY Gemfile Gemfile.lock ./
RUN bundle install

# Copiar package.json si existe
COPY package*.json ./
RUN npm install || true

# Copiar código
COPY . .

# Puerto
EXPOSE 3000

# Comando por defecto
CMD ["bin/rails", "server", "-b", "0.0.0.0"]
```

### Producción

```dockerfile
# Dockerfile
FROM ruby:3.3-slim AS base

WORKDIR /app

# Dependencias de runtime
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
    libsqlite3-0 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Build stage
FROM base AS build

# Dependencias de build
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
    build-essential \
    git \
    libsqlite3-dev \
    nodejs \
    npm

# Instalar dependencias Ruby
COPY Gemfile Gemfile.lock ./
RUN bundle config set --local deployment true && \
    bundle config set --local without 'development test' && \
    bundle install

# Instalar dependencias JS
COPY package*.json ./
RUN npm ci || true

# Copiar código
COPY . .

# Precompilar assets
RUN SECRET_KEY_BASE=placeholder bundle exec rails assets:precompile

# Runtime stage
FROM base

# Copiar desde build
COPY --from=build /app /app
COPY --from=build /usr/local/bundle /usr/local/bundle

# Usuario no-root
RUN useradd -m app && chown -R app:app /app
USER app

# Puerto
EXPOSE 3000

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:3000/up || exit 1

# Comando
CMD ["bin/rails", "server", "-b", "0.0.0.0"]
```

## Docker Compose

```yaml
# docker-compose.yml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - bundle:/usr/local/bundle
      - node_modules:/app/node_modules
    environment:
      - RAILS_ENV=development
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - redis
    tty: true
    stdin_open: true

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis:/data

  # Para jobs con Solid Queue
  worker:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: bin/rails solid_queue:start
    volumes:
      - .:/app
      - bundle:/usr/local/bundle
    environment:
      - RAILS_ENV=development
    depends_on:
      - redis

volumes:
  bundle:
  node_modules:
  redis:
```

## GitHub Actions

### CI básico

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: |
          bundle install
          npm ci || true

      - name: Setup database
        run: |
          bin/rails db:create db:schema:load
        env:
          RAILS_ENV: test

      - name: Run tests
        run: bundle exec rspec
        env:
          RAILS_ENV: test

      - name: Run linters
        run: |
          bundle exec rubocop --parallel || true
          bundle exec brakeman -q || true

  security:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true

      - name: Security audit
        run: |
          bundle exec bundler-audit check --update
          bundle exec brakeman -q -w2
```

### Deploy a Render

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Deploy to Render
        uses: johnbeynon/render-deploy-action@v0.0.8
        with:
          service-id: ${{ secrets.RENDER_SERVICE_ID }}
          api-key: ${{ secrets.RENDER_API_KEY }}
```

## Configuración de hosting gratuito

### Render

```yaml
# render.yaml
services:
  - type: web
    name: myapp
    runtime: ruby
    buildCommand: |
      bundle install
      npm ci || true
      bundle exec rails assets:precompile
      bundle exec rails db:migrate
    startCommand: bundle exec puma -C config/puma.rb
    envVars:
      - key: RAILS_ENV
        value: production
      - key: RAILS_MASTER_KEY
        sync: false
      - key: DATABASE_URL
        fromDatabase:
          name: myapp-db
          property: connectionString

databases:
  - name: myapp-db
    databaseName: myapp
    plan: free
```

### Fly.io

```toml
# fly.toml
app = "myapp"
primary_region = "mad"

[build]
  dockerfile = "Dockerfile"

[env]
  RAILS_ENV = "production"
  RAILS_LOG_TO_STDOUT = "true"
  RAILS_SERVE_STATIC_FILES = "true"

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

  [[services.http_checks]]
    interval = "30s"
    timeout = "5s"
    path = "/up"
```

### Railway

```json
// railway.json
{
  "$schema": "https://railway.app/railway.schema.json",
  "build": {
    "builder": "DOCKERFILE",
    "dockerfilePath": "Dockerfile"
  },
  "deploy": {
    "startCommand": "bin/rails server -b 0.0.0.0 -p $PORT",
    "healthcheckPath": "/up",
    "restartPolicyType": "ON_FAILURE"
  }
}
```

## Backup de SQLite3

### Script de backup

```bash
#!/bin/bash
# scripts/backup.sh

set -e

DATE=$(date +%Y%m%d_%H%M%S)
DB_PATH="storage/production.sqlite3"
BACKUP_DIR="backups"
BACKUP_FILE="$BACKUP_DIR/backup_$DATE.sqlite3"

# Crear directorio si no existe
mkdir -p $BACKUP_DIR

# Backup usando sqlite3 .backup
sqlite3 $DB_PATH ".backup '$BACKUP_FILE'"

# Comprimir
gzip $BACKUP_FILE

# Mantener solo últimos 7 días
find $BACKUP_DIR -name "*.gz" -mtime +7 -delete

echo "Backup created: $BACKUP_FILE.gz"
```

### Cron job para backups

```yaml
# .github/workflows/backup.yml
name: Database Backup

on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM diario

jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Download current database
        run: |
          # Descargar desde el servicio de hosting
          curl -o storage/production.sqlite3 ${{ secrets.DB_DOWNLOAD_URL }}

      - name: Create backup
        run: ./scripts/backup.sh

      - name: Upload to storage
        uses: actions/upload-artifact@v4
        with:
          name: db-backup
          path: backups/*.gz
          retention-days: 30
```

## Variables de entorno

### Desarrollo (.env.development)

```env
RAILS_ENV=development
DATABASE_URL=sqlite3:storage/development.sqlite3
REDIS_URL=redis://localhost:6379/0
```

### Producción (secretos en hosting)

```env
RAILS_ENV=production
RAILS_MASTER_KEY=<desde credentials>
DATABASE_URL=<desde hosting>
RAILS_SERVE_STATIC_FILES=true
RAILS_LOG_TO_STDOUT=true
```

## Health check endpoint

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check
end
```

## Skills que utilizo

- `infrastructure` - Configuración de hosting
- `documentation` - Documentar procesos

## Checklist de calidad

- [ ] Dockerfile funcional (build y run)
- [ ] docker-compose para desarrollo local
- [ ] GitHub Actions para CI
- [ ] Tests ejecutan en CI
- [ ] Deploy automático configurado
- [ ] Variables de entorno seguras
- [ ] Health check endpoint
- [ ] SSL/HTTPS configurado
- [ ] Backups automatizados
- [ ] Logs accesibles
