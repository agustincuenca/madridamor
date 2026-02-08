# Skill: Debugging

## Purpose

Técnicas y herramientas de depuración para encontrar y solucionar bugs en aplicaciones Ruby on Rails de forma eficiente.

## Herramientas de Debugging

### Debug Gem (Rails 7+)

```ruby
# Gemfile (incluido por defecto en Rails 7+)
group :development, :test do
  gem "debug", platforms: %i[mri mingw x64_mingw]
end
```

```ruby
# En cualquier parte del código
def process_order(order)
  debugger  # Pausa ejecución aquí

  total = calculate_total(order)
  # ...
end
```

### Byebug

```ruby
# Gemfile
group :development, :test do
  gem "byebug"
end
```

```ruby
# Uso
def problematic_method
  byebug  # Punto de parada

  result = complex_calculation
  result
end
```

### Pry

```ruby
# Gemfile
group :development, :test do
  gem "pry-rails"
  gem "pry-byebug"  # Combina pry con debugging
end
```

```ruby
# Uso
def analyze_data(data)
  binding.pry  # Abre consola interactiva

  processed = data.map { |d| transform(d) }
  processed
end
```

### Binding.irb (Sin gems adicionales)

```ruby
# Disponible en Ruby 2.5+
def investigate_bug
  binding.irb  # Abre IRB en este punto

  # código a investigar
end
```

## Comandos del Debugger

### Navegación

| Comando | Alias | Descripción |
|---------|-------|-------------|
| `step` | `s` | Entrar en el método |
| `next` | `n` | Siguiente línea (sin entrar) |
| `finish` | `fin` | Salir del método actual |
| `continue` | `c` | Continuar ejecución |
| `up` | | Subir en el stack frame |
| `down` | | Bajar en el stack frame |

### Inspección

| Comando | Descripción |
|---------|-------------|
| `where` / `bt` | Ver backtrace completo |
| `list` / `l` | Ver código alrededor |
| `info locals` | Ver variables locales |
| `info args` | Ver argumentos del método |
| `p expression` | Evaluar expresión |
| `pp object` | Pretty print de objeto |

### Breakpoints

```ruby
# En debug gem
break app/models/user.rb:25        # Línea específica
break User#validate                 # Método específico
break if user.admin?               # Breakpoint condicional
delete 1                           # Eliminar breakpoint 1
info breakpoints                   # Listar breakpoints

# En byebug
break 25                           # Línea 25 del archivo actual
break User#save                    # Método save de User
condition 1 user.email.nil?        # Condición en breakpoint 1
```

### Ejemplo de sesión de debugging

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  def calculate_total
    debugger

    subtotal = line_items.sum(&:total)
    tax = subtotal * tax_rate
    shipping = calculate_shipping

    subtotal + tax + shipping
  end
end

# En el debugger:
(rdbg) p subtotal           # => 100.0
(rdbg) p tax_rate           # => 0.21
(rdbg) p tax                # => 21.0
(rdbg) n                    # Siguiente línea
(rdbg) p shipping           # => nil  # Aquí está el bug!
(rdbg) s                    # Entrar en calculate_shipping
(rdbg) where                # Ver stack trace
(rdbg) c                    # Continuar
```

## Rails Console

### Comandos útiles

```ruby
# Iniciar consola
rails console
rails c

# Consola en sandbox (rollback al salir)
rails console --sandbox

# Consola en producción (cuidado!)
RAILS_ENV=production rails console

# Recargar código modificado
reload!

# Acceder al último valor retornado
_

# Ver SQL generado
ActiveRecord::Base.logger = Logger.new(STDOUT)

# O para una query específica
User.where(active: true).to_sql
```

### Helper y app

```ruby
# Acceder a helpers de vistas
helper.number_to_currency(1234.50)
helper.time_ago_in_words(3.days.ago)
helper.link_to("Home", "/")

# Hacer requests HTTP
app.get "/"
app.response.status  # => 200
app.response.body    # HTML

app.post "/users", params: { user: { name: "Test" } }

# Acceder a rutas
app.users_path
app.root_url
```

### Inspección de objetos

```ruby
# Ver métodos de un objeto
User.new.methods.sort
User.new.methods.grep(/name/)

# Ver source de un método
User.instance_method(:full_name).source_location
User.method(:find).source_location

# Con pry
$ User#full_name    # Ver source
? User#full_name    # Ver documentación

# Ver ancestros
User.ancestors

# Ver asociaciones
User.reflect_on_all_associations.map(&:name)

# Ver columnas
User.column_names
User.columns_hash["email"]
```

## Logging

### Niveles de log

```ruby
# config/environments/development.rb
config.log_level = :debug  # :debug, :info, :warn, :error, :fatal

