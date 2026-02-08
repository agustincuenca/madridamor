# Skill: Functional Programming in Ruby

## Purpose
Apply functional programming concepts in Ruby and Rails to write more predictable, testable, and maintainable code.

---

## Immutability

### Freeze Objects to Prevent Mutation
```ruby
# Mutable (dangerous)
CONFIG = { api_key: "secret", timeout: 30 }
CONFIG[:api_key] = "hacked"  # Oops, modified!

# Immutable (safe)
CONFIG = { api_key: "secret", timeout: 30 }.freeze
CONFIG[:api_key] = "hacked"  # => FrozenError

# Deep freeze for nested structures
module DeepFreeze
  def self.freeze(obj)
    case obj
    when Hash
      obj.each_value { |v| freeze(v) }
    when Array
      obj.each { |v| freeze(v) }
    end
    obj.freeze
  end
end

SETTINGS = DeepFreeze.freeze({
  database: { host: "localhost", port: 5432 },
  features: ["auth", "api"]
})
```

### Check if Object is Frozen
```ruby
string = "hello"
string.frozen?  # => false

string.freeze
string.frozen?  # => true

# Many Ruby methods return new objects instead of mutating
original = "hello"
uppercase = original.upcase  # Returns new string
original  # => "hello" (unchanged)

# Methods ending with ! typically mutate
original = "hello"
original.upcase!  # Mutates in place
original  # => "HELLO"
```

### Prefer Non-Mutating Methods
```ruby
# Mutating (avoid when possible)
array = [3, 1, 2]
array.sort!    # Mutates array
array.reverse! # Mutates array

# Non-mutating (preferred)
array = [3, 1, 2]
sorted = array.sort     # Returns new array
reversed = array.reverse # Returns new array
array # => [3, 1, 2] (unchanged)

# String methods that don't mutate
name = "john doe"
formatted = name.split.map(&:capitalize).join(" ")
# name is still "john doe"

# Rails: dup for ActiveRecord
user = User.find(1)
draft = user.dup
draft.name = "New Name"
# user.name unchanged
```

### Immutable Data Structures with Structs
```ruby
# Regular struct is mutable
Point = Struct.new(:x, :y)
point = Point.new(1, 2)
point.x = 10  # Mutable

# Immutable with Data class (Ruby 3.2+)
Point = Data.define(:x, :y)
point = Point.new(x: 1, y: 2)
point.x = 10  # => NoMethodError

# Create modified copy
new_point = point.with(x: 10)  # => Point(x: 10, y: 2)
point  # => Point(x: 1, y: 2) (unchanged)

# For older Ruby, use custom immutable struct
class ImmutablePoint
  attr_reader :x, :y

  def initialize(x, y)
    @x = x
    @y = y
    freeze
  end

  def with(x: @x, y: @y)
    self.class.new(x, y)
  end
end
```

---

## Pure Functions

### Definition: Same Input Always Produces Same Output, No Side Effects
```ruby
# Impure - depends on external state
@tax_rate = 0.1

def calculate_total(price)
  price * (1 + @tax_rate)  # Depends on @tax_rate
end

# Impure - modifies external state
def process_order(order)
  @order_count += 1  # Side effect
  order.save         # Side effect (database)
  send_email(order)  # Side effect (I/O)
end

# Pure - no dependencies, no side effects
def calculate_total(price, tax_rate)
  price * (1 + tax_rate)
end

# Always returns same result for same input
calculate_total(100, 0.1)  # => 110.0
calculate_total(100, 0.1)  # => 110.0 (always)
```

### Benefits of Pure Functions
```ruby
# 1. Easy to test - no setup required
def add(a, b)
  a + b
end

# Test
expect(add(2, 3)).to eq(5)  # Simple, no mocks

# 2. Easy to reason about
def full_name(first, last)
  "#{first} #{last}"
end

# 3. Cacheable (memoization)
def fibonacci(n)
  return n if n <= 1
  fibonacci(n - 1) + fibonacci(n - 2)
end

# 4. Parallelizable - no shared state
results = items.map { |item| pure_transform(item) }
# Can safely run in parallel

# 5. Composable
def double(x); x * 2; end
def add_one(x); x + 1; end

double(add_one(5))  # => 12
```

