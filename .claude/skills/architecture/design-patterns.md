# Skill: Design Patterns (GoF)

## Purpose
Implement Gang of Four design patterns in Ruby and Rails to solve common software design problems with proven, reusable solutions.

---

## Creational Patterns

### Factory Method
**Problem:** Need to create objects without specifying their exact class.
**When to use:** When a class cannot anticipate the type of objects it needs to create.

```ruby
# Base creator
class NotificationFactory
  def self.create(type, recipient, message)
    case type
    when :email
      EmailNotification.new(recipient, message)
    when :sms
      SmsNotification.new(recipient, message)
    when :push
      PushNotification.new(recipient, message)
    else
      raise ArgumentError, "Unknown notification type: #{type}"
    end
  end
end

# Product classes
class EmailNotification
  def initialize(recipient, message)
    @recipient = recipient
    @message = message
  end

  def deliver
    # Send email
  end
end

class SmsNotification
  def initialize(recipient, message)
    @recipient = recipient
    @message = message
  end

  def deliver
    # Send SMS
  end
end

# Usage
notification = NotificationFactory.create(:email, user.email, "Welcome!")
notification.deliver
```

**Rails Example:**
```ruby
# app/services/payment_processor_factory.rb
class PaymentProcessorFactory
  PROCESSORS = {
    stripe: StripeProcessor,
    paypal: PaypalProcessor,
    braintree: BraintreeProcessor
  }.freeze

  def self.for(gateway)
    PROCESSORS.fetch(gateway) do
      raise ArgumentError, "Unknown payment gateway: #{gateway}"
    end.new
  end
end

# Usage in controller
class PaymentsController < ApplicationController
  def create
    processor = PaymentProcessorFactory.for(params[:gateway].to_sym)
    result = processor.charge(current_user, amount)
  end
end
```

---

### Abstract Factory
**Problem:** Need to create families of related objects without specifying their concrete classes.
**When to use:** When your system needs to work with multiple families of products.

```ruby
# Abstract factory
class UIFactory
  def create_button
    raise NotImplementedError
  end

  def create_input
    raise NotImplementedError
  end
end

# Concrete factories
class MaterialUIFactory < UIFactory
  def create_button
    MaterialButton.new
  end

  def create_input
    MaterialInput.new
  end
end

class BootstrapUIFactory < UIFactory
  def create_button
    BootstrapButton.new
  end

  def create_input
    BootstrapInput.new
  end
end

# Usage
factory = Rails.application.config.ui_framework == :material ?
          MaterialUIFactory.new :
          BootstrapUIFactory.new

button = factory.create_button
input = factory.create_input
```

**Rails Example:**
```ruby
# app/services/export_factory.rb
class ExportFactory
  def self.for(format)
    case format
    when :pdf
      PdfExportFactory.new
    when :excel
      ExcelExportFactory.new
    when :csv
      CsvExportFactory.new
    end
  end
end

class PdfExportFactory
  def create_document
    PdfDocument.new
  end

  def create_formatter
    PdfFormatter.new
  end

  def create_writer
    PdfWriter.new
  end
end

# Usage
factory = ExportFactory.for(:pdf)
doc = factory.create_document
formatter = factory.create_formatter
writer = factory.create_writer
```

---

### Builder
**Problem:** Need to construct complex objects step by step.
**When to use:** When an object requires many steps to be created or has many optional parameters.

```ruby
class QueryBuilder
  def initialize
    @select = []
    @where = []
    @order = nil
    @limit = nil
  end

  def select(*columns)
    @select.concat(columns)
    self
  end

  def where(condition)
    @where << condition
    self
  end

  def order(column, direction = :asc)
    @order = { column: column, direction: direction }
    self
  end

  def limit(count)
    @limit = count
    self
  end

  def build
    query = "SELECT #{@select.join(', ')} FROM users"
    query += " WHERE #{@where.join(' AND ')}" if @where.any?
    query += " ORDER BY #{@order[:column]} #{@order[:direction]}" if @order
    query += " LIMIT #{@limit}" if @limit
    query
  end
end

# Usage with method chaining
query = QueryBuilder.new
  .select(:id, :name, :email)
  .where("active = true")
  .where("created_at > '2024-01-01'")
  .order(:created_at, :desc)
  .limit(10)
  .build
```