# En código
Rails.logger.debug "Valor de x: #{x}"
Rails.logger.info "Usuario creado: #{user.id}"
Rails.logger.warn "Parámetro deprecado usado"
Rails.logger.error "Fallo en payment: #{error.message}"
Rails.logger.fatal "Sistema no puede continuar"
```

### Tagged Logging

```ruby
# config/application.rb
config.log_tags = [:request_id, :remote_ip]

# Uso manual
Rails.logger.tagged("OrderProcessor", "User:#{user.id}") do
  Rails.logger.info "Procesando orden"
  # [OrderProcessor] [User:123] Procesando orden
end
```

### Custom Logger

```ruby
# lib/debug_logger.rb
class DebugLogger
  def self.log(context, data = {})
    return unless Rails.env.development?

    message = {
      timestamp: Time.current.iso8601,
      context: context,
      caller: caller[0],
      **data
    }

    Rails.logger.debug message.to_json
  end
end

# Uso
DebugLogger.log("PaymentService", amount: 100, user_id: user.id)
```

### Silenciar logs temporalmente

```ruby
# Silenciar ActiveRecord
ActiveRecord::Base.logger.silence do
  # Queries aquí no se loguean
  User.find_each { |u| process(u) }
end

# Log a archivo específico
debug_log = Logger.new(Rails.root.join("log/debug.log"))
debug_log.info "Investigando bug..."
```

## Stack Traces

### Leer un stack trace

```
NoMethodError: undefined method 'total' for nil:NilClass

app/models/order.rb:25:in `calculate_total'
app/services/checkout_service.rb:15:in `process'
app/controllers/orders_controller.rb:10:in `create'
```

**Leer de abajo hacia arriba:**
1. El error empezó en `orders_controller.rb:10`
2. Llamó a `checkout_service.rb:15`
3. Que llamó a `order.rb:25` donde ocurrió el error

### Filtrar stack trace

```ruby
# config/initializers/backtrace_silencers.rb
Rails.backtrace_cleaner.add_silencer { |line| line.include?("/gems/") }

# Ver backtrace completo
Rails.backtrace_cleaner.remove_silencers!

# En una excepción
begin
  risky_operation
rescue => e
  puts e.message
  puts e.backtrace.first(10).join("\n")

  # Con Rails cleaner
  puts Rails.backtrace_cleaner.clean(e.backtrace).join("\n")
end
```

### Capturar contexto

```ruby
# Guardar contexto cuando ocurre un error
class ApplicationController < ActionController::Base
  rescue_from StandardError do |exception|
    context = {
      user_id: current_user&.id,
      params: params.to_unsafe_h,
      session: session.to_h,
      url: request.url,
      method: request.method
    }

    Rails.logger.error "Error: #{exception.message}"
    Rails.logger.error "Context: #{context.to_json}"
    Rails.logger.error exception.backtrace.first(20).join("\n")

    raise exception
  end
end
```

## Estrategias de Debugging

### Divide y vencerás

```ruby
# Bug: el resultado es incorrecto
def complex_calculation(data)
  # Dividir en pasos y verificar cada uno
  step1 = transform_data(data)
  Rails.logger.debug "After step1: #{step1.inspect}"

  step2 = filter_data(step1)
  Rails.logger.debug "After step2: #{step2.inspect}"

  step3 = aggregate_data(step2)
  Rails.logger.debug "After step3: #{step3.inspect}"

  step3
end
```

### Rubber Duck Debugging

```ruby
# Explicar el código línea por línea:
def find_discount(user, cart)
  # 1. Obtengo todos los cupones activos del usuario
  coupons = user.coupons.active

  # 2. Filtro los que aplican a los productos del carrito
  applicable = coupons.select { |c| c.applies_to?(cart) }

  # 3. Ordeno por descuento y tomo el mejor
  # ESPERA... ¿qué pasa si applicable está vacío?
  best = applicable.max_by(&:discount_percent)

  # 4. Retorno el descuento
  best.discount_percent  # NoMethodError si best es nil!
end

# Fix:
best&.discount_percent || 0
```

### Binary Search (para bugs de regresión)

```ruby
# Si algo funcionaba antes y ahora no:

# 1. Usar git bisect
# git bisect start
# git bisect bad HEAD
# git bisect good v1.0.0
# git bisect run bundle exec rspec spec/models/order_spec.rb

# 2. Manual: comentar mitad del código
def process_order(order)
  validate_order(order)
  # calculate_taxes(order)
  # apply_discounts(order)
  # process_payment(order)
  # send_confirmation(order)
end
# Si funciona, el bug está en la mitad comentada
# Seguir dividiendo hasta encontrarlo
```

### Añadir assertions temporales