### Pushing Side Effects to the Edges
```ruby
# Bad - side effects mixed with logic
class OrderProcessor
  def process(order)
    return if order.items.empty?

    total = order.items.sum(&:price)
    tax = total * 0.1
    shipping = total > 100 ? 0 : 10
    final_total = total + tax + shipping

    order.update!(total: final_total)  # Side effect mixed in
    OrderMailer.confirmation(order).deliver_later  # Side effect
    InventoryService.reduce_stock(order.items)  # Side effect

    final_total
  end
end

# Good - pure calculation, side effects at edges
class OrderCalculator
  # Pure function - no side effects
  def calculate(items, tax_rate: 0.1, free_shipping_threshold: 100)
    return nil if items.empty?

    subtotal = items.sum(&:price)
    tax = subtotal * tax_rate
    shipping = subtotal > free_shipping_threshold ? 0 : 10

    {
      subtotal: subtotal,
      tax: tax,
      shipping: shipping,
      total: subtotal + tax + shipping
    }
  end
end

class OrderProcessor
  def initialize(calculator: OrderCalculator.new)
    @calculator = calculator
  end

  def process(order)
    # Pure calculation
    totals = @calculator.calculate(order.items)
    return false unless totals

    # Side effects at the edge
    persist_order(order, totals)
    send_notifications(order)
    update_inventory(order)

    true
  end

  private

  def persist_order(order, totals)
    order.update!(totals)
  end

  def send_notifications(order)
    OrderMailer.confirmation(order).deliver_later
  end

  def update_inventory(order)
    InventoryService.reduce_stock(order.items)
  end
end
```

---

## Higher-Order Functions

### Functions That Receive Functions
```ruby
# map - transform each element
numbers = [1, 2, 3, 4, 5]
doubled = numbers.map { |n| n * 2 }  # => [2, 4, 6, 8, 10]

# select/filter - keep elements matching condition
evens = numbers.select { |n| n.even? }  # => [2, 4]

# reject - remove elements matching condition
odds = numbers.reject { |n| n.even? }  # => [1, 3, 5]

# reduce/inject - combine elements into single value
sum = numbers.reduce(0) { |acc, n| acc + n }  # => 15
sum = numbers.reduce(:+)  # Shorthand => 15

# find/detect - first element matching condition
first_even = numbers.find { |n| n.even? }  # => 2

# any? - at least one matches
numbers.any? { |n| n > 4 }  # => true

# all? - all match
numbers.all? { |n| n > 0 }  # => true

# none? - none match
numbers.none? { |n| n > 10 }  # => true

# count - count matching elements
numbers.count { |n| n.even? }  # => 2

# partition - split into two arrays
evens, odds = numbers.partition(&:even?)
# evens => [2, 4], odds => [1, 3, 5]

# group_by - group into hash
grouped = numbers.group_by { |n| n.even? ? :even : :odd }
# => { odd: [1, 3, 5], even: [2, 4] }

# sort_by - sort by transformation
users.sort_by { |u| u.created_at }
users.sort_by(&:created_at)  # Shorthand
```

### Functions That Return Functions
```ruby
# Factory for validators
def minimum_length_validator(min)
  ->(string) { string.length >= min }
end

validate_password = minimum_length_validator(8)
validate_password.call("hello")      # => false
validate_password.call("hello123!")  # => true

# Factory for comparators
def compare_by(attribute)
  ->(a, b) { a.send(attribute) <=> b.send(attribute) }
end

users.sort(&compare_by(:name))
users.sort(&compare_by(:created_at))

# Decorator/wrapper functions
def with_logging(func)
  ->(*args) do
    puts "Calling with args: #{args}"
    result = func.call(*args)
    puts "Result: #{result}"
    result
  end
end

add = ->(a, b) { a + b }
logged_add = with_logging(add)
logged_add.call(2, 3)
# Output:
# Calling with args: [2, 3]
# Result: 5
```

---

## Blocks, Procs, and Lambdas

### Blocks
```ruby
# Implicit block with yield
def with_timing
  start = Time.now
  result = yield
  elapsed = Time.now - start
  puts "Took #{elapsed} seconds"
  result
end

with_timing { sleep(1); "done" }

# Explicit block with &block
def transform(items, &block)
  items.map(&block)
end

transform([1, 2, 3]) { |n| n * 2 }  # => [2, 4, 6]

# Check if block given
def maybe_yield
  if block_given?
    yield
  else
    "No block given"
  end
end
```

