# Skill: Clean Code Principles

## Purpose
Write readable, maintainable, and self-documenting Ruby and Rails code following Uncle Bob's Clean Code principles.

---

## Naming Conventions

### Use Intention-Revealing Names
Names should tell you why something exists, what it does, and how it's used.

```ruby
# Bad
d = Time.now - user.created_at  # elapsed time in days?
list = User.where("age > 18")
temp = calculate_something

# Good
days_since_registration = (Time.now - user.created_at).to_i / 1.day
adult_users = User.where("age >= ?", 18)
monthly_revenue = calculate_monthly_revenue
```

### Use Pronounceable Names
```ruby
# Bad
genymdhms = Time.now.strftime("%Y%m%d%H%M%S")
cstmr_lst = []
modymdhms = record.updated_at

# Good
generation_timestamp = Time.now.strftime("%Y%m%d%H%M%S")
customers = []
modification_timestamp = record.updated_at
```

### Use Searchable Names
```ruby
# Bad - magic numbers are not searchable
if status == 4
  user.update(role: 2)
end

# Good - constants are searchable and meaningful
STATUS_APPROVED = 4
ROLE_ADMIN = 2

if status == STATUS_APPROVED
  user.update(role: ROLE_ADMIN)
end

# Even better in Rails - use enums
class User < ApplicationRecord
  enum status: { pending: 0, approved: 1, rejected: 2 }
  enum role: { user: 0, moderator: 1, admin: 2 }
end

if user.approved?
  user.admin!
end
```

### Naming Conventions by Type

```ruby
# Classes/Modules: Nouns, PascalCase
class UserAccount; end
class PaymentProcessor; end
module Searchable; end

# Methods: Verbs, snake_case
def calculate_total; end
def send_notification; end
def validate_email; end

# Predicate methods: End with ?
def valid?; end
def admin?; end
def can_edit?; end

# Dangerous methods: End with !
def save!; end      # Raises exception on failure
def delete!; end    # Destructive operation
def normalize!; end # Mutates in place

# Variables: snake_case, descriptive
current_user = User.find(id)
total_amount = cart.items.sum(&:price)
is_authenticated = session[:user_id].present?  # or: authenticated?

# Constants: SCREAMING_SNAKE_CASE
MAX_LOGIN_ATTEMPTS = 5
DEFAULT_PAGE_SIZE = 25
API_BASE_URL = "https://api.example.com"
```

### Avoid Mental Mapping
```ruby
# Bad - reader must remember what a, b, c mean
def calculate(a, b, c)
  a * b * (1 + c)
end

# Good - names are self-documenting
def calculate_total_with_tax(quantity, unit_price, tax_rate)
  quantity * unit_price * (1 + tax_rate)
end
```

---

## Functions / Methods

### Small and Focused
Methods should do one thing, do it well, and do it only.

```ruby
# Bad - method does too many things
def process_order(order)
  # Validate
  return false if order.items.empty?
  return false unless order.user.active?

  # Calculate totals
  subtotal = order.items.sum(&:price)
  tax = subtotal * 0.1
  shipping = subtotal > 100 ? 0 : 10
  total = subtotal + tax + shipping

  # Process payment
  payment_result = PaymentGateway.charge(total, order.payment_info)
  return false unless payment_result.success?

  # Update inventory
  order.items.each do |item|
    product = Product.find(item.product_id)
    product.decrement!(:stock, item.quantity)
  end

  # Create records
  order.update(
    status: :paid,
    total: total,
    paid_at: Time.current
  )

  # Send notifications
  OrderMailer.confirmation(order).deliver_later
  AdminNotifier.new_order(order)

  true
end

# Good - each method does one thing
def process_order(order)
  return failure("Empty order") if order.items.empty?
  return failure("Inactive user") unless order.user.active?

  totals = calculate_totals(order)
  payment = process_payment(order, totals[:total])

  return failure(payment.error) unless payment.success?

  finalize_order(order, totals, payment)
  send_notifications(order)

  success(order)
end

private

def calculate_totals(order)
  subtotal = order.items.sum(&:price)
  {
    subtotal: subtotal,
    tax: calculate_tax(subtotal),
    shipping: calculate_shipping(subtotal),
    total: subtotal + tax + shipping
  }
end

def process_payment(order, amount)
  PaymentGateway.charge(amount, order.payment_info)
end

def finalize_order(order, totals, payment)
  reduce_inventory(order.items)
  order.update(status: :paid, total: totals[:total], paid_at: Time.current)
end

def send_notifications(order)
  OrderMailer.confirmation(order).deliver_later
  AdminNotifier.new_order(order)
end
```

