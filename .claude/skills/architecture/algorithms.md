# Skill: Algorithms and Data Structures

## Purpose
Understand algorithmic complexity and choose appropriate data structures for efficient Ruby and Rails applications.

---

## Big O Notation

### Common Complexities
```
O(1)       - Constant    - Hash lookup, array access by index
O(log n)   - Logarithmic - Binary search, balanced tree operations
O(n)       - Linear      - Array iteration, linear search
O(n log n) - Linearithmic - Efficient sorting (merge sort, quick sort)
O(n²)      - Quadratic   - Nested loops, bubble sort
O(2ⁿ)      - Exponential - Recursive fibonacci (naive), power set
O(n!)      - Factorial   - Permutations, traveling salesman (brute force)
```

### Visual Comparison
```
n=10:    O(1)=1    O(log n)=3    O(n)=10    O(n log n)=33    O(n²)=100
n=100:   O(1)=1    O(log n)=7    O(n)=100   O(n log n)=664   O(n²)=10,000
n=1000:  O(1)=1    O(log n)=10   O(n)=1000  O(n log n)=9,966 O(n²)=1,000,000
```

### Examples in Ruby
```ruby
# O(1) - Constant time
hash = { a: 1, b: 2, c: 3 }
hash[:b]          # Hash lookup
array[5]          # Array access by index
array.push(item)  # Array append
array.pop         # Array pop

# O(log n) - Logarithmic
sorted_array.bsearch { |x| x >= target }  # Binary search
# Each step cuts the search space in half

# O(n) - Linear
array.each { |item| process(item) }  # Iterate all elements
array.include?(item)                  # Linear search
array.find { |x| x == target }       # Find first match
array.max                             # Find maximum
array.sum                             # Sum all elements

# O(n log n) - Linearithmic
array.sort                # Sorting
array.sort_by { |x| x.name }

# O(n²) - Quadratic
array.each do |a|
  array.each do |b|       # Nested loop
    compare(a, b)
  end
end

# O(2ⁿ) - Exponential (avoid!)
def fibonacci(n)
  return n if n <= 1
  fibonacci(n - 1) + fibonacci(n - 2)  # Each call spawns 2 more
end
```

### Analyzing Complexity
```ruby
# Rule 1: Drop constants
# O(2n) -> O(n)
# O(n/2) -> O(n)

# Rule 2: Drop non-dominant terms
# O(n² + n) -> O(n²)
# O(n + log n) -> O(n)

# Rule 3: Different inputs = different variables
def compare_lists(list_a, list_b)
  list_a.each do |a|           # O(a)
    list_b.each do |b|         # O(b)
      puts a == b
    end
  end
end
# Total: O(a * b), not O(n²)

# Rule 4: Sequential operations add
def process(array)
  array.each { |x| step1(x) }  # O(n)
  array.each { |x| step2(x) }  # O(n)
end
# Total: O(n + n) = O(2n) = O(n)

# Rule 5: Nested operations multiply
def process(array)
  array.each do |x|            # O(n)
    array.each { |y| work(x, y) }  # O(n)
  end
end
# Total: O(n * n) = O(n²)
```

---

## Arrays vs Hashes

### When to Use Arrays
```ruby
# Arrays are best for:
# - Ordered collections
# - Accessing by position
# - Iterating in sequence
# - Small collections with simple lookups

# Array operations and complexity
array = [1, 2, 3, 4, 5]

array[0]          # O(1) - Access by index
array.push(6)     # O(1) - Append
array.pop         # O(1) - Remove last
array.shift       # O(n) - Remove first (shifts all elements)
array.unshift(0)  # O(n) - Add to front (shifts all elements)
array.include?(3) # O(n) - Search
array.index(3)    # O(n) - Find index
array.insert(2, x) # O(n) - Insert at position

# Good use case: maintaining order
recent_items = []
recent_items.push(item)
recent_items.shift if recent_items.size > 10  # Keep last 10
```

### When to Use Hashes
```ruby
# Hashes are best for:
# - Key-value lookups
# - Checking membership
# - Counting occurrences
# - Caching/memoization

# Hash operations and complexity
hash = { a: 1, b: 2, c: 3 }

hash[:a]           # O(1) - Lookup
hash[:d] = 4       # O(1) - Insert
hash.delete(:a)    # O(1) - Delete
hash.key?(:b)      # O(1) - Check key exists
hash.value?(2)     # O(n) - Check value exists (must scan all)
hash.keys          # O(n) - Get all keys
hash.values        # O(n) - Get all values

# Good use case: counting
def count_words(text)
  counts = Hash.new(0)
  text.split.each { |word| counts[word] += 1 }
  counts
end

# Good use case: caching
def expensive_lookup(id)
  @cache ||= {}
  @cache[id] ||= Database.find(id)
end

# Good use case: checking membership
allowed_ids = { 1 => true, 2 => true, 3 => true }
allowed_ids[user_id]  # O(1) vs array.include? O(n)
```

