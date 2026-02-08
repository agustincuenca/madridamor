# Skill: SOLID Principles

## Purpose
Apply SOLID principles to write maintainable, extensible, and testable Ruby and Rails code.

---

## S - Single Responsibility Principle (SRP)

**Definition:** A class should have only one reason to change. Each class should do one thing and do it well.

### Bad Example
```ruby
# This class does too many things: data access, validation, notifications, formatting
class User < ApplicationRecord
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :name, presence: true

  def full_name
    "#{first_name} #{last_name}"
  end

  def send_welcome_email
    UserMailer.welcome(self).deliver_later
  end

  def send_password_reset
    token = generate_reset_token
    UserMailer.password_reset(self, token).deliver_later
  end

  def generate_monthly_report
    orders = orders.where(created_at: 1.month.ago..Time.current)
    total = orders.sum(:total)
    # Generate PDF report...
  end

  def export_to_csv
    CSV.generate do |csv|
      csv << %w[id name email created_at]
      csv << [id, full_name, email, created_at]
    end
  end

  def calculate_loyalty_points
    orders.sum(:total) / 10
  end
end
```

### Good Example
```ruby
# Model handles only data and validations
class User < ApplicationRecord
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :name, presence: true

  def full_name
    "#{first_name} #{last_name}"
  end
end

# Separate service for notifications
class UserNotificationService
  def initialize(user)
    @user = user
  end

  def send_welcome_email
    UserMailer.welcome(@user).deliver_later
  end

  def send_password_reset
    token = PasswordResetToken.generate_for(@user)
    UserMailer.password_reset(@user, token).deliver_later
  end
end

# Separate service for reports
class UserReportService
  def initialize(user)
    @user = user
  end

  def monthly_report
    orders = @user.orders.where(created_at: 1.month.ago..Time.current)
    MonthlyReportGenerator.new(orders).generate
  end
end

# Separate exporter
class UserCsvExporter
  def initialize(users)
    @users = users
  end

  def export
    CSV.generate do |csv|
      csv << headers
      @users.each { |user| csv << row(user) }
    end
  end

  private

  def headers
    %w[id name email created_at]
  end

  def row(user)
    [user.id, user.full_name, user.email, user.created_at]
  end
end

# Separate calculator for loyalty
class LoyaltyPointsCalculator
  def initialize(user)
    @user = user
  end

  def calculate
    @user.orders.sum(:total) / 10
  end
end
```

### How to Apply in Rails
- **Models:** Only data, validations, associations, scopes
- **Controllers:** Only HTTP handling, delegate to services
- **Services:** One business operation per service
- **Jobs:** One task per job
- **Mailers:** Only email composition

---

## O - Open/Closed Principle (OCP)

**Definition:** Classes should be open for extension but closed for modification. Add new functionality by adding new code, not changing existing code.

### Bad Example
```ruby
class PaymentProcessor
  def process(payment)
    case payment.method
    when :credit_card
      process_credit_card(payment)
    when :paypal
      process_paypal(payment)
    when :stripe
      process_stripe(payment)
    when :apple_pay
      process_apple_pay(payment)
    # Every new payment method requires modifying this class
    end
  end

  private

  def process_credit_card(payment)
    # Credit card logic
  end

  def process_paypal(payment)
    # PayPal logic
  end

  def process_stripe(payment)
    # Stripe logic
  end

  def process_apple_pay(payment)
    # Apple Pay logic
  end
end
```

