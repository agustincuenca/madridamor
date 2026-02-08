# Comando: /backup

## Descripción

Gestiona backups de la base de datos y archivos de la aplicación.

## Uso

```
/backup [acción] [opciones]
```

## Acciones

| Acción | Descripción |
|--------|-------------|
| `crear` | Crea un backup completo (default) |
| `listar` | Lista backups disponibles |
| `restaurar` | Restaura un backup específico |
| `limpiar` | Elimina backups antiguos |

## Proceso

### 1. Backup de SQLite

```bash
#!/bin/bash
# bin/backup

set -e

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="backups"
DB_PATH="storage/production.sqlite3"

mkdir -p $BACKUP_DIR

echo "📦 Creando backup..."

# Método 1: sqlite3 .backup (recomendado - consistente)
sqlite3 $DB_PATH ".backup '$BACKUP_DIR/db_$TIMESTAMP.sqlite3'"

# Comprimir
gzip "$BACKUP_DIR/db_$TIMESTAMP.sqlite3"

echo "✅ Backup creado: $BACKUP_DIR/db_$TIMESTAMP.sqlite3.gz"

# Limpiar backups antiguos (mantener últimos 7 días)
find $BACKUP_DIR -name "*.gz" -mtime +7 -delete

echo "🧹 Backups antiguos eliminados"
```

### 2. Rake tasks completas

```ruby
# lib/tasks/backup.rake
namespace :backup do
  desc "Crear backup de base de datos"
  task db: :environment do
    require "fileutils"

    timestamp = Time.current.strftime("%Y%m%d_%H%M%S")
    backup_dir = Rails.root.join("backups")
    FileUtils.mkdir_p(backup_dir)

    db_config = ActiveRecord::Base.connection_db_config
    db_path = db_config.database

    backup_file = backup_dir.join("db_#{timestamp}.sqlite3")

    puts "📦 Creando backup de base de datos..."

    # Usar sqlite3 .backup para consistencia
    system("sqlite3 #{db_path} '.backup #{backup_file}'")

    # Comprimir
    system("gzip #{backup_file}")

    puts "✅ Backup creado: #{backup_file}.gz"
    puts "   Tamaño: #{File.size("#{backup_file}.gz") / 1024} KB"
  end

  desc "Crear backup de archivos (Active Storage)"
  task files: :environment do
    require "fileutils"

    timestamp = Time.current.strftime("%Y%m%d_%H%M%S")
    backup_dir = Rails.root.join("backups")
    storage_dir = Rails.root.join("storage")

    backup_file = backup_dir.join("files_#{timestamp}.tar.gz")

    puts "📦 Creando backup de archivos..."

    system("tar -czf #{backup_file} -C #{storage_dir} .")

    puts "✅ Backup de archivos creado: #{backup_file}"
    puts "   Tamaño: #{File.size(backup_file) / 1024 / 1024} MB"
  end

  desc "Crear backup completo (BD + archivos)"
  task full: [:db, :files] do
    puts "✅ Backup completo finalizado"
  end

  desc "Listar backups disponibles"
  task list: :environment do
    backup_dir = Rails.root.join("backups")

    unless backup_dir.exist?
      puts "No hay backups disponibles"
      return
    end

    puts "\n📋 Backups disponibles:\n\n"

    Dir.glob(backup_dir.join("*.gz")).sort.reverse.each do |file|
      name = File.basename(file)
      size = File.size(file) / 1024
      date = File.mtime(file).strftime("%Y-%m-%d %H:%M")

      puts "  #{name}"
      puts "    Fecha: #{date}"
      puts "    Tamaño: #{size} KB"
      puts ""
    end
  end

  desc "Restaurar backup de base de datos"
  task :restore, [:filename] => :environment do |t, args|
    unless args[:filename]
      puts "❌ Especifica el archivo: rake backup:restore[db_20240101_120000.sqlite3.gz]"
      exit 1
    end

    backup_file = Rails.root.join("backups", args[:filename])

    unless File.exist?(backup_file)
      puts "❌ Archivo no encontrado: #{backup_file}"
      exit 1
    end

    db_config = ActiveRecord::Base.connection_db_config
    db_path = db_config.database

    puts "⚠️  ADVERTENCIA: Esto reemplazará la base de datos actual"
    puts "   Base de datos: #{db_path}"
    puts "   Backup: #{backup_file}"
    print "\n¿Continuar? (yes/no): "

    confirmation = STDIN.gets.chomp
    unless confirmation == "yes"
      puts "Cancelado"
      exit 0
    end

    puts "\n🔄 Restaurando..."

    # Crear backup del estado actual
    current_backup = "#{db_path}.before_restore"
    FileUtils.cp(db_path, current_backup)
    puts "   Backup actual guardado en: #{current_backup}"

    # Descomprimir si es necesario
    if backup_file.to_s.end_with?(".gz")
      system("gunzip -k #{backup_file}")
      backup_file = backup_file.to_s.gsub(".gz", "")
    end

    # Restaurar
    FileUtils.cp(backup_file, db_path)

    # Limpiar archivo descomprimido temporal
    FileUtils.rm(backup_file) if File.exist?(backup_file) && backup_file != args[:filename]

    puts "✅ Base de datos restaurada"
    puts "\n💡 Si hay problemas, el backup anterior está en: #{current_backup}"
  end

  desc "Limpiar backups antiguos"
  task :clean, [:days] => :environment do |t, args|
    days = (args[:days] || 7).to_i
    backup_dir = Rails.root.join("backups")

    puts "🧹 Eliminando backups de más de #{days} días..."

    count = 0
    Dir.glob(backup_dir.join("*.gz")).each do |file|
      if File.mtime(file) < days.days.ago
        FileUtils.rm(file)
        count += 1
        puts "   Eliminado: #{File.basename(file)}"
      end
    end

    puts "✅ #{count} backups eliminados"
  end
end
```