### Procs
```ruby
# Create a Proc
double = Proc.new { |x| x * 2 }
double = proc { |x| x * 2 }  # Shorthand

# Call a Proc
double.call(5)  # => 10
double.(5)      # => 10
double[5]       # => 10

# Procs are lenient with arguments
flexible = proc { |a, b| [a, b] }
flexible.call(1)        # => [1, nil]
flexible.call(1, 2, 3)  # => [1, 2]

# Procs return from enclosing method
def test_proc
  my_proc = proc { return "from proc" }
  my_proc.call
  "after proc"  # Never reached
end
test_proc  # => "from proc"
```

### Lambdas
```ruby
# Create a lambda
double = lambda { |x| x * 2 }
double = ->(x) { x * 2 }  # Stabby lambda

# Call a lambda
double.call(5)  # => 10
double.(5)      # => 10
double[5]       # => 10

# Lambdas are strict with arguments
strict = ->(a, b) { [a, b] }
strict.call(1)        # => ArgumentError
strict.call(1, 2, 3)  # => ArgumentError
strict.call(1, 2)     # => [1, 2]

# Lambdas return to caller
def test_lambda
  my_lambda = -> { return "from lambda" }
  my_lambda.call
  "after lambda"  # This IS reached
end
test_lambda  # => "after lambda"

# Lambdas with multiple arguments
calculate = ->(a, b, operation) { operation.call(a, b) }
add = ->(x, y) { x + y }
multiply = ->(x, y) { x * y }

calculate.call(2, 3, add)       # => 5
calculate.call(2, 3, multiply)  # => 6
```

### When to Use Each
```ruby
# Use blocks for:
# - One-time use, inline code
# - Iterators (each, map, select)
[1, 2, 3].map { |n| n * 2 }

# Use Procs for:
# - Storing blocks for later
# - When you need lenient argument handling
# - When you want return to exit enclosing method
callback = proc { |result| handle_result(result) }

# Use Lambdas for:
# - Strict argument checking
# - First-class functions
# - Functional programming patterns
validator = ->(value) { value.present? && value.length > 3 }
```

### Symbol to Proc
```ruby
# Long form
users.map { |user| user.name }

# Short form with &
users.map(&:name)

# How it works:
# &:name calls :name.to_proc
# :name.to_proc returns a proc that calls .name on its argument

# Custom to_proc
class Integer
  def to_proc
    ->(obj) { obj * self }
  end
end

[1, 2, 3].map(&2)  # => [2, 4, 6]
```

---

## Method Chaining

### Building Fluent Interfaces
```ruby
# ActiveRecord is a great example
User
  .where(active: true)
  .where("created_at > ?", 1.month.ago)
  .includes(:orders)
  .order(created_at: :desc)
  .limit(10)

# Custom chainable query builder
class ProductQuery
  def initialize(scope = Product.all)
    @scope = scope
  end

  def active
    chain { @scope.where(active: true) }
  end

  def in_category(category)
    chain { @scope.where(category: category) }
  end

  def price_between(min, max)
    chain { @scope.where(price: min..max) }
  end

  def search(query)
    return self if query.blank?
    chain { @scope.where("name ILIKE ?", "%#{query}%") }
  end

  def to_a
    @scope.to_a
  end

  def count
    @scope.count
  end

  private

  def chain
    self.class.new(yield)
  end
end

# Usage
products = ProductQuery.new
  .active
  .in_category("Electronics")
  .price_between(100, 500)
  .search(params[:q])
  .to_a
```

### Tap for Side Effects in Chains
```ruby
# tap yields self and returns self
User.new
  .tap { |u| u.name = "John" }
  .tap { |u| u.email = "john@example.com" }
  .tap { |u| puts "Created user: #{u.name}" }
  .save

# Useful for debugging chains
users
  .select(&:active?)
  .tap { |list| puts "Active users: #{list.count}" }
  .map(&:email)
  .tap { |emails| puts "Emails: #{emails}" }
  .join(", ")

# then/yield_self for transformations
"hello"
  .then { |s| s.upcase }
  .then { |s| "#{s}!" }
  # => "HELLO!"
```

---

## Currying and Partial Application

### Currying
Transforms a function that takes multiple arguments into a sequence of functions that each take a single argument.

```ruby
# Regular function
add = ->(a, b, c) { a + b + c }
add.call(1, 2, 3)  # => 6

# Curried function
curried_add = add.curry
curried_add.call(1).call(2).call(3)  # => 6
curried_add.(1).(2).(3)              # => 6

# Partial application with currying
add_one = curried_add.(1)      # Returns a lambda waiting for 2 more args
add_one_and_two = add_one.(2)  # Returns a lambda waiting for 1 more arg
add_one_and_two.(3)            # => 6

# Practical example
multiply = ->(x, y) { x * y }.curry

double = multiply.(2)
triple = multiply.(3)

[1, 2, 3, 4].map(&double)  # => [2, 4, 6, 8]
[1, 2, 3, 4].map(&triple)  # => [3, 6, 9, 12]
```

