# Skill: Shell Scripting

## Purpose

Automatizar tareas repetitivas y operaciones de DevOps usando Bash scripting para proyectos Rails.

## Basics

### Variables

```bash
#!/bin/bash

# Asignación (sin espacios alrededor de =)
NAME="myapp"
VERSION="1.0.0"
COUNT=42

# Uso de variables
echo "Deploying $NAME version $VERSION"
echo "Deploying ${NAME} version ${VERSION}"  # Más explícito

# Variables de solo lectura
readonly DB_NAME="production"

# Variables por defecto
RAILS_ENV="${RAILS_ENV:-development}"  # development si no está definida
PORT="${PORT:=3000}"                    # Asigna 3000 si no está definida

# Resultado de comando
CURRENT_DATE=$(date +%Y-%m-%d)
GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD)

# Aritmética
COUNT=$((COUNT + 1))
TOTAL=$((10 * 5))
```

### Arrays

```bash
#!/bin/bash

# Declarar array
SERVERS=("web1" "web2" "web3")
TASKS=("migrate" "assets" "restart")

# Acceder a elementos
echo "${SERVERS[0]}"          # web1
echo "${SERVERS[@]}"          # Todos los elementos
echo "${#SERVERS[@]}"         # Número de elementos (3)
echo "${SERVERS[@]:1:2}"      # Slice: web2 web3

# Añadir elementos
SERVERS+=("web4")

# Iterar
for server in "${SERVERS[@]}"; do
  echo "Deploying to $server"
done

# Array asociativo (Bash 4+)
declare -A CONFIG
CONFIG[host]="localhost"
CONFIG[port]="5432"
CONFIG[database]="myapp_production"

echo "Connecting to ${CONFIG[host]}:${CONFIG[port]}"
```

### Strings

```bash
#!/bin/bash

STR="Hello World"

# Longitud
echo "${#STR}"                # 11

# Substring
echo "${STR:0:5}"             # Hello
echo "${STR:6}"               # World

# Reemplazo
echo "${STR/World/Rails}"     # Hello Rails
echo "${STR//o/0}"            # Hell0 W0rld (todos)

# Eliminar patrón
FILE="app/models/user.rb"
echo "${FILE%.rb}"            # app/models/user (elimina suffix)
echo "${FILE##*/}"            # user.rb (elimina prefix hasta /)
echo "${FILE%/*}"             # app/models (directorio)

# Mayúsculas/minúsculas (Bash 4+)
echo "${STR,,}"               # hello world
echo "${STR^^}"               # HELLO WORLD

# Verificar si está vacío
if [[ -z "$VAR" ]]; then
  echo "VAR está vacía"
fi

if [[ -n "$VAR" ]]; then
  echo "VAR tiene valor"
fi
```

## Control Flow

### If/Else

```bash
#!/bin/bash

# Básico
if [[ $RAILS_ENV == "production" ]]; then
  echo "Production mode"
elif [[ $RAILS_ENV == "staging" ]]; then
  echo "Staging mode"
else
  echo "Development mode"
fi

# Operadores de comparación (strings)
[[ $a == $b ]]    # Igual
[[ $a != $b ]]    # Diferente
[[ $a < $b ]]     # Menor (alfabético)
[[ $a =~ ^[0-9]+$ ]]  # Regex match

# Operadores de comparación (números)
[[ $a -eq $b ]]   # Igual
[[ $a -ne $b ]]   # Diferente
[[ $a -lt $b ]]   # Menor que
[[ $a -le $b ]]   # Menor o igual
[[ $a -gt $b ]]   # Mayor que
[[ $a -ge $b ]]   # Mayor o igual

# Operadores lógicos
[[ $a && $b ]]    # AND
[[ $a || $b ]]    # OR
[[ ! $a ]]        # NOT

# Verificar archivos
[[ -f "$FILE" ]]  # Archivo existe
[[ -d "$DIR" ]]   # Directorio existe
[[ -r "$FILE" ]]  # Legible
[[ -w "$FILE" ]]  # Escribible
[[ -x "$FILE" ]]  # Ejecutable
[[ -s "$FILE" ]]  # Tamaño > 0

# One-liner
[[ -f config.yml ]] && echo "Config exists" || echo "Config missing"
```

### Case