### Few Arguments (Ideally Zero to Two)
```ruby
# Bad - too many arguments
def create_user(first_name, last_name, email, password, role, department, manager_id, start_date)
  # ...
end

# Good - use a hash or object
def create_user(attributes)
  User.create!(attributes)
end

# Or named parameters
def create_user(email:, password:, name: nil, role: :user)
  User.create!(email: email, password: password, name: name, role: role)
end

# Best - use a form object or builder
class UserRegistration
  include ActiveModel::Model

  attr_accessor :email, :password, :name, :role

  validates :email, :password, presence: true

  def save
    return false unless valid?
    User.create!(attributes)
  end
end

registration = UserRegistration.new(params[:user])
registration.save
```

### Avoid Flag Arguments
```ruby
# Bad - boolean parameter
def render_page(content, include_sidebar)
  if include_sidebar
    render_with_sidebar(content)
  else
    render_without_sidebar(content)
  end
end

render_page(content, true)  # What does true mean?

# Good - separate methods
def render_page_with_sidebar(content)
  render_with_sidebar(content)
end

def render_page_without_sidebar(content)
  render_without_sidebar(content)
end

# Or use named parameters
def render_page(content, sidebar: false)
  sidebar ? render_with_sidebar(content) : render_without_sidebar(content)
end

render_page(content, sidebar: true)  # Clear intent
```

### Command Query Separation
Methods should either do something (command) or return something (query), not both.

```ruby
# Bad - does something AND returns something
def set_and_check_name(name)
  @name = name
  @name.present?
end

# Good - separate command and query
def set_name(name)
  @name = name
end

def name_present?
  @name.present?
end

# Usage
set_name("John")
if name_present?
  # ...
end
```

---

## Comments

### Code Should Be Self-Documenting
```ruby
# Bad - comment explains what (obvious from code)
# Check if user is an adult
if user.age >= 18
  # Allow access
  grant_access
end

# Good - code explains itself
if user.adult?
  grant_access
end

# In User model
def adult?
  age >= ADULT_AGE
end
```

### Comment the "Why", Not the "What"
```ruby
# Bad - explains what the code does (obvious)
# Increment counter by 1
counter += 1

# Good - explains why
# We add a buffer day to account for timezone differences
# when calculating subscription expiry
expiry_date = subscription.end_date + 1.day

# Performance optimization: batch processing to avoid memory issues
# with large datasets (see issue #1234)
User.find_each(batch_size: 1000) do |user|
  process(user)
end
```

### Good Uses of Comments
```ruby
# TODO: Refactor this when we upgrade to Rails 8
# FIXME: Race condition possible with concurrent updates
# HACK: Workaround for third-party API bug (ticket #XYZ)
# NOTE: This algorithm is O(n^2), acceptable for small datasets

# Legal/copyright headers (required by license)
# frozen_string_literal: true

# Public API documentation
# Calculates the compound interest for an investment.
#
# @param principal [Float] The initial investment amount
# @param rate [Float] Annual interest rate (e.g., 0.05 for 5%)
# @param years [Integer] Number of years
# @return [Float] The final amount after compound interest
#
# @example
#   compound_interest(1000, 0.05, 10) #=> 1628.89
def compound_interest(principal, rate, years)
  principal * (1 + rate) ** years
end

# Warning about consequences
# WARNING: This will delete all user data permanently.
# Only call this in development/test environments.
def reset_database!
  raise "Not in production!" if Rails.env.production?
  # ...
end
```

### Avoid Commented-Out Code
```ruby
# Bad - commented code is confusing and clutters
def calculate_discount(price)
  # old_discount = price * 0.1
  # if customer.vip?
  #   old_discount *= 2
  # end
  # return old_discount

  # new implementation
  DiscountCalculator.new(price, customer).calculate
end

# Good - remove dead code, use git for history
def calculate_discount(price)
  DiscountCalculator.new(price, customer).calculate
end
```

---

## Formatting

### Consistent Indentation
```ruby
# Use 2 spaces for indentation in Ruby (standard)
class User
  def full_name
    "#{first_name} #{last_name}"
  end

  def admin?
    role == :admin
  end
end
```

