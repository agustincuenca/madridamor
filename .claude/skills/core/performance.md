# Skill: Performance

## Purpose
Optimize Rails application performance through database queries, caching, and efficient code.

## Database Optimization

### N+1 Query Prevention

```ruby
# BAD - N+1 queries
@articles = Article.all
@articles.each do |article|
  puts article.user.name  # New query for each article
end

# GOOD - Eager loading
@articles = Article.includes(:user)
@articles.each do |article|
  puts article.user.name  # No additional queries
end
```

### Eager Loading Methods
```ruby
# includes - Separate queries, smart loading
Article.includes(:user, :comments)

# preload - Always separate queries
Article.preload(:user)

# eager_load - Single LEFT OUTER JOIN
Article.eager_load(:user)

# Nested associations
Article.includes(comments: :user)
Article.includes(:user, comments: [:user, :likes])
```

### Bullet Gem (Development)
```ruby
# Gemfile
group :development do
  gem "bullet"
end

# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.alert = true
  Bullet.bullet_logger = true
  Bullet.rails_logger = true
end
```

### Select Only Needed Columns
```ruby
# BAD - Loads all columns
User.all.map(&:name)

# GOOD - Only loads name
User.pluck(:name)

# GOOD - Select specific columns
User.select(:id, :name, :email)
```

### Batch Processing
```ruby
# BAD - Loads all records into memory
User.all.each { |user| process(user) }

# GOOD - Loads in batches of 1000
User.find_each { |user| process(user) }

# Custom batch size
User.find_each(batch_size: 500) { |user| process(user) }

# With batches
User.in_batches(of: 1000) do |batch|
  batch.update_all(processed: true)
end
```

### Database Indexes
```ruby
# Migration with indexes
class CreateArticles < ActiveRecord::Migration[8.0]
  def change
    create_table :articles do |t|
      t.string :title, null: false
      t.references :user, null: false, foreign_key: true
      t.boolean :published, default: false
      t.datetime :published_at
      t.timestamps
    end

    # Single column indexes
    add_index :articles, :published
    add_index :articles, :published_at

    # Composite index (order matters!)
    add_index :articles, [:user_id, :published]

    # Unique index
    add_index :articles, :slug, unique: true

    # Partial index
    add_index :articles, :published_at, where: "published = true"
  end
end
```

### Query Optimization
```ruby
# Use exists? instead of present?
User.where(admin: true).exists?  # SELECT 1 ... LIMIT 1
User.where(admin: true).present? # SELECT * ... (loads all)

# Use count vs size vs length
relation.count   # Always SQL COUNT
relation.size    # COUNT if not loaded, length if loaded
relation.length  # Always loads records, then counts

# Use find_by instead of where.first
User.find_by(email: "test@example.com")  # Optimized
User.where(email: "test@example.com").first  # Less optimal
```

## Caching

### Fragment Caching
```erb
<%# app/views/articles/index.html.erb %>
<% @articles.each do |article| %>
  <%= cache article do %>
    <%= render article %>
  <% end %>
<% end %>

<%# With explicit key %>
<%= cache ["v1", article, current_user.admin?] do %>
  <%= render article %>
<% end %>
```

### Collection Caching
```erb
<%# Cached collection rendering %>
<%= render partial: "article", collection: @articles, cached: true %>

<%# With cache key prefix %>
<%= render partial: "article",
           collection: @articles,
           cached: ->(article) { ["v2", article] } %>
```

### Russian Doll Caching
```erb
<%# Parent caches children %>
<%= cache @article do %>
  <h1><%= @article.title %></h1>

  <% @article.comments.each do |comment| %>
    <%= cache comment do %>
      <%= render comment %>
    <% end %>
  <% end %>
<% end %>
```