**Rails Example:**
```ruby
# app/builders/user_builder.rb
class UserBuilder
  def initialize
    @user = User.new
  end

  def with_email(email)
    @user.email = email
    self
  end

  def with_name(first, last)
    @user.first_name = first
    @user.last_name = last
    self
  end

  def with_role(role)
    @user.role = role
    self
  end

  def with_profile(attributes = {})
    @user.build_profile(attributes)
    self
  end

  def with_preferences(prefs)
    @user.preferences = prefs
    self
  end

  def build
    @user
  end

  def create!
    @user.save!
    @user
  end
end

# Usage
user = UserBuilder.new
  .with_email("john@example.com")
  .with_name("John", "Doe")
  .with_role(:admin)
  .with_profile(bio: "Developer")
  .create!
```

---

### Singleton
**Problem:** Ensure a class has only one instance with global access.
**When to use:** When exactly one instance is needed (configurations, connection pools, caches).

```ruby
# Ruby's built-in Singleton module
require 'singleton'

class Configuration
  include Singleton

  attr_accessor :api_key, :debug_mode, :timeout

  def initialize
    @api_key = nil
    @debug_mode = false
    @timeout = 30
  end
end

# Usage
config = Configuration.instance
config.api_key = "secret123"
config.debug_mode = true

# Anywhere else in the app
Configuration.instance.api_key # => "secret123"
```

**Rails Example:**
```ruby
# Rails already provides singletons via configuration
# config/application.rb
module MyApp
  class Application < Rails::Application
    config.my_setting = "value"
  end
end

# Access anywhere
Rails.application.config.my_setting

# Custom singleton service
class FeatureFlags
  include Singleton

  def initialize
    @flags = {}
    load_flags
  end

  def enabled?(feature)
    @flags[feature.to_sym] == true
  end

  def enable(feature)
    @flags[feature.to_sym] = true
  end

  private

  def load_flags
    @flags = Rails.cache.fetch("feature_flags", expires_in: 1.hour) do
      Feature.pluck(:name, :enabled).to_h.symbolize_keys
    end
  end
end

# Usage
FeatureFlags.instance.enabled?(:new_checkout)
```

---

### Prototype
**Problem:** Create new objects by copying existing ones.
**When to use:** When creating an object is expensive and you can clone existing ones.

```ruby
class Document
  attr_accessor :title, :content, :styles, :metadata

  def initialize
    @styles = {}
    @metadata = {}
  end

  def clone
    copy = super
    copy.styles = @styles.dup
    copy.metadata = @metadata.dup
    copy
  end
end

# Usage
template = Document.new
template.title = "Template"
template.styles = { font: "Arial", size: 12 }
template.metadata = { author: "System", version: 1 }

# Create copies
doc1 = template.clone
doc1.title = "Report Q1"

doc2 = template.clone
doc2.title = "Report Q2"
```

**Rails Example:**
```ruby
# Using ActiveRecord's dup
class Order < ApplicationRecord
  has_many :order_items

  def duplicate
    new_order = dup
    new_order.status = :draft
    new_order.created_at = nil

    order_items.each do |item|
      new_order.order_items << item.dup
    end

    new_order
  end
end

# Usage
original_order = Order.find(1)
new_order = original_order.duplicate
new_order.save!
```

---

## Structural Patterns

### Adapter
**Problem:** Make incompatible interfaces work together.
**When to use:** When you need to use an existing class with an incompatible interface.

```ruby
# Target interface expected by client
class PaymentGateway
  def charge(amount, card)
    raise NotImplementedError
  end
end

# Adaptee - legacy system with different interface
class LegacyPaymentSystem
  def process_payment(cents, card_number, expiry, cvv)
    # Legacy implementation
  end
end

# Adapter
class LegacyPaymentAdapter < PaymentGateway
  def initialize
    @legacy = LegacyPaymentSystem.new
  end

  def charge(amount, card)
    cents = (amount * 100).to_i
    @legacy.process_payment(
      cents,
      card[:number],
      card[:expiry],
      card[:cvv]
    )
  end
end

# Usage
gateway = LegacyPaymentAdapter.new
gateway.charge(99.99, { number: "4111...", expiry: "12/25", cvv: "123" })
```

**Rails Example:**
```ruby
# app/adapters/external_user_adapter.rb
class ExternalUserAdapter
  def initialize(external_data)
    @data = external_data
  end

  def to_user_params
    {
      email: @data["email_address"],
      first_name: @data["given_name"],
      last_name: @data["family_name"],
      phone: format_phone(@data["phone_number"]),
      external_id: @data["id"]
    }
  end

  private

  def format_phone(phone)
    phone&.gsub(/\D/, "")
  end
end

# Usage
external_data = ExternalApi.fetch_user(123)
adapter = ExternalUserAdapter.new(external_data)
User.create!(adapter.to_user_params)
```

---

### Bridge
**Problem:** Separate abstraction from implementation so both can vary independently.
**When to use:** When you want to avoid a permanent binding between abstraction and implementation.

