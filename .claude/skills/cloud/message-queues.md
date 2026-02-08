# Skill: Message Queues y Sistemas de Mensajeria

## Purpose

Implementar sistemas de colas y mensajeria para procesamiento asincrono, desacoplamiento de servicios y arquitecturas event-driven en aplicaciones Rails.

## Conceptos Fundamentales

### Terminologia

```markdown
## Componentes principales

| Concepto | Descripcion |
|----------|-------------|
| Producer | Servicio que envia mensajes a la cola |
| Consumer | Servicio que procesa mensajes de la cola |
| Queue | Cola FIFO donde se almacenan mensajes |
| Topic | Canal para distribuir mensajes a multiples suscriptores |
| Exchange | Router que dirige mensajes a las colas (RabbitMQ) |
| Partition | Division de un topic para paralelismo (Kafka) |
| Offset | Posicion del consumidor en una particion |
| Acknowledgment | Confirmacion de que un mensaje fue procesado |
| Dead Letter Queue | Cola para mensajes que fallaron |

## Patrones de mensajeria

| Patron | Descripcion |
|--------|-------------|
| Point-to-Point | Un mensaje es procesado por un solo consumidor |
| Pub/Sub | Un mensaje es distribuido a multiples suscriptores |
| Request-Reply | Comunicacion sincrona sobre mensajeria |
| Fan-out | Un mensaje genera multiples mensajes derivados |
| Saga | Transacciones distribuidas con compensacion |
```

## Solid Queue (Rails 8)

### Configuracion basica

```ruby
# Gemfile (incluido en Rails 8)
gem "solid_queue"

# Instalar
bin/rails solid_queue:install
bin/rails db:migrate

# config/application.rb
config.active_job.queue_adapter = :solid_queue
```

### Configuracion de workers

```yaml
# config/queue.yml
default: &default
  dispatchers:
    - polling_interval: 1
      batch_size: 500
  workers:
    - queues: "*"
      threads: 3
      processes: 1
      polling_interval: 0.1

development:
  <<: *default

production:
  <<: *default
  dispatchers:
    - polling_interval: 0.5
      batch_size: 1000
  workers:
    # Worker para tareas criticas
    - queues: [critical]
      threads: 5
      processes: 2
      polling_interval: 0.1

    # Worker para tareas normales
    - queues: [default, mailers]
      threads: 5
      processes: 2
      polling_interval: 0.5

    # Worker para tareas de baja prioridad
    - queues: [low, reports]
      threads: 3
      processes: 1
      polling_interval: 2
```

### Jobs con prioridad

```ruby
# app/jobs/critical_payment_job.rb
class CriticalPaymentJob < ApplicationJob
  queue_as :critical

  retry_on PaymentGatewayError, wait: 5.seconds, attempts: 5
  discard_on PaymentDeclinedError

  def perform(payment_id)
    payment = Payment.find(payment_id)
    PaymentProcessor.new(payment).process!
  end
end

# app/jobs/send_report_job.rb
class SendReportJob < ApplicationJob
  queue_as :low

  def perform(user_id, report_type)
    user = User.find(user_id)
    report = ReportGenerator.new(user, report_type).generate
    ReportMailer.send_report(user, report).deliver_now
  end
end
```

### Recurring jobs

```yaml
# config/recurring.yml
production:
  cleanup_sessions:
    class: CleanupSessionsJob
    schedule: every day at 3am

  daily_digest:
    class: DailyDigestJob
    schedule: every day at 8am

  sync_inventory:
    class: SyncInventoryJob
    schedule: every 15 minutes

  generate_reports:
    class: GenerateReportsJob
    schedule: every monday at 6am
    args: [weekly]
```

## Sidekiq

### Configuracion

```ruby
# Gemfile
gem "sidekiq"
gem "sidekiq-scheduler"  # Para jobs recurrentes

# config/application.rb
config.active_job.queue_adapter = :sidekiq

# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = {
    url: ENV.fetch("REDIS_URL") { "redis://localhost:6379/0" },
    pool_timeout: 5
  }

  # Cargar scheduler
  config.on(:startup) do
    Sidekiq.schedule = YAML.load_file(Rails.root.join("config/sidekiq_schedule.yml"))
    Sidekiq::Scheduler.reload_schedule!
  end
end

Sidekiq.configure_client do |config|
  config.redis = {
    url: ENV.fetch("REDIS_URL") { "redis://localhost:6379/0" },
    pool_timeout: 5
  }
end
```