### Good Example
```ruby
# Base class or interface
class PaymentMethod
  def process(payment)
    raise NotImplementedError, "#{self.class} must implement #process"
  end
end

# Each payment method is a separate class
class CreditCardPayment < PaymentMethod
  def process(payment)
    # Credit card specific logic
    gateway = CreditCardGateway.new
    gateway.charge(payment.amount, payment.card_details)
  end
end

class PaypalPayment < PaymentMethod
  def process(payment)
    # PayPal specific logic
    client = PaypalClient.new
    client.execute_payment(payment.paypal_order_id)
  end
end

class StripePayment < PaymentMethod
  def process(payment)
    # Stripe specific logic
    Stripe::Charge.create(
      amount: payment.amount_in_cents,
      source: payment.stripe_token
    )
  end
end

# Adding new payment method doesn't require changing existing code
class ApplePayPayment < PaymentMethod
  def process(payment)
    # Apple Pay specific logic
  end
end

# Processor uses dependency injection
class PaymentProcessor
  METHODS = {
    credit_card: CreditCardPayment,
    paypal: PaypalPayment,
    stripe: StripePayment,
    apple_pay: ApplePayPayment
  }.freeze

  def process(payment)
    method_class = METHODS.fetch(payment.method) do
      raise ArgumentError, "Unknown payment method: #{payment.method}"
    end

    method_class.new.process(payment)
  end
end

# Or use a registry pattern for runtime registration
class PaymentMethodRegistry
  class << self
    def register(name, handler)
      handlers[name] = handler
    end

    def for(name)
      handlers.fetch(name) { raise "Unknown payment method: #{name}" }
    end

    private

    def handlers
      @handlers ||= {}
    end
  end
end

# Register handlers
PaymentMethodRegistry.register(:credit_card, CreditCardPayment)
PaymentMethodRegistry.register(:paypal, PaypalPayment)
```

### How to Apply in Rails
- Use **polymorphism** instead of conditionals
- Create **service objects** for each variant
- Use **concerns** to add behavior to models
- Use **decorators** to extend functionality
- Define **interfaces** (abstract classes) for extensibility

---

## L - Liskov Substitution Principle (LSP)

**Definition:** Objects of a superclass should be replaceable with objects of its subclasses without breaking the application. Subclasses must honor the contract of the parent class.

### Bad Example
```ruby
class Bird
  def fly
    puts "Flying..."
  end
end

class Sparrow < Bird
  def fly
    puts "Sparrow flying..."
  end
end

# Violates LSP: Penguin is a bird but can't fly
class Penguin < Bird
  def fly
    raise NotImplementedError, "Penguins can't fly!"
  end
end

# This code breaks with Penguin
def make_bird_fly(bird)
  bird.fly # Raises error for Penguin
end

make_bird_fly(Sparrow.new) # Works
make_bird_fly(Penguin.new) # Breaks!
```

### Good Example
```ruby
# Better hierarchy based on capabilities
class Bird
  def move
    raise NotImplementedError
  end
end

class FlyingBird < Bird
  def move
    fly
  end

  def fly
    puts "Flying..."
  end
end

class WalkingBird < Bird
  def move
    walk
  end

  def walk
    puts "Walking..."
  end
end

class Sparrow < FlyingBird
  def fly
    puts "Sparrow flying..."
  end
end

class Penguin < WalkingBird
  def walk
    puts "Penguin waddling..."
  end

  def swim
    puts "Penguin swimming..."
  end
end

# Now this works for all birds
def make_bird_move(bird)
  bird.move
end

make_bird_move(Sparrow.new) # "Sparrow flying..."
make_bird_move(Penguin.new) # "Penguin waddling..."
```

### Rails Example
```ruby
# Bad: Subclass changes behavior in unexpected ways
class Document < ApplicationRecord
  def publish
    update(published_at: Time.current, status: :published)
  end
end

class DraftDocument < Document
  def publish
    # Violates LSP: silently does nothing instead of publishing
    false
  end
end

# Good: Use state machine or explicit contract
class Document < ApplicationRecord
  include AASM

  aasm column: :status do
    state :draft, initial: true
    state :published

    event :publish do
      transitions from: :draft, to: :published
    end
  end
end

# Or use composition over inheritance
class Document < ApplicationRecord
  def publish
    publishing_strategy.publish(self)
  end

  def publishing_strategy
    PublishingStrategy.for(self)
  end
end

class PublishingStrategy
  def self.for(document)
    case document.document_type
    when "standard"
      StandardPublishing.new
    when "review_required"
      ReviewRequiredPublishing.new
    end
  end
end

class StandardPublishing
  def publish(document)
    document.update(published_at: Time.current, status: :published)
  end
end

class ReviewRequiredPublishing
  def publish(document)
    document.update(status: :pending_review)
    NotifyReviewersJob.perform_later(document.id)
  end
end
```