### Vertical Spacing
```ruby
class OrderService
  # Group related methods together
  # Separate groups with blank lines

  # Public interface
  def create_order(user, cart)
    validate_cart(cart)
    order = build_order(user, cart)
    process_order(order)
    order
  end

  def cancel_order(order)
    refund_payment(order)
    restore_inventory(order)
    order.cancel!
  end

  private

  # Validation helpers
  def validate_cart(cart)
    raise EmptyCartError if cart.empty?
  end

  # Order building
  def build_order(user, cart)
    Order.new(user: user, items: cart.items)
  end

  # Processing
  def process_order(order)
    charge_payment(order)
    reserve_inventory(order)
    send_confirmation(order)
  end

  # Payment operations
  def charge_payment(order)
    # ...
  end

  def refund_payment(order)
    # ...
  end

  # Inventory operations
  def reserve_inventory(order)
    # ...
  end

  def restore_inventory(order)
    # ...
  end

  # Notifications
  def send_confirmation(order)
    # ...
  end
end
```

### Line Length
```ruby
# Keep lines under 80-120 characters
# Break long method chains
User
  .where(active: true)
  .includes(:orders)
  .order(created_at: :desc)
  .limit(10)

# Break long argument lists
create_user(
  email: "john@example.com",
  first_name: "John",
  last_name: "Doe",
  role: :admin,
  department: "Engineering"
)

# Break long conditionals
if user.active? &&
   user.email_verified? &&
   user.subscription.valid? &&
   user.two_factor_enabled?
  grant_premium_access
end

# Or extract to methods
if eligible_for_premium_access?(user)
  grant_premium_access
end
```

---

## Error Handling

### Use Exceptions, Not Return Codes
```ruby
# Bad - return codes
def withdraw(amount)
  return -1 if amount <= 0
  return -2 if amount > balance
  @balance -= amount
  0  # success
end

result = account.withdraw(100)
case result
when 0 then puts "Success"
when -1 then puts "Invalid amount"
when -2 then puts "Insufficient funds"
end

# Good - exceptions
class InvalidAmountError < StandardError; end
class InsufficientFundsError < StandardError; end

def withdraw(amount)
  raise InvalidAmountError, "Amount must be positive" if amount <= 0
  raise InsufficientFundsError, "Balance too low" if amount > balance
  @balance -= amount
end

begin
  account.withdraw(100)
  puts "Success"
rescue InvalidAmountError => e
  puts "Error: #{e.message}"
rescue InsufficientFundsError => e
  puts "Error: #{e.message}"
end
```

### Fail Fast
```ruby
# Bad - deeply nested conditionals
def process_payment(order)
  if order.present?
    if order.items.any?
      if order.user.active?
        if order.payment_method.valid?
          # Finally, the actual logic
          charge_customer(order)
        else
          log_error("Invalid payment method")
        end
      else
        log_error("Inactive user")
      end
    else
      log_error("Empty order")
    end
  else
    log_error("No order")
  end
end

# Good - guard clauses / fail fast
def process_payment(order)
  raise ArgumentError, "Order required" unless order.present?
  raise EmptyOrderError unless order.items.any?
  raise InactiveUserError unless order.user.active?
  raise InvalidPaymentError unless order.payment_method.valid?

  charge_customer(order)
end
```

### Define Custom Exceptions
```ruby
# app/errors/application_error.rb
class ApplicationError < StandardError
  attr_reader :code, :details

  def initialize(message = nil, code: nil, details: {})
    @code = code
    @details = details
    super(message)
  end
end

# Domain-specific errors
class PaymentError < ApplicationError; end
class PaymentDeclinedError < PaymentError; end
class PaymentGatewayError < PaymentError; end

class OrderError < ApplicationError; end
class OutOfStockError < OrderError; end
class InvalidOrderError < OrderError; end

# Usage with details
raise OutOfStockError.new(
  "Product not available",
  code: "OUT_OF_STOCK",
  details: { product_id: product.id, requested: 5, available: 2 }
)
```

### Handle Errors at the Right Level
```ruby
# In Rails controller - handle and respond
class OrdersController < ApplicationController
  rescue_from OrderError, with: :handle_order_error
  rescue_from PaymentError, with: :handle_payment_error

  def create
    @order = OrderService.new.create(order_params)
    redirect_to @order, notice: "Order placed!"
  end

  private

  def handle_order_error(error)
    flash.now[:alert] = error.message
    render :new, status: :unprocessable_entity
  end

  def handle_payment_error(error)
    flash.now[:alert] = "Payment failed: #{error.message}"
    render :checkout, status: :payment_required
  end
end

# In service - raise, don't handle
class OrderService
  def create(params)
    order = Order.new(params)
    validate!(order)
    process_payment!(order)
    order.save!
    order
  end

  private

  def validate!(order)
    raise InvalidOrderError, "No items" if order.items.empty?
  end

  def process_payment!(order)
    result = PaymentGateway.charge(order.total)
    raise PaymentDeclinedError, result.message unless result.success?
  end
end
```