```ruby
# Implementation interface
class Renderer
  def render_title(text)
    raise NotImplementedError
  end

  def render_body(text)
    raise NotImplementedError
  end
end

# Concrete implementations
class HtmlRenderer < Renderer
  def render_title(text)
    "<h1>#{text}</h1>"
  end

  def render_body(text)
    "<p>#{text}</p>"
  end
end

class MarkdownRenderer < Renderer
  def render_title(text)
    "# #{text}\n"
  end

  def render_body(text)
    "#{text}\n"
  end
end

# Abstraction
class Document
  def initialize(renderer)
    @renderer = renderer
  end

  def render
    raise NotImplementedError
  end
end

# Refined abstraction
class Article < Document
  attr_accessor :title, :body

  def render
    @renderer.render_title(@title) + @renderer.render_body(@body)
  end
end

# Usage
html_article = Article.new(HtmlRenderer.new)
html_article.title = "Hello"
html_article.body = "World"
html_article.render # => "<h1>Hello</h1><p>World</p>"

md_article = Article.new(MarkdownRenderer.new)
md_article.title = "Hello"
md_article.body = "World"
md_article.render # => "# Hello\nWorld\n"
```

---

### Composite
**Problem:** Compose objects into tree structures and treat individual objects and compositions uniformly.
**When to use:** When you need to represent part-whole hierarchies.

```ruby
# Component
class MenuComponent
  def add(component)
    raise NotImplementedError
  end

  def render
    raise NotImplementedError
  end
end

# Leaf
class MenuItem < MenuComponent
  attr_reader :name, :url

  def initialize(name, url)
    @name = name
    @url = url
  end

  def render
    "<a href='#{@url}'>#{@name}</a>"
  end
end

# Composite
class Menu < MenuComponent
  attr_reader :name

  def initialize(name)
    @name = name
    @children = []
  end

  def add(component)
    @children << component
  end

  def render
    html = "<ul class='menu'><li>#{@name}"
    html += "<ul>"
    @children.each { |child| html += "<li>#{child.render}</li>" }
    html += "</ul></li></ul>"
    html
  end
end

# Usage
main_menu = Menu.new("Main")
main_menu.add(MenuItem.new("Home", "/"))
main_menu.add(MenuItem.new("About", "/about"))

products_menu = Menu.new("Products")
products_menu.add(MenuItem.new("Software", "/products/software"))
products_menu.add(MenuItem.new("Hardware", "/products/hardware"))

main_menu.add(products_menu)
main_menu.render
```

**Rails Example:**
```ruby
# app/models/category.rb - Tree structure with acts_as_tree or closure_tree
class Category < ApplicationRecord
  has_many :children, class_name: "Category", foreign_key: "parent_id"
  belongs_to :parent, class_name: "Category", optional: true
  has_many :products

  def all_products
    Product.where(category_id: subtree_ids)
  end

  def subtree_ids
    [id] + children.flat_map(&:subtree_ids)
  end

  def render_tree(level = 0)
    indent = "  " * level
    output = "#{indent}- #{name}\n"
    children.each { |child| output += child.render_tree(level + 1) }
    output
  end
end
```

---

### Decorator
**Problem:** Add responsibilities to objects dynamically without altering their structure.
**When to use:** When you need to add behavior to objects without subclassing.

```ruby
# Component interface
class Coffee
  def cost
    raise NotImplementedError
  end

  def description
    raise NotImplementedError
  end
end

# Concrete component
class SimpleCoffee < Coffee
  def cost
    2.0
  end

  def description
    "Simple coffee"
  end
end

# Decorator base
class CoffeeDecorator < Coffee
  def initialize(coffee)
    @coffee = coffee
  end

  def cost
    @coffee.cost
  end

  def description
    @coffee.description
  end
end

# Concrete decorators
class Milk < CoffeeDecorator
  def cost
    @coffee.cost + 0.5
  end

  def description
    "#{@coffee.description}, milk"
  end
end

class Sugar < CoffeeDecorator
  def cost
    @coffee.cost + 0.2
  end

  def description
    "#{@coffee.description}, sugar"
  end
end

# Usage
coffee = SimpleCoffee.new
coffee = Milk.new(coffee)
coffee = Sugar.new(coffee)
coffee.cost # => 2.7
coffee.description # => "Simple coffee, milk, sugar"
```

**Rails Example:**
```ruby
# Using SimpleDelegator
class UserPresenter < SimpleDelegator
  def full_name
    "#{first_name} #{last_name}"
  end

  def formatted_created_at
    created_at.strftime("%B %d, %Y")
  end

  def avatar_url
    avatar.attached? ? avatar : "default_avatar.png"
  end
end

# Usage in view
presenter = UserPresenter.new(@user)
presenter.email # delegates to user
presenter.full_name # decorator method
```