```yaml
# config/sidekiq.yml
:concurrency: 10
:queues:
  - [critical, 3]
  - [default, 2]
  - [low, 1]

production:
  :concurrency: 25
```

### Workers nativos de Sidekiq

```ruby
# app/workers/heavy_processing_worker.rb
class HeavyProcessingWorker
  include Sidekiq::Worker

  sidekiq_options queue: :low,
                  retry: 5,
                  backtrace: true,
                  dead: true

  sidekiq_retry_in do |count, exception|
    case exception
    when RateLimitError
      60 * (count + 1)  # 1min, 2min, 3min...
    else
      (count ** 4) + 15  # Exponential backoff
    end
  end

  def perform(record_id, options = {})
    record = Record.find(record_id)
    # Procesamiento pesado...
  end
end

# Encolar
HeavyProcessingWorker.perform_async(record.id)
HeavyProcessingWorker.perform_in(1.hour, record.id)
HeavyProcessingWorker.perform_at(Date.tomorrow.noon, record.id)
```

### Batches (Sidekiq Pro)

```ruby
# Procesar multiples items como batch
batch = Sidekiq::Batch.new
batch.description = "Import users from CSV"
batch.on(:complete, ImportCallbacks, file_id: file.id)
batch.on(:success, ImportCallbacks)

batch.jobs do
  users_data.each do |user_data|
    ImportUserWorker.perform_async(user_data)
  end
end

# Callbacks
class ImportCallbacks
  def on_complete(status, options)
    file = ImportFile.find(options["file_id"])
    file.update!(status: "completed", processed: status.total)
  end

  def on_success(status, options)
    AdminNotifier.import_success(options["file_id"]).deliver_later
  end
end
```

## RabbitMQ

### Configuracion con Bunny

```ruby
# Gemfile
gem "bunny"
gem "sneakers"  # Para workers

# config/initializers/rabbitmq.rb
require "bunny"

RABBITMQ = Bunny.new(
  host: ENV.fetch("RABBITMQ_HOST") { "localhost" },
  port: ENV.fetch("RABBITMQ_PORT") { 5672 },
  user: ENV.fetch("RABBITMQ_USER") { "guest" },
  password: ENV.fetch("RABBITMQ_PASSWORD") { "guest" },
  vhost: ENV.fetch("RABBITMQ_VHOST") { "/" },
  automatically_recover: true,
  recovery_attempts: 10
)
RABBITMQ.start

# Crear canal para publicar
RABBITMQ_CHANNEL = RABBITMQ.create_channel
```

### Exchanges y Queues

```ruby
# app/services/rabbitmq_publisher.rb
class RabbitmqPublisher
  def initialize
    @channel = RABBITMQ.create_channel
  end

  # Direct exchange - mensajes a una cola especifica
  def publish_direct(queue_name, message, options = {})
    queue = @channel.queue(queue_name, durable: true)
    @channel.default_exchange.publish(
      message.to_json,
      routing_key: queue_name,
      persistent: true,
      content_type: "application/json",
      **options
    )
  end

  # Fanout exchange - mensajes a todas las colas conectadas
  def publish_fanout(exchange_name, message)
    exchange = @channel.fanout(exchange_name, durable: true)
    exchange.publish(
      message.to_json,
      persistent: true,
      content_type: "application/json"
    )
  end

  # Topic exchange - mensajes basados en routing key pattern
  def publish_topic(exchange_name, routing_key, message)
    exchange = @channel.topic(exchange_name, durable: true)
    exchange.publish(
      message.to_json,
      routing_key: routing_key,
      persistent: true,
      content_type: "application/json"
    )
  end

  # Headers exchange - mensajes basados en headers
  def publish_headers(exchange_name, headers, message)
    exchange = @channel.headers(exchange_name, durable: true)
    exchange.publish(
      message.to_json,
      headers: headers,
      persistent: true,
      content_type: "application/json"
    )
  end
end

# Uso
publisher = RabbitmqPublisher.new
publisher.publish_topic("orders", "order.created.premium", { order_id: 123 })
```