```ruby
def transfer_money(from, to, amount)
  # Assertions para encontrar estado inesperado
  raise "from is nil!" if from.nil?
  raise "to is nil!" if to.nil?
  raise "amount must be positive: #{amount}" unless amount.positive?
  raise "insufficient funds: #{from.balance} < #{amount}" if from.balance < amount

  from.balance -= amount
  to.balance += amount

  # Verificar invariantes
  raise "from balance negative!" if from.balance.negative?
end
```

## Profiling

### Rack Mini Profiler

```ruby
# Gemfile
group :development do
  gem "rack-mini-profiler"
  gem "memory_profiler"
  gem "stackprof"
end
```

```ruby
# En cualquier request, aparece badge con timing
# Añadir ?pp=flamegraph para flamegraph
# Añadir ?pp=analyze-memory para memoria
```

### Memory Profiler

```ruby
# En consola o código
require "memory_profiler"

report = MemoryProfiler.report do
  # Código a analizar
  1000.times { User.all.to_a }
end

report.pretty_print(to_file: "tmp/memory_report.txt")

# Métricas clave:
# - Total allocated: memoria usada
# - Total retained: memoria no liberada (posible leak)
```

### StackProf (CPU profiling)

```ruby
require "stackprof"

# Profile bloque de código
StackProf.run(mode: :cpu, out: "tmp/stackprof.dump") do
  # Código lento
  heavy_computation
end

# Analizar resultados
# stackprof tmp/stackprof.dump --text
# stackprof tmp/stackprof.dump --method 'Object#slow_method'

# Profile request completo
# En config/initializers/stackprof.rb
if Rails.env.development?
  require "stackprof"
  use StackProf::Middleware, enabled: true,
                             mode: :cpu,
                             interval: 1000,
                             save_every: 5
end
```

### Benchmark

```ruby
require "benchmark"

# Medir tiempo
time = Benchmark.measure do
  User.all.each { |u| process(u) }
end
puts time  # 0.234567

# Comparar alternativas
Benchmark.bm(20) do |x|
  x.report("each:") { users.each { |u| process(u) } }
  x.report("find_each:") { User.find_each { |u| process(u) } }
  x.report("in_batches:") { User.in_batches { |batch| batch.each { |u| process(u) } } }
end

# Benchmark/ips para comparaciones más precisas
require "benchmark/ips"

Benchmark.ips do |x|
  x.report("string +") { "hello" + " " + "world" }
  x.report("string interpolation") { "hello #{'world'}" }
  x.compare!
end
```

## N+1 Queries

### Bullet Gem

```ruby
# Gemfile
group :development do
  gem "bullet"
end

# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.alert = true            # Popup en browser
  Bullet.bullet_logger = true    # Log a bullet.log
  Bullet.console = true          # Console.log
  Bullet.rails_logger = true     # Rails logger
  Bullet.add_footer = true       # Footer en página
end
```

### Detectar manualmente

```ruby
# Activar logging de SQL
ActiveRecord::Base.logger = Logger.new(STDOUT)

# En consola
User.all.each { |u| puts u.posts.count }
# Verás N+1 queries: 1 para users + N para posts

# Usar explain
User.includes(:posts).explain
# Muestra plan de ejecución
```

### Solucionar N+1

```ruby
# ANTES (N+1)
@users = User.all
# En vista: user.posts.count genera query por usuario

# DESPUÉS (eager loading)
@users = User.includes(:posts)
# o
@users = User.preload(:posts)
# o para joins
@users = User.eager_load(:posts)

# Diferencias:
# includes: Rails decide si usar JOIN o queries separadas
# preload: Siempre queries separadas
# eager_load: Siempre LEFT OUTER JOIN

# Counter cache para counts
class Post < ApplicationRecord
  belongs_to :user, counter_cache: true
end
# Requiere columna posts_count en users
```

## Error Tracking

### Sentry

```ruby
# Gemfile
gem "sentry-ruby"
gem "sentry-rails"

# config/initializers/sentry.rb
Sentry.init do |config|
  config.dsn = Rails.application.credentials.sentry_dsn
  config.environment = Rails.env
  config.breadcrumbs_logger = [:active_support_logger, :http_logger]

  # Sample rate (0.0 a 1.0)
  config.traces_sample_rate = 0.5

  # Filtrar datos sensibles
  config.before_send = lambda do |event, hint|
    event.extra.delete(:password)
    event
  end
end

# Capturar errores manualmente
begin
  risky_operation
rescue => e
  Sentry.capture_exception(e, extra: { user_id: current_user.id })
  raise
end

# Añadir contexto
Sentry.set_user(id: user.id, email: user.email)
Sentry.set_tags(feature: "checkout")
Sentry.set_context("order", { id: order.id, total: order.total })
```