### 3. Backup automático con cron

```ruby
# config/schedule.rb (con whenever gem)
every 1.day, at: "3:00 am" do
  rake "backup:full"
end

every :sunday, at: "4:00 am" do
  rake "backup:clean[30]"
end
```

### 4. Backup a S3/R2

```ruby
# lib/tasks/backup.rake
namespace :backup do
  desc "Subir backup a S3"
  task upload: :environment do
    require "aws-sdk-s3"

    backup_dir = Rails.root.join("backups")
    latest_backup = Dir.glob(backup_dir.join("db_*.gz")).max_by { |f| File.mtime(f) }

    unless latest_backup
      puts "❌ No hay backups para subir"
      exit 1
    end

    s3 = Aws::S3::Resource.new(
      region: Rails.application.credentials.dig(:aws, :region),
      access_key_id: Rails.application.credentials.dig(:aws, :access_key_id),
      secret_access_key: Rails.application.credentials.dig(:aws, :secret_access_key)
    )

    bucket = s3.bucket(Rails.application.credentials.dig(:aws, :backup_bucket))
    filename = File.basename(latest_backup)

    puts "📤 Subiendo #{filename} a S3..."

    obj = bucket.object("backups/#{filename}")
    obj.upload_file(latest_backup)

    puts "✅ Backup subido exitosamente"
  end

  desc "Backup completo con upload a S3"
  task remote: [:full, :upload] do
    puts "✅ Backup remoto completado"
  end
end
```

### 5. Backup en Fly.io

```bash
#!/bin/bash
# bin/backup-fly

# Crear backup en la máquina
fly ssh console -C "bin/rails backup:db"

# Descargar backup
fly ssh sftp get /rails/backups/$(fly ssh console -C "ls -t /rails/backups | head -1")
```

### 6. Modelo de auditoría de backups

```ruby
# app/models/backup_log.rb
class BackupLog < ApplicationRecord
  enum :status, { started: 0, completed: 1, failed: 2 }
  enum :backup_type, { database: 0, files: 1, full: 2 }

  validates :backup_type, presence: true

  def self.log_backup(type:, &block)
    log = create!(backup_type: type, status: :started, started_at: Time.current)

    begin
      result = yield
      log.update!(
        status: :completed,
        completed_at: Time.current,
        file_path: result[:file_path],
        file_size: result[:file_size]
      )
    rescue => e
      log.update!(
        status: :failed,
        completed_at: Time.current,
        error_message: e.message
      )
      raise
    end

    log
  end
end

# Uso en rake task
BackupLog.log_backup(type: :database) do
  # ... crear backup ...
  { file_path: backup_file, file_size: File.size(backup_file) }
end
```

## Ejemplo de salida

### /backup crear

```
📦 Iniciando backup completo...

🗃️ Base de datos:
   → Archivo: db_20240115_143022.sqlite3.gz
   → Tamaño: 2.3 MB
   → ✅ Completado

📁 Archivos (Active Storage):
   → Archivo: files_20240115_143022.tar.gz
   → Tamaño: 156 MB
   → ✅ Completado

✅ Backup completo finalizado

📍 Ubicación: /backups/
   - db_20240115_143022.sqlite3.gz
   - files_20240115_143022.tar.gz
```

### /backup listar

```
📋 Backups disponibles:

  db_20240115_143022.sqlite3.gz
    Fecha: 2024-01-15 14:30
    Tamaño: 2.3 MB

  db_20240114_030000.sqlite3.gz
    Fecha: 2024-01-14 03:00
    Tamaño: 2.1 MB

  db_20240113_030000.sqlite3.gz
    Fecha: 2024-01-13 03:00
    Tamaño: 2.0 MB

Total: 3 backups (6.4 MB)
```

### /backup restaurar

```
⚠️  RESTAURAR BACKUP

Esto reemplazará la base de datos actual con:
  db_20240114_030000.sqlite3.gz (2024-01-14 03:00)

¿Estás seguro? (sí/no): sí

🔄 Restaurando...
   → Backup actual guardado en: production.sqlite3.before_restore
   → Descomprimiendo backup...
   → Reemplazando base de datos...

✅ Base de datos restaurada exitosamente

💡 Si hay problemas, ejecuta:
   cp storage/production.sqlite3.before_restore storage/production.sqlite3
```

## Respuesta del agente

Al ejecutar `/backup`, el agente:

1. **Determina acción**
   - Si no se especifica, asume "crear"
   - Parsea opciones adicionales

2. **Ejecuta backup**
   - Verifica espacio disponible
   - Crea backup consistente de SQLite
   - Comprime archivos
   - Opcionalmente sube a storage remoto

3. **Registra resultado**
   - Guarda log de backup
   - Muestra ubicación y tamaño
   - Limpia backups antiguos si está configurado

4. **Maneja restauración**
   - Pide confirmación explícita
   - Crea backup del estado actual antes de restaurar
   - Proporciona instrucciones de rollback
