# Skill: Authorization

## Purpose
Implement role-based access control using Pundit for fine-grained authorization.

## Setup

### Install Pundit
```ruby
# Gemfile
gem "pundit"
```

```bash
bundle install
rails generate pundit:install
```

### Include in Application Controller
```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Pundit::Authorization

  rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized

  private

  def user_not_authorized
    flash[:alert] = "You are not authorized to perform this action."
    redirect_back(fallback_location: root_path)
  end
end
```

## User Roles

### Add Role to User
```ruby
# Migration
class AddRoleToUsers < ActiveRecord::Migration[8.0]
  def change
    add_column :users, :role, :string, default: "user", null: false
    add_index :users, :role
  end
end
```

### User Model with Roles
```ruby
# app/models/user.rb
class User < ApplicationRecord
  ROLES = %w[user admin moderator].freeze

  validates :role, inclusion: { in: ROLES }

  def admin?
    role == "admin"
  end

  def moderator?
    role == "moderator"
  end

  def at_least_moderator?
    admin? || moderator?
  end
end
```

## Policies

### Application Policy (Base)
```ruby
# app/policies/application_policy.rb
class ApplicationPolicy
  attr_reader :user, :record

  def initialize(user, record)
    @user = user
    @record = record
  end

  def index?
    true
  end

  def show?
    true
  end

  def create?
    user.present?
  end

  def new?
    create?
  end

  def update?
    owner_or_admin?
  end

  def edit?
    update?
  end

  def destroy?
    owner_or_admin?
  end

  private

  def owner?
    record.respond_to?(:user) && record.user == user
  end

  def owner_or_admin?
    owner? || user&.admin?
  end

  def admin?
    user&.admin?
  end

  class Scope
    def initialize(user, scope)
      @user = user
      @scope = scope
    end

    def resolve
      scope.all
    end

    private

    attr_reader :user, :scope
  end
end
```

### Resource Policy
```ruby
# app/policies/article_policy.rb
class ArticlePolicy < ApplicationPolicy
  def index?
    true
  end

  def show?
    record.published? || owner_or_admin?
  end

  def create?
    user.present?
  end

  def update?
    owner_or_admin?
  end

  def destroy?
    owner_or_admin?
  end

  def publish?
    owner_or_admin?
  end

  class Scope < ApplicationPolicy::Scope
    def resolve
      if user&.admin?
        scope.all
      elsif user
        scope.where(published: true).or(scope.where(user: user))
      else
        scope.where(published: true)
      end
    end
  end
end
```

### Admin-Only Policy
```ruby
# app/policies/admin/user_policy.rb
module Admin
  class UserPolicy < ApplicationPolicy
    def index?
      admin?
    end

    def show?
      admin?
    end

    def update?
      admin? && record != user # Can't edit self
    end

    def destroy?
      admin? && record != user # Can't delete self
    end
  end
end
```

## Controller Usage

### Basic Authorization
```ruby
class ArticlesController < ApplicationController
  def show
    @article = Article.find(params[:id])
    authorize @article
  end

  def create
    @article = current_user.articles.build(article_params)
    authorize @article

    if @article.save
      redirect_to @article
    else
      render :new, status: :unprocessable_entity
    end
  end

  def update
    @article = Article.find(params[:id])
    authorize @article

    if @article.update(article_params)
      redirect_to @article
    else
      render :edit, status: :unprocessable_entity
    end
  end
end
```

### Policy Scopes
```ruby
class ArticlesController < ApplicationController
  def index
    @articles = policy_scope(Article).recent
  end
end
```

### Authorize in Before Action
```ruby
class ArticlesController < ApplicationController
  before_action :set_article, only: [:show, :edit, :update, :destroy]

  def edit
    # authorize called in set_article
  end

  private

  def set_article
    @article = Article.find(params[:id])
    authorize @article
  end
end
```

## View Helpers

### Conditional Display
```erb
<% if policy(@article).edit? %>
  <%= link_to "Edit", edit_article_path(@article) %>
<% end %>

<% if policy(@article).destroy? %>
  <%= button_to "Delete", @article, method: :delete %>
<% end %>
```

### New Record Check
```erb
<% if policy(Article).create? %>
  <%= link_to "New Article", new_article_path %>
<% end %>
```

### Admin-Only Section
```erb
<% if policy([:admin, User]).index? %>
  <%= link_to "Manage Users", admin_users_path %>
<% end %>
```

## Headless Policies

For actions not tied to a model:

```ruby
# app/policies/dashboard_policy.rb
class DashboardPolicy < ApplicationPolicy
  def initialize(user, _record = nil)
    @user = user
  end

  def show?
    user.present?
  end

  def admin?
    user&.admin?
  end
end
```

```ruby
# Controller
class DashboardController < ApplicationController
  def show
    authorize :dashboard, :show?
  end

  def admin
    authorize :dashboard, :admin?
  end
end
```

## Testing Policies

```ruby
# spec/policies/article_policy_spec.rb
require "rails_helper"

RSpec.describe ArticlePolicy do
  subject { described_class }

  let(:user) { create(:user) }
  let(:admin) { create(:user, role: "admin") }
  let(:article) { create(:article, user: user) }
  let(:other_article) { create(:article) }

  permissions :update?, :destroy? do
    it "allows owner" do
      expect(subject).to permit(user, article)
    end

    it "denies non-owner" do
      expect(subject).not_to permit(user, other_article)
    end

    it "allows admin" do
      expect(subject).to permit(admin, other_article)
    end
  end

  describe "Scope" do
    let!(:published) { create(:article, published: true) }
    let!(:draft) { create(:article, published: false) }
    let!(:own_draft) { create(:article, published: false, user: user) }

    it "shows only published to guests" do
      scope = ArticlePolicy::Scope.new(nil, Article).resolve
      expect(scope).to contain_exactly(published)
    end

    it "shows published + own drafts to users" do
      scope = ArticlePolicy::Scope.new(user, Article).resolve
      expect(scope).to contain_exactly(published, own_draft)
    end

    it "shows all to admin" do
      scope = ArticlePolicy::Scope.new(admin, Article).resolve
      expect(scope).to contain_exactly(published, draft, own_draft)
    end
  end
end
```

## Best Practices

1. **Authorize early** - Call authorize at the start of actions
2. **Use scopes** - Filter collections with policy_scope
3. **Test policies** - Write specs for all permission scenarios
4. **Keep policies focused** - One policy per resource
5. **Use permitted_attributes** - Let policy control params