---

### Facade
**Problem:** Provide a simplified interface to a complex subsystem.
**When to use:** When you need a simple interface to a complex set of classes.

```ruby
# Complex subsystem classes
class Inventory
  def check_stock(product_id)
    # Check inventory
  end

  def reserve(product_id, quantity)
    # Reserve stock
  end
end

class PaymentProcessor
  def authorize(amount, card)
    # Authorize payment
  end

  def capture(authorization_id)
    # Capture payment
  end
end

class ShippingService
  def calculate_cost(address, weight)
    # Calculate shipping
  end

  def create_shipment(order, address)
    # Create shipment
  end
end

class NotificationService
  def send_confirmation(user, order)
    # Send email
  end
end

# Facade
class OrderFacade
  def initialize
    @inventory = Inventory.new
    @payment = PaymentProcessor.new
    @shipping = ShippingService.new
    @notification = NotificationService.new
  end

  def place_order(user, cart, address, payment_info)
    # Check stock for all items
    cart.items.each do |item|
      unless @inventory.check_stock(item.product_id)
        raise OutOfStockError, item.name
      end
    end

    # Reserve inventory
    cart.items.each { |item| @inventory.reserve(item.product_id, item.quantity) }

    # Process payment
    auth = @payment.authorize(cart.total, payment_info)
    @payment.capture(auth.id)

    # Create shipment
    @shipping.create_shipment(order, address)

    # Notify user
    @notification.send_confirmation(user, order)

    order
  end
end

# Usage - simple interface to complex process
facade = OrderFacade.new
facade.place_order(current_user, cart, address, payment_info)
```

---

### Proxy
**Problem:** Provide a surrogate or placeholder for another object.
**When to use:** For lazy loading, access control, logging, or caching.

```ruby
# Subject interface
class Image
  def display
    raise NotImplementedError
  end
end

# Real subject
class RealImage < Image
  def initialize(filename)
    @filename = filename
    load_from_disk
  end

  def display
    puts "Displaying #{@filename}"
  end

  private

  def load_from_disk
    puts "Loading #{@filename} from disk..."
    sleep(2) # Expensive operation
  end
end

# Proxy - lazy loading
class ImageProxy < Image
  def initialize(filename)
    @filename = filename
    @real_image = nil
  end

  def display
    @real_image ||= RealImage.new(@filename)
    @real_image.display
  end
end

# Usage
image = ImageProxy.new("large_photo.jpg")
# Image not loaded yet
image.display # Loads and displays
image.display # Just displays (cached)
```

**Rails Example:**
```ruby
# Caching proxy
class CachedProductRepository
  def initialize(repository = ProductRepository.new)
    @repository = repository
    @cache = Rails.cache
  end

  def find(id)
    @cache.fetch("product:#{id}", expires_in: 1.hour) do
      @repository.find(id)
    end
  end

  def all
    @cache.fetch("products:all", expires_in: 15.minutes) do
      @repository.all
    end
  end

  def invalidate(id)
    @cache.delete("product:#{id}")
    @cache.delete("products:all")
  end
end
```

---

## Behavioral Patterns

### Chain of Responsibility
**Problem:** Pass requests along a chain of handlers.
**When to use:** When multiple objects may handle a request and the handler isn't known beforehand.

```ruby
class Handler
  attr_accessor :next_handler

  def handle(request)
    if can_handle?(request)
      process(request)
    elsif next_handler
      next_handler.handle(request)
    else
      default_handle(request)
    end
  end

  def can_handle?(request)
    raise NotImplementedError
  end

  def process(request)
    raise NotImplementedError
  end

  def default_handle(request)
    raise "No handler for request: #{request}"
  end
end

class SmallOrderHandler < Handler
  def can_handle?(request)
    request[:amount] < 100
  end

  def process(request)
    puts "Processing small order: $#{request[:amount]}"
    :approved
  end
end

class MediumOrderHandler < Handler
  def can_handle?(request)
    request[:amount] < 1000
  end

  def process(request)
    puts "Processing medium order: $#{request[:amount]} - needs review"
    :pending_review
  end
end

class LargeOrderHandler < Handler
  def can_handle?(request)
    request[:amount] >= 1000
  end

  def process(request)
    puts "Processing large order: $#{request[:amount]} - needs approval"
    :pending_approval
  end
end

# Setup chain
small = SmallOrderHandler.new
medium = MediumOrderHandler.new
large = LargeOrderHandler.new

small.next_handler = medium
medium.next_handler = large

# Usage
small.handle({ amount: 50 })    # Small handler
small.handle({ amount: 500 })   # Medium handler
small.handle({ amount: 5000 })  # Large handler
```

