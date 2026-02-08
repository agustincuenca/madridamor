# Skill: Rails Scaffold

## Purpose
Generate complete Rails resources with models, controllers, views, and routes following Rails conventions.

## When to Use
- Creating new resources/entities
- Setting up CRUD operations
- Initial feature scaffolding

## Implementation

### Standard Scaffold Command
```bash
rails generate scaffold ResourceName field:type field:type --skip-jbuilder
```

### Common Field Types
```
string      - Short text (name, title)
text        - Long text (description, content)
integer     - Whole numbers
decimal     - Money, precise numbers
float       - Scientific numbers
boolean     - True/false
date        - Date only
datetime    - Date and time
time        - Time only
references  - Foreign key (belongs_to)
```

### Example: Complete Resource
```bash
# Generate scaffold
rails generate scaffold Article title:string body:text published:boolean user:references --skip-jbuilder

# Run migration
rails db:migrate
```

### Generated Files
```
app/models/article.rb
app/controllers/articles_controller.rb
app/views/articles/
├── _article.html.erb
├── _form.html.erb
├── edit.html.erb
├── index.html.erb
├── new.html.erb
└── show.html.erb
db/migrate/TIMESTAMP_create_articles.rb
config/routes.rb (modified)
```

## Post-Scaffold Checklist

### 1. Model Enhancements
```ruby
class Article < ApplicationRecord
  # Associations
  belongs_to :user
  has_many :comments, dependent: :destroy

  # Validations
  validates :title, presence: true, length: { maximum: 255 }
  validates :body, presence: true

  # Scopes
  scope :published, -> { where(published: true) }
  scope :recent, -> { order(created_at: :desc) }
end
```

### 2. Controller Security
```ruby
class ArticlesController < ApplicationController
  before_action :authenticate_user!
  before_action :set_article, only: %i[show edit update destroy]

  # Add authorization
  def edit
    authorize @article
  end
end
```

### 3. View Improvements
- Add Tailwind CSS classes
- Implement responsive design
- Add Turbo Frame tags for interactivity
- Ensure accessibility

### 4. Route Customization
```ruby
resources :articles do
  member do
    patch :publish
    patch :unpublish
  end
  collection do
    get :drafts
  end
end
```

## Scaffold Alternatives

### Model Only
```bash
rails generate model Article title:string body:text
```

### Controller Only
```bash
rails generate controller Articles index show new create edit update destroy
```

### Migration Only
```bash
rails generate migration AddCategoryToArticles category:string
```

## Best Practices

1. **Always add validations** after scaffolding
2. **Add indexes** for foreign keys and frequently queried fields
3. **Set up associations** immediately
4. **Remove unused actions** if not needed
5. **Style views** with Tailwind before showing to user
6. **Add tests** for the new resource