### Workers con Sneakers

```ruby
# config/initializers/sneakers.rb
Sneakers.configure(
  connection: RABBITMQ,
  workers: 4,
  threads: 10,
  prefetch: 10,
  timeout_job_after: 60,
  ack: true,
  heartbeat: 30,
  hooks: {
    before_fork: -> { ActiveRecord::Base.connection_pool.disconnect! },
    after_fork: -> { ActiveRecord::Base.establish_connection }
  }
)

# app/workers/order_worker.rb
class OrderWorker
  include Sneakers::Worker

  from_queue "orders",
    exchange: "orders_exchange",
    exchange_type: :topic,
    routing_key: "order.#",
    durable: true

  def work(raw_message)
    message = JSON.parse(raw_message)

    case message["event"]
    when "order.created"
      handle_order_created(message["data"])
    when "order.paid"
      handle_order_paid(message["data"])
    end

    ack!
  rescue StandardError => e
    Rails.logger.error "OrderWorker error: #{e.message}"
    reject!  # Requeue message
  end

  private

  def handle_order_created(data)
    order = Order.find(data["order_id"])
    OrderConfirmationMailer.notify(order).deliver_now
  end

  def handle_order_paid(data)
    order = Order.find(data["order_id"])
    FulfillmentService.new(order).process
  end
end
```

## Apache Kafka

### Configuracion

```ruby
# Gemfile
gem "ruby-kafka"
# O para Rails
gem "karafka"

# config/initializers/kafka.rb
require "kafka"

KAFKA = Kafka.new(
  seed_brokers: ENV.fetch("KAFKA_BROKERS") { "localhost:9092" }.split(","),
  client_id: "rails-app",
  logger: Rails.logger,
  ssl_ca_cert: ENV["KAFKA_SSL_CA_CERT"],
  ssl_client_cert: ENV["KAFKA_SSL_CLIENT_CERT"],
  ssl_client_cert_key: ENV["KAFKA_SSL_CLIENT_KEY"]
)
```

### Producer

```ruby
# app/services/kafka_producer.rb
class KafkaProducer
  def initialize
    @producer = KAFKA.producer(
      ack_timeout: 5,
      required_acks: :all,
      max_retries: 3,
      retry_backoff: 1
    )
  end

  def publish(topic, message, key: nil, partition_key: nil, headers: {})
    @producer.produce(
      message.to_json,
      topic: topic,
      key: key,
      partition_key: partition_key,
      headers: headers.merge(
        "produced_at" => Time.current.iso8601,
        "producer" => "rails-app"
      )
    )
  end

  def deliver
    @producer.deliver_messages
  rescue Kafka::DeliveryFailed => e
    Rails.logger.error "Kafka delivery failed: #{e.message}"
    raise
  end

  def publish_sync(topic, message, **options)
    publish(topic, message, **options)
    deliver
  end

  def shutdown
    @producer.shutdown
  end
end

# Uso
producer = KafkaProducer.new
producer.publish("user-events", { event: "user.created", user_id: user.id })
producer.publish("user-events", { event: "user.updated", user_id: user.id })
producer.deliver  # Enviar batch
```

### Consumer

