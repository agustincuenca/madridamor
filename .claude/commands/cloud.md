# Comando: /cloud

Despliegue y gestión de aplicaciones en la nube.

## Uso

```
/cloud [provider] [acción]
```

## Providers soportados

| Provider | Descripción | Tier gratuito |
|----------|-------------|---------------|
| `fly` | Fly.io (recomendado) | 3 VMs pequeñas gratis |
| `render` | Render.com | Web service gratis |
| `railway` | Railway.app | $5 crédito mensual |
| `aws` | Amazon Web Services | 12 meses free tier |
| `gcp` | Google Cloud Platform | $300 crédito inicial |
| `azure` | Microsoft Azure | $200 crédito inicial |

## Acciones disponibles

| Acción | Descripción |
|--------|-------------|
| `setup` | Configura el proyecto para el provider |
| `deploy` | Despliega la aplicación |
| `logs` | Muestra logs de producción |
| `status` | Estado de la aplicación |
| `scale [n]` | Escala a n instancias |
| `costs` | Estimación de costos |

## Flujo de trabajo

### `/cloud` (sin argumentos)

Muestra estado actual y opciones disponibles.

```markdown
## Cloud Status

### Configuración detectada
- 🟢 Fly.io (fly.toml encontrado)
- ⚪ Render (no configurado)
- ⚪ Railway (no configurado)

### Recomendación
Para este proyecto, recomiendo **Fly.io** porque:
- Tier gratuito generoso
- SQLite-friendly (volúmenes persistentes)
- Deploy simple con CLI
- Servers en Europa disponibles

¿Qué te gustaría hacer?
1. `/cloud fly deploy` - Desplegar a Fly.io
2. `/cloud fly logs` - Ver logs
3. `/cloud render setup` - Configurar Render como alternativa
```

### `/cloud [provider] setup`

Configura el proyecto para el provider seleccionado.

#### Fly.io Setup

```bash
# Instalar CLI si no está
curl -L https://fly.io/install.sh | sh

# Login
fly auth login

# Crear app
fly launch --no-deploy
```

**Genera `fly.toml`:**

```toml
app = "mi-app"
primary_region = "mad"  # Madrid

[build]

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

[[vm]]
  memory = "512mb"
  cpu_kind = "shared"
  cpus = 1

[mounts]
  source = "data"
  destination = "/rails/storage"
```

**Configura secrets:**

```bash
fly secrets set RAILS_MASTER_KEY=$(cat config/master.key)
fly secrets set SECRET_KEY_BASE=$(rails secret)
```

#### Render Setup

**Genera `render.yaml`:**

```yaml
services:
  - type: web
    name: mi-app
    runtime: ruby
    buildCommand: |
      bundle install
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
          name: mi-app-db
          property: connectionString

databases:
  - name: mi-app-db
    plan: free
    databaseName: mi_app_production
```

#### Railway Setup

```bash
# Instalar CLI
npm i -g @railway/cli

# Login y crear proyecto
railway login
railway init

# Configurar variables
railway variables set RAILS_ENV=production
railway variables set RAILS_MASTER_KEY=$(cat config/master.key)
```

### `/cloud [provider] deploy`

Despliega la aplicación al provider.

**Pre-checks:**

```markdown
## Pre-deploy Checklist

- [ ] Tests pasando
- [ ] Assets compilan
- [ ] Migraciones listas
- [ ] Secrets configurados
- [ ] .gitignore correcto
```

#### Deploy a Fly.io

```bash
# Deploy
fly deploy

# Ejecutar migraciones
fly ssh console -C "bin/rails db:migrate"

# Verificar
fly status
```

**Salida:**

```markdown
## Deploy a Fly.io

### Proceso
```
📦 Building image...
   → Using Dockerfile
   → Build successful

🚀 Deploying...
   → Creating release v5
   → Updating machines
   → Waiting for health checks

✅ Deploy successful!
```

### Detalles
- **URL**: https://mi-app.fly.dev
- **Region**: Madrid (mad)
- **Versión**: v5
- **Tiempo**: 2m 34s

### Health Check
- ✅ HTTP responding
- ✅ Database connected
- ✅ Assets serving

### Próximos pasos
- Verificar la app: https://mi-app.fly.dev
- Ver logs: `/cloud fly logs`
- Escalar si es necesario: `/cloud fly scale 2`
```

### `/cloud [provider] logs`

Muestra logs de producción.

```bash
# Fly.io
fly logs --app mi-app

# Render
# Usar dashboard web

# Railway
railway logs
```

**Filtros disponibles:**

```markdown
## Logs de Producción

### Opciones de filtrado
- `/cloud fly logs` - Últimos logs en tiempo real
- `/cloud fly logs --error` - Solo errores
- `/cloud fly logs --today` - Logs de hoy
- `/cloud fly logs --search "texto"` - Buscar texto

### Logs recientes
```
2024-01-15 10:23:45 [info] GET /articles 200 in 45ms
2024-01-15 10:23:46 [info] GET /articles/1 200 in 32ms
2024-01-15 10:23:50 [error] NoMethodError in ArticlesController#show
2024-01-15 10:23:50 [error] undefined method `name' for nil:NilClass
```