---

## DRY - Don't Repeat Yourself

### Extract Common Patterns
```ruby
# Bad - repeated code
class UsersController < ApplicationController
  def show
    @user = User.find(params[:id])
    authorize @user
    respond_to do |format|
      format.html
      format.json { render json: @user }
    end
  rescue ActiveRecord::RecordNotFound
    redirect_to users_path, alert: "User not found"
  end

  def edit
    @user = User.find(params[:id])
    authorize @user
    respond_to do |format|
      format.html
      format.json { render json: @user }
    end
  rescue ActiveRecord::RecordNotFound
    redirect_to users_path, alert: "User not found"
  end
end

# Good - DRY with callbacks and concerns
class UsersController < ApplicationController
  before_action :set_user, only: [:show, :edit, :update, :destroy]

  def show
    respond_with @user
  end

  def edit
    respond_with @user
  end

  private

  def set_user
    @user = User.find(params[:id])
    authorize @user
  end
end

# Or with a concern for common behavior
module Findable
  extend ActiveSupport::Concern

  class_methods do
    def set_resource(name, scope: nil)
      before_action only: [:show, :edit, :update, :destroy] do
        model = name.to_s.classify.constantize
        query = scope ? instance_exec(&scope) : model
        instance_variable_set("@#{name}", query.find(params[:id]))
      end
    end
  end
end

class UsersController < ApplicationController
  include Findable
  set_resource :user
end
```

### But Don't Over-DRY
```ruby
# Sometimes repetition is okay if:
# 1. The code is simple enough
# 2. The concepts are different even if similar
# 3. DRY would create unnecessary coupling

# Okay to repeat - different domains
class User < ApplicationRecord
  def full_name
    "#{first_name} #{last_name}"
  end
end

class Author < ApplicationRecord
  def full_name
    "#{first_name} #{last_name}"
  end
end

# These look similar but represent different concepts
# Extracting to a module might create unwanted coupling
# If full_name changes for Author (e.g., add title), User shouldn't be affected
```

---

## KISS - Keep It Simple, Stupid

### Prefer Simple Solutions
```ruby
# Bad - over-engineered
class StringReverser
  def initialize(strategy: DefaultReversalStrategy.new)
    @strategy = strategy
  end

  def reverse(string)
    @strategy.reverse(string)
  end

  class DefaultReversalStrategy
    def reverse(string)
      string.chars.reverse.join
    end
  end
end

reverser = StringReverser.new
reverser.reverse("hello")

# Good - simple
"hello".reverse
```

### Avoid Premature Optimization
```ruby
# Bad - premature optimization
class UserSearch
  def initialize
    @cache = LRUCache.new(1000)
    @index = BloomFilter.new
    @thread_pool = ThreadPool.new(4)
  end

  def search(query)
    return @cache.get(query) if @cache.has?(query)
    # Complex parallel search...
  end
end

# Good - start simple, optimize when needed
class UserSearch
  def search(query)
    User.where("name ILIKE ?", "%#{query}%").limit(20)
  end
end

# Add complexity only when you have evidence it's needed:
# - Slow queries in production logs
# - Performance tests showing bottlenecks
# - Clear requirements for scale
```

---

## YAGNI - You Aren't Gonna Need It

### Don't Build Features You Don't Need Yet
```ruby
# Bad - building for hypothetical future requirements
class Report
  def generate(format = :pdf)
    data = fetch_data

    case format
    when :pdf then generate_pdf(data)
    when :csv then generate_csv(data)
    when :excel then generate_excel(data)
    when :xml then generate_xml(data)        # Never requested
    when :json then generate_json(data)       # Never requested
    when :html then generate_html(data)       # Never requested
    when :markdown then generate_markdown(data) # Never requested
    end
  end
end

# Good - build what's needed now
class Report
  def generate(format = :pdf)
    data = fetch_data

    case format
    when :pdf then generate_pdf(data)
    when :csv then generate_csv(data)
    else raise ArgumentError, "Unsupported format: #{format}"
    end
  end
end
# Add other formats when/if they're actually needed
```

