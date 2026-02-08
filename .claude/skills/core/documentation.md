# Skill: Documentation

## Purpose
Create and maintain clear, useful documentation for both technical and non-technical audiences.

## Documentation Types

### 1. User Documentation
For end users who use the application.

### 2. Developer Documentation
For developers who work on the codebase.

### 3. API Documentation
For developers integrating with the API.

### 4. Inline Documentation
Code comments and annotations.

## User Documentation

### Getting Started Guide
```markdown
# Getting Started with [App Name]

Welcome! This guide will help you start using [App Name] in just a few minutes.

## Creating Your Account

1. Go to [app URL]
2. Click "Sign Up"
3. Enter your email and choose a password
4. Check your email for verification link
5. Click the link to activate your account

## Your First [Action]

Now that you're logged in, let's create your first [item]:

1. Click the "+ New" button
2. Fill in the details:
   - **Title**: Give it a descriptive name
   - **Description**: Add details (optional)
3. Click "Create"

Congratulations! You've created your first [item].

## Next Steps

- [Link to feature guide]
- [Link to advanced features]
- [Link to FAQ]

## Need Help?

- Email: support@example.com
- [Link to help center]
```

### Feature Guide
```markdown
# [Feature Name] Guide

## What is [Feature]?

[One paragraph explanation of what this feature does and why it's useful]

## How to Use [Feature]

### Step 1: [Action]
[Instructions with screenshot if helpful]

### Step 2: [Action]
[Instructions]

## Tips and Best Practices

- **Tip 1**: [Helpful advice]
- **Tip 2**: [Helpful advice]

## Frequently Asked Questions

**Q: [Common question]?**
A: [Clear answer]

**Q: [Another question]?**
A: [Clear answer]

## Troubleshooting

### Problem: [Issue description]
**Solution**: [How to fix it]

### Problem: [Another issue]
**Solution**: [How to fix it]
```

## Developer Documentation

### README.md
```markdown
# [Project Name]

[One-line description of what the project does]

## Requirements

- Ruby 3.3+
- Rails 8.1+
- SQLite3
- Node.js 18+ (for assets)

## Quick Start

```bash
# Clone the repository
git clone https://github.com/user/project.git
cd project

# Install dependencies
bundle install

# Setup database
rails db:setup

# Start the server
bin/dev
```

Visit http://localhost:3000

## Configuration

Copy the example environment file:

```bash
cp .env.example .env
```

Configure the following variables:
- `SECRET_KEY_BASE`: Rails secret key
- `DATABASE_URL`: Database connection string (optional)

## Testing

```bash
# Run all tests
bundle exec rspec

# Run specific tests
bundle exec rspec spec/models
```

## Deployment

See [DEPLOYMENT.md](DEPLOYMENT.md) for deployment instructions.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for contribution guidelines.

## License

[License name] - see [LICENSE](LICENSE) for details.
```

### Architecture Documentation
```markdown
# Architecture Overview

## System Architecture

```
┌─────────────────────────────────────────────────┐
│                    Client                        │
│  (Browser / Mobile App via Hotwire Native)      │
└─────────────────────┬───────────────────────────┘
                      │ HTTPS
┌─────────────────────▼───────────────────────────┐
│                 Rails App                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │Controllers│  │  Models  │  │  Views   │      │
│  └──────────┘  └──────────┘  └──────────┘      │
│                      │                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │  Hotwire │  │Background │  │ Mailers  │      │
│  │(Turbo/   │  │   Jobs   │  │          │      │
│  │ Stimulus)│  │          │  │          │      │
│  └──────────┘  └──────────┘  └──────────┘      │
└─────────────────────┬───────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────┐
│               SQLite Database                    │
└─────────────────────────────────────────────────┘
```

## Key Components

### Models
- `User` - Authentication and user data
- `Article` - Main content entity
- `Comment` - User-generated comments

### Controllers
- RESTful design following Rails conventions
- Authentication via Rails 8 built-in auth
- Authorization via Pundit policies

### Views
- ERB templates with Tailwind CSS
- Hotwire (Turbo + Stimulus) for interactivity
- Mobile-first responsive design

## Data Flow

1. User makes request
2. Router directs to controller
3. Controller loads data via model
4. Model queries database
5. Controller renders view
6. View returns HTML (or Turbo Stream)
7. Browser updates DOM

## Background Jobs

Using Solid Queue for:
- Email delivery
- Data exports
- Scheduled tasks
```