### Partial Application
```ruby
# Using curry for partial application
greet = ->(greeting, name) { "#{greeting}, #{name}!" }
say_hello = greet.curry.("Hello")
say_goodbye = greet.curry.("Goodbye")

say_hello.("World")   # => "Hello, World!"
say_goodbye.("World") # => "Goodbye, World!"

# Manual partial application
def partial(func, *fixed_args)
  ->(*args) { func.call(*fixed_args, *args) }
end

log = ->(level, message) { puts "[#{level}] #{message}" }
info = partial(log, "INFO")
error = partial(log, "ERROR")

info.("Application started")  # => [INFO] Application started
error.("Something went wrong") # => [ERROR] Something went wrong
```

---

## Function Composition

### Composing Functions
```ruby
# Manual composition
double = ->(x) { x * 2 }
add_one = ->(x) { x + 1 }

# Compose: add_one after double
composed = ->(x) { add_one.(double.(x)) }
composed.(5)  # => 11 (5 * 2 + 1)

# Compose helper
def compose(*functions)
  ->(arg) { functions.reverse.reduce(arg) { |result, fn| fn.(result) } }
end

pipeline = compose(add_one, double, add_one)
pipeline.(5)  # => 13 ((5 + 1) * 2 + 1)

# Using >> and << operators (Ruby 2.6+)
double = ->(x) { x * 2 }
add_one = ->(x) { x + 1 }

# >> composes left to right
(double >> add_one).call(5)  # => 11 (5 * 2, then + 1)

# << composes right to left
(double << add_one).call(5)  # => 12 (5 + 1, then * 2)

# Chain multiple
square = ->(x) { x ** 2 }
pipeline = double >> add_one >> square
pipeline.(3)  # => 49 ((3 * 2 + 1) ** 2)
```

### Practical Composition in Rails
```ruby
# Compose transformations
class DataPipeline
  def initialize
    @steps = []
  end

  def add_step(step)
    @steps << step
    self
  end

  def process(data)
    @steps.reduce(data) { |result, step| step.call(result) }
  end
end

# Define transformation steps
strip_whitespace = ->(data) { data.transform_values(&:strip) }
downcase_email = ->(data) { data.merge(email: data[:email]&.downcase) }
validate = ->(data) { data[:email]&.include?("@") ? data : raise("Invalid email") }
normalize_phone = ->(data) { data.merge(phone: data[:phone]&.gsub(/\D/, "")) }

# Build pipeline
pipeline = DataPipeline.new
  .add_step(strip_whitespace)
  .add_step(downcase_email)
  .add_step(normalize_phone)
  .add_step(validate)

# Use
clean_data = pipeline.process({
  name: "  John  ",
  email: "  JOHN@EXAMPLE.COM  ",
  phone: "(555) 123-4567"
})
# => { name: "John", email: "john@example.com", phone: "5551234567" }
```

---

## Lazy Evaluation

### Enumerator::Lazy
```ruby
# Eager - processes all elements immediately
(1..Float::INFINITY)
  .select { |n| n.even? }
  .first(5)
# => Never finishes! (infinite sequence)

# Lazy - processes elements on demand
(1..Float::INFINITY)
  .lazy
  .select { |n| n.even? }
  .first(5)
# => [2, 4, 6, 8, 10]

# Useful for large files
File.open("huge_file.csv")
  .lazy
  .map { |line| line.split(",") }
  .select { |row| row[0] == "active" }
  .first(100)
  # Only reads enough lines to get 100 matches

# Chaining lazy operations
numbers = (1..1_000_000).lazy

result = numbers
  .select { |n| n % 3 == 0 }      # Divisible by 3
  .map { |n| n * 2 }               # Double it
  .select { |n| n.to_s.include?("7") }  # Contains 7
  .first(10)
# Only processes what's needed to find 10 matches
```

### Custom Lazy Enumerators
```ruby
# Lazy API paginator
class LazyApiPaginator
  include Enumerable

  def initialize(client, resource)
    @client = client
    @resource = resource
  end

  def each
    return to_enum(:each).lazy unless block_given?

    page = 1
    loop do
      response = @client.get(@resource, page: page)
      break if response.data.empty?

      response.data.each { |item| yield item }
      page += 1
    end
  end
end

# Usage - only fetches pages as needed
paginator = LazyApiPaginator.new(api_client, "/users")
paginator.lazy.select { |u| u["active"] }.first(10)
# Stops fetching pages once we have 10 active users
```