### Rules for LSP
1. **Preconditions cannot be strengthened** in subclass
2. **Postconditions cannot be weakened** in subclass
3. **Invariants must be preserved** in subclass
4. **No new exceptions** that parent doesn't throw
5. **Return types** must be compatible (same or more specific)

---

## I - Interface Segregation Principle (ISP)

**Definition:** Clients should not be forced to depend on interfaces they don't use. Many specific interfaces are better than one general-purpose interface.

### Bad Example
```ruby
# Fat interface - forces all implementations to define everything
class Worker
  def work
    raise NotImplementedError
  end

  def eat
    raise NotImplementedError
  end

  def sleep
    raise NotImplementedError
  end

  def manage_team
    raise NotImplementedError
  end

  def attend_meetings
    raise NotImplementedError
  end
end

class Developer < Worker
  def work
    puts "Writing code..."
  end

  def eat
    puts "Eating lunch..."
  end

  def sleep
    puts "Sleeping..."
  end

  # Forced to implement but doesn't use
  def manage_team
    raise "Developers don't manage teams"
  end

  def attend_meetings
    puts "Attending standup..."
  end
end

class Robot < Worker
  def work
    puts "Processing..."
  end

  # Robots don't eat or sleep - forced to implement useless methods
  def eat
    raise "Robots don't eat"
  end

  def sleep
    raise "Robots don't sleep"
  end

  def manage_team
    raise "Robots don't manage"
  end

  def attend_meetings
    raise "Robots don't attend meetings"
  end
end
```

### Good Example
```ruby
# Segregated interfaces using modules
module Workable
  def work
    raise NotImplementedError
  end
end

module Eatable
  def eat
    raise NotImplementedError
  end
end

module Sleepable
  def sleep
    raise NotImplementedError
  end
end

module Manageable
  def manage_team
    raise NotImplementedError
  end
end

module MeetingAttendable
  def attend_meetings
    raise NotImplementedError
  end
end

# Classes include only what they need
class Developer
  include Workable
  include Eatable
  include Sleepable
  include MeetingAttendable

  def work
    puts "Writing code..."
  end

  def eat
    puts "Eating lunch..."
  end

  def sleep
    puts "Sleeping..."
  end

  def attend_meetings
    puts "Attending standup..."
  end
end

class Robot
  include Workable

  def work
    puts "Processing..."
  end
end

class Manager
  include Workable
  include Eatable
  include Sleepable
  include Manageable
  include MeetingAttendable

  def work
    puts "Managing projects..."
  end

  def eat
    puts "Business lunch..."
  end

  def sleep
    puts "Sleeping..."
  end

  def manage_team
    puts "Leading team..."
  end

  def attend_meetings
    puts "Running meetings..."
  end
end
```

### Rails Example
```ruby
# Bad: One giant concern with everything
module Reportable
  extend ActiveSupport::Concern

  def generate_pdf_report
    # PDF generation
  end

  def generate_csv_report
    # CSV generation
  end

  def generate_excel_report
    # Excel generation
  end

  def send_report_by_email
    # Email sending
  end

  def schedule_report
    # Scheduling
  end
end

# Good: Segregated concerns
module PdfExportable
  extend ActiveSupport::Concern

  def to_pdf
    PdfGenerator.new(self).generate
  end
end

module CsvExportable
  extend ActiveSupport::Concern

  def to_csv
    CsvGenerator.new(self).generate
  end
end

module ExcelExportable
  extend ActiveSupport::Concern

  def to_excel
    ExcelGenerator.new(self).generate
  end
end

module Schedulable
  extend ActiveSupport::Concern

  def schedule(at:)
    ScheduledJob.set(wait_until: at).perform_later(self)
  end
end

# Models include only what they need
class Invoice < ApplicationRecord
  include PdfExportable
  include CsvExportable
  # Doesn't need Excel or scheduling
end

class Report < ApplicationRecord
  include PdfExportable
  include ExcelExportable
  include Schedulable
  # Doesn't need CSV
end
```

