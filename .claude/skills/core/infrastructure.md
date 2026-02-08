# Skill: Infrastructure

## Purpose
Set up and manage deployment infrastructure for Rails applications.

## Development Environment

### Prerequisites
```bash
# Ruby (via rbenv)
brew install rbenv ruby-build
rbenv install 3.3.0
rbenv global 3.3.0

# Node.js (for asset compilation)
brew install node

# SQLite3
brew install sqlite3
```

### Project Setup
```bash
# Create new Rails app
rails new myapp --database=sqlite3 --css=tailwind --skip-jbuilder

# Or with existing project
cd myapp
bundle install
rails db:setup
```

### Development Server
```bash
# Start server with all processes
bin/dev

# Or manually
rails server
```

## Production Configuration

### Database (SQLite3 for Production)
```yaml
# config/database.yml
production:
  adapter: sqlite3
  database: storage/production.sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000
```

### Environment Variables
```bash
# .env.production (not committed)
RAILS_ENV=production
SECRET_KEY_BASE=your-secret-key-here
RAILS_MASTER_KEY=your-master-key
```

### Production Settings
```ruby
# config/environments/production.rb
Rails.application.configure do
  config.cache_classes = true
  config.eager_load = true
  config.consider_all_requests_local = false
  config.action_controller.perform_caching = true

  # Force SSL
  config.force_ssl = true

  # Logging
  config.log_level = :info
  config.log_tags = [:request_id]

  # Asset serving
  config.public_file_server.enabled = ENV["RAILS_SERVE_STATIC_FILES"].present?

  # Active Storage
  config.active_storage.service = :local
end
```

## Deployment Options

### Option 1: Render.com (Recommended Free Option)

```yaml
# render.yaml
services:
  - type: web
    name: myapp
    env: ruby
    buildCommand: bundle install && bundle exec rails assets:precompile
    startCommand: bundle exec puma -C config/puma.rb
    envVars:
      - key: RAILS_MASTER_KEY
        sync: false
      - key: SECRET_KEY_BASE
        generateValue: true
    disk:
      name: data
      mountPath: /app/storage
      sizeGB: 1
```

### Option 2: Fly.io

```toml
# fly.toml
app = "myapp"
primary_region = "iad"

[build]
  builder = "heroku/buildpacks:22"

[env]
  RAILS_ENV = "production"
  RAILS_LOG_TO_STDOUT = "true"

[http_service]
  internal_port = 3000
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true

[mounts]
  source = "data"
  destination = "/app/storage"
```

```bash
# Deploy commands
fly launch
fly secrets set RAILS_MASTER_KEY=xxx
fly deploy
```

### Option 3: Railway

```json
// railway.json
{
  "build": {
    "builder": "nixpacks"
  },
  "deploy": {
    "startCommand": "bundle exec rails server -b 0.0.0.0 -p $PORT",
    "healthcheckPath": "/up",
    "healthcheckTimeout": 100
  }
}
```

### Option 4: VPS (DigitalOcean, Linode, etc.)

```bash
# Server setup script
#!/bin/bash

# Install dependencies
apt update && apt upgrade -y
apt install -y curl git build-essential libssl-dev zlib1g-dev

# Install Ruby via rbenv
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
source ~/.bashrc

rbenv install 3.3.0
rbenv global 3.3.0

# Install bundler
gem install bundler

# Clone and setup app
git clone your-repo /var/www/myapp
cd /var/www/myapp
bundle install --deployment
RAILS_ENV=production rails db:migrate
RAILS_ENV=production rails assets:precompile
```

## Puma Configuration

```ruby
# config/puma.rb
max_threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
min_threads_count = ENV.fetch("RAILS_MIN_THREADS") { max_threads_count }
threads min_threads_count, max_threads_count

port ENV.fetch("PORT") { 3000 }

environment ENV.fetch("RAILS_ENV") { "development" }

# Workers for production
workers ENV.fetch("WEB_CONCURRENCY") { 2 } if ENV["RAILS_ENV"] == "production"

preload_app!

# Solid Queue integration
plugin :solid_queue if ENV["RAILS_ENV"] == "production"
```

## Background Jobs in Production

### Solid Queue Setup
```bash
# Procfile
web: bundle exec puma -C config/puma.rb
worker: bundle exec rake solid_queue:start
```

### Or with Puma Plugin
```ruby
# config/puma.rb
plugin :solid_queue
```

## Health Checks

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check
end
```

## SSL/HTTPS

### Force SSL in Production
```ruby
# config/environments/production.rb
config.force_ssl = true
```

### Let's Encrypt (VPS)
```bash
# Install certbot
apt install certbot python3-certbot-nginx

# Get certificate
certbot --nginx -d yourdomain.com
```

## Monitoring

### Basic Logging
```ruby
# config/environments/production.rb
config.log_level = :info
config.log_formatter = ::Logger::Formatter.new

# Log to STDOUT for container environments
if ENV["RAILS_LOG_TO_STDOUT"].present?
  logger = ActiveSupport::Logger.new(STDOUT)
  logger.formatter = config.log_formatter
  config.logger = ActiveSupport::TaggedLogging.new(logger)
end
```

### Error Tracking (Optional)
```ruby
# Gemfile
gem "sentry-ruby"
gem "sentry-rails"

# config/initializers/sentry.rb
Sentry.init do |config|
  config.dsn = ENV["SENTRY_DSN"]
  config.environment = Rails.env
end
```

## Backup Strategy

### Database Backup Script
```bash
#!/bin/bash
# backup.sh

BACKUP_DIR="/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DB_FILE="/app/storage/production.sqlite3"

# Create backup
cp $DB_FILE "$BACKUP_DIR/production_$TIMESTAMP.sqlite3"

# Keep only last 7 days
find $BACKUP_DIR -name "*.sqlite3" -mtime +7 -delete
```

### Cron Job
```bash
# crontab -e
0 2 * * * /app/backup.sh
```

## Deployment Checklist

### Pre-deployment
- [ ] All tests pass
- [ ] No security vulnerabilities (brakeman)
- [ ] Environment variables set
- [ ] Database migrations ready
- [ ] Assets precompiled

### Deployment
- [ ] Database backup taken
- [ ] Deploy code
- [ ] Run migrations
- [ ] Restart application
- [ ] Verify health check

### Post-deployment
- [ ] Verify application loads
- [ ] Check error logs
- [ ] Test critical flows
- [ ] Monitor performance

## Rollback Plan

```bash
# If using Git-based deployment
git revert HEAD
git push origin main

# Restore database if needed
cp /backups/latest.sqlite3 /app/storage/production.sqlite3

# Restart application
systemctl restart myapp
```