### Converting Between Them
```ruby
# Array to Hash
array = [[:a, 1], [:b, 2]]
hash = array.to_h  # => { a: 1, b: 2 }

# Array to Hash (for membership checking)
ids = [1, 2, 3, 4, 5]
id_set = ids.to_h { |id| [id, true] }
id_set[3]  # O(1) lookup

# Hash to Array
hash = { a: 1, b: 2 }
hash.to_a     # => [[:a, 1], [:b, 2]]
hash.keys     # => [:a, :b]
hash.values   # => [1, 2]
```

---

## Sorting

### Built-in Sorting
```ruby
# sort - uses <=> operator
[3, 1, 4, 1, 5].sort  # => [1, 1, 3, 4, 5]
["banana", "apple"].sort  # => ["apple", "banana"]

# sort with custom comparator
[3, 1, 4].sort { |a, b| b <=> a }  # Descending: [4, 3, 1]

# sort_by - more efficient for complex keys
users.sort_by { |u| u.name }
users.sort_by(&:name)  # Shorthand

# sort_by with multiple keys
users.sort_by { |u| [u.last_name, u.first_name] }

# Descending sort_by
users.sort_by { |u| -u.age }  # Numeric
users.sort_by { |u| u.name }.reverse  # Any type

# In-place sorting (mutates)
array.sort!
array.sort_by!(&:name)
```

### Custom Comparators
```ruby
# The <=> operator returns:
# -1 if a < b
#  0 if a == b
#  1 if a > b

class Person
  include Comparable

  attr_reader :name, :age

  def initialize(name, age)
    @name = name
    @age = age
  end

  def <=>(other)
    age <=> other.age
  end
end

people = [Person.new("Bob", 30), Person.new("Alice", 25)]
people.sort  # Sorted by age

# Complex sorting
products.sort do |a, b|
  # First by category, then by price descending
  comparison = a.category <=> b.category
  comparison = b.price <=> a.price if comparison == 0
  comparison
end
```

### Sorting Stability
```ruby
# Ruby's sort is NOT stable (equal elements may change order)
# Use sort_by for stable-ish sorting or add a tiebreaker

# Stable sort workaround
users_with_index = users.each_with_index.map { |u, i| [u, i] }
sorted = users_with_index.sort_by { |u, i| [u.name, i] }
result = sorted.map(&:first)
```

---

## Searching

### Linear Search
```ruby
# O(n) - checks each element
array = [3, 1, 4, 1, 5, 9, 2, 6]

# find - returns first match or nil
array.find { |x| x > 4 }  # => 5

# find_index/index - returns index of first match
array.find_index { |x| x > 4 }  # => 4
array.index(4)  # => 2

# include? - checks presence
array.include?(5)  # => true

# any?/all?/none?
array.any? { |x| x > 8 }   # => true
array.all? { |x| x > 0 }   # => true
array.none? { |x| x > 10 } # => true
```

### Binary Search
```ruby
# O(log n) - requires sorted array
sorted = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# bsearch - returns element
sorted.bsearch { |x| x >= 5 }   # => 5
sorted.bsearch { |x| x >= 5.5 } # => 6

# bsearch_index - returns index
sorted.bsearch_index { |x| x >= 5 }  # => 4

# Find exact match
sorted.bsearch { |x| x <=> 5 }  # => 5 (or nil if not found)

# Custom binary search
def binary_search(array, target)
  low = 0
  high = array.length - 1

  while low <= high
    mid = (low + high) / 2
    case array[mid] <=> target
    when 0 then return mid
    when -1 then low = mid + 1
    when 1 then high = mid - 1
    end
  end

  nil  # Not found
end
```

### Search in Hashes and Sets
```ruby
# Hash - O(1) average
users_by_email = users.index_by(&:email)
user = users_by_email["john@example.com"]  # O(1)

# Set - O(1) for membership
require 'set'
valid_ids = Set.new([1, 2, 3, 4, 5])
valid_ids.include?(3)  # O(1)

# vs Array.include? which is O(n)
```

---

## Recursion