### How to Apply in Rails
- Create **small, focused concerns** instead of large ones
- Use **modules** as interfaces
- **Service objects** should do one thing
- **Policies** should be granular (CanRead, CanWrite vs CanDoEverything)

---

## D - Dependency Inversion Principle (DIP)

**Definition:** High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details.

### Bad Example
```ruby
# High-level module depends directly on low-level module
class OrderProcessor
  def initialize
    # Direct dependency on concrete implementation
    @email_sender = SmtpEmailSender.new
    @payment_gateway = StripeGateway.new
    @inventory = MySqlInventory.new
  end

  def process(order)
    return false unless @inventory.check_stock(order.items)

    @payment_gateway.charge(order.total, order.payment_info)
    @inventory.reduce_stock(order.items)
    @email_sender.send_confirmation(order)

    true
  end
end

# Problems:
# - Can't easily test without real SMTP, Stripe, MySQL
# - Can't swap implementations
# - Hard to reuse in different contexts
```

### Good Example
```ruby
# Define abstractions (interfaces)
class EmailSender
  def send_email(to:, subject:, body:)
    raise NotImplementedError
  end
end

class PaymentGateway
  def charge(amount, payment_info)
    raise NotImplementedError
  end
end

class InventoryService
  def check_stock(items)
    raise NotImplementedError
  end

  def reduce_stock(items)
    raise NotImplementedError
  end
end

# Concrete implementations
class SmtpEmailSender < EmailSender
  def send_email(to:, subject:, body:)
    # SMTP implementation
  end
end

class StripeGateway < PaymentGateway
  def charge(amount, payment_info)
    Stripe::Charge.create(amount: amount, source: payment_info[:token])
  end
end

class DatabaseInventory < InventoryService
  def check_stock(items)
    items.all? { |item| Product.find(item.id).stock >= item.quantity }
  end

  def reduce_stock(items)
    items.each { |item| Product.find(item.id).decrement!(:stock, item.quantity) }
  end
end

# High-level module depends on abstractions via dependency injection
class OrderProcessor
  def initialize(email_sender:, payment_gateway:, inventory:)
    @email_sender = email_sender
    @payment_gateway = payment_gateway
    @inventory = inventory
  end

  def process(order)
    return false unless @inventory.check_stock(order.items)

    @payment_gateway.charge(order.total, order.payment_info)
    @inventory.reduce_stock(order.items)
    @email_sender.send_email(
      to: order.user.email,
      subject: "Order Confirmation",
      body: "Your order has been processed"
    )

    true
  end
end

# Usage in production
processor = OrderProcessor.new(
  email_sender: SmtpEmailSender.new,
  payment_gateway: StripeGateway.new,
  inventory: DatabaseInventory.new
)

# Usage in tests - easy to mock
class MockEmailSender < EmailSender
  attr_reader :sent_emails

  def initialize
    @sent_emails = []
  end

  def send_email(to:, subject:, body:)
    @sent_emails << { to: to, subject: subject, body: body }
  end
end

class MockPaymentGateway < PaymentGateway
  def charge(amount, payment_info)
    { success: true, charge_id: "test_123" }
  end
end

# Test
processor = OrderProcessor.new(
  email_sender: MockEmailSender.new,
  payment_gateway: MockPaymentGateway.new,
  inventory: MockInventory.new
)
```

