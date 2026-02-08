# Skill: Background Jobs

## Purpose
Implement asynchronous job processing using Solid Queue (Rails 8 default).

## Overview

Solid Queue is Rails 8's default Active Job backend, using the database for job storage. No Redis required.

## Configuration

### Database Setup
```bash
rails generate solid_queue:install
rails db:migrate
```

### Configure Active Job
```ruby
# config/application.rb
config.active_job.queue_adapter = :solid_queue
```

### Queue Configuration
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
  workers:
    - queues: [critical, default, low]
      threads: 5
      processes: 2
      polling_interval: 0.1
```

## Creating Jobs

### Basic Job
```ruby
# app/jobs/send_welcome_email_job.rb
class SendWelcomeEmailJob < ApplicationJob
  queue_as :default

  def perform(user_id)
    user = User.find(user_id)
    UserMailer.welcome(user).deliver_now
  end
end
```

### Job with Options
```ruby
# app/jobs/process_payment_job.rb
class ProcessPaymentJob < ApplicationJob
  queue_as :critical

  retry_on ActiveRecord::Deadlocked, wait: 5.seconds, attempts: 3
  discard_on ActiveJob::DeserializationError

  def perform(order)
    PaymentService.process(order)
  end
end
```

### Job with Callbacks
```ruby
# app/jobs/import_data_job.rb
class ImportDataJob < ApplicationJob
  queue_as :low

  before_enqueue do |job|
    Rails.logger.info "Enqueueing import for #{job.arguments.first}"
  end

  around_perform do |job, block|
    start_time = Time.current
    block.call
    duration = Time.current - start_time
    Rails.logger.info "Import completed in #{duration.round(2)}s"
  end

  def perform(file_path, user_id)
    user = User.find(user_id)
    ImportService.new(file_path, user).call
  end
end
```

## Enqueueing Jobs

### Immediate Execution
```ruby
# Enqueue to run as soon as possible
SendWelcomeEmailJob.perform_later(user.id)
```

### Delayed Execution
```ruby
# Run in 1 hour
SendReminderJob.set(wait: 1.hour).perform_later(user.id)

# Run at specific time
SendReportJob.set(wait_until: Date.tomorrow.noon).perform_later
```

### Queue Priority
```ruby
# Override queue at enqueue time
ProcessOrderJob.set(queue: :critical).perform_later(order)
```

## Common Job Patterns

### Mailer Job
```ruby
# Instead of creating a job, use deliver_later
UserMailer.welcome(user).deliver_later
UserMailer.notification(user).deliver_later(wait: 5.minutes)
```

### Batch Processing
```ruby
# app/jobs/batch_process_job.rb
class BatchProcessJob < ApplicationJob
  queue_as :low

  def perform(record_ids)
    Record.where(id: record_ids).find_each do |record|
      process_record(record)
    end
  end

  private

  def process_record(record)
    # Processing logic
  end
end

# Enqueue in batches
Record.pending.pluck(:id).each_slice(100) do |batch_ids|
  BatchProcessJob.perform_later(batch_ids)
end
```

### Webhook Processing
```ruby
# app/jobs/process_webhook_job.rb
class ProcessWebhookJob < ApplicationJob
  queue_as :critical

  retry_on Net::OpenTimeout, wait: :polynomially_longer, attempts: 5

  def perform(webhook_data)
    case webhook_data["event"]
    when "payment.completed"
      handle_payment_completed(webhook_data)
    when "subscription.cancelled"
      handle_subscription_cancelled(webhook_data)
    end
  end

  private

  def handle_payment_completed(data)
    # Process payment
  end

  def handle_subscription_cancelled(data)
    # Handle cancellation
  end
end
```

### Progress Tracking
```ruby
# app/jobs/export_job.rb
class ExportJob < ApplicationJob
  queue_as :default

  def perform(export_id)
    export = Export.find(export_id)
    export.update!(status: "processing")

    records = export.user.records
    total = records.count

    records.find_each.with_index do |record, index|
      process_record(record, export)

      # Update progress every 100 records
      if (index + 1) % 100 == 0
        progress = ((index + 1).to_f / total * 100).round
        export.update!(progress: progress)
      end
    end

    export.update!(status: "completed", progress: 100)
  rescue StandardError => e
    export.update!(status: "failed", error_message: e.message)
    raise
  end
end
```

## Recurring Jobs

### Using Solid Queue Recurring
```yaml
# config/recurring.yml
production:
  cleanup_old_sessions:
    class: CleanupSessionsJob
    schedule: every day at 3am

  send_daily_digest:
    class: DailyDigestJob
    schedule: every day at 8am

  sync_inventory:
    class: SyncInventoryJob
    schedule: every 15 minutes
```

### Manual Scheduling
```ruby
# app/jobs/schedule_daily_tasks_job.rb
class ScheduleDailyTasksJob < ApplicationJob
  def perform
    # Schedule tomorrow's run
    ScheduleDailyTasksJob.set(wait_until: Date.tomorrow.midnight).perform_later

    # Run daily tasks
    CleanupJob.perform_later
    ReportJob.perform_later
  end
end
```

## Error Handling

### Retry Configuration
```ruby
class UnreliableApiJob < ApplicationJob
  # Retry with exponential backoff
  retry_on ApiError, wait: :polynomially_longer, attempts: 5

  # Retry specific errors
  retry_on Timeout::Error, wait: 10.seconds, attempts: 3

  # Don't retry certain errors
  discard_on InvalidDataError

  def perform(data)
    ExternalApi.call(data)
  end
end
```

### Custom Error Handling
```ruby
class ImportJob < ApplicationJob
  def perform(file_path)
    ImportService.new(file_path).call
  rescue ImportService::ValidationError => e
    # Log and notify, but don't retry
    Rails.logger.error "Import validation failed: #{e.message}"
    AdminNotifier.import_failed(e).deliver_later
  end
end
```

## Testing Jobs

```ruby
# spec/jobs/send_welcome_email_job_spec.rb
require "rails_helper"

RSpec.describe SendWelcomeEmailJob, type: :job do
  let(:user) { create(:user) }

  describe "#perform" do
    it "sends welcome email" do
      expect {
        described_class.perform_now(user.id)
      }.to change { ActionMailer::Base.deliveries.count }.by(1)
    end
  end

  describe "enqueueing" do
    it "enqueues job" do
      expect {
        described_class.perform_later(user.id)
      }.to have_enqueued_job(described_class).with(user.id)
    end

    it "enqueues on correct queue" do
      expect {
        described_class.perform_later(user.id)
      }.to have_enqueued_job.on_queue(:default)
    end
  end
end
```

## Running Workers

```bash
# Development (auto-started with rails server)
rails server

# Production
bundle exec rake solid_queue:start

# With Procfile
# Procfile
web: bundle exec puma -C config/puma.rb
worker: bundle exec rake solid_queue:start
```

## Best Practices

1. **Pass IDs, not objects** - Serialize primitive values
2. **Keep jobs idempotent** - Safe to run multiple times
3. **Use appropriate queues** - Separate by priority
4. **Handle failures gracefully** - Log and notify
5. **Monitor job performance** - Track duration and failures