```bash
#!/bin/bash

case "$COMMAND" in
  start)
    echo "Starting server..."
    rails server -d
    ;;
  stop)
    echo "Stopping server..."
    pkill -f "rails server"
    ;;
  restart)
    $0 stop
    $0 start
    ;;
  status)
    pgrep -f "rails server" && echo "Running" || echo "Stopped"
    ;;
  *)
    echo "Usage: $0 {start|stop|restart|status}"
    exit 1
    ;;
esac
```

### For Loop

```bash
#!/bin/bash

# Lista simple
for env in development staging production; do
  echo "Environment: $env"
done

# Rango
for i in {1..5}; do
  echo "Iteration $i"
done

# Rango con paso
for i in {0..10..2}; do
  echo "Even: $i"
done

# Estilo C
for ((i=0; i<10; i++)); do
  echo "Index: $i"
done

# Archivos
for file in app/models/*.rb; do
  echo "Processing $file"
done

# Líneas de archivo
while IFS= read -r line; do
  echo "Line: $line"
done < config/servers.txt

# Resultado de comando
for user in $(cat /etc/passwd | cut -d: -f1); do
  echo "User: $user"
done
```

### While Loop

```bash
#!/bin/bash

# Contador
count=0
while [[ $count -lt 5 ]]; do
  echo "Count: $count"
  ((count++))
done

# Esperar condición
while ! pg_isready -h localhost -p 5432; do
  echo "Waiting for PostgreSQL..."
  sleep 2
done
echo "PostgreSQL is ready!"

# Loop infinito con break
while true; do
  read -p "Enter command (quit to exit): " cmd
  [[ $cmd == "quit" ]] && break
  eval "$cmd"
done

# Until (opuesto a while)
until [[ -f /tmp/ready.flag ]]; do
  echo "Waiting for ready flag..."
  sleep 1
done
```

## Funciones

```bash
#!/bin/bash

# Función simple
greet() {
  echo "Hello, World!"
}

# Función con argumentos
deploy() {
  local server=$1
  local branch=${2:-main}  # Default: main

  echo "Deploying $branch to $server"
  ssh "$server" "cd /app && git pull origin $branch && bundle install"
}

# Uso
deploy "web1.example.com" "feature/new-design"
deploy "web2.example.com"  # Usa branch main

# Retornar valor (0-255)
is_production() {
  [[ $RAILS_ENV == "production" ]]
  return $?  # 0 si true, 1 si false
}

if is_production; then
  echo "Production checks enabled"
fi

# Retornar string
get_version() {
  cat VERSION
}
VERSION=$(get_version)

# Variables locales
calculate() {
  local result=$(($1 + $2))
  echo $result
}
SUM=$(calculate 5 3)  # 8

# Función con validación
backup_database() {
  local db_name=$1
  local backup_path=$2

  if [[ -z "$db_name" ]] || [[ -z "$backup_path" ]]; then
    echo "Usage: backup_database <db_name> <backup_path>" >&2
    return 1
  fi

  pg_dump "$db_name" > "$backup_path"
  return $?
}
```

## Pipes y Redirección

### Pipes

```bash
#!/bin/bash

# Encadenar comandos
cat access.log | grep "POST" | wc -l

# Múltiples pipes
ps aux | grep rails | grep -v grep | awk '{print $2}'

# Pipe a while
cat servers.txt | while read server; do
  echo "Checking $server"
  ping -c 1 "$server"
done

# tee: escribir a archivo y stdout
bundle install 2>&1 | tee install.log
```

### Redirección

```bash
#!/bin/bash

# Redireccionar stdout
echo "Log message" > app.log       # Sobrescribir
echo "Another line" >> app.log     # Añadir

# Redireccionar stderr
command 2> error.log               # Solo stderr
command 2>> error.log              # Añadir stderr

# Ambos a mismo archivo
command > output.log 2>&1          # stdout y stderr
command &> output.log              # Forma corta (Bash)

# Separar stdout y stderr
command > stdout.log 2> stderr.log

# Descartar output
command > /dev/null                # Ignorar stdout
command 2> /dev/null               # Ignorar stderr
command &> /dev/null               # Ignorar todo

# Here document
cat > config.yml <<EOF
database:
  host: localhost
  port: 5432
  name: $DB_NAME
EOF

# Here string
grep "error" <<< "$LOG_CONTENT"

# Process substitution
diff <(cat file1.txt) <(cat file2.txt)
```

