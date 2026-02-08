# Skill: Authentication

## Purpose
Implement user authentication using Rails 8's built-in authentication generator.

## Rails 8 Authentication Generator

### Generate Authentication
```bash
rails generate authentication
```

### Generated Files
```
app/models/user.rb
app/models/session.rb
app/models/current.rb
app/controllers/sessions_controller.rb
app/controllers/passwords_controller.rb
app/views/sessions/new.html.erb
app/views/passwords/new.html.erb
app/views/passwords/edit.html.erb
db/migrate/TIMESTAMP_create_users.rb
db/migrate/TIMESTAMP_create_sessions.rb
```

## User Model

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy

  normalizes :email_address, with: ->(e) { e.strip.downcase }

  validates :email_address, presence: true, uniqueness: true,
            format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, length: { minimum: 8 }, allow_nil: true
end
```

## Session Model

```ruby
# app/models/session.rb
class Session < ApplicationRecord
  belongs_to :user

  before_create do
    self.token = SecureRandom.urlsafe_base64(32)
  end
end
```

## Current Model

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :session
  delegate :user, to: :session, allow_nil: true
end
```

## Authentication Concern

```ruby
# app/controllers/concerns/authentication.rb
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :require_authentication
    helper_method :signed_in?
  end

  class_methods do
    def allow_unauthenticated_access(**options)
      skip_before_action :require_authentication, **options
    end
  end

  private

  def signed_in?
    Current.session.present?
  end

  def require_authentication
    resume_session || request_authentication
  end

  def resume_session
    Current.session = find_session_by_cookie
  end

  def find_session_by_cookie
    Session.find_by(token: cookies.signed[:session_token])
  end

  def request_authentication
    session[:return_to_after_authenticating] = request.url
    redirect_to new_session_path, alert: "Please sign in to continue."
  end

  def after_authentication_url
    session.delete(:return_to_after_authenticating) || root_url
  end

  def start_new_session_for(user)
    user.sessions.create!.tap do |session|
      Current.session = session
      cookies.signed.permanent[:session_token] = {
        value: session.token,
        httponly: true,
        same_site: :lax
      }
    end
  end

  def terminate_session
    Current.session.destroy
    cookies.delete(:session_token)
  end
end
```

## Sessions Controller

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  allow_unauthenticated_access only: %i[new create]
  rate_limit to: 10, within: 3.minutes, only: :create, with: -> {
    redirect_to new_session_url, alert: "Too many attempts. Try again later."
  }

  def new
  end

  def create
    if user = User.authenticate_by(email_address: params[:email_address], password: params[:password])
      start_new_session_for user
      redirect_to after_authentication_url, notice: "Signed in successfully."
    else
      redirect_to new_session_path, alert: "Invalid email or password."
    end
  end

  def destroy
    terminate_session
    redirect_to new_session_path, notice: "Signed out successfully."
  end
end
```

## Registration

### Add Registration Controller
```ruby
# app/controllers/registrations_controller.rb
class RegistrationsController < ApplicationController
  allow_unauthenticated_access

  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)

    if @user.save
      start_new_session_for @user
      redirect_to root_path, notice: "Welcome! Your account has been created."
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def user_params
    params.require(:user).permit(:email_address, :password, :password_confirmation)
  end
end
```

### Registration View
```erb
<%# app/views/registrations/new.html.erb %>
<div class="max-w-md mx-auto mt-10">
  <h1 class="text-2xl font-bold mb-6">Create Account</h1>

  <%= form_with model: @user, url: registration_path, class: "space-y-4" do |f| %>
    <% if @user.errors.any? %>
      <div class="bg-red-50 border border-red-200 rounded p-4">
        <ul class="list-disc list-inside text-red-700 text-sm">
          <% @user.errors.full_messages.each do |error| %>
            <li><%= error %></li>
          <% end %>
        </ul>
      </div>
    <% end %>

    <div>
      <%= f.label :email_address, class: "block text-sm font-medium text-gray-700" %>
      <%= f.email_field :email_address, required: true, autofocus: true,
          class: "mt-1 w-full rounded-lg border-gray-300" %>
    </div>

    <div>
      <%= f.label :password, class: "block text-sm font-medium text-gray-700" %>
      <%= f.password_field :password, required: true,
          class: "mt-1 w-full rounded-lg border-gray-300" %>
    </div>

    <div>
      <%= f.label :password_confirmation, class: "block text-sm font-medium text-gray-700" %>
      <%= f.password_field :password_confirmation, required: true,
          class: "mt-1 w-full rounded-lg border-gray-300" %>
    </div>

    <%= f.submit "Create Account",
        class: "w-full bg-blue-600 hover:bg-blue-700 text-white py-2 rounded-lg" %>
  <% end %>

  <p class="mt-4 text-center text-sm text-gray-600">
    Already have an account?
    <%= link_to "Sign in", new_session_path, class: "text-blue-600 hover:underline" %>
  </p>
</div>
```

### Routes
```ruby
# config/routes.rb
Rails.application.routes.draw do
  resource :session, only: [:new, :create, :destroy]
  resource :registration, only: [:new, :create]
  resource :password, only: [:new, :create, :edit, :update]

  # ...
end
```

## Login View

```erb
<%# app/views/sessions/new.html.erb %>
<div class="max-w-md mx-auto mt-10">
  <h1 class="text-2xl font-bold mb-6">Sign In</h1>

  <%= form_with url: session_path, class: "space-y-4" do |f| %>
    <div>
      <%= f.label :email_address, class: "block text-sm font-medium text-gray-700" %>
      <%= f.email_field :email_address, required: true, autofocus: true,
          class: "mt-1 w-full rounded-lg border-gray-300" %>
    </div>

    <div>
      <%= f.label :password, class: "block text-sm font-medium text-gray-700" %>
      <%= f.password_field :password, required: true,
          class: "mt-1 w-full rounded-lg border-gray-300" %>
    </div>

    <%= f.submit "Sign In",
        class: "w-full bg-blue-600 hover:bg-blue-700 text-white py-2 rounded-lg" %>
  <% end %>

  <div class="mt-4 text-center text-sm text-gray-600">
    <%= link_to "Forgot password?", new_password_path, class: "text-blue-600 hover:underline" %>
    <span class="mx-2">|</span>
    <%= link_to "Create account", new_registration_path, class: "text-blue-600 hover:underline" %>
  </div>
</div>
```

## Application Controller

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Authentication
end
```

## Current User Helper

```erb
<%# In views %>
<% if signed_in? %>
  <span>Welcome, <%= Current.user.email_address %></span>
  <%= button_to "Sign Out", session_path, method: :delete %>
<% else %>
  <%= link_to "Sign In", new_session_path %>
<% end %>
```

## Security Best Practices

1. **Rate limiting** - Prevent brute force attacks
2. **Secure cookies** - HttpOnly, SameSite, Secure in production
3. **Password requirements** - Minimum 8 characters
4. **Session management** - Token-based, revocable
5. **Email normalization** - Consistent email handling