### Rails Example with Dependency Injection
```ruby
# config/initializers/dependencies.rb
Rails.application.config.after_initialize do
  Rails.application.config.dependencies = {
    email_sender: -> { SmtpEmailSender.new },
    payment_gateway: -> { StripeGateway.new },
    inventory: -> { DatabaseInventory.new }
  }
end

# Helper to resolve dependencies
module DependencyResolver
  def resolve(name)
    Rails.application.config.dependencies[name].call
  end
end

# Usage in service
class OrderProcessor
  include DependencyResolver

  def initialize(
    email_sender: resolve(:email_sender),
    payment_gateway: resolve(:payment_gateway),
    inventory: resolve(:inventory)
  )
    @email_sender = email_sender
    @payment_gateway = payment_gateway
    @inventory = inventory
  end
end

# Or using a container (dry-container gem)
class Container
  extend Dry::Container::Mixin

  register :email_sender do
    SmtpEmailSender.new
  end

  register :payment_gateway do
    StripeGateway.new
  end
end

# config/environments/test.rb
class TestContainer < Container
  register :email_sender do
    MockEmailSender.new
  end

  register :payment_gateway do
    MockPaymentGateway.new
  end
end
```

### How to Apply in Rails
- **Inject dependencies** through constructor or method parameters
- **Use configuration** to wire up implementations
- **Default to production** implementations with ability to override
- **Create adapters** for external services
- **Test doubles** become trivial to use

---

## SOLID Cheat Sheet

| Principle | One-liner | Code Smell |
|-----------|-----------|------------|
| **S**ingle Responsibility | One class, one reason to change | Class has multiple unrelated methods |
| **O**pen/Closed | Add features by adding code, not changing it | Switch statements for types |
| **L**iskov Substitution | Subclasses must be substitutable | Subclass throws unexpected errors |
| **I**nterface Segregation | Small, focused interfaces | Fat classes/modules with unused methods |
| **D**ependency Inversion | Depend on abstractions, not concretions | `new` inside business logic |

---

## Applying SOLID in Rails Architecture

```
app/
├── models/          # S: Only data, validations, associations
├── controllers/     # S: Only HTTP handling
├── services/        # S: One business operation each
│                    # O: Base service + specialized variants
│                    # D: Inject dependencies
├── policies/        # I: Granular authorization
├── adapters/        # D: Wrap external services
│   ├── payment/     # O: PaymentAdapter interface + implementations
│   └── email/       # O: EmailAdapter interface + implementations
├── presenters/      # S: View-specific formatting
└── validators/      # S: One validation concern each
```

### Example Service Following All Principles
```ruby
# app/services/checkout_service.rb
class CheckoutService
  # D: Dependencies injected, defaults for production
  def initialize(
    payment_processor: PaymentProcessor.new,
    inventory_service: InventoryService.new,
    notification_service: NotificationService.new
  )
    @payment_processor = payment_processor
    @inventory_service = inventory_service
    @notification_service = notification_service
  end

  # S: Single responsibility - orchestrate checkout
  def call(cart:, user:, payment_info:)
    # L: All services honor their contracts
    validate_stock!(cart)
    charge = process_payment(cart, payment_info)
    order = create_order(cart, user, charge)
    fulfill_order(order)
    notify_user(user, order)

    Result.success(order)
  rescue CheckoutError => e
    Result.failure(e.message)
  end

  private

  def validate_stock!(cart)
    cart.items.each do |item|
      unless @inventory_service.available?(item.product_id, item.quantity)
        raise CheckoutError, "#{item.name} is out of stock"
      end
    end
  end

  # O: Different payment methods handled by PaymentProcessor variants
  def process_payment(cart, payment_info)
    @payment_processor.charge(cart.total, payment_info)
  end

  def fulfill_order(order)
    @inventory_service.reserve(order.items)
  end

  def notify_user(user, order)
    @notification_service.order_confirmed(user, order)
  end
end

# I: Focused interfaces
class PaymentProcessor
  def charge(amount, payment_info)
    raise NotImplementedError
  end
end

class StripePaymentProcessor < PaymentProcessor
  def charge(amount, payment_info)
    Stripe::Charge.create(amount: amount, source: payment_info[:token])
  end
end

class PaypalPaymentProcessor < PaymentProcessor
  def charge(amount, payment_info)
    PaypalClient.new.execute(payment_info[:order_id])
  end
end
```