```ruby
# app/consumers/kafka_consumer.rb
class KafkaConsumer
  def initialize(topics, group_id: "rails-app-consumer")
    @consumer = KAFKA.consumer(
      group_id: group_id,
      offset_commit_interval: 10,
      offset_commit_threshold: 100,
      heartbeat_interval: 10
    )

    topics.each do |topic|
      @consumer.subscribe(topic, start_from_beginning: false)
    end
  end

  def consume
    @consumer.each_message do |message|
      process_message(message)
    rescue StandardError => e
      handle_error(e, message)
    end
  end

  def consume_batch(batch_size: 100)
    @consumer.each_batch(max_wait_time: 5) do |batch|
      batch.messages.each do |message|
        process_message(message)
      end
    end
  end

  private

  def process_message(message)
    payload = JSON.parse(message.value)

    Rails.logger.info "Processing: topic=#{message.topic} " \
                      "partition=#{message.partition} " \
                      "offset=#{message.offset}"

    handler = find_handler(message.topic, payload["event"])
    handler.call(payload)
  end

  def find_handler(topic, event)
    handlers = {
      "user-events" => {
        "user.created" => ->(data) { UserCreatedHandler.new(data).call },
        "user.updated" => ->(data) { UserUpdatedHandler.new(data).call }
      },
      "order-events" => {
        "order.placed" => ->(data) { OrderPlacedHandler.new(data).call }
      }
    }

    handlers.dig(topic, event) || ->(data) { Rails.logger.warn "Unknown event: #{event}" }
  end

  def handle_error(error, message)
    Rails.logger.error "Consumer error: #{error.message}"
    Rails.logger.error "Failed message: #{message.value}"

    # Enviar a dead letter topic
    dead_letter_producer = KafkaProducer.new
    dead_letter_producer.publish_sync(
      "#{message.topic}-dlq",
      {
        original_message: message.value,
        error: error.message,
        failed_at: Time.current.iso8601
      }
    )
  end
end

# Iniciar consumer (en proceso separado)
# bin/kafka_consumer
consumer = KafkaConsumer.new(["user-events", "order-events"])
consumer.consume
```

### Karafka (Framework para Kafka)

```ruby
# Gemfile
gem "karafka"

# karafka.rb
class KarafkaApp < Karafka::App
  setup do |config|
    config.kafka = {
      "bootstrap.servers": ENV.fetch("KAFKA_BROKERS") { "localhost:9092" }
    }
    config.client_id = "rails-app"
    config.consumer_persistence = true
  end

  routes.draw do
    topic :user_events do
      consumer UserEventsConsumer
    end

    topic :order_events do
      consumer OrderEventsConsumer
      dead_letter_queue(topic: :order_events_dlq, max_retries: 3)
    end

    topic :notifications do
      consumer NotificationsConsumer
      batch true
      max_messages 100
    end
  end
end

# app/consumers/user_events_consumer.rb
class UserEventsConsumer < Karafka::BaseConsumer
  def consume
    messages.each do |message|
      payload = message.payload

      case payload["event"]
      when "user.created"
        handle_user_created(payload)
      when "user.deleted"
        handle_user_deleted(payload)
      end
    end
  end

  private

  def handle_user_created(payload)
    WelcomeEmailJob.perform_later(payload["user_id"])
  end

  def handle_user_deleted(payload)
    DataCleanupJob.perform_later(payload["user_id"])
  end
end
```

## AWS SQS/SNS

### SQS (Simple Queue Service)

```ruby
# Gemfile
gem "aws-sdk-sqs"

# config/initializers/sqs.rb
require "aws-sdk-sqs"

SQS = Aws::SQS::Client.new(
  region: ENV.fetch("AWS_REGION") { "us-east-1" },
  credentials: Aws::Credentials.new(
    ENV["AWS_ACCESS_KEY_ID"],
    ENV["AWS_SECRET_ACCESS_KEY"]
  )
)

# app/services/sqs_publisher.rb
class SqsPublisher
  def initialize(queue_url)
    @queue_url = queue_url
    @client = SQS
  end

  def publish(message, delay_seconds: 0, message_attributes: {})
    @client.send_message(
      queue_url: @queue_url,
      message_body: message.to_json,
      delay_seconds: delay_seconds,
      message_attributes: build_attributes(message_attributes)
    )
  end

  def publish_batch(messages)
    entries = messages.map.with_index do |msg, idx|
      {
        id: idx.to_s,
        message_body: msg.to_json
      }
    end

    @client.send_message_batch(
      queue_url: @queue_url,
      entries: entries
    )
  end

  private

  def build_attributes(attrs)
    attrs.transform_values do |value|
      { string_value: value.to_s, data_type: "String" }
    end
  end
end

# app/services/sqs_consumer.rb
class SqsConsumer
  def initialize(queue_url, visibility_timeout: 30)
    @queue_url = queue_url
    @visibility_timeout = visibility_timeout
    @client = SQS
  end

  def poll(max_messages: 10, wait_time: 20)
    loop do
      response = @client.receive_message(
        queue_url: @queue_url,
        max_number_of_messages: max_messages,
        wait_time_seconds: wait_time,
        visibility_timeout: @visibility_timeout
      )

      response.messages.each do |message|
        process_message(message)
      end
    end
  end

  private

  def process_message(message)
    payload = JSON.parse(message.body)
    yield payload if block_given?

    @client.delete_message(
      queue_url: @queue_url,
      receipt_handle: message.receipt_handle
    )
  rescue StandardError => e
    Rails.logger.error "SQS processing error: #{e.message}"
    # El mensaje volvera a estar visible despues del timeout
  end
end
```