### Honeybadger

```ruby
# Gemfile
gem "honeybadger"

# config/honeybadger.yml
api_key: <%= Rails.application.credentials.honeybadger_api_key %>
env: <%= Rails.env %>

# Uso
Honeybadger.notify(exception, context: { user_id: user.id })
```

### Rollbar

```ruby
# Gemfile
gem "rollbar"

# config/initializers/rollbar.rb
Rollbar.configure do |config|
  config.access_token = Rails.application.credentials.rollbar_token
  config.environment = Rails.env
end

# Uso
Rollbar.error(exception, user_id: user.id)
```

## Reproducir Bugs

### Seeds para estado específico

```ruby
# db/seeds/bug_reproduction.rb
# Crear datos que reproducen el bug

user = User.create!(
  email: "test@example.com",
  password: "password123"
)

# Estado específico que causa el bug
order = Order.create!(
  user: user,
  status: "pending",
  total: 0  # Edge case que causa división por cero
)

puts "Bug reproduction data created"
puts "User ID: #{user.id}"
puts "Order ID: #{order.id}"
```

### Fixtures de test

```yaml
# spec/fixtures/users.yml
bug_user:
  email: user_with_bug@example.com
  name: Bug User

# spec/fixtures/orders.yml
problematic_order:
  user: bug_user
  status: pending
  total: 0
```

### Script de reproducción

```ruby
#!/usr/bin/env ruby
# scripts/reproduce_bug_123.rb

require_relative "../config/environment"

puts "Reproduciendo Bug #123..."

# 1. Crear estado inicial
user = User.create!(email: "test#{Time.now.to_i}@test.com", password: "password")
order = user.orders.create!(status: "pending")

# 2. Ejecutar acción que causa el bug
begin
  CheckoutService.new(order).process
rescue => e
  puts "Bug reproducido!"
  puts "Error: #{e.message}"
  puts e.backtrace.first(5).join("\n")
end

# 3. Limpiar
user.destroy
```

### Pasos claros

```markdown
## Bug #123: Error al procesar orden vacía

### Pasos para reproducir:
1. Crear usuario: `user = User.create!(email: "test@test.com", password: "pass")`
2. Crear orden sin items: `order = user.orders.create!()`
3. Intentar checkout: `CheckoutService.new(order).process`

### Resultado esperado:
Error claro indicando que la orden está vacía

### Resultado actual:
`ZeroDivisionError: divided by 0`

### Causa raíz:
En `order.rb:45`, se divide total entre número de items sin verificar que no sea cero.

### Fix:
```ruby
def average_item_price
  return 0 if line_items.empty?
  total / line_items.count
end
```
```

## Debugging en Producción

### Logs estructurados

```ruby
# config/environments/production.rb
config.log_formatter = proc do |severity, time, progname, msg|
  {
    timestamp: time.iso8601,
    level: severity,
    message: msg,
    app: "myapp",
    env: Rails.env
  }.to_json + "\n"
end
```

### Investigar sin reproducir

```ruby
# Añadir logging temporal
class OrderProcessor
  def process(order)
    Rails.logger.info({
      event: "order_processing_start",
      order_id: order.id,
      user_id: order.user_id,
      items_count: order.line_items.count,
      total: order.total
    }.to_json)

    result = do_processing(order)

    Rails.logger.info({
      event: "order_processing_complete",
      order_id: order.id,
      result: result
    }.to_json)

    result
  rescue => e
    Rails.logger.error({
      event: "order_processing_error",
      order_id: order.id,
      error: e.message,
      backtrace: e.backtrace.first(10)
    }.to_json)

    raise
  end
end
```

### Console en producción (con cuidado)

```bash
# Conectar a producción
RAILS_ENV=production rails console

# SIEMPRE usar transacciones para investigar
ActiveRecord::Base.transaction do
  # Investigar...
  order = Order.find(123)
  order.calculate_total

  raise ActiveRecord::Rollback  # No guardar cambios
end

# Solo lectura
Order.find(123).attributes
User.where(created_at: 1.day.ago..).count
```

## Checklist de Debugging

### Cuando encuentras un bug:

1. [ ] Reproducir el bug de forma consistente
2. [ ] Aislar el problema (quitar código hasta que desaparezca)
3. [ ] Añadir logging/breakpoints en puntos clave
4. [ ] Verificar entrada vs salida esperada
5. [ ] Revisar código reciente (git log, git blame)
6. [ ] Buscar patrones similares en el código
7. [ ] Escribir test que falle
8. [ ] Aplicar fix
9. [ ] Verificar que el test pase
10. [ ] Verificar que no haya regresiones