### Avoid Over-Abstraction
```ruby
# Bad - abstraction for one use case
class NotificationStrategyFactory
  def self.create(type)
    case type
    when :email
      EmailNotificationStrategy.new
    end
    # Only email exists, but we built a whole factory
  end
end

# Good - just send the email
class NotificationService
  def notify(user, message)
    UserMailer.notification(user, message).deliver_later
  end
end

# When you actually need SMS and push:
class NotificationService
  def notify(user, message, via: :email)
    case via
    when :email then UserMailer.notification(user, message).deliver_later
    when :sms then SmsClient.send(user.phone, message)
    when :push then PushService.send(user.device_token, message)
    end
  end
end
```

---

## Boy Scout Rule

### Leave Code Better Than You Found It

```ruby
# Before - you're fixing a bug in this method
def process_order(order)
  if order.items.count > 0
    total = 0
    for item in order.items
      total = total + item.price
    end
    order.total = total
    order.save
    UserMailer.order_confirmation(order.user, order).deliver
    return true
  else
    return false
  end
end

# After - fixed the bug AND improved the code
def process_order(order)
  return false if order.items.empty?

  order.update!(total: order.items.sum(&:price))
  OrderMailer.confirmation(order).deliver_later

  true
end

# Improvements made while fixing the bug:
# 1. Guard clause instead of wrapping if
# 2. Idiomatic Ruby (empty?, sum)
# 3. Bang method for save (fail fast)
# 4. deliver_later instead of deliver
# 5. Simplified mailer call
# 6. Removed unnecessary variable
```

### Small Incremental Improvements
```ruby
# Don't try to rewrite everything at once
# Make small improvements as you touch code:

# If you see: outdated syntax, improve it
# Before
{ :key => "value" }
# After
{ key: "value" }

# If you see: unclear names, clarify them
# Before
def calc(x, y)
# After
def calculate_total(price, quantity)

# If you see: missing tests, add them
# Before: no tests
# After: at least cover the method you're modifying

# If you see: dead code, remove it
# Before: commented-out code from 2019
# After: delete it (git has history)
```

---

## Rails-Specific Clean Code

### Fat Models, Skinny Controllers (But Not Too Fat)
```ruby
# Controller: Only HTTP concerns
class OrdersController < ApplicationController
  def create
    result = CreateOrder.call(order_params, current_user)

    if result.success?
      redirect_to result.order, notice: "Order placed!"
    else
      @order = result.order
      flash.now[:alert] = result.error
      render :new, status: :unprocessable_entity
    end
  end

  private

  def order_params
    params.require(:order).permit(:shipping_address, items: [:product_id, :quantity])
  end
end

# Service: Business logic
class CreateOrder
  def self.call(params, user)
    new(params, user).call
  end

  def initialize(params, user)
    @params = params
    @user = user
  end

  def call
    order = @user.orders.build(@params)

    if order.valid? && process_payment(order)
      order.save!
      notify(order)
      Result.new(success: true, order: order)
    else
      Result.new(success: false, order: order, error: order.errors.full_messages.join(", "))
    end
  end

  private

  def process_payment(order)
    # ...
  end

  def notify(order)
    # ...
  end

  Result = Struct.new(:success, :order, :error, keyword_init: true) do
    def success?
      success
    end
  end
end

# Model: Data, validations, associations, scopes
class Order < ApplicationRecord
  belongs_to :user
  has_many :items, class_name: "OrderItem"

  validates :shipping_address, presence: true

  scope :recent, -> { order(created_at: :desc) }
  scope :pending, -> { where(status: :pending) }

  def total
    items.sum(&:subtotal)
  end
end
```

### Use Query Objects for Complex Queries
```ruby
# Instead of putting complex queries in models
class UserSearch
  def initialize(params)
    @params = params
  end

  def call
    scope = User.all
    scope = filter_by_status(scope)
    scope = filter_by_role(scope)
    scope = filter_by_date_range(scope)
    scope = search_by_name(scope)
    scope = apply_sorting(scope)
    scope
  end

  private

  def filter_by_status(scope)
    return scope unless @params[:status].present?
    scope.where(status: @params[:status])
  end

  def filter_by_role(scope)
    return scope unless @params[:role].present?
    scope.where(role: @params[:role])
  end

  def filter_by_date_range(scope)
    scope = scope.where("created_at >= ?", @params[:from]) if @params[:from]
    scope = scope.where("created_at <= ?", @params[:to]) if @params[:to]
    scope
  end

  def search_by_name(scope)
    return scope unless @params[:q].present?
    scope.where("name ILIKE ?", "%#{@params[:q]}%")
  end

  def apply_sorting(scope)
    column = @params[:sort] || :created_at
    direction = @params[:direction] || :desc
    scope.order(column => direction)
  end
end

# Usage
users = UserSearch.new(params).call
```