### SNS (Simple Notification Service)

```ruby
# Gemfile
gem "aws-sdk-sns"

# app/services/sns_publisher.rb
class SnsPublisher
  def initialize
    @client = Aws::SNS::Client.new(
      region: ENV.fetch("AWS_REGION") { "us-east-1" }
    )
  end

  def publish(topic_arn, message, subject: nil, attributes: {})
    @client.publish(
      topic_arn: topic_arn,
      message: message.to_json,
      subject: subject,
      message_attributes: build_attributes(attributes)
    )
  end

  # Publicar a multiples suscriptores con filtro
  def publish_with_filter(topic_arn, message, filter_key:, filter_value:)
    @client.publish(
      topic_arn: topic_arn,
      message: message.to_json,
      message_attributes: {
        filter_key => {
          data_type: "String",
          string_value: filter_value
        }
      }
    )
  end

  private

  def build_attributes(attrs)
    attrs.transform_values do |value|
      { data_type: "String", string_value: value.to_s }
    end
  end
end
```

### Shoryuken (Sidekiq-like para SQS)

```ruby
# Gemfile
gem "shoryuken"

# config/shoryuken.yml
concurrency: 25
queues:
  - [critical, 3]
  - [default, 2]
  - [low, 1]

# app/workers/sqs_worker.rb
class SqsWorker
  include Shoryuken::Worker

  shoryuken_options queue: "default",
                    auto_delete: true,
                    body_parser: :json

  def perform(sqs_msg, body)
    case body["event"]
    when "order.created"
      process_order(body["data"])
    when "user.registered"
      send_welcome_email(body["data"])
    end
  end

  private

  def process_order(data)
    order = Order.find(data["order_id"])
    OrderProcessor.new(order).process
  end

  def send_welcome_email(data)
    user = User.find(data["user_id"])
    WelcomeMailer.welcome(user).deliver_now
  end
end
```

## Redis Pub/Sub

### Publisher

```ruby
# app/services/redis_publisher.rb
class RedisPublisher
  def initialize
    @redis = Redis.new(url: ENV["REDIS_URL"])
  end

  def publish(channel, message)
    @redis.publish(channel, message.to_json)
  end

  def publish_event(event_type, payload)
    message = {
      event: event_type,
      payload: payload,
      timestamp: Time.current.iso8601,
      producer: "rails-app"
    }
    publish("events:#{event_type.split('.').first}", message)
  end
end

# Uso
publisher = RedisPublisher.new
publisher.publish_event("order.created", { order_id: 123, total: 99.99 })
```

### Subscriber

```ruby
# app/services/redis_subscriber.rb
class RedisSubscriber
  def initialize(channels)
    @redis = Redis.new(url: ENV["REDIS_URL"])
    @channels = channels
  end

  def subscribe
    @redis.subscribe(*@channels) do |on|
      on.message do |channel, message|
        process_message(channel, JSON.parse(message))
      end

      on.subscribe do |channel, subscriptions|
        Rails.logger.info "Subscribed to #{channel} (#{subscriptions} subscriptions)"
      end
    end
  end

  def psubscribe(patterns)
    @redis.psubscribe(*patterns) do |on|
      on.pmessage do |pattern, channel, message|
        process_message(channel, JSON.parse(message), pattern: pattern)
      end
    end
  end

  private

  def process_message(channel, message, pattern: nil)
    Rails.logger.info "Received on #{channel}: #{message['event']}"

    handler_class = find_handler(message["event"])
    handler_class.new(message["payload"]).call if handler_class
  rescue StandardError => e
    Rails.logger.error "Redis subscriber error: #{e.message}"
  end

  def find_handler(event_type)
    {
      "order.created" => OrderCreatedHandler,
      "user.registered" => UserRegisteredHandler
    }[event_type]
  end
end

# Iniciar subscriber (proceso separado)
# bin/redis_subscriber
subscriber = RedisSubscriber.new(["events:order", "events:user"])
subscriber.subscribe
```