**Rails Example:**
```ruby
# Middleware-style chain
class AuthenticationMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    if authenticated?(env)
      @app.call(env)
    else
      [401, {}, ["Unauthorized"]]
    end
  end
end

# Or as service objects
class ValidationChain
  def self.build
    chain = RequiredFieldsValidator.new
    chain.next_handler = EmailFormatValidator.new
    chain.next_handler.next_handler = UniqueEmailValidator.new
    chain
  end
end
```

---

### Command
**Problem:** Encapsulate a request as an object.
**When to use:** For undo/redo, queuing operations, or logging requests.

```ruby
# Command interface
class Command
  def execute
    raise NotImplementedError
  end

  def undo
    raise NotImplementedError
  end
end

# Concrete commands
class AddItemCommand < Command
  def initialize(cart, item)
    @cart = cart
    @item = item
  end

  def execute
    @cart.add(@item)
  end

  def undo
    @cart.remove(@item)
  end
end

class RemoveItemCommand < Command
  def initialize(cart, item)
    @cart = cart
    @item = item
  end

  def execute
    @cart.remove(@item)
  end

  def undo
    @cart.add(@item)
  end
end

# Invoker with history
class CommandInvoker
  def initialize
    @history = []
  end

  def execute(command)
    command.execute
    @history << command
  end

  def undo
    return if @history.empty?
    command = @history.pop
    command.undo
  end
end

# Usage
invoker = CommandInvoker.new
invoker.execute(AddItemCommand.new(cart, product))
invoker.execute(AddItemCommand.new(cart, another_product))
invoker.undo # Removes another_product
```

**Rails Example:**
```ruby
# app/commands/create_order_command.rb
class CreateOrderCommand
  def initialize(user:, cart:, address:)
    @user = user
    @cart = cart
    @address = address
  end

  def execute
    ActiveRecord::Base.transaction do
      @order = Order.create!(
        user: @user,
        address: @address,
        total: @cart.total
      )

      @cart.items.each do |item|
        @order.order_items.create!(
          product: item.product,
          quantity: item.quantity,
          price: item.price
        )
      end

      @cart.clear
      OrderMailer.confirmation(@order).deliver_later
    end

    @order
  end

  def undo
    return unless @order
    @order.cancel!
  end
end
```

---

### Iterator
**Problem:** Access elements of a collection sequentially without exposing underlying representation.
**When to use:** When you need to traverse a collection in different ways.

```ruby
# Ruby's Enumerable module provides iterator pattern
class Library
  include Enumerable

  def initialize
    @books = []
  end

  def add(book)
    @books << book
  end

  def each(&block)
    @books.each(&block)
  end

  # Custom iterators
  def each_by_author(author, &block)
    @books.select { |b| b.author == author }.each(&block)
  end

  def each_published_after(year, &block)
    @books.select { |b| b.year > year }.each(&block)
  end
end

# Usage
library = Library.new
library.add(Book.new("1984", "Orwell", 1949))
library.add(Book.new("Brave New World", "Huxley", 1932))

library.each { |book| puts book.title }
library.map(&:title)
library.select { |b| b.year > 1940 }
library.each_by_author("Orwell") { |b| puts b.title }
```

**Rails Example:**
```ruby
# ActiveRecord already implements Enumerable
User.all.each { |user| process(user) }
User.find_each(batch_size: 1000) { |user| process(user) } # Memory efficient

# Custom iterator for API pagination
class PaginatedIterator
  include Enumerable

  def initialize(client, resource)
    @client = client
    @resource = resource
  end

  def each
    page = 1
    loop do
      response = @client.get(@resource, page: page)
      break if response.data.empty?

      response.data.each { |item| yield item }
      page += 1
    end
  end
end
```

---

### Mediator
**Problem:** Reduce chaotic dependencies between objects by having them communicate through a mediator.
**When to use:** When objects have complex many-to-many relationships.

```ruby
class ChatRoom
  def initialize
    @users = []
  end

  def register(user)
    @users << user
    user.chat_room = self
  end

  def send_message(sender, message, recipient = nil)
    if recipient
      # Private message
      recipient.receive(message, sender)
    else
      # Broadcast
      @users.each do |user|
        user.receive(message, sender) unless user == sender
      end
    end
  end
end

class User
  attr_reader :name
  attr_accessor :chat_room

  def initialize(name)
    @name = name
  end

  def send(message, recipient = nil)
    @chat_room.send_message(self, message, recipient)
  end

  def receive(message, sender)
    puts "#{@name} received from #{sender.name}: #{message}"
  end
end

# Usage
room = ChatRoom.new
alice = User.new("Alice")
bob = User.new("Bob")

room.register(alice)
room.register(bob)

alice.send("Hello everyone!") # Broadcast
alice.send("Hi Bob!", bob)    # Private
```

