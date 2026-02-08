# Skill: Testing

## Purpose
Write comprehensive tests using RSpec, FactoryBot, Faker, and Shoulda Matchers.

## Setup

### Gemfile
```ruby
group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
end

group :test do
  gem "shoulda-matchers"
  gem "capybara"
  gem "selenium-webdriver"
end
```

### Install RSpec
```bash
rails generate rspec:install
```

### Configure RSpec
```ruby
# spec/rails_helper.rb
require "spec_helper"
ENV["RAILS_ENV"] ||= "test"
require_relative "../config/environment"

abort("Running in production!") if Rails.env.production?
require "rspec/rails"

Dir[Rails.root.join("spec/support/**/*.rb")].sort.each { |f| require f }

RSpec.configure do |config|
  config.fixture_paths = [Rails.root.join("spec/fixtures")]
  config.use_transactional_fixtures = true
  config.infer_spec_type_from_file_location!
  config.filter_rails_from_backtrace!

  config.include FactoryBot::Syntax::Methods
end

Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

## Factories

### Basic Factory
```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    email_address { Faker::Internet.unique.email }
    password { "password123" }
    password_confirmation { "password123" }

    trait :admin do
      role { "admin" }
    end

    trait :with_articles do
      transient do
        articles_count { 3 }
      end

      after(:create) do |user, evaluator|
        create_list(:article, evaluator.articles_count, user: user)
      end
    end
  end
end
```

### Associated Factory
```ruby
# spec/factories/articles.rb
FactoryBot.define do
  factory :article do
    title { Faker::Lorem.sentence }
    body { Faker::Lorem.paragraphs(number: 3).join("\n\n") }
    published { false }
    user

    trait :published do
      published { true }
    end

    trait :with_comments do
      transient do
        comments_count { 5 }
      end

      after(:create) do |article, evaluator|
        create_list(:comment, evaluator.comments_count, article: article)
      end
    end
  end
end
```

## Model Specs

```ruby
# spec/models/article_spec.rb
require "rails_helper"

RSpec.describe Article, type: :model do
  describe "validations" do
    it { should validate_presence_of(:title) }
    it { should validate_length_of(:title).is_at_most(255) }
    it { should validate_presence_of(:body) }
  end

  describe "associations" do
    it { should belong_to(:user) }
    it { should have_many(:comments).dependent(:destroy) }
    it { should have_many(:taggings).dependent(:destroy) }
    it { should have_many(:tags).through(:taggings) }
  end

  describe "scopes" do
    describe ".published" do
      it "returns only published articles" do
        published = create(:article, :published)
        draft = create(:article, published: false)

        expect(Article.published).to include(published)
        expect(Article.published).not_to include(draft)
      end
    end

    describe ".recent" do
      it "orders by created_at descending" do
        old = create(:article, created_at: 1.week.ago)
        new = create(:article, created_at: 1.day.ago)

        expect(Article.recent).to eq([new, old])
      end
    end
  end

  describe "#publish!" do
    it "marks article as published" do
      article = create(:article, published: false)

      article.publish!

      expect(article.reload).to be_published
    end
  end

  describe "callbacks" do
    it "generates slug before save" do
      article = build(:article, title: "Hello World")

      article.save

      expect(article.slug).to eq("hello-world")
    end
  end
end
```

## Request Specs

```ruby
# spec/requests/articles_spec.rb
require "rails_helper"