## Patrones Avanzados

### Saga Pattern

```ruby
# app/sagas/order_saga.rb
class OrderSaga
  include Wisper::Publisher

  STEPS = [
    :reserve_inventory,
    :process_payment,
    :create_shipment,
    :send_confirmation
  ].freeze

  def initialize(order)
    @order = order
    @completed_steps = []
  end

  def execute
    STEPS.each do |step|
      begin
        send(step)
        @completed_steps << step
      rescue StandardError => e
        Rails.logger.error "Saga step #{step} failed: #{e.message}"
        compensate
        raise SagaFailedError, "Failed at #{step}: #{e.message}"
      end
    end

    broadcast(:order_saga_completed, @order)
  end

  private

  def reserve_inventory
    InventoryService.reserve(@order.line_items)
  end

  def process_payment
    PaymentService.charge(@order.payment_method, @order.total)
  end

  def create_shipment
    ShipmentService.create(@order)
  end

  def send_confirmation
    OrderMailer.confirmation(@order).deliver_now
  end

  # Compensaciones (rollback)
  def compensate
    @completed_steps.reverse.each do |step|
      compensation_method = "compensate_#{step}"
      send(compensation_method) if respond_to?(compensation_method, true)
    end
  end

  def compensate_reserve_inventory
    InventoryService.release(@order.line_items)
  end

  def compensate_process_payment
    PaymentService.refund(@order.payment_id)
  end

  def compensate_create_shipment
    ShipmentService.cancel(@order.shipment_id)
  end
end
```

### Outbox Pattern

```ruby
# db/migrate/xxx_create_outbox_messages.rb
class CreateOutboxMessages < ActiveRecord::Migration[8.0]
  def change
    create_table :outbox_messages do |t|
      t.string :topic, null: false
      t.string :key
      t.jsonb :payload, null: false
      t.string :status, default: "pending"
      t.datetime :processed_at
      t.integer :retry_count, default: 0
      t.timestamps

      t.index [:status, :created_at]
    end
  end
end

# app/models/outbox_message.rb
class OutboxMessage < ApplicationRecord
  scope :pending, -> { where(status: "pending") }
  scope :failed, -> { where(status: "failed") }
  scope :processable, -> { pending.order(created_at: :asc) }

  def mark_processed!
    update!(status: "processed", processed_at: Time.current)
  end

  def mark_failed!
    update!(status: "failed", retry_count: retry_count + 1)
  end
end

# app/services/outbox_publisher.rb
class OutboxPublisher
  def self.publish(topic:, payload:, key: nil)
    OutboxMessage.create!(
      topic: topic,
      key: key,
      payload: payload
    )
  end
end

# app/jobs/outbox_processor_job.rb
class OutboxProcessorJob < ApplicationJob
  queue_as :critical

  def perform
    OutboxMessage.processable.find_each do |message|
      begin
        publish_to_kafka(message)
        message.mark_processed!
      rescue StandardError => e
        Rails.logger.error "Outbox publish failed: #{e.message}"
        message.mark_failed!
      end
    end
  end

  private

  def publish_to_kafka(message)
    producer = KafkaProducer.new
    producer.publish_sync(
      message.topic,
      message.payload,
      key: message.key
    )
  end
end

# Uso - transaccion atomica
ActiveRecord::Base.transaction do
  order = Order.create!(params)
  OutboxPublisher.publish(
    topic: "orders",
    payload: { event: "order.created", order_id: order.id },
    key: order.id.to_s
  )
end
```

### Dead Letter Queue