**Rails Example:**
```ruby
# Event-driven mediator using ActiveSupport::Notifications
class OrderMediator
  def self.setup
    ActiveSupport::Notifications.subscribe("order.placed") do |*args|
      event = ActiveSupport::Notifications::Event.new(*args)
      order = event.payload[:order]

      InventoryService.reserve(order)
      PaymentService.capture(order)
      NotificationService.confirm(order)
    end
  end
end

# In order creation
ActiveSupport::Notifications.instrument("order.placed", order: @order)
```

---

### Memento
**Problem:** Capture and restore an object's internal state.
**When to use:** For undo/redo, snapshots, or checkpoints.

```ruby
# Memento
class EditorMemento
  attr_reader :content, :cursor_position

  def initialize(content, cursor_position)
    @content = content.dup
    @cursor_position = cursor_position
  end
end

# Originator
class TextEditor
  attr_accessor :content, :cursor_position

  def initialize
    @content = ""
    @cursor_position = 0
  end

  def type(text)
    @content.insert(@cursor_position, text)
    @cursor_position += text.length
  end

  def save
    EditorMemento.new(@content, @cursor_position)
  end

  def restore(memento)
    @content = memento.content
    @cursor_position = memento.cursor_position
  end
end

# Caretaker
class EditorHistory
  def initialize
    @mementos = []
  end

  def push(memento)
    @mementos << memento
  end

  def pop
    @mementos.pop
  end
end

# Usage
editor = TextEditor.new
history = EditorHistory.new

editor.type("Hello")
history.push(editor.save)

editor.type(" World")
history.push(editor.save)

editor.type("!!!")
puts editor.content # => "Hello World!!!"

editor.restore(history.pop)
puts editor.content # => "Hello World"
```

**Rails Example:**
```ruby
# Using paper_trail gem or custom versioning
class Document < ApplicationRecord
  has_paper_trail

  def restore_version(version_id)
    version = versions.find(version_id)
    version.reify.save!
  end
end

# Or manual memento
class DocumentMemento < ApplicationRecord
  belongs_to :document
  serialize :state, coder: JSON
end

class Document < ApplicationRecord
  has_many :mementos, class_name: "DocumentMemento"

  def create_snapshot
    mementos.create!(state: attributes)
  end

  def restore_snapshot(memento)
    update!(memento.state.except("id", "created_at", "updated_at"))
  end
end
```

---

### Observer
**Problem:** Define a one-to-many dependency so that when one object changes state, all dependents are notified.
**When to use:** For event handling, notifications, or decoupled updates.

```ruby
# Using Ruby's Observable module
require 'observer'

class Stock
  include Observable

  attr_reader :name, :price

  def initialize(name, price)
    @name = name
    @price = price
  end

  def price=(new_price)
    @price = new_price
    changed
    notify_observers(self)
  end
end

class StockAlert
  def update(stock)
    puts "Alert: #{stock.name} is now $#{stock.price}"
  end
end

class StockLogger
  def update(stock)
    puts "[LOG] #{Time.now}: #{stock.name} changed to $#{stock.price}"
  end
end

# Usage
stock = Stock.new("AAPL", 150)
stock.add_observer(StockAlert.new)
stock.add_observer(StockLogger.new)

stock.price = 155 # Both observers notified
```

**Rails Example:**
```ruby
# ActiveRecord callbacks as observers
class Order < ApplicationRecord
  after_create :notify_observers

  private

  def notify_observers
    OrderCreatedJob.perform_later(id)
    SlackNotifier.order_placed(self)
    AnalyticsService.track("order_created", order_id: id)
  end
end

# Using ActiveSupport::Notifications
class Order < ApplicationRecord
  after_create do
    ActiveSupport::Notifications.instrument("order.created", order: self)
  end
end

# Subscribers
ActiveSupport::Notifications.subscribe("order.created") do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  OrderMailer.confirmation(event.payload[:order]).deliver_later
end
```

---

### State
**Problem:** Allow an object to alter its behavior when its internal state changes.
**When to use:** When an object's behavior depends on its state and must change at runtime.