### Basic Recursion
```ruby
# Every recursive function needs:
# 1. Base case - when to stop
# 2. Recursive case - how to break down the problem

# Factorial: n! = n * (n-1) * (n-2) * ... * 1
def factorial(n)
  return 1 if n <= 1          # Base case
  n * factorial(n - 1)        # Recursive case
end

factorial(5)  # => 120

# Fibonacci (naive - O(2^n), don't use!)
def fibonacci(n)
  return n if n <= 1           # Base case
  fibonacci(n - 1) + fibonacci(n - 2)  # Two recursive calls!
end

# Sum of array
def sum(array)
  return 0 if array.empty?    # Base case
  array.first + sum(array[1..])  # Recursive case
end
```

### Tail Recursion
```ruby
# Regular recursion - builds up call stack
def factorial(n)
  return 1 if n <= 1
  n * factorial(n - 1)  # Must wait for recursive call to complete
end
# Stack: factorial(5) -> factorial(4) -> factorial(3) -> ...

# Tail recursion - last operation is the recursive call
def factorial_tail(n, accumulator = 1)
  return accumulator if n <= 1
  factorial_tail(n - 1, n * accumulator)  # Tail position
end
# Can be optimized to O(1) space (Ruby doesn't do this automatically)

# Enable tail call optimization (experimental)
RubyVM::InstructionSequence.compile_option = {
  tailcall_optimization: true,
  trace_instruction: false
}
```

### Converting Recursion to Iteration
```ruby
# Recursive
def sum_recursive(array)
  return 0 if array.empty?
  array.first + sum_recursive(array[1..])
end

# Iterative (usually more efficient in Ruby)
def sum_iterative(array)
  total = 0
  array.each { |x| total += x }
  total
end

# Or simply
array.sum

# Recursive tree traversal
def traverse_recursive(node)
  return if node.nil?
  process(node)
  node.children.each { |child| traverse_recursive(child) }
end

# Iterative with stack
def traverse_iterative(root)
  stack = [root]
  while stack.any?
    node = stack.pop
    process(node)
    stack.concat(node.children.reverse)
  end
end
```

---

## Memoization

### Using ||= for Simple Memoization
```ruby
class ExpensiveCalculator
  def result
    @result ||= calculate  # Only calculates once
  end

  # Handle nil and false
  def value
    return @value if defined?(@value)
    @value = calculate_value
  end

  private

  def calculate
    sleep(2)  # Expensive operation
    42
  end
end

# Memoizing methods with arguments
class Fibonacci
  def initialize
    @cache = {}
  end

  def calculate(n)
    @cache[n] ||= begin
      return n if n <= 1
      calculate(n - 1) + calculate(n - 2)
    end
  end
end

fib = Fibonacci.new
fib.calculate(100)  # Fast! O(n) instead of O(2^n)
```

### Using Rails.cache
```ruby
# Simple caching
def expensive_query
  Rails.cache.fetch("expensive_query", expires_in: 1.hour) do
    User.includes(:orders, :subscriptions).where(active: true).to_a
  end
end

# Cache with key based on data
def user_stats(user)
  Rails.cache.fetch("user_stats/#{user.id}/#{user.updated_at.to_i}") do
    calculate_stats(user)
  end
end

# Russian doll caching in views
<% cache @product do %>
  <% @product.variants.each do |variant| %>
    <% cache variant do %>
      <%= render variant %>
    <% end %>
  <% end %>
<% end %>
```

### Memoization Patterns
```ruby
# Class-level memoization for constants
class TaxCalculator
  def self.rates
    @rates ||= TaxRate.all.index_by(&:region).freeze
  end
end

# Thread-safe memoization
class Configuration
  def self.settings
    @settings_mutex ||= Mutex.new
    @settings ||= @settings_mutex.synchronize do
      @settings || load_settings
    end
  end
end

# Per-request memoization (Rails)
class ApplicationController
  def current_user
    @current_user ||= User.find(session[:user_id])
  end
end
```

---

## Data Structures

### Stack (LIFO)
```ruby
# Using Array as stack
stack = []
stack.push(1)    # Add to top
stack.push(2)
stack.push(3)
stack.pop        # => 3 (remove from top)
stack.last       # => 2 (peek)
stack.empty?     # => false

# Use case: parsing nested structures
def valid_parentheses?(string)
  stack = []
  pairs = { ")" => "(", "]" => "[", "}" => "{" }

  string.each_char do |char|
    if pairs.values.include?(char)
      stack.push(char)
    elsif pairs.keys.include?(char)
      return false if stack.pop != pairs[char]
    end
  end

  stack.empty?
end

valid_parentheses?("([{}])")  # => true
valid_parentheses?("([)]")    # => false
```

