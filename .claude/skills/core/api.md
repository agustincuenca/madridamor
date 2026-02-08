# Skill: API

## Purpose
Build RESTful JSON APIs for external integrations and mobile apps.

## API Controller Setup

### Base Controller
```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ApplicationController
      skip_before_action :verify_authenticity_token
      before_action :authenticate_api_request

      rescue_from ActiveRecord::RecordNotFound, with: :not_found
      rescue_from ActiveRecord::RecordInvalid, with: :unprocessable_entity
      rescue_from ActionController::ParameterMissing, with: :bad_request

      private

      def authenticate_api_request
        token = request.headers["Authorization"]&.split(" ")&.last
        @current_api_user = User.find_by(api_token: token)

        render_unauthorized unless @current_api_user
      end

      def current_api_user
        @current_api_user
      end

      def render_unauthorized
        render json: { error: "Unauthorized" }, status: :unauthorized
      end

      def not_found(exception)
        render json: { error: exception.message }, status: :not_found
      end

      def unprocessable_entity(exception)
        render json: {
          error: "Validation failed",
          details: exception.record.errors.full_messages
        }, status: :unprocessable_entity
      end

      def bad_request(exception)
        render json: { error: exception.message }, status: :bad_request
      end
    end
  end
end
```

### Resource Controller
```ruby
# app/controllers/api/v1/articles_controller.rb
module Api
  module V1
    class ArticlesController < BaseController
      before_action :set_article, only: [:show, :update, :destroy]

      def index
        @articles = Article.published
                          .includes(:user, :tags)
                          .order(created_at: :desc)
                          .page(params[:page])
                          .per(params[:per_page] || 20)

        render json: {
          articles: @articles.map { |a| article_json(a) },
          meta: pagination_meta(@articles)
        }
      end

      def show
        render json: { article: article_json(@article, full: true) }
      end

      def create
        @article = current_api_user.articles.create!(article_params)
        render json: { article: article_json(@article) }, status: :created
      end

      def update
        authorize_owner!(@article)
        @article.update!(article_params)
        render json: { article: article_json(@article) }
      end

      def destroy
        authorize_owner!(@article)
        @article.destroy
        head :no_content
      end

      private

      def set_article
        @article = Article.find(params[:id])
      end

      def article_params
        params.require(:article).permit(:title, :body, :published, tag_ids: [])
      end

      def authorize_owner!(resource)
        unless resource.user == current_api_user
          render json: { error: "Forbidden" }, status: :forbidden
        end
      end

      def article_json(article, full: false)
        json = {
          id: article.id,
          title: article.title,
          published: article.published,
          created_at: article.created_at.iso8601,
          author: {
            id: article.user.id,
            name: article.user.name
          },
          tags: article.tags.pluck(:name)
        }

        if full
          json[:body] = article.body
          json[:comments_count] = article.comments_count
        end

        json
      end

      def pagination_meta(collection)
        {
          current_page: collection.current_page,
          total_pages: collection.total_pages,
          total_count: collection.total_count,
          per_page: collection.limit_value
        }
      end
    end
  end
end
```

## Routes

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :articles, only: [:index, :show, :create, :update, :destroy] do
        resources :comments, only: [:index, :create, :destroy]
      end
      resources :users, only: [:show, :update]
      resource :session, only: [:create, :destroy]
    end
  end
end
```

## Authentication

### Token-Based Auth
```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_token :api_token

  def regenerate_api_token!
    regenerate_api_token
    save!
  end
end

# Migration
class AddApiTokenToUsers < ActiveRecord::Migration[8.0]
  def change
    add_column :users, :api_token, :string
    add_index :users, :api_token, unique: true
  end
end
```

### Session Controller
```ruby
# app/controllers/api/v1/sessions_controller.rb
module Api
  module V1
    class SessionsController < BaseController
      skip_before_action :authenticate_api_request, only: [:create]

      def create
        user = User.authenticate_by(
          email_address: params[:email],
          password: params[:password]
        )

        if user
          user.regenerate_api_token! unless user.api_token
          render json: {
            user: user_json(user),
            token: user.api_token
          }
        else
          render json: { error: "Invalid credentials" }, status: :unauthorized
        end
      end

      def destroy
        current_api_user.regenerate_api_token!
        head :no_content
      end

      private

      def user_json(user)
        {
          id: user.id,
          email: user.email_address,
          name: user.name
        }
      end
    end
  end