```ruby
# State interface
class OrderState
  def process(order)
    raise NotImplementedError
  end

  def ship(order)
    raise NotImplementedError
  end

  def cancel(order)
    raise NotImplementedError
  end
end

# Concrete states
class PendingState < OrderState
  def process(order)
    puts "Processing payment..."
    order.state = PaidState.new
  end

  def ship(order)
    puts "Cannot ship - payment pending"
  end

  def cancel(order)
    puts "Order cancelled"
    order.state = CancelledState.new
  end
end

class PaidState < OrderState
  def process(order)
    puts "Already paid"
  end

  def ship(order)
    puts "Shipping order..."
    order.state = ShippedState.new
  end

  def cancel(order)
    puts "Refunding and cancelling..."
    order.state = CancelledState.new
  end
end

class ShippedState < OrderState
  def process(order)
    puts "Already processed and shipped"
  end

  def ship(order)
    puts "Already shipped"
  end

  def cancel(order)
    puts "Cannot cancel - already shipped"
  end
end

# Context
class Order
  attr_accessor :state

  def initialize
    @state = PendingState.new
  end

  def process
    @state.process(self)
  end

  def ship
    @state.ship(self)
  end

  def cancel
    @state.cancel(self)
  end
end

# Usage
order = Order.new
order.process  # => "Processing payment..."
order.ship     # => "Shipping order..."
order.cancel   # => "Cannot cancel - already shipped"
```

**Rails Example:**
```ruby
# Using AASM gem
class Order < ApplicationRecord
  include AASM

  aasm column: :status do
    state :pending, initial: true
    state :paid, :shipped, :delivered, :cancelled

    event :pay do
      transitions from: :pending, to: :paid
      after { OrderMailer.payment_received(self).deliver_later }
    end

    event :ship do
      transitions from: :paid, to: :shipped
      after { ShippingService.create_label(self) }
    end

    event :deliver do
      transitions from: :shipped, to: :delivered
    end

    event :cancel do
      transitions from: [:pending, :paid], to: :cancelled
      after { RefundService.process(self) if paid? }
    end
  end
end
```

---

### Strategy
**Problem:** Define a family of algorithms and make them interchangeable.
**When to use:** When you have multiple ways to do something and want to switch at runtime.

```ruby
# Strategy interface
class ShippingStrategy
  def calculate(package)
    raise NotImplementedError
  end
end

# Concrete strategies
class GroundShipping < ShippingStrategy
  def calculate(package)
    package.weight * 1.5
  end
end

class AirShipping < ShippingStrategy
  def calculate(package)
    package.weight * 3.0
  end
end

class ExpressShipping < ShippingStrategy
  def calculate(package)
    package.weight * 5.0 + 10.0
  end
end

# Context
class ShippingCalculator
  attr_accessor :strategy

  def initialize(strategy)
    @strategy = strategy
  end

  def calculate(package)
    @strategy.calculate(package)
  end
end

# Usage
package = OpenStruct.new(weight: 10)
calculator = ShippingCalculator.new(GroundShipping.new)
puts calculator.calculate(package) # => 15.0

calculator.strategy = ExpressShipping.new
puts calculator.calculate(package) # => 60.0
```

**Rails Example:**
```ruby
# app/strategies/pricing_strategy.rb
class PricingStrategy
  def self.for(user)
    case user.membership
    when "premium"
      PremiumPricing.new
    when "wholesale"
      WholesalePricing.new
    else
      StandardPricing.new
    end
  end
end

class StandardPricing
  def calculate(product)
    product.base_price
  end
end

class PremiumPricing
  def calculate(product)
    product.base_price * 0.9 # 10% discount
  end
end

class WholesalePricing
  def calculate(product)
    product.base_price * 0.7 # 30% discount
  end
end

# Usage in service
class CartService
  def total(user, cart)
    strategy = PricingStrategy.for(user)
    cart.items.sum { |item| strategy.calculate(item.product) * item.quantity }
  end
end
```

---

### Template Method
**Problem:** Define the skeleton of an algorithm and let subclasses override specific steps.
**When to use:** When you have a common algorithm with varying implementations.

```ruby
# Abstract class with template method
class DataExporter
  # Template method
  def export(data)
    validate(data)
    formatted = format(data)
    output = generate_output(formatted)
    save(output)
    notify
  end

  private

  def validate(data)
    raise ArgumentError, "Data cannot be empty" if data.empty?
  end

  def format(data)
    raise NotImplementedError
  end

  def generate_output(formatted)
    raise NotImplementedError
  end

  def save(output)
    File.write(filename, output)
  end

  def notify
    puts "Export completed: #{filename}"
  end

  def filename
    raise NotImplementedError
  end
end

# Concrete implementations
class CsvExporter < DataExporter
  def format(data)
    data.map { |row| row.values }
  end

  def generate_output(formatted)
    formatted.map { |row| row.join(",") }.join("\n")
  end

  def filename
    "export.csv"
  end
end

class JsonExporter < DataExporter
  def format(data)
    data
  end

  def generate_output(formatted)
    JSON.pretty_generate(formatted)
  end

  def filename
    "export.json"
  end
end

# Usage
data = [{ name: "Alice", age: 30 }, { name: "Bob", age: 25 }]
CsvExporter.new.export(data)
JsonExporter.new.export(data)
```