## Comandos Útiles

### grep

```bash
#!/bin/bash

# Búsqueda básica
grep "error" app.log
grep -i "error" app.log            # Case insensitive
grep -n "error" app.log            # Mostrar números de línea
grep -c "error" app.log            # Contar coincidencias
grep -v "debug" app.log            # Invertir (excluir)

# Regex
grep -E "error|warning" app.log    # Extended regex
grep "user_[0-9]+" app.log         # Pattern con números

# Archivos
grep -r "TODO" app/                # Recursivo
grep -l "class User" app/models/   # Solo nombres de archivo
grep -L "test" spec/               # Archivos sin match

# Contexto
grep -A 3 "error" app.log          # 3 líneas después
grep -B 3 "error" app.log          # 3 líneas antes
grep -C 3 "error" app.log          # 3 líneas antes y después
```

### awk

```bash
#!/bin/bash

# Imprimir columnas
awk '{print $1}' file.txt          # Primera columna
awk '{print $1, $3}' file.txt      # Columnas 1 y 3
awk '{print $NF}' file.txt         # Última columna

# Con delimitador
awk -F: '{print $1}' /etc/passwd   # Delimitador :
awk -F',' '{print $2}' data.csv    # Delimitador ,

# Condiciones
awk '$3 > 100 {print $1}' data.txt
awk '/error/ {print}' app.log

# Cálculos
awk '{sum += $1} END {print sum}' numbers.txt
awk '{count++} END {print count}' file.txt

# Variables
awk -v threshold=100 '$3 > threshold {print}' data.txt

# Ejemplo: analizar logs de Rails
awk '/Completed [45][0-9]{2}/ {print $4, $0}' production.log
```

### sed

```bash
#!/bin/bash

# Reemplazar texto
sed 's/old/new/' file.txt          # Primera ocurrencia por línea
sed 's/old/new/g' file.txt         # Todas las ocurrencias
sed -i 's/old/new/g' file.txt      # In-place (modifica archivo)
sed -i.bak 's/old/new/g' file.txt  # Con backup

# Eliminar líneas
sed '/pattern/d' file.txt          # Líneas con pattern
sed '5d' file.txt                  # Línea 5
sed '1,10d' file.txt               # Líneas 1-10

# Insertar/añadir
sed '3i\New line' file.txt         # Insertar antes de línea 3
sed '3a\New line' file.txt         # Añadir después de línea 3

# Múltiples comandos
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt

# Ejemplo: actualizar config
sed -i "s/DB_HOST=.*/DB_HOST=$NEW_HOST/" .env
```

### xargs

```bash
#!/bin/bash

# Ejecutar comando con cada línea
cat files.txt | xargs rm
find . -name "*.log" | xargs rm

# Con placeholder
cat servers.txt | xargs -I {} ssh {} "uptime"

# Paralelo
cat urls.txt | xargs -P 4 -I {} curl -s {}

# Con confirmación
find . -name "*.tmp" | xargs -p rm

# Manejar espacios en nombres
find . -name "*.rb" -print0 | xargs -0 wc -l
```

### find

```bash
#!/bin/bash

# Por nombre
find . -name "*.rb"                # Archivos .rb
find . -iname "*.rb"               # Case insensitive
find . -name "user*"               # Empiezan con user

# Por tipo
find . -type f                     # Solo archivos
find . -type d                     # Solo directorios

# Por tiempo
find . -mtime -7                   # Modificados últimos 7 días
find . -mmin -60                   # Modificados última hora

# Por tamaño
find . -size +10M                  # Mayores de 10MB
find . -size -1k                   # Menores de 1KB

# Ejecutar comando
find . -name "*.rb" -exec wc -l {} \;
find . -name "*.log" -exec rm {} +

# Combinaciones
find app -name "*.rb" -type f -mtime -7

# Excluir directorios
find . -name "*.rb" -not -path "./vendor/*"
```

## Script Structure

### Shebang y header

```bash
#!/bin/bash
#
# Script: deploy.sh
# Description: Deploy Rails application
# Author: Your Name
# Date: 2024-01-15
# Usage: ./deploy.sh [environment]
#

set -euo pipefail  # Exit on error, undefined vars, pipe failures
IFS=$'\n\t'        # Safer word splitting
```