### Low-Level Caching
```ruby
class Article < ApplicationRecord
  def expensive_calculation
    Rails.cache.fetch("article/#{id}/calculation", expires_in: 1.hour) do
      # Expensive operation
      perform_complex_calculation
    end
  end
end

# Manual cache operations
Rails.cache.write("key", value, expires_in: 1.hour)
Rails.cache.read("key")
Rails.cache.delete("key")
Rails.cache.exist?("key")
```

### Counter Caches
```ruby
# Migration
class AddCommentsCountToArticles < ActiveRecord::Migration[8.0]
  def change
    add_column :articles, :comments_count, :integer, default: 0, null: false

    # Populate existing counts
    Article.find_each do |article|
      Article.reset_counters(article.id, :comments)
    end
  end
end

# Model
class Comment < ApplicationRecord
  belongs_to :article, counter_cache: true
end

# Usage - no query needed!
@article.comments_count
```

## View Performance

### Turbo Frames for Partial Updates
```erb
<%# Only update what changed %>
<%= turbo_frame_tag "article_#{@article.id}" do %>
  <%= render @article %>
<% end %>
```

### Lazy Loading
```erb
<%# Load comments only when scrolled to %>
<%= turbo_frame_tag "comments",
    src: article_comments_path(@article),
    loading: :lazy do %>
  <p>Loading comments...</p>
<% end %>
```

### Pagination
```ruby
# Controller with Pagy
class ArticlesController < ApplicationController
  include Pagy::Backend

  def index
    @pagy, @articles = pagy(Article.published.recent, items: 20)
  end
end
```

```erb
<%# View %>
<%= render @articles %>
<%== pagy_nav(@pagy) %>
```

### Asset Optimization
```ruby
# config/environments/production.rb
config.assets.css_compressor = :sass
config.assets.js_compressor = :terser

# Use CDN
config.asset_host = "https://cdn.example.com"
```

## Background Processing

### Move Slow Operations to Jobs
```ruby
# BAD - Blocks request
def create
  @article = Article.create!(article_params)
  ArticleMailer.published(@article).deliver_now
  SearchIndex.update(@article)
  Analytics.track("article_created", @article.id)
end

# GOOD - Process async
def create
  @article = Article.create!(article_params)
  ArticlePublishedJob.perform_later(@article.id)
end
```

## Memory Optimization

### Avoid Loading Large Datasets
```ruby
# BAD - Loads all into memory
csv_data = Article.all.map do |a|
  [a.id, a.title, a.user.name]
end

# GOOD - Stream processing
def generate_csv
  Enumerator.new do |yielder|
    yielder << CSV.generate_line(["ID", "Title", "Author"])

    Article.includes(:user).find_each do |article|
      yielder << CSV.generate_line([article.id, article.title, article.user.name])
    end
  end
end
```

### Pluck vs Map
```ruby
# BAD - Instantiates all AR objects
User.all.map(&:email)

# GOOD - Returns array of values
User.pluck(:email)

# Multiple columns
User.pluck(:id, :email, :name)
```

## Monitoring

### Request Logging
```ruby
# config/environments/production.rb
config.log_level = :info
config.log_tags = [:request_id]
```

### Rack Mini Profiler (Development)
```ruby
# Gemfile
gem "rack-mini-profiler"

# Access at /?pp=help for options
```

### Performance Testing
```ruby
# spec/performance/articles_spec.rb
require "rails_helper"
require "benchmark"

RSpec.describe "Article performance", type: :request do
  let!(:articles) { create_list(:article, 100, :with_comments) }

  it "loads index quickly" do
    time = Benchmark.measure { get articles_path }
    expect(time.real).to be < 0.5
  end

  it "avoids N+1 queries" do
    expect {
      get articles_path
    }.to make_database_queries(count: 0..5)
  end
end
```

## Checklist

- [ ] No N+1 queries (use Bullet in development)
- [ ] Proper database indexes for queries
- [ ] Fragment caching for expensive views
- [ ] Counter caches for counts
- [ ] Pagination for lists
- [ ] Background jobs for slow operations
- [ ] Batch processing for large datasets
- [ ] Select only needed columns
