# Skill: Controllers

## Purpose
Create RESTful controllers with proper authentication, authorization, parameter handling, and responses.

## Controller Structure

```ruby
class ArticlesController < ApplicationController
  # 1. Callbacks
  before_action :authenticate_user!
  before_action :set_article, only: %i[show edit update destroy]

  # 2. RESTful actions
  def index
    @articles = Article.includes(:user).recent.page(params[:page])
  end

  def show
  end

  def new
    @article = current_user.articles.build
  end

  def create
    @article = current_user.articles.build(article_params)

    if @article.save
      redirect_to @article, notice: "Article created successfully."
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
  end

  def update
    if @article.update(article_params)
      redirect_to @article, notice: "Article updated successfully."
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @article.destroy
    redirect_to articles_path, notice: "Article deleted."
  end

  private

  def set_article
    @article = Article.find(params[:id])
  end

  def article_params
    params.require(:article).permit(:title, :body, :published)
  end
end
```

## Strong Parameters

```ruby
# Basic
def article_params
  params.require(:article).permit(:title, :body)
end

# With nested attributes
def article_params
  params.require(:article).permit(
    :title,
    :body,
    tags_attributes: [:id, :name, :_destroy],
    images: []  # Array of files
  )
end

# Conditional
def article_params
  permitted = [:title, :body]
  permitted << :published if current_user.admin?
  params.require(:article).permit(permitted)
end
```

## Before Actions

```ruby
class ArticlesController < ApplicationController
  # Authentication
  before_action :authenticate_user!
  before_action :authenticate_user!, except: [:index, :show]

  # Resource loading
  before_action :set_article, only: %i[show edit update destroy]

  # Authorization
  before_action :authorize_article, only: %i[edit update destroy]

  private

  def set_article
    @article = Article.find(params[:id])
  end

  def authorize_article
    redirect_to articles_path, alert: "Not authorized" unless @article.user == current_user
  end
end
```

## Response Formats

### HTML with Turbo
```ruby
def create
  @article = current_user.articles.build(article_params)

  if @article.save
    respond_to do |format|
      format.html { redirect_to @article, notice: "Created!" }
      format.turbo_stream
    end
  else
    render :new, status: :unprocessable_entity
  end
end
```

### JSON API
```ruby
def index
  @articles = Article.all

  respond_to do |format|
    format.html
    format.json { render json: @articles }
  end
end
```

## Flash Messages

```ruby
# Standard messages
redirect_to @article, notice: "Article created."
redirect_to articles_path, alert: "Something went wrong."

# Custom flash types
flash[:success] = "Great job!"
flash.now[:error] = "Please fix the errors."  # For render
```

## Turbo Stream Responses

```ruby
# app/views/articles/create.turbo_stream.erb
<%= turbo_stream.prepend "articles", @article %>
<%= turbo_stream.update "flash", partial: "shared/flash" %>

# Or inline in controller
def create
  @article = current_user.articles.build(article_params)

  if @article.save
    render turbo_stream: [
      turbo_stream.prepend("articles", @article),
      turbo_stream.update("form", partial: "form", locals: { article: Article.new })
    ]
  else
    render :new, status: :unprocessable_entity
  end
end
```

## Authorization with Pundit

```ruby
class ArticlesController < ApplicationController
  include Pundit::Authorization

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
end
```

## Error Handling

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from Pundit::NotAuthorizedError, with: :forbidden

  private

  def not_found
    render file: Rails.root.join("public/404.html"), status: :not_found, layout: false
  end

  def forbidden
    redirect_to root_path, alert: "You are not authorized to perform this action."
  end
end
```

## RESTful Custom Actions

```ruby
# routes.rb
resources :articles do
  member do
    patch :publish
    patch :archive
  end
  collection do
    get :drafts
    get :search
  end
end

# controller
def publish
  @article = Article.find(params[:id])
  @article.update(published: true)
  redirect_to @article, notice: "Article published."
end

def drafts
  @articles = current_user.articles.where(published: false)
  render :index
end
```

## Pagination

```ruby
# With Pagy (recommended for Rails 8)
class ArticlesController < ApplicationController
  include Pagy::Backend

  def index
    @pagy, @articles = pagy(Article.recent, items: 20)
  end
end

# In view
<%== pagy_nav(@pagy) %>
```

## Concerns

```ruby
# app/controllers/concerns/searchable.rb
module Searchable
  extend ActiveSupport::Concern

  def search
    @results = model_class.search(params[:q])
    render :index
  end

  private

  def model_class
    controller_name.classify.constantize
  end
end

# Usage
class ArticlesController < ApplicationController
  include Searchable
end
```