### Exit codes

```bash
#!/bin/bash

# Códigos estándar
# 0 = éxito
# 1 = error general
# 2 = uso incorrecto

deploy() {
  if [[ -z "$1" ]]; then
    echo "Error: Environment required" >&2
    exit 2
  fi

  if ! ssh "$1" "cd /app && git pull"; then
    echo "Error: Deployment failed" >&2
    exit 1
  fi

  echo "Deployment successful"
  exit 0
}

deploy "$@"
```

### Error handling

```bash
#!/bin/bash
set -euo pipefail

# Trap para cleanup
cleanup() {
  local exit_code=$?
  echo "Cleaning up..."
  rm -f /tmp/deploy.lock
  exit $exit_code
}
trap cleanup EXIT

# Trap para errores
on_error() {
  echo "Error on line $1" >&2
}
trap 'on_error $LINENO' ERR

# Verificar dependencias
check_dependencies() {
  local deps=("git" "ruby" "bundle")
  for dep in "${deps[@]}"; do
    if ! command -v "$dep" &> /dev/null; then
      echo "Error: $dep is required but not installed" >&2
      exit 1
    fi
  done
}

# Función con manejo de errores
safe_deploy() {
  local server=$1

  echo "Deploying to $server..."

  if ! ssh "$server" "cd /app && git pull origin main"; then
    echo "Git pull failed" >&2
    return 1
  fi

  if ! ssh "$server" "cd /app && bundle install"; then
    echo "Bundle install failed" >&2
    return 1
  fi

  return 0
}

# Main
main() {
  check_dependencies

  for server in "${SERVERS[@]}"; do
    if ! safe_deploy "$server"; then
      echo "Deployment to $server failed, aborting" >&2
      exit 1
    fi
  done

  echo "All deployments successful!"
}

main "$@"
```

## Variables de Entorno

### Export y acceso

```bash
#!/bin/bash

# Definir variables de entorno
export RAILS_ENV="production"
export DATABASE_URL="postgres://localhost/myapp"

# Cargar desde archivo
set -a  # Auto-export
source .env
set +a

# O manualmente
while IFS='=' read -r key value; do
  # Ignorar comentarios y líneas vacías
  [[ $key =~ ^#.*$ ]] && continue
  [[ -z "$key" ]] && continue
  export "$key=$value"
done < .env

# Verificar si está definida
if [[ -v DATABASE_URL ]]; then
  echo "DATABASE_URL is set"
fi

# Unset
unset TEMP_VAR
```

### dotenv

```bash
# .env
RAILS_ENV=production
DATABASE_URL=postgres://localhost/myapp
SECRET_KEY_BASE=abc123
REDIS_URL=redis://localhost:6379

# .env.local (no commitear)
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...

# .env.example (commitear como template)
RAILS_ENV=development
DATABASE_URL=
SECRET_KEY_BASE=
```

```bash
#!/bin/bash
# Función para cargar .env
load_env() {
  local env_file=${1:-.env}

  if [[ ! -f "$env_file" ]]; then
    echo "Warning: $env_file not found" >&2
    return 1
  fi

  while IFS='=' read -r key value; do
    # Trim whitespace
    key=$(echo "$key" | xargs)
    value=$(echo "$value" | xargs)

    # Skip comments and empty lines
    [[ -z "$key" || $key == \#* ]] && continue

    # Remove quotes from value
    value="${value%\"}"
    value="${value#\"}"

    export "$key=$value"
  done < "$env_file"
}

load_env ".env"
load_env ".env.local" 2>/dev/null || true
```

## Cron Jobs

### Sintaxis de crontab

```
# ┌───────────── minuto (0 - 59)
# │ ┌───────────── hora (0 - 23)
# │ │ ┌───────────── día del mes (1 - 31)
# │ │ │ ┌───────────── mes (1 - 12)
# │ │ │ │ ┌───────────── día de la semana (0 - 6) (Domingo = 0)
# │ │ │ │ │
# * * * * * comando
```

### Ejemplos comunes