end
```

## Request/Response Format

### Request Headers
```
Content-Type: application/json
Authorization: Bearer <token>
Accept: application/json
```

### Response Format
```json
// Success (single resource)
{
  "article": {
    "id": 1,
    "title": "Hello World",
    "body": "Content here..."
  }
}

// Success (collection)
{
  "articles": [...],
  "meta": {
    "current_page": 1,
    "total_pages": 5,
    "total_count": 100
  }
}

// Error
{
  "error": "Validation failed",
  "details": [
    "Title can't be blank",
    "Body is too short"
  ]
}
```

## Serializers (Optional)

### Using ActiveModel Serializers
```ruby
# Gemfile
gem "active_model_serializers"

# app/serializers/article_serializer.rb
class ArticleSerializer < ActiveModel::Serializer
  attributes :id, :title, :body, :published, :created_at

  belongs_to :user
  has_many :tags

  def created_at
    object.created_at.iso8601
  end
end

# Controller
def show
  render json: @article
end
```

### Using Blueprinter
```ruby
# Gemfile
gem "blueprinter"

# app/blueprints/article_blueprint.rb
class ArticleBlueprint < Blueprinter::Base
  identifier :id

  fields :title, :published

  field :created_at do |article|
    article.created_at.iso8601
  end

  association :user, blueprint: UserBlueprint
  association :tags, blueprint: TagBlueprint

  view :full do
    field :body
    field :comments_count
  end
end

# Controller
def show
  render json: ArticleBlueprint.render(@article, view: :full)
end
```

## Versioning

### URL Versioning
```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :articles
  end

  namespace :v2 do
    resources :articles
  end
end
```

### Header Versioning
```ruby
# app/controllers/api/base_controller.rb
class Api::BaseController < ApplicationController
  before_action :set_api_version

  private

  def set_api_version
    @api_version = request.headers["Api-Version"] || "v1"
  end
end
```

## Rate Limiting

```ruby
# app/controllers/api/v1/base_controller.rb
class Api::V1::BaseController < ApplicationController
  include RateLimiting

  rate_limit to: 100, within: 1.minute
end

# app/controllers/concerns/rate_limiting.rb
module RateLimiting
  extend ActiveSupport::Concern

  class_methods do
    def rate_limit(to:, within:)
      before_action -> { check_rate_limit(to, within) }
    end
  end

  private

  def check_rate_limit(limit, period)
    key = "rate_limit:#{current_api_user&.id || request.ip}"
    count = Rails.cache.increment(key, 1, expires_in: period)

    if count > limit
      render json: {
        error: "Rate limit exceeded",
        retry_after: period.to_i
      }, status: :too_many_requests
    end
  end
end
```

## Testing

```ruby
# spec/requests/api/v1/articles_spec.rb
require "rails_helper"

RSpec.describe "API V1 Articles", type: :request do
  let(:user) { create(:user) }
  let(:headers) do
    {
      "Authorization" => "Bearer #{user.api_token}",
      "Content-Type" => "application/json"
    }
  end

  describe "GET /api/v1/articles" do
    let!(:articles) { create_list(:article, 3, :published) }

    it "returns articles" do
      get "/api/v1/articles", headers: headers

      expect(response).to have_http_status(:ok)
      expect(json_response["articles"].size).to eq(3)
    end
  end

  describe "POST /api/v1/articles" do
    let(:valid_params) do
      { article: { title: "Test", body: "Content" } }
    end

    it "creates article" do
      post "/api/v1/articles",
           params: valid_params.to_json,
           headers: headers

      expect(response).to have_http_status(:created)
      expect(json_response["article"]["title"]).to eq("Test")
    end
  end

  def json_response
    JSON.parse(response.body)
  end
end
```

## Documentation

### Request/Response Examples
```ruby
# doc/api/articles.md
## Articles API

### List Articles
GET /api/v1/articles

#### Parameters
| Name | Type | Description |
|------|------|-------------|
| page | integer | Page number |
| per_page | integer | Items per page (max 100) |

#### Response
```json
{
  "articles": [...],
  "meta": { "current_page": 1, "total_pages": 5 }
}
```
```

## Best Practices

1. **Version your API** - Support old clients during transitions
2. **Use proper HTTP status codes** - 200, 201, 204, 400, 401, 403, 404, 422, 429
3. **Paginate collections** - Don't return unbounded lists
4. **Include metadata** - Pagination info, timestamps
5. **Handle errors consistently** - Same format for all errors
6. **Rate limit** - Protect against abuse
7. **Document everything** - Request/response examples