### Errores detectados
⚠️ 1 error en los últimos 5 minutos

¿Quieres que analice el error con `/debug`?
```

### `/cloud [provider] status`

Muestra estado detallado de la aplicación.

```markdown
## Estado de mi-app en Fly.io

### General
- **Estado**: 🟢 Running
- **URL**: https://mi-app.fly.dev
- **Region**: Madrid (mad)
- **Uptime**: 5 días, 3 horas

### Instancias
| ID | Estado | CPU | Memoria | Region |
|----|--------|-----|---------|--------|
| abc123 | running | 2% | 256MB/512MB | mad |

### Métricas (últimas 24h)
- **Requests**: 1,234
- **Errores**: 2 (0.16%)
- **Tiempo respuesta**: 45ms avg

### Volúmenes
| Nombre | Tamaño | Usado |
|--------|--------|-------|
| data | 1GB | 45MB |

### Secrets configurados
- RAILS_MASTER_KEY ✅
- SECRET_KEY_BASE ✅

### Próximas acciones sugeridas
- Todo está funcionando correctamente
- Considera activar auto-scaling si esperas más tráfico
```

### `/cloud [provider] scale [n]`

Escala la aplicación a n instancias.

```bash
# Fly.io
fly scale count 2

# Render
# Usar dashboard (free tier = 1 instancia)

# Railway
# Automático según uso
```

**Salida:**

```markdown
## Escalado de mi-app

### Configuración actual
- Instancias: 1
- Memoria por instancia: 512MB
- CPU: shared

### Nuevo estado (después de escalar a 2)
- Instancias: 2
- Memoria total: 1GB
- Regiones: mad (2 instancias)

### Impacto en costos
- Antes: $0/mes (free tier)
- Después: ~$5/mes (segundo VM no es gratis)

⚠️ Esto excederá el tier gratuito de Fly.io.

¿Continuar? (s/n)
```

### `/cloud [provider] costs`

Muestra estimación de costos.

```markdown
## Estimación de Costos - Fly.io

### Uso actual
| Recurso | Cantidad | Precio | Total |
|---------|----------|--------|-------|
| VM shared-cpu-1x | 1 | $0 (free) | $0 |
| Memoria 512MB | 512MB | Incluido | $0 |
| Volumen | 1GB | $0.15/GB | $0.15 |
| Bandwidth | 5GB | Incluido | $0 |

**Total mensual estimado: $0.15**

### Si escalaras a 2 instancias
| Recurso | Cantidad | Precio | Total |
|---------|----------|--------|-------|
| VM shared-cpu-1x | 2 | $1.94/mes | $3.88 |
| Memoria 512MB | 1GB | Incluido | $0 |
| Volumen | 1GB | $0.15/GB | $0.15 |

**Total mensual estimado: $4.03**

### Comparación con otros providers

| Provider | Config similar | Costo/mes |
|----------|---------------|-----------|
| Fly.io | 1 VM + volumen | $0.15 |
| Render | Free tier | $0 |
| Railway | Free tier | $0-5 |
| Heroku | Eco dyno | $5 |
| AWS (EC2) | t3.micro | ~$8 |

### Recomendación
Para tu nivel de tráfico actual, el tier gratuito es suficiente.
Considera escalar cuando tengas >1000 usuarios activos diarios.
```

## Configuración por provider

### Fly.io (Recomendado)

**Ventajas:**
- SQLite funciona bien con volúmenes
- Deploy rápido
- CLI excelente
- Tier gratuito generoso

**Dockerfile para Rails:**

```dockerfile
FROM ruby:3.3-slim

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
    build-essential libsqlite3-dev nodejs

WORKDIR /rails

COPY Gemfile Gemfile.lock ./
RUN bundle install

COPY . .

RUN bundle exec rails assets:precompile

EXPOSE 3000
CMD ["./bin/rails", "server", "-b", "0.0.0.0"]
```

### Render.com

**Ventajas:**
- Setup más simple
- PostgreSQL gratuito
- Auto-deploy desde GitHub

**Limitaciones free tier:**
- Se apaga después de 15min inactividad
- Solo 1 instancia

### Railway

**Ventajas:**
- UX muy pulido
- Fácil de usar
- PostgreSQL incluido

**Limitaciones:**
- $5 crédito mensual
- Puede no ser suficiente para uso intensivo

## Rollback

Si algo sale mal:

```bash
# Fly.io
fly releases
fly deploy --image registry.fly.io/mi-app:v4  # versión anterior

# Render
# Desde dashboard, seleccionar deploy anterior

# Railway
railway rollback
```

## Notas importantes

- Siempre tener backup antes de deploy
- Probar en staging antes de producción
- Monitorear logs después de cada deploy
- Configurar alertas para errores
- Documentar configuración de cada provider