```bash
# crontab -e

# Cada minuto
* * * * * /app/scripts/check_health.sh

# Cada 5 minutos
*/5 * * * * /app/scripts/sync_data.sh

# Cada hora
0 * * * * /app/scripts/hourly_task.sh

# Diario a las 2:00 AM
0 2 * * * /app/scripts/daily_backup.sh

# Lunes a las 9:00 AM
0 9 * * 1 /app/scripts/weekly_report.sh

# Primer día del mes
0 0 1 * * /app/scripts/monthly_cleanup.sh

# Múltiples horarios
0 8,12,18 * * * /app/scripts/three_times_daily.sh

# Rango
0 9-17 * * 1-5 /app/scripts/business_hours.sh
```

### Whenever gem (Ruby)

```ruby
# Gemfile
gem "whenever", require: false

# config/schedule.rb
set :output, "log/cron.log"
set :environment, "production"

# Cada día a las 2am
every 1.day, at: "2:00 am" do
  rake "db:backup"
end

# Cada hora
every 1.hour do
  runner "DataSync.perform"
end

# Cada 5 minutos
every 5.minutes do
  rake "health:check"
end

# Solo en producción
if environment == "production"
  every 1.day, at: "4:00 am" do
    rake "cleanup:old_sessions"
  end
end
```

```bash
# Generar crontab
bundle exec whenever

# Instalar en crontab
bundle exec whenever --update-crontab myapp

# Eliminar de crontab
bundle exec whenever --clear-crontab myapp
```

## Aliases y Funciones (.bashrc/.zshrc)

```bash
# ~/.bashrc o ~/.zshrc

# Aliases para Rails
alias be="bundle exec"
alias rs="bundle exec rails server"
alias rc="bundle exec rails console"
alias rr="bundle exec rails routes"
alias rdbm="bundle exec rails db:migrate"
alias rdbr="bundle exec rails db:rollback"
alias rdbs="bundle exec rails db:seed"
alias rspec="bundle exec rspec"

# Alias para Git
alias gs="git status"
alias gd="git diff"
alias gc="git commit"
alias gp="git push"
alias gl="git pull"
alias gco="git checkout"
alias gb="git branch"
alias glog="git log --oneline --graph --decorate -20"

# Funciones útiles
# Crear y cambiar a directorio
mkcd() {
  mkdir -p "$1" && cd "$1"
}

# Buscar en archivos Ruby
rgr() {
  grep -rn "$1" --include="*.rb" .
}

# Rails: reset database
dbreset() {
  bundle exec rails db:drop db:create db:migrate db:seed
}

# Git: crear branch y pushear
gcb() {
  git checkout -b "$1" && git push -u origin "$1"
}

# Docker: entrar a contenedor
dexec() {
  docker exec -it "$1" /bin/bash
}

# Logs de Rails en tiempo real
railslogs() {
  tail -f log/"${RAILS_ENV:-development}".log
}

# Matar proceso en puerto
killport() {
  lsof -ti:"$1" | xargs kill -9
}
```

## Scripts Útiles para Rails

### Deploy script

```bash
#!/bin/bash
# scripts/deploy.sh

set -euo pipefail

APP_DIR="/var/www/myapp"
BRANCH="${1:-main}"
RAILS_ENV="${RAILS_ENV:-production}"

echo "=== Deploying $BRANCH to $RAILS_ENV ==="

cd "$APP_DIR"

echo "-> Pulling latest code..."
git fetch origin
git checkout "$BRANCH"
git pull origin "$BRANCH"

echo "-> Installing dependencies..."
bundle install --deployment --without development test

echo "-> Compiling assets..."
RAILS_ENV=$RAILS_ENV bundle exec rails assets:precompile

echo "-> Running migrations..."
RAILS_ENV=$RAILS_ENV bundle exec rails db:migrate

echo "-> Restarting application..."
sudo systemctl restart myapp

echo "-> Warming up..."
sleep 5
curl -s http://localhost:3000/health || exit 1

echo "=== Deployment complete! ==="
```

### Backup script

