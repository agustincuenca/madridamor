# Skill: Models

## Purpose
Create and manage Active Record models with validations, associations, scopes, and callbacks.

## Model Structure

```ruby
class ModelName < ApplicationRecord
  # 1. Constants
  STATUSES = %w[draft published archived].freeze

  # 2. Associations
  belongs_to :user
  has_many :comments, dependent: :destroy
  has_one :profile, dependent: :destroy
  has_many :tags, through: :taggings
  has_one_attached :avatar
  has_many_attached :images

  # 3. Validations
  validates :title, presence: true, length: { maximum: 255 }
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :status, inclusion: { in: STATUSES }

  # 4. Scopes
  scope :published, -> { where(published: true) }
  scope :recent, -> { order(created_at: :desc) }
  scope :by_user, ->(user) { where(user: user) }

  # 5. Callbacks
  before_validation :normalize_data
  after_create :send_notification

  # 6. Class methods
  def self.search(query)
    where("title LIKE ?", "%#{query}%")
  end

  # 7. Instance methods
  def full_name
    "#{first_name} #{last_name}"
  end

  private

  def normalize_data
    self.email = email&.downcase&.strip
  end
end
```

## Associations

### Basic Associations
```ruby
# One-to-many
class User < ApplicationRecord
  has_many :posts, dependent: :destroy
end

class Post < ApplicationRecord
  belongs_to :user
end

# Many-to-many
class Article < ApplicationRecord
  has_many :taggings, dependent: :destroy
  has_many :tags, through: :taggings
end

class Tag < ApplicationRecord
  has_many :taggings, dependent: :destroy
  has_many :articles, through: :taggings
end

class Tagging < ApplicationRecord
  belongs_to :article
  belongs_to :tag
end
```

### Association Options
```ruby
has_many :comments, -> { order(created_at: :desc) }, dependent: :destroy
belongs_to :author, class_name: "User", foreign_key: "user_id"
has_one :profile, dependent: :destroy, inverse_of: :user
```

## Validations

### Common Validations
```ruby
# Presence
validates :name, presence: true

# Length
validates :title, length: { minimum: 5, maximum: 255 }
validates :password, length: { in: 8..72 }

# Format
validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
validates :slug, format: { with: /\A[a-z0-9-]+\z/ }

# Uniqueness
validates :email, uniqueness: { case_sensitive: false }
validates :slug, uniqueness: { scope: :user_id }

# Numericality
validates :price, numericality: { greater_than: 0 }
validates :quantity, numericality: { only_integer: true }

# Inclusion
validates :status, inclusion: { in: %w[draft published] }

# Custom
validate :end_date_after_start_date

def end_date_after_start_date
  return if end_date.blank? || start_date.blank?
  if end_date < start_date
    errors.add(:end_date, "must be after start date")
  end
end
```

## Scopes

```ruby
# Basic scopes
scope :active, -> { where(active: true) }
scope :recent, -> { order(created_at: :desc) }
scope :limit_recent, ->(n) { recent.limit(n) }

# Composed scopes
scope :published_recently, -> { published.where("created_at > ?", 1.week.ago) }

# With joins
scope :with_comments, -> { joins(:comments).distinct }

# Default scope (use sparingly)
default_scope { order(created_at: :desc) }
```

## Callbacks

```ruby
# Available callbacks (in order)
before_validation
after_validation
before_save
around_save
after_save
before_create / before_update
around_create / around_update
after_create / after_update
after_commit / after_rollback

# Example usage
before_save :generate_slug
after_create :send_welcome_email
after_commit :update_search_index, on: [:create, :update]

private

def generate_slug
  self.slug = title.parameterize if slug.blank?
end
```

## Migrations

### Create Table
```ruby
class CreateArticles < ActiveRecord::Migration[8.0]
  def change
    create_table :articles do |t|
      t.string :title, null: false
      t.text :body
      t.references :user, null: false, foreign_key: true
      t.boolean :published, default: false
      t.timestamps
    end

    add_index :articles, :title
    add_index :articles, [:user_id, :created_at]
  end
end
```

### Modify Table
```ruby
class AddCategoryToArticles < ActiveRecord::Migration[8.0]
  def change
    add_column :articles, :category, :string
    add_index :articles, :category
  end
end
```

## Query Interface

```ruby
# Find
User.find(1)
User.find_by(email: "test@example.com")
User.find_or_create_by(email: "new@example.com")

# Where
User.where(active: true)
User.where("created_at > ?", 1.week.ago)
User.where(role: ["admin", "moderator"])

# Ordering
User.order(created_at: :desc)
User.order(:last_name, :first_name)

# Eager loading (N+1 prevention)
User.includes(:posts).where(posts: { published: true })
User.preload(:comments)
User.eager_load(:profile)

# Aggregations
User.count
User.average(:age)
User.sum(:balance)
User.group(:status).count
```