**Rails Example:**
```ruby
# app/services/base_importer.rb
class BaseImporter
  def import(file)
    records = parse(file)
    validate_records(records)

    imported = 0
    records.each do |record|
      next unless valid?(record)
      create_record(record)
      imported += 1
    end

    finalize(imported)
  end

  private

  def parse(file)
    raise NotImplementedError
  end

  def validate_records(records)
    raise ImportError, "No records found" if records.empty?
  end

  def valid?(record)
    true # Override in subclass
  end

  def create_record(record)
    raise NotImplementedError
  end

  def finalize(count)
    Rails.logger.info "Imported #{count} records"
  end
end

class UserImporter < BaseImporter
  private

  def parse(file)
    CSV.read(file, headers: true).map(&:to_h)
  end

  def valid?(record)
    record["email"].present?
  end

  def create_record(record)
    User.create!(
      email: record["email"],
      name: record["name"]
    )
  end
end
```

---

### Visitor
**Problem:** Add new operations to existing object structures without modifying them.
**When to use:** When you need to perform many unrelated operations on objects.

```ruby
# Visitor interface
class ShapeVisitor
  def visit_circle(circle)
    raise NotImplementedError
  end

  def visit_rectangle(rectangle)
    raise NotImplementedError
  end
end

# Concrete visitors
class AreaCalculator < ShapeVisitor
  def visit_circle(circle)
    Math::PI * circle.radius ** 2
  end

  def visit_rectangle(rectangle)
    rectangle.width * rectangle.height
  end
end

class DrawingVisitor < ShapeVisitor
  def visit_circle(circle)
    puts "Drawing circle with radius #{circle.radius}"
  end

  def visit_rectangle(rectangle)
    puts "Drawing rectangle #{rectangle.width}x#{rectangle.height}"
  end
end

# Elements
class Circle
  attr_reader :radius

  def initialize(radius)
    @radius = radius
  end

  def accept(visitor)
    visitor.visit_circle(self)
  end
end

class Rectangle
  attr_reader :width, :height

  def initialize(width, height)
    @width = width
    @height = height
  end

  def accept(visitor)
    visitor.visit_rectangle(self)
  end
end

# Usage
shapes = [Circle.new(5), Rectangle.new(4, 6), Circle.new(3)]

area_calc = AreaCalculator.new
total_area = shapes.sum { |shape| shape.accept(area_calc) }
puts "Total area: #{total_area}"

drawer = DrawingVisitor.new
shapes.each { |shape| shape.accept(drawer) }
```

**Rails Example:**
```ruby
# Reporting visitor for different model types
class ReportVisitor
  def visit_order(order)
    {
      type: "order",
      total: order.total,
      items: order.items.count
    }
  end

  def visit_refund(refund)
    {
      type: "refund",
      amount: refund.amount,
      reason: refund.reason
    }
  end

  def visit_subscription(subscription)
    {
      type: "subscription",
      plan: subscription.plan.name,
      mrr: subscription.monthly_amount
    }
  end
end

# Models include visitable concern
module Visitable
  def accept(visitor)
    method_name = "visit_#{self.class.name.underscore}"
    visitor.send(method_name, self)
  end
end

class Order < ApplicationRecord
  include Visitable
end

# Generate report
visitor = ReportVisitor.new
transactions = Order.all + Refund.all + Subscription.all
report = transactions.map { |t| t.accept(visitor) }
```

---

## Pattern Selection Guide

| Problem | Pattern |
|---------|---------|
| Create objects without specifying class | Factory Method |
| Create families of related objects | Abstract Factory |
| Construct complex objects step by step | Builder |
| Single instance with global access | Singleton |
| Clone existing objects | Prototype |
| Make incompatible interfaces work together | Adapter |
| Separate abstraction from implementation | Bridge |
| Tree structures with uniform interface | Composite |
| Add behavior dynamically | Decorator |
| Simplify complex subsystem | Facade |
| Control access, lazy loading, caching | Proxy |
| Pass request through handler chain | Chain of Responsibility |
| Encapsulate request as object | Command |
| Traverse collections uniformly | Iterator |
| Reduce dependencies between objects | Mediator |
| Capture and restore state | Memento |
| Notify dependents of changes | Observer |
| Behavior depends on state | State |
| Interchangeable algorithms | Strategy |
| Algorithm skeleton with customizable steps | Template Method |
| Add operations without modifying objects | Visitor |