```bash
#!/bin/bash
# scripts/backup.sh

set -euo pipefail

# Configuration
BACKUP_DIR="/backups"
DB_PATH="/app/storage/production.sqlite3"
RETENTION_DAYS=30
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

echo "=== Starting backup ==="

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Database backup
echo "-> Backing up database..."
cp "$DB_PATH" "$BACKUP_DIR/db_$TIMESTAMP.sqlite3"

# Compress
gzip "$BACKUP_DIR/db_$TIMESTAMP.sqlite3"

# Upload backup
echo "-> Uploading to S3..."
aws s3 cp "$BACKUP_DIR/db_$TIMESTAMP.sqlite3.gz" \
  "s3://myapp-backups/database/"

# Cleanup old backups
echo "-> Cleaning old backups..."
find "$BACKUP_DIR" -name "*.gz" -mtime +$RETENTION_DAYS -delete

echo "=== Backup complete: db_$TIMESTAMP.sqlite3.gz ==="
```

### Migrate script

```bash
#!/bin/bash
# scripts/migrate.sh

set -euo pipefail

RAILS_ENV="${RAILS_ENV:-production}"

echo "=== Running migrations for $RAILS_ENV ==="

# Check pending migrations
PENDING=$(RAILS_ENV=$RAILS_ENV bundle exec rails db:migrate:status | grep "down" | wc -l)

if [[ $PENDING -eq 0 ]]; then
  echo "No pending migrations"
  exit 0
fi

echo "Found $PENDING pending migrations"

# Backup before migration
echo "-> Creating backup..."
cp storage/production.sqlite3 "storage/pre_migration_$(date +%Y%m%d_%H%M%S).sqlite3"

# Run migrations
echo "-> Running migrations..."
RAILS_ENV=$RAILS_ENV bundle exec rails db:migrate

# Verify
echo "-> Verifying..."
REMAINING=$(RAILS_ENV=$RAILS_ENV bundle exec rails db:migrate:status | grep "down" | wc -l)

if [[ $REMAINING -gt 0 ]]; then
  echo "ERROR: $REMAINING migrations still pending!"
  exit 1
fi

echo "=== Migrations complete ==="
```

### Health check script

```bash
#!/bin/bash
# scripts/health_check.sh

set -euo pipefail

URL="${1:-http://localhost:3000/up}"
TIMEOUT=10
RETRIES=3

check_health() {
  curl -sf --max-time $TIMEOUT "$URL" > /dev/null
}

for i in $(seq 1 $RETRIES); do
  if check_health; then
    echo "Health check passed"
    exit 0
  fi
  echo "Attempt $i failed, retrying..."
  sleep 2
done

echo "Health check failed after $RETRIES attempts"
exit 1
```

### Console wrapper

```bash
#!/bin/bash
# scripts/console.sh

set -euo pipefail

RAILS_ENV="${1:-production}"

echo "Opening Rails console for $RAILS_ENV"
echo "Type 'exit' to quit"
echo ""

if [[ $RAILS_ENV == "production" ]]; then
  echo "WARNING: You are in PRODUCTION console!"
  echo "Be careful with data modifications."
  echo ""
  read -p "Are you sure? (yes/no): " confirm
  [[ $confirm != "yes" ]] && exit 1
fi

RAILS_ENV=$RAILS_ENV bundle exec rails console
```

## Template de Script

```bash
#!/bin/bash
#
# Script: script_name.sh
# Description: Brief description
# Usage: ./script_name.sh [options] <arguments>
#

set -euo pipefail

# ==============================================================================
# Configuration
# ==============================================================================
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "$0")"

# ==============================================================================
# Functions
# ==============================================================================
usage() {
  cat <<EOF
Usage: $SCRIPT_NAME [options] <argument>

Options:
  -h, --help      Show this help message
  -v, --verbose   Enable verbose output
  -e, --env       Environment (default: development)

Examples:
  $SCRIPT_NAME -e production
  $SCRIPT_NAME --verbose

EOF
}

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

error() {
  echo "[ERROR] $*" >&2
}

cleanup() {
  # Cleanup tasks
  :
}

# ==============================================================================
# Main
# ==============================================================================
main() {
  local verbose=false
  local env="development"

  # Parse arguments
  while [[ $# -gt 0 ]]; do
    case "$1" in
      -h|--help)
        usage
        exit 0
        ;;
      -v|--verbose)
        verbose=true
        shift
        ;;
      -e|--env)
        env="$2"
        shift 2
        ;;
      *)
        error "Unknown option: $1"
        usage
        exit 1
        ;;
    esac
  done

  trap cleanup EXIT

  # Script logic here
  log "Starting script with env=$env verbose=$verbose"
}

main "$@"
```