### Queue (FIFO)
```ruby
# Using Array as queue (inefficient for large queues)
queue = []
queue.push(1)     # Enqueue
queue.push(2)
queue.shift       # => 1 (dequeue)

# Better: Use Queue class for thread safety
require 'thread'
queue = Queue.new
queue << 1        # Enqueue
queue.pop         # Dequeue (blocks if empty)
queue.pop(true)   # Non-blocking (raises if empty)

# Use case: BFS traversal
def breadth_first_search(root, target)
  queue = [root]

  while queue.any?
    node = queue.shift
    return node if node.value == target
    queue.concat(node.children)
  end

  nil
end
```

### Linked List
```ruby
# Ruby arrays are dynamic, but linked lists are useful to understand
class Node
  attr_accessor :value, :next_node

  def initialize(value)
    @value = value
    @next_node = nil
  end
end

class LinkedList
  def initialize
    @head = nil
  end

  def append(value)
    new_node = Node.new(value)
    if @head.nil?
      @head = new_node
    else
      current = @head
      current = current.next_node while current.next_node
      current.next_node = new_node
    end
  end

  def each
    current = @head
    while current
      yield current.value
      current = current.next_node
    end
  end
end
```

### Tree
```ruby
class TreeNode
  attr_accessor :value, :children

  def initialize(value)
    @value = value
    @children = []
  end

  def add_child(value)
    child = TreeNode.new(value)
    @children << child
    child
  end

  # Depth-first traversal
  def each_dfs(&block)
    block.call(self)
    @children.each { |child| child.each_dfs(&block) }
  end

  # Breadth-first traversal
  def each_bfs
    queue = [self]
    while queue.any?
      node = queue.shift
      yield node
      queue.concat(node.children)
    end
  end
end

# Binary Tree
class BinaryTreeNode
  attr_accessor :value, :left, :right

  def initialize(value)
    @value = value
  end

  # In-order traversal (left, root, right)
  def in_order(&block)
    @left&.in_order(&block)
    block.call(@value)
    @right&.in_order(&block)
  end

  # Pre-order traversal (root, left, right)
  def pre_order(&block)
    block.call(@value)
    @left&.pre_order(&block)
    @right&.pre_order(&block)
  end

  # Post-order traversal (left, right, root)
  def post_order(&block)
    @left&.post_order(&block)
    @right&.post_order(&block)
    block.call(@value)
  end
end
```

### Graph
```ruby
class Graph
  def initialize
    @adjacency_list = Hash.new { |h, k| h[k] = [] }
  end

  def add_edge(from, to, directed: false)
    @adjacency_list[from] << to
    @adjacency_list[to] << from unless directed
  end

  def neighbors(node)
    @adjacency_list[node]
  end

  # Breadth-First Search
  def bfs(start)
    visited = Set.new([start])
    queue = [start]
    result = []

    while queue.any?
      node = queue.shift
      result << node

      neighbors(node).each do |neighbor|
        unless visited.include?(neighbor)
          visited << neighbor
          queue << neighbor
        end
      end
    end

    result
  end

  # Depth-First Search
  def dfs(start, visited = Set.new)
    return [] if visited.include?(start)

    visited << start
    result = [start]

    neighbors(start).each do |neighbor|
      result.concat(dfs(neighbor, visited))
    end

    result
  end
end

# Usage
graph = Graph.new
graph.add_edge("A", "B")
graph.add_edge("A", "C")
graph.add_edge("B", "D")
graph.add_edge("C", "D")

graph.bfs("A")  # => ["A", "B", "C", "D"]
graph.dfs("A")  # => ["A", "B", "D", "C"]
```

---

## Ruby Collections

### Set
```ruby
require 'set'

# Create a set
set = Set.new([1, 2, 3, 2, 1])  # => #<Set: {1, 2, 3}>

# Operations - all O(1) average
set.add(4)           # Add element
set.delete(2)        # Remove element
set.include?(3)      # Check membership

# Set operations
a = Set.new([1, 2, 3])
b = Set.new([2, 3, 4])

a | b  # Union: {1, 2, 3, 4}
a & b  # Intersection: {2, 3}
a - b  # Difference: {1}
a ^ b  # Symmetric difference: {1, 4}

# Use case: efficient membership checking
valid_statuses = Set.new(%w[pending active completed])
valid_statuses.include?(params[:status])

# Use case: removing duplicates while preserving order
array = [3, 1, 2, 1, 3]
Set.new(array).to_a  # => [3, 1, 2]
# Or simply:
array.uniq  # => [3, 1, 2]
```