## API Documentation

### Endpoint Documentation
```markdown
# API Reference

## Authentication

All API requests require authentication via Bearer token.

```
Authorization: Bearer <your-api-token>
```

Get your API token from Settings > API Access.

---

## Articles

### List Articles

```
GET /api/v1/articles
```

**Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| page | integer | No | Page number (default: 1) |
| per_page | integer | No | Items per page (default: 20, max: 100) |
| published | boolean | No | Filter by published status |

**Response**

```json
{
  "articles": [
    {
      "id": 1,
      "title": "Article Title",
      "published": true,
      "created_at": "2024-01-15T10:30:00Z",
      "author": {
        "id": 5,
        "name": "John Doe"
      }
    }
  ],
  "meta": {
    "current_page": 1,
    "total_pages": 5,
    "total_count": 100
  }
}
```

---

### Get Article

```
GET /api/v1/articles/:id
```

**Response**

```json
{
  "article": {
    "id": 1,
    "title": "Article Title",
    "body": "Full article content...",
    "published": true,
    "created_at": "2024-01-15T10:30:00Z",
    "author": {
      "id": 5,
      "name": "John Doe"
    },
    "comments_count": 12
  }
}
```

---

### Create Article

```
POST /api/v1/articles
```

**Request Body**

```json
{
  "article": {
    "title": "New Article",
    "body": "Article content...",
    "published": false
  }
}
```

**Response** (201 Created)

```json
{
  "article": {
    "id": 2,
    "title": "New Article",
    ...
  }
}
```

---

## Error Responses

### 400 Bad Request
```json
{
  "error": "Missing required parameter: title"
}
```

### 401 Unauthorized
```json
{
  "error": "Invalid or missing API token"
}
```

### 422 Unprocessable Entity
```json
{
  "error": "Validation failed",
  "details": [
    "Title can't be blank",
    "Body is too short (minimum is 10 characters)"
  ]
}
```
```

## Inline Documentation

### Ruby/Rails Comments
```ruby
# Processes payment for the given order.
#
# @param order [Order] the order to process payment for
# @param payment_method [String] the payment method ('card', 'bank')
# @return [PaymentResult] result object with success status and transaction ID
# @raise [PaymentError] if payment processing fails
#
# @example
#   result = PaymentService.process(order, 'card')
#   if result.success?
#     redirect_to success_path
#   end
#
def self.process(order, payment_method)
  # Implementation
end
```

### Complex Logic Comments
```ruby
def calculate_discount(order)
  # Apply tiered discount based on order total:
  # - Orders over $100: 10% off
  # - Orders over $200: 15% off
  # - Orders over $500: 20% off
  # Note: Discount is applied before tax calculation
  discount_rate = case order.subtotal
                  when 500.. then 0.20
                  when 200..499.99 then 0.15
                  when 100..199.99 then 0.10
                  else 0
                  end

  order.subtotal * discount_rate
end
```

## Changelog

### CHANGELOG.md Format
```markdown
# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Added
- New feature description

### Changed
- Change description

### Fixed
- Bug fix description

## [1.2.0] - 2024-01-15

### Added
- User profile pictures
- Dark mode support

### Changed
- Improved search performance
- Updated email templates

### Fixed
- Fixed login redirect issue
- Fixed mobile navigation bug

## [1.1.0] - 2024-01-01

### Added
- Comment system for articles
- Email notifications

[Unreleased]: https://github.com/user/repo/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/user/repo/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/user/repo/releases/tag/v1.1.0
```

## Best Practices

1. **Write for your audience** - Adjust language and detail level
2. **Keep it updated** - Outdated docs are worse than no docs
3. **Use examples** - Show, don't just tell
4. **Be concise** - Respect readers' time
5. **Structure for scanning** - Headers, lists, code blocks
6. **Link related content** - Help users find more info
7. **Include troubleshooting** - Address common problems