```ruby
# app/services/dead_letter_handler.rb
class DeadLetterHandler
  MAX_RETRIES = 3
  RETRY_DELAYS = [60, 300, 900]  # 1min, 5min, 15min

  def initialize(original_queue:, dlq:)
    @original_queue = original_queue
    @dlq = dlq
  end

  def process_with_retry(message)
    retry_count = message.dig("metadata", "retry_count") || 0

    begin
      yield message["payload"]
    rescue StandardError => e
      if retry_count < MAX_RETRIES
        schedule_retry(message, retry_count, e)
      else
        send_to_dlq(message, e)
      end
    end
  end

  private

  def schedule_retry(message, retry_count, error)
    delay = RETRY_DELAYS[retry_count]
    message["metadata"] ||= {}
    message["metadata"]["retry_count"] = retry_count + 1
    message["metadata"]["last_error"] = error.message

    RetryMessageJob.set(wait: delay.seconds).perform_later(
      @original_queue,
      message
    )
  end

  def send_to_dlq(message, error)
    message["metadata"] ||= {}
    message["metadata"]["final_error"] = error.message
    message["metadata"]["failed_at"] = Time.current.iso8601

    @dlq.publish(message)

    # Notificar a admins
    AdminNotifier.dlq_message(@dlq.name, message).deliver_later
  end
end
```

## Monitoreo

### Metricas para colas

```ruby
# app/services/queue_metrics.rb
class QueueMetrics
  def self.record_job_execution(job_class, duration_ms, success:)
    labels = { job: job_class, success: success }

    # Incrementar contador
    Rails.cache.increment("metrics:jobs:#{job_class}:count")

    # Registrar duracion
    Rails.cache.write(
      "metrics:jobs:#{job_class}:last_duration",
      duration_ms,
      expires_in: 1.hour
    )

    # Para Prometheus/Grafana
    # StatsD.timing("job.duration", duration_ms, tags: labels)
    # StatsD.increment("job.count", tags: labels)
  end

  def self.record_queue_depth(queue_name, depth)
    Rails.cache.write(
      "metrics:queue:#{queue_name}:depth",
      depth,
      expires_in: 1.minute
    )
  end

  def self.dashboard_data
    {
      solid_queue: solid_queue_stats,
      sidekiq: sidekiq_stats,
      dead_jobs: dead_jobs_count
    }
  end

  private

  def self.solid_queue_stats
    {
      ready: SolidQueue::ReadyExecution.count,
      scheduled: SolidQueue::ScheduledExecution.count,
      failed: SolidQueue::FailedExecution.count
    }
  end

  def self.sidekiq_stats
    return {} unless defined?(Sidekiq)

    stats = Sidekiq::Stats.new
    {
      processed: stats.processed,
      failed: stats.failed,
      enqueued: stats.enqueued,
      retry_size: stats.retry_size
    }
  end
end
```

### Alertas

```ruby
# app/jobs/queue_health_check_job.rb
class QueueHealthCheckJob < ApplicationJob
  queue_as :critical

  THRESHOLDS = {
    queue_depth: 1000,
    failed_jobs: 50,
    oldest_job_age: 30.minutes
  }.freeze

  def perform
    check_queue_depth
    check_failed_jobs
    check_oldest_job
  end

  private

  def check_queue_depth
    depth = SolidQueue::ReadyExecution.count
    if depth > THRESHOLDS[:queue_depth]
      AdminNotifier.queue_depth_alert(depth).deliver_now
    end
  end

  def check_failed_jobs
    failed = SolidQueue::FailedExecution.count
    if failed > THRESHOLDS[:failed_jobs]
      AdminNotifier.failed_jobs_alert(failed).deliver_now
    end
  end

  def check_oldest_job
    oldest = SolidQueue::ReadyExecution.order(:created_at).first
    return unless oldest

    age = Time.current - oldest.created_at
    if age > THRESHOLDS[:oldest_job_age]
      AdminNotifier.stale_jobs_alert(age).deliver_now
    end
  end
end
```

## Checklist

- [ ] Queue adapter configurado (Solid Queue/Sidekiq)
- [ ] Colas separadas por prioridad
- [ ] Retry strategy definida
- [ ] Dead letter queue configurada
- [ ] Jobs idempotentes
- [ ] Monitoreo de profundidad de cola
- [ ] Alertas para jobs fallidos
- [ ] Backpressure handling
- [ ] Graceful shutdown
- [ ] Tests para jobs
