# Skill: Seeds

## Purpose
Generate realistic test data using Faker for development and testing environments.

## Basic Seeds

### Simple Seed File
```ruby
# db/seeds.rb

puts "Cleaning database..."
# Clean in reverse order of dependencies
Comment.destroy_all
Article.destroy_all
User.destroy_all

puts "Creating users..."
admin = User.create!(
  email_address: "admin@example.com",
  password: "password123",
  role: "admin"
)

users = 10.times.map do
  User.create!(
    email_address: Faker::Internet.unique.email,
    password: "password123"
  )
end

puts "Creating articles..."
users.each do |user|
  rand(1..5).times do
    user.articles.create!(
      title: Faker::Lorem.sentence(word_count: 4),
      body: Faker::Lorem.paragraphs(number: rand(3..8)).join("\n\n"),
      published: [true, true, true, false].sample,
      created_at: Faker::Time.between(from: 6.months.ago, to: Time.current)
    )
  end
end

puts "Creating comments..."
Article.published.each do |article|
  rand(0..10).times do
    article.comments.create!(
      user: users.sample,
      body: Faker::Lorem.paragraph,
      created_at: Faker::Time.between(from: article.created_at, to: Time.current)
    )
  end
end

puts "Seed complete!"
puts "  - #{User.count} users"
puts "  - #{Article.count} articles"
puts "  - #{Comment.count} comments"
```

## Modular Seeds

### Organized Structure
```
db/
├── seeds.rb
└── seeds/
    ├── 01_users.rb
    ├── 02_categories.rb
    ├── 03_articles.rb
    ├── 04_comments.rb
    └── development_only.rb
```

### Main Seed File
```ruby
# db/seeds.rb

# Load seed files in order
Dir[Rails.root.join("db/seeds/*.rb")].sort.each do |file|
  puts "Loading #{File.basename(file)}..."
  load file
end

puts "\nSeeding complete!"
```

### Individual Seed File
```ruby
# db/seeds/01_users.rb

# Create admin user
User.find_or_create_by!(email_address: "admin@example.com") do |user|
  user.password = "password123"
  user.role = "admin"
end

# Create test users
unless User.where.not(role: "admin").exists?
  20.times do |i|
    User.create!(
      email_address: Faker::Internet.unique.email,
      password: "password123",
      role: "user",
      created_at: Faker::Time.between(from: 1.year.ago, to: 1.month.ago)
    )
  end
end

puts "  Created #{User.count} users"
```

## Environment-Specific Seeds

```ruby
# db/seeds.rb

# Always run
load Rails.root.join("db/seeds/essential.rb")

# Development only
if Rails.env.development?
  load Rails.root.join("db/seeds/development.rb")
end

# Production (minimal data)
if Rails.env.production?
  load Rails.root.join("db/seeds/production.rb")
end
```

```ruby
# db/seeds/development.rb

puts "Creating development data..."

# More extensive fake data for development
100.times do
  user = User.create!(
    email_address: Faker::Internet.unique.email,
    password: "password123"
  )

  # Each user has varying activity
  rand(0..20).times do
    user.articles.create!(
      title: Faker::Book.title,
      body: Faker::Lorem.paragraphs(number: 5).join("\n\n"),
      published: Faker::Boolean.boolean(true_ratio: 0.7)
    )
  end
end
```

## Faker Examples

### User Data
```ruby
Faker::Name.name                    # "John Smith"
Faker::Internet.email               # "john@example.com"
Faker::Internet.unique.email        # Guaranteed unique
Faker::Internet.username            # "john_smith"
Faker::PhoneNumber.phone_number     # "555-123-4567"
Faker::Address.full_address         # "123 Main St, City, ST 12345"
Faker::Avatar.image                 # URL to avatar image
```

### Content
```ruby
Faker::Lorem.sentence               # "Lorem ipsum dolor sit."
Faker::Lorem.paragraph              # Full paragraph
Faker::Lorem.paragraphs(number: 3)  # Array of paragraphs
Faker::Markdown.emphasis            # "*emphasis*"
Faker::Markdown.sandwich            # Headers + paragraphs
Faker::Book.title                   # "The Great Gatsby"
Faker::Quote.famous_last_words      # Famous quote
```

### Dates and Times
```ruby
Faker::Date.between(from: 1.year.ago, to: Date.today)
Faker::Time.between(from: 1.month.ago, to: Time.current)
Faker::Date.birthday(min_age: 18, max_age: 65)
Faker::Time.forward(days: 30)       # Random future time
Faker::Time.backward(days: 30)      # Random past time
```

### Numbers and Booleans
```ruby
Faker::Number.between(from: 1, to: 100)
Faker::Number.decimal(l_digits: 2, r_digits: 2)
Faker::Boolean.boolean                      # true/false
Faker::Boolean.boolean(true_ratio: 0.8)     # 80% true
rand(1..10)                                 # Ruby native
[true, false].sample                        # Random pick
```

### Commerce/Business
```ruby
Faker::Commerce.product_name        # "Ergonomic Steel Chair"
Faker::Commerce.price               # 29.99
Faker::Company.name                 # "Acme Inc"
Faker::Company.catch_phrase         # "Innovative solution"
Faker::Job.title                    # "Software Engineer"
```

### Images (URLs)
```ruby
Faker::LoremFlickr.image(size: "300x300", search_terms: ['nature'])
Faker::Placeholdit.image(size: "300x300")
Faker::Avatar.image(slug: "user-#{id}", size: "100x100")
```

## Idempotent Seeds

```ruby
# Find or create pattern
admin = User.find_or_create_by!(email_address: "admin@example.com") do |user|
  user.password = "password123"
  user.role = "admin"
end

# Check before creating
unless Article.count >= 50
  50.times do
    Article.create!(
      title: Faker::Lorem.sentence,
      body: Faker::Lorem.paragraphs(number: 3).join("\n\n"),
      user: User.all.sample
    )
  end
end

# Using find_or_initialize_by
category = Category.find_or_initialize_by(name: "Technology")
category.update!(description: "Tech articles") if category.new_record?
```

## With Attachments

```ruby
# Using remote URLs
user = User.create!(
  email_address: Faker::Internet.email,
  password: "password123"
)
user.avatar.attach(
  io: URI.open(Faker::Avatar.image),
  filename: "avatar.jpg",
  content_type: "image/jpeg"
)

# Using local files
article = Article.create!(
  title: Faker::Lorem.sentence,
  body: Faker::Lorem.paragraphs.join("\n\n"),
  user: users.sample
)
article.images.attach(
  io: File.open(Rails.root.join("db/seed_files/sample.jpg")),
  filename: "sample.jpg",
  content_type: "image/jpeg"
)
```

## Running Seeds

```bash
# Run all seeds
rails db:seed

# Reset and seed
rails db:reset  # drop, create, migrate, seed

# Specific environment
RAILS_ENV=production rails db:seed

# Custom seed task
rails db:seed:development
```

### Custom Rake Task
```ruby
# lib/tasks/seed.rake
namespace :db do
  namespace :seed do
    desc "Load development seeds"
    task development: :environment do
      load Rails.root.join("db/seeds/development.rb")
    end

    desc "Load minimal production seeds"
    task production: :environment do
      load Rails.root.join("db/seeds/production.rb")
    end
  end
end
```

## Best Practices

1. **Make seeds idempotent** - Safe to run multiple times
2. **Use Faker::UniqueGenerator** - Avoid duplicates
3. **Order by dependencies** - Create parents before children
4. **Print progress** - Show what's being created
5. **Handle attachments** - Include sample files or URLs
6. **Separate environments** - Different data needs