RSpec.describe "Articles", type: :request do
  let(:user) { create(:user) }
  let(:article) { create(:article, user: user) }

  describe "GET /articles" do
    it "returns success" do
      get articles_path
      expect(response).to have_http_status(:success)
    end

    it "displays articles" do
      article = create(:article, :published)
      get articles_path
      expect(response.body).to include(article.title)
    end
  end

  describe "GET /articles/:id" do
    context "when article is published" do
      it "returns success" do
        article = create(:article, :published)
        get article_path(article)
        expect(response).to have_http_status(:success)
      end
    end

    context "when article is draft" do
      it "redirects unauthorized users" do
        article = create(:article, published: false)
        get article_path(article)
        expect(response).to redirect_to(new_session_path)
      end
    end
  end

  describe "POST /articles" do
    context "when authenticated" do
      before { sign_in(user) }

      it "creates article with valid params" do
        expect {
          post articles_path, params: { article: { title: "Test", body: "Content" } }
        }.to change(Article, :count).by(1)
      end

      it "does not create with invalid params" do
        expect {
          post articles_path, params: { article: { title: "", body: "" } }
        }.not_to change(Article, :count)
      end
    end

    context "when not authenticated" do
      it "redirects to login" do
        post articles_path, params: { article: { title: "Test", body: "Content" } }
        expect(response).to redirect_to(new_session_path)
      end
    end
  end

  describe "PATCH /articles/:id" do
    before { sign_in(user) }

    it "updates article" do
      patch article_path(article), params: { article: { title: "Updated" } }
      expect(article.reload.title).to eq("Updated")
    end
  end

  describe "DELETE /articles/:id" do
    before { sign_in(user) }

    it "destroys article" do
      article # create it
      expect {
        delete article_path(article)
      }.to change(Article, :count).by(-1)
    end
  end
end
```

## System Specs

```ruby
# spec/system/articles_spec.rb
require "rails_helper"

RSpec.describe "Articles", type: :system do
  let(:user) { create(:user) }

  before do
    driven_by(:selenium_chrome_headless)
  end

  describe "creating an article" do
    before { sign_in(user) }

    it "allows user to create article" do
      visit new_article_path

      fill_in "Title", with: "My Article"
      fill_in "Body", with: "Article content here"
      click_button "Create Article"

      expect(page).to have_content("Article created successfully")
      expect(page).to have_content("My Article")
    end

    it "shows validation errors" do
      visit new_article_path

      click_button "Create Article"

      expect(page).to have_content("can't be blank")
    end
  end

  describe "editing an article" do
    let!(:article) { create(:article, user: user, title: "Original") }

    before { sign_in(user) }

    it "allows owner to edit" do
      visit article_path(article)
      click_link "Edit"

      fill_in "Title", with: "Updated Title"
      click_button "Update Article"

      expect(page).to have_content("Updated Title")
    end
  end

  describe "with Turbo interactions" do
    before { sign_in(user) }

    it "adds comment without page reload", js: true do
      article = create(:article, :published)
      visit article_path(article)

      fill_in "Comment", with: "Great article!"
      click_button "Post Comment"

      within("#comments") do
        expect(page).to have_content("Great article!")
      end
    end
  end
end
```

## Test Helpers

```ruby
# spec/support/authentication_helper.rb
module AuthenticationHelper
  def sign_in(user)
    post session_path, params: {
      email_address: user.email_address,
      password: "password123"
    }
  end
end

RSpec.configure do |config|
  config.include AuthenticationHelper, type: :request
end

# spec/support/system_helper.rb
module SystemHelper
  def sign_in(user)
    visit new_session_path
    fill_in "Email", with: user.email_address
    fill_in "Password", with: "password123"
    click_button "Sign In"
  end
end

RSpec.configure do |config|
  config.include SystemHelper, type: :system
end
```

## Running Tests

```bash
# All tests
bundle exec rspec

# Specific file
bundle exec rspec spec/models/article_spec.rb

# Specific test
bundle exec rspec spec/models/article_spec.rb:25

# By type
bundle exec rspec spec/models
bundle exec rspec spec/requests
bundle exec rspec spec/system

# With format
bundle exec rspec --format documentation
```

## Best Practices

1. **One assertion per example** when possible
2. **Use let/let!** for setup, not before blocks
3. **Use traits** for variations
4. **Test behavior, not implementation**
5. **Use factories, not fixtures**
6. **Keep tests fast** - mock external services
