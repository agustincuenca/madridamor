# Skill: Code Review

## Purpose
Systematically review code for quality, security, performance, and maintainability.

## Review Checklist

### 1. Functionality
- [ ] Code does what it's supposed to do
- [ ] Edge cases are handled
- [ ] Error handling is appropriate
- [ ] No obvious bugs

### 2. Security
- [ ] No SQL injection vulnerabilities
- [ ] No XSS vulnerabilities
- [ ] Authentication/authorization checked
- [ ] Sensitive data not logged
- [ ] CSRF protection in place
- [ ] Strong parameters used
- [ ] No hardcoded secrets

### 3. Performance
- [ ] No N+1 queries
- [ ] Proper indexes exist
- [ ] No unnecessary database calls
- [ ] Efficient algorithms used
- [ ] Caching where appropriate

### 4. Code Quality
- [ ] Follows Rails conventions
- [ ] DRY - no unnecessary duplication
- [ ] Single responsibility principle
- [ ] Clear naming
- [ ] Appropriate comments (not excessive)

### 5. Testing
- [ ] Tests exist for new code
- [ ] Tests cover edge cases
- [ ] Tests are meaningful (not just coverage)
- [ ] All tests pass

### 6. Accessibility
- [ ] Semantic HTML used
- [ ] Form labels present
- [ ] ARIA attributes where needed
- [ ] Color contrast sufficient

## Security Review

### SQL Injection
```ruby
# BAD - SQL injection vulnerable
User.where("name = '#{params[:name]}'")

# GOOD - Parameterized
User.where(name: params[:name])
User.where("name = ?", params[:name])
```

### XSS Prevention
```erb
<%# BAD - Raw HTML output %>
<%= raw @article.body %>
<%= @article.body.html_safe %>

<%# GOOD - Escaped by default %>
<%= @article.body %>

<%# GOOD - Sanitized HTML %>
<%= sanitize @article.body, tags: %w[p br strong em] %>
```

### Mass Assignment
```ruby
# BAD - Permits everything
params.require(:user).permit!

# GOOD - Explicit whitelist
params.require(:user).permit(:name, :email)

# GOOD - Conditional permits
def user_params
  permitted = [:name, :email]
  permitted << :role if current_user.admin?
  params.require(:user).permit(permitted)
end
```

### Authorization
```ruby
# BAD - No authorization check
def update
  @article = Article.find(params[:id])
  @article.update(article_params)
end

# GOOD - Ownership check
def update
  @article = current_user.articles.find(params[:id])
  @article.update(article_params)
end

# GOOD - Policy check
def update
  @article = Article.find(params[:id])
  authorize @article
  @article.update(article_params)
end
```

## Performance Review

### N+1 Queries
```ruby
# BAD - N+1 query
@articles = Article.all
@articles.each { |a| puts a.user.name }

# GOOD - Eager loading
@articles = Article.includes(:user)
@articles.each { |a| puts a.user.name }
```

### Efficient Queries
```ruby
# BAD - Loads all records
User.all.count
User.all.any?

# GOOD - Database operations
User.count
User.exists?
```

### Avoid Memory Bloat
```ruby
# BAD - Loads all into memory
User.all.each { |u| process(u) }

# GOOD - Batch processing
User.find_each { |u| process(u) }
```

## Code Quality Review

### Single Responsibility
```ruby
# BAD - Controller doing too much
def create
  @user = User.new(user_params)
  @user.generate_token
  @user.save
  UserMailer.welcome(@user).deliver_now
  Analytics.track("signup", @user.id)
  redirect_to dashboard_path
end

# GOOD - Delegated to service/job
def create
  @user = User.new(user_params)
  if @user.save
    UserRegistrationJob.perform_later(@user.id)
    redirect_to dashboard_path
  else
    render :new
  end
end
```

### Clear Naming
```ruby
# BAD - Unclear names
def process(d)
  d.map { |x| x * 2 }
end

# GOOD - Descriptive names
def double_quantities(order_items)
  order_items.map { |item| item.quantity * 2 }
end
```

### Avoid Deep Nesting
```ruby
# BAD - Too nested
if user
  if user.active?
    if user.verified?
      do_something
    end
  end
end

# GOOD - Early returns
return unless user
return unless user.active?
return unless user.verified?
do_something
```

## Rails Conventions Review

### RESTful Actions
```ruby
# BAD - Non-RESTful action names
def fetch_articles
def do_publish

# GOOD - RESTful
def index
def show
def publish  # Custom action on member
```

### Callbacks vs Explicit Calls
```ruby
# QUESTIONABLE - Hidden behavior
class Article < ApplicationRecord
  after_save :notify_subscribers
  after_save :update_search_index
  after_save :clear_cache
end

# BETTER - Explicit service object
class ArticlePublisher
  def call(article)
    article.save!
    notify_subscribers(article)
    update_search_index(article)
    clear_cache(article)
  end
end
```

### Fat Models, Skinny Controllers
```ruby
# BAD - Logic in controller
def create
  @order = Order.new(order_params)
  @order.total = @order.items.sum(&:price) * 1.1  # Tax
  @order.status = "pending"
  @order.save
end

# GOOD - Logic in model
def create
  @order = Order.new(order_params)
  @order.save
end

class Order < ApplicationRecord
  before_save :calculate_total

  def calculate_total
    self.total = items.sum(&:price) * tax_rate
  end
end
```

## Review Report Format

```markdown
# Code Review: [PR/Feature Name]
Reviewer: [Name]
Date: [Date]

## Summary
[Brief description of what was reviewed]

## Findings

### Critical Issues
1. **[Issue Title]**
   - File: `path/to/file.rb:123`
   - Problem: [Description]
   - Suggestion: [How to fix]

### Warnings
1. **[Issue Title]**
   - File: `path/to/file.rb:456`
   - Problem: [Description]
   - Suggestion: [How to fix]

### Suggestions
1. **[Improvement]**
   - File: `path/to/file.rb:789`
   - Current: [Current approach]
   - Suggested: [Better approach]

## Positive Notes
- [What was done well]
- [Good patterns used]

## Tests
- [ ] Tests exist and pass
- [ ] Coverage adequate

## Verdict
- [ ] Approved
- [ ] Approved with minor changes
- [ ] Changes requested
```

## Automated Checks

### Linting (RuboCop)
```yaml
# .rubocop.yml
require:
  - rubocop-rails
  - rubocop-rspec

AllCops:
  NewCops: enable
  TargetRubyVersion: 3.3

Rails:
  Enabled: true
```

### Security Scanning (Brakeman)
```bash
# Run security scan
brakeman -o brakeman-report.html
```

### Dependency Audit
```bash
# Check for vulnerable gems
bundle audit check --update
```

## Best Practices

1. **Review small PRs** - Easier to catch issues
2. **Use automated tools first** - Let robots catch obvious issues
3. **Focus on logic** - Don't nitpick style (let linters handle it)
4. **Ask questions** - Don't assume, clarify intent
5. **Be constructive** - Suggest solutions, not just problems
6. **Prioritize security** - Always check auth and data handling