### SortedSet (Removed in Ruby 3.0+)
```ruby
# For sorted sets, use a sorted array or gem
# Or implement with binary search insertion

class SortedArray
  def initialize
    @array = []
  end

  def add(value)
    index = @array.bsearch_index { |x| x >= value } || @array.length
    @array.insert(index, value)
  end

  def include?(value)
    @array.bsearch { |x| x <=> value }
  end

  def to_a
    @array.dup
  end
end
```

---

## Rails-Specific Optimizations

### Efficient Queries
```ruby
# Bad: N+1 query
users.each do |user|
  puts user.orders.count  # Query for each user
end

# Good: Eager loading
users = User.includes(:orders)
users.each do |user|
  puts user.orders.size  # No additional queries
end

# Better: Counter cache
# In migration
add_column :users, :orders_count, :integer, default: 0
User.reset_counters(user.id, :orders)

# In model
class Order < ApplicationRecord
  belongs_to :user, counter_cache: true
end

# Now O(1) to get count
user.orders_count
```

### Batch Processing
```ruby
# Bad: Loads all records into memory
User.all.each { |user| process(user) }

# Good: Process in batches
User.find_each(batch_size: 1000) do |user|
  process(user)
end

# With conditions
User.where(active: true).find_each do |user|
  process(user)
end

# Parallel processing with batches
User.in_batches(of: 1000) do |batch|
  batch.update_all(processed: true)
end

# For large exports
def export_users_csv
  CSV.generate do |csv|
    csv << %w[id email name]
    User.find_each do |user|
      csv << [user.id, user.email, user.name]
    end
  end
end
```

### Pluck vs Select
```ruby
# select loads ActiveRecord objects
emails = User.select(:email).map(&:email)  # Creates User objects

# pluck returns just the values
emails = User.pluck(:email)  # Just array of strings

# Multiple columns
data = User.pluck(:id, :email, :name)
# => [[1, "a@b.com", "Alice"], [2, "c@d.com", "Bob"]]

# For calculations
User.count               # COUNT(*)
User.sum(:balance)       # SUM(balance)
User.average(:age)       # AVG(age)
User.maximum(:age)       # MAX(age)
User.minimum(:created_at)# MIN(created_at)
```

### Index Optimization
```ruby
# Add indexes for:
# 1. Foreign keys
add_index :orders, :user_id

# 2. Columns used in WHERE clauses
add_index :users, :email
add_index :users, :status

# 3. Columns used in ORDER BY
add_index :posts, :created_at

# 4. Composite indexes for common query patterns
add_index :orders, [:user_id, :created_at]
add_index :products, [:category_id, :price]

# 5. Partial indexes for filtered queries
add_index :users, :email, where: "active = true"

# Check if query uses index
User.where(email: "test@example.com").explain
```

### Avoiding Common Performance Issues
```ruby
# Issue: String concatenation in loops
# Bad
result = ""
1000.times { |i| result += i.to_s }

# Good
result = []
1000.times { |i| result << i.to_s }
result.join

# Issue: Repeated calculations
# Bad
orders.each do |order|
  tax_rate = TaxService.fetch_rate(order.region)  # Called each time
  order.update(tax: order.total * tax_rate)
end

# Good
rates = TaxService.fetch_all_rates
orders.each do |order|
  order.update(tax: order.total * rates[order.region])
end

# Issue: Loading unnecessary data
# Bad
user = User.find(id)  # Loads all columns
user.email

# Good
email = User.where(id: id).pick(:email)  # Just one column, one value

# Issue: Serialization in hot paths
# Bad
class User < ApplicationRecord
  serialize :preferences, coder: JSON  # Parsing on every access
end

# Good: Use jsonb column (PostgreSQL) or separate table
```

### Algorithm Selection Guide

| Problem | Best Approach | Time |
|---------|--------------|------|
| Check if item exists | Hash/Set | O(1) |
| Find item by key | Hash | O(1) |
| Find item in sorted list | Binary search | O(log n) |
| Find item in unsorted list | Linear search | O(n) |
| Sort data | sort/sort_by | O(n log n) |
| Process all items | Each/map | O(n) |
| Find duplicates | Hash/Set | O(n) |
| Top K items | Partial sort | O(n log k) |
| Sliding window | Two pointers | O(n) |
| Tree/graph traversal | BFS/DFS | O(V + E) |