### Memoization as Lazy Evaluation
```ruby
# Using ||= for lazy initialization
class ExpensiveCalculator
  def result
    @result ||= calculate_expensive_value
  end

  private

  def calculate_expensive_value
    puts "Calculating..."
    sleep(2)
    42
  end
end

calc = ExpensiveCalculator.new
calc.result  # Calculates (takes 2 seconds)
calc.result  # Returns cached value (instant)
```

---

## Functional Patterns in Rails

### Service Objects as Pure Functions
```ruby
class CalculateShipping
  def self.call(weight:, destination:, expedited: false)
    new(weight, destination, expedited).call
  end

  def initialize(weight, destination, expedited)
    @weight = weight
    @destination = destination
    @expedited = expedited
  end

  def call
    base_rate = calculate_base_rate
    distance_factor = calculate_distance_factor
    expedited_factor = @expedited ? 1.5 : 1.0

    (base_rate * distance_factor * expedited_factor).round(2)
  end

  private

  def calculate_base_rate
    case @weight
    when 0..1 then 5.0
    when 1..5 then 10.0
    when 5..20 then 20.0
    else 30.0 + (@weight - 20) * 1.5
    end
  end

  def calculate_distance_factor
    DISTANCE_FACTORS.fetch(@destination, 1.0)
  end

  DISTANCE_FACTORS = {
    local: 1.0,
    regional: 1.25,
    national: 1.5,
    international: 2.5
  }.freeze
end

# Pure - same inputs always produce same output
CalculateShipping.call(weight: 3, destination: :regional)  # => 12.5
```

### Transformation Pipelines
```ruby
class ImportUsersPipeline
  TRANSFORMATIONS = [
    :parse_csv,
    :normalize_data,
    :validate_records,
    :deduplicate,
    :transform_to_users
  ].freeze

  def call(file_content)
    TRANSFORMATIONS.reduce(file_content) do |data, transformation|
      send(transformation, data)
    end
  end

  private

  def parse_csv(content)
    CSV.parse(content, headers: true).map(&:to_h)
  end

  def normalize_data(records)
    records.map do |record|
      record.transform_keys(&:downcase)
            .transform_values(&:strip)
    end
  end

  def validate_records(records)
    records.select { |r| r["email"].present? }
  end

  def deduplicate(records)
    records.uniq { |r| r["email"].downcase }
  end

  def transform_to_users(records)
    records.map { |r| UserBuilder.from_hash(r) }
  end
end
```

### Functional Error Handling with Result Objects
```ruby
class Result
  attr_reader :value, :error

  def self.success(value)
    new(value: value, success: true)
  end

  def self.failure(error)
    new(error: error, success: false)
  end

  def initialize(value: nil, error: nil, success:)
    @value = value
    @error = error
    @success = success
  end

  def success?
    @success
  end

  def failure?
    !@success
  end

  # Monadic bind/flatmap
  def and_then(&block)
    return self if failure?
    block.call(@value)
  end

  # Functor map
  def map(&block)
    return self if failure?
    Result.success(block.call(@value))
  end

  # Handle error
  def or_else(&block)
    return self if success?
    block.call(@error)
  end
end

# Usage with chaining
class RegisterUser
  def call(params)
    validate(params)
      .and_then { |p| check_uniqueness(p) }
      .and_then { |p| create_user(p) }
      .and_then { |user| send_welcome_email(user) }
  end

  private

  def validate(params)
    if params[:email].present? && params[:password].length >= 8
      Result.success(params)
    else
      Result.failure("Invalid params")
    end
  end

  def check_uniqueness(params)
    if User.exists?(email: params[:email])
      Result.failure("Email already taken")
    else
      Result.success(params)
    end
  end

  def create_user(params)
    user = User.create(params)
    if user.persisted?
      Result.success(user)
    else
      Result.failure(user.errors.full_messages.join(", "))
    end
  end

  def send_welcome_email(user)
    UserMailer.welcome(user).deliver_later
    Result.success(user)
  end
end

# In controller
result = RegisterUser.new.call(user_params)

if result.success?
  redirect_to dashboard_path, notice: "Welcome!"
else
  flash.now[:alert] = result.error
  render :new
end
```
