# Skill: Notifications

## Purpose

Implementar sistema de notificaciones completo: email, in-app, y push notifications para aplicaciones Rails.

## Email Notifications (Action Mailer)

### Configuración

```ruby
# config/environments/production.rb
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings = {
  address: "smtp.example.com",
  port: 587,
  domain: "myapp.com",
  user_name: Rails.application.credentials.dig(:smtp, :user_name),
  password: Rails.application.credentials.dig(:smtp, :password),
  authentication: "plain",
  enable_starttls_auto: true
}

config.action_mailer.default_url_options = { host: "myapp.com" }
config.action_mailer.perform_deliveries = true

# config/environments/development.rb
config.action_mailer.delivery_method = :letter_opener
config.action_mailer.default_url_options = { host: "localhost", port: 3000 }

# Gemfile (desarrollo)
group :development do
  gem "letter_opener"
end
```

### Mailer base

```ruby
# app/mailers/application_mailer.rb
class ApplicationMailer < ActionMailer::Base
  default from: "Mi App <noreply@myapp.com>"
  layout "mailer"

  # Helper para incluir assets
  helper :application

  # Callback para tracking
  after_action :track_email_sent

  private

  def track_email_sent
    EmailLog.create!(
      to: message.to&.join(", "),
      subject: message.subject,
      mailer: self.class.name,
      action: action_name
    )
  end
end
```

### Mailer de ejemplo

```ruby
# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
  def welcome(user)
    @user = user
    @login_url = new_session_url

    mail(
      to: @user.email,
      subject: t(".subject", name: @user.name)
    )
  end

  def password_reset(user)
    @user = user
    @reset_url = edit_password_url(token: @user.password_reset_token)

    mail(
      to: @user.email,
      subject: t(".subject")
    )
  end

  def notification(user, notification)
    @user = user
    @notification = notification

    mail(
      to: @user.email,
      subject: @notification.title
    )
  end
end
```

### Templates

```erb
<%# app/views/user_mailer/welcome.html.erb %>
<h1><%= t(".greeting", name: @user.name) %></h1>

<p><%= t(".welcome_message") %></p>

<p>
  <%= link_to t(".login_button"), @login_url, class: "button" %>
</p>

<p><%= t(".help_text") %></p>
```

```erb
<%# app/views/layouts/mailer.html.erb %>
<!DOCTYPE html>
<html>
<head>
  <meta content="text/html; charset=UTF-8" http-equiv="Content-Type" />
  <style>
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
      line-height: 1.6;
      color: #333;
      max-width: 600px;
      margin: 0 auto;
      padding: 20px;
    }
    .button {
      display: inline-block;
      padding: 12px 24px;
      background-color: #3B82F6;
      color: white;
      text-decoration: none;
      border-radius: 6px;
    }
    .footer {
      margin-top: 40px;
      padding-top: 20px;
      border-top: 1px solid #eee;
      font-size: 12px;
      color: #666;
    }
  </style>
</head>
<body>
  <%= yield %>

  <div class="footer">
    <p><%= t("mailer.footer.company_name") %></p>
    <p><%= link_to t("mailer.footer.unsubscribe"), unsubscribe_url %></p>
  </div>
</body>
</html>
```

### Envío asíncrono

```ruby
# Enviar en background (recomendado)
UserMailer.welcome(user).deliver_later

# Enviar con prioridad
UserMailer.password_reset(user).deliver_later(queue: :high_priority)

# Enviar con delay
UserMailer.reminder(user).deliver_later(wait: 1.hour)

# Enviar a hora específica
UserMailer.digest(user).deliver_later(wait_until: Date.tomorrow.noon)
```

## In-App Notifications

### Modelo de notificaciones

```ruby
# db/migrate/xxx_create_notifications.rb
class CreateNotifications < ActiveRecord::Migration[8.0]
  def change
    create_table :notifications do |t|
      t.references :recipient, null: false, foreign_key: { to_table: :users }
      t.references :actor, foreign_key: { to_table: :users }
      t.references :notifiable, polymorphic: true
      t.string :action, null: false
      t.string :title, null: false
      t.text :body
      t.string :url
      t.datetime :read_at
      t.datetime :seen_at

      t.timestamps
    end

    add_index :notifications, [:recipient_id, :read_at]
    add_index :notifications, [:recipient_id, :created_at]
  end
end

# app/models/notification.rb
class Notification < ApplicationRecord
  belongs_to :recipient, class_name: "User"
  belongs_to :actor, class_name: "User", optional: true
  belongs_to :notifiable, polymorphic: true, optional: true

  scope :unread, -> { where(read_at: nil) }
  scope :unseen, -> { where(seen_at: nil) }
  scope :recent, -> { order(created_at: :desc).limit(20) }

  after_create_commit :broadcast_to_recipient

  def read?
    read_at.present?
  end

  def mark_as_read!
    update!(read_at: Time.current) unless read?
  end

  def mark_as_seen!
    update!(seen_at: Time.current) unless seen_at?
  end

  private

  def broadcast_to_recipient
    broadcast_prepend_to(
      "notifications_#{recipient_id}",
      target: "notifications",
      partial: "notifications/notification",
      locals: { notification: self }
    )

    # Actualizar contador
    broadcast_replace_to(
      "notifications_#{recipient_id}",
      target: "notifications_count",
      partial: "notifications/count",
      locals: { count: recipient.notifications.unread.count }
    )
  end
end

# app/models/user.rb
class User < ApplicationRecord
  has_many :notifications, foreign_key: :recipient_id, dependent: :destroy

  def notify!(action:, title:, body: nil, url: nil, actor: nil, notifiable: nil)
    notifications.create!(
      action: action,
      title: title,
      body: body,
      url: url,
      actor: actor,
      notifiable: notifiable
    )
  end
end
```

### Service para crear notificaciones

```ruby
# app/services/notification_service.rb
class NotificationService
  class << self
    def notify_new_comment(comment)
      return if comment.user == comment.post.user

      comment.post.user.notify!(
        action: "new_comment",
        title: I18n.t("notifications.new_comment.title", user: comment.user.name),
        body: comment.body.truncate(100),
        url: Rails.application.routes.url_helpers.post_path(comment.post, anchor: "comment_#{comment.id}"),
        actor: comment.user,
        notifiable: comment
      )
    end

    def notify_new_follower(follow)
      follow.followed.notify!(
        action: "new_follower",
        title: I18n.t("notifications.new_follower.title", user: follow.follower.name),
        url: Rails.application.routes.url_helpers.user_path(follow.follower),
        actor: follow.follower,
        notifiable: follow
      )
    end

    def notify_mention(mentionable, mentioned_user, mentioner)
      mentioned_user.notify!(
        action: "mention",
        title: I18n.t("notifications.mention.title", user: mentioner.name),
        body: mentionable.body.truncate(100),
        url: polymorphic_url(mentionable),
        actor: mentioner,
        notifiable: mentionable
      )
    end
  end

  private

  def self.polymorphic_url(record)
    Rails.application.routes.url_helpers.polymorphic_path(record)
  end
end
```

### Controller

```ruby
# app/controllers/notifications_controller.rb
class NotificationsController < ApplicationController
  before_action :authenticate_user!

  def index
    @notifications = current_user.notifications.recent.includes(:actor, :notifiable)

    # Marcar como vistas
    current_user.notifications.unseen.update_all(seen_at: Time.current)
  end

  def mark_as_read
    @notification = current_user.notifications.find(params[:id])
    @notification.mark_as_read!

    respond_to do |format|
      format.html { redirect_to @notification.url || notifications_path }
      format.turbo_stream
    end
  end

  def mark_all_as_read
    current_user.notifications.unread.update_all(read_at: Time.current)

    respond_to do |format|
      format.html { redirect_to notifications_path, notice: t(".success") }
      format.turbo_stream
    end
  end
end
```

### Vistas con Turbo

```erb
<%# app/views/notifications/index.html.erb %>
<div class="max-w-2xl mx-auto py-8">
  <div class="flex justify-between items-center mb-6">
    <h1 class="text-2xl font-bold"><%= t(".title") %></h1>
    <% if @notifications.unread.any? %>
      <%= button_to t(".mark_all_read"),
                    mark_all_as_read_notifications_path,
                    method: :post,
                    class: "text-blue-600 hover:text-blue-800" %>
    <% end %>
  </div>

  <%= turbo_stream_from "notifications_#{current_user.id}" %>

  <div id="notifications" class="space-y-2">
    <%= render @notifications %>

    <% if @notifications.empty? %>
      <div class="text-center py-12 text-gray-500">
        <%= t(".empty") %>
      </div>
    <% end %>
  </div>
</div>
```

```erb
<%# app/views/notifications/_notification.html.erb %>
<%= turbo_frame_tag dom_id(notification) do %>
  <div class="flex items-start p-4 rounded-lg <%= notification.read? ? 'bg-white' : 'bg-blue-50' %>">
    <% if notification.actor %>
      <%= image_tag notification.actor.avatar_url,
                    class: "w-10 h-10 rounded-full mr-3",
                    alt: notification.actor.name %>
    <% end %>

    <div class="flex-1">
      <p class="font-medium text-gray-900"><%= notification.title %></p>
      <% if notification.body.present? %>
        <p class="text-gray-600 text-sm mt-1"><%= notification.body %></p>
      <% end %>
      <p class="text-gray-400 text-xs mt-1">
        <%= time_ago_in_words(notification.created_at) %>
      </p>
    </div>

    <% unless notification.read? %>
      <%= button_to mark_as_read_notification_path(notification),
                    method: :patch,
                    class: "text-gray-400 hover:text-gray-600",
                    title: t("notifications.mark_as_read") do %>
        <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 20 20">
          <circle cx="10" cy="10" r="4"/>
        </svg>
      <% end %>
    <% end %>
  </div>
<% end %>
```

### Dropdown de notificaciones (navbar)

```erb
<%# app/views/shared/_notifications_dropdown.html.erb %>
<div data-controller="dropdown" class="relative">
  <button data-action="dropdown#toggle"
          class="relative p-2 text-gray-600 hover:text-gray-900">
    <svg class="w-6 h-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
            d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9"/>
    </svg>

    <%# Contador %>
    <span id="notifications_count">
      <%= render "notifications/count", count: current_user.notifications.unread.count %>
    </span>
  </button>

  <div data-dropdown-target="menu"
       class="hidden absolute right-0 mt-2 w-80 bg-white rounded-lg shadow-lg z-50">
    <div class="p-4 border-b">
      <h3 class="font-semibold"><%= t("notifications.title") %></h3>
    </div>

    <div id="notifications_preview" class="max-h-96 overflow-y-auto">
      <%= turbo_stream_from "notifications_#{current_user.id}" %>
      <%= render current_user.notifications.recent.limit(5) %>
    </div>

    <div class="p-3 border-t text-center">
      <%= link_to t("notifications.view_all"), notifications_path,
                  class: "text-blue-600 hover:text-blue-800 text-sm" %>
    </div>
  </div>
</div>
```

```erb
<%# app/views/notifications/_count.html.erb %>
<% if count > 0 %>
  <span class="absolute -top-1 -right-1 bg-red-500 text-white text-xs
               rounded-full h-5 w-5 flex items-center justify-center">
    <%= count > 99 ? "99+" : count %>
  </span>
<% end %>
```

## Push Notifications (Web Push)

### Setup

```ruby
# Gemfile
gem "web-push"
```

```bash
# Generar VAPID keys
rails c
> vapid_key = WebPush.generate_key
> puts vapid_key.public_key
> puts vapid_key.private_key
```

```yaml
# config/credentials.yml.enc
web_push:
  public_key: xxx
  private_key: xxx
```

### Modelo para subscripciones

```ruby
# db/migrate/xxx_create_push_subscriptions.rb
class CreatePushSubscriptions < ActiveRecord::Migration[8.0]
  def change
    create_table :push_subscriptions do |t|
      t.references :user, null: false, foreign_key: true
      t.string :endpoint, null: false
      t.string :p256dh_key, null: false
      t.string :auth_key, null: false

      t.timestamps
    end

    add_index :push_subscriptions, :endpoint, unique: true
  end
end

# app/models/push_subscription.rb
class PushSubscription < ApplicationRecord
  belongs_to :user

  validates :endpoint, presence: true, uniqueness: true
  validates :p256dh_key, :auth_key, presence: true
end
```

### Service Worker

```javascript
// public/service-worker.js
self.addEventListener("push", (event) => {
  const data = event.data.json()

  const options = {
    body: data.body,
    icon: "/icon-192.png",
    badge: "/badge.png",
    data: { url: data.url },
    actions: data.actions || []
  }

  event.waitUntil(
    self.registration.showNotification(data.title, options)
  )
})

self.addEventListener("notificationclick", (event) => {
  event.notification.close()

  if (event.notification.data?.url) {
    event.waitUntil(
      clients.openWindow(event.notification.data.url)
    )
  }
})
```

### JavaScript para subscription

```javascript
// app/javascript/controllers/push_notifications_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { vapidPublicKey: String }

  async connect() {
    if (!("serviceWorker" in navigator) || !("PushManager" in window)) {
      console.log("Push notifications not supported")
      return
    }

    this.registration = await navigator.serviceWorker.register("/service-worker.js")
  }

  async subscribe() {
    const permission = await Notification.requestPermission()
    if (permission !== "granted") return

    const subscription = await this.registration.pushManager.subscribe({
      userVisibleOnly: true,
      applicationServerKey: this.urlBase64ToUint8Array(this.vapidPublicKeyValue)
    })

    // Enviar al servidor
    const response = await fetch("/push_subscriptions", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-CSRF-Token": document.querySelector("[name='csrf-token']").content
      },
      body: JSON.stringify({ subscription: subscription.toJSON() })
    })

    if (response.ok) {
      this.element.textContent = "Notifications enabled"
    }
  }

  urlBase64ToUint8Array(base64String) {
    const padding = "=".repeat((4 - (base64String.length % 4)) % 4)
    const base64 = (base64String + padding).replace(/-/g, "+").replace(/_/g, "/")
    const rawData = window.atob(base64)
    const outputArray = new Uint8Array(rawData.length)
    for (let i = 0; i < rawData.length; ++i) {
      outputArray[i] = rawData.charCodeAt(i)
    }
    return outputArray
  }
}
```

### Service para enviar push

```ruby
# app/services/push_notification_service.rb
class PushNotificationService
  def self.send_to_user(user, title:, body:, url: nil)
    user.push_subscriptions.find_each do |subscription|
      send_notification(subscription, title: title, body: body, url: url)
    end
  end

  def self.send_notification(subscription, title:, body:, url: nil)
    message = {
      title: title,
      body: body,
      url: url
    }.to_json

    WebPush.payload_send(
      message: message,
      endpoint: subscription.endpoint,
      p256dh: subscription.p256dh_key,
      auth: subscription.auth_key,
      vapid: {
        public_key: Rails.application.credentials.dig(:web_push, :public_key),
        private_key: Rails.application.credentials.dig(:web_push, :private_key)
      }
    )
  rescue WebPush::ExpiredSubscription
    subscription.destroy
  rescue WebPush::InvalidSubscription
    subscription.destroy
  end
end
```

## Preferencias de notificación

```ruby
# db/migrate/xxx_create_notification_preferences.rb
class CreateNotificationPreferences < ActiveRecord::Migration[8.0]
  def change
    create_table :notification_preferences do |t|
      t.references :user, null: false, foreign_key: true
      t.boolean :email_new_comment, default: true
      t.boolean :email_new_follower, default: true
      t.boolean :email_mentions, default: true
      t.boolean :email_digest, default: true
      t.boolean :push_new_comment, default: true
      t.boolean :push_new_follower, default: false
      t.boolean :push_mentions, default: true

      t.timestamps
    end
  end
end

# app/models/notification_preference.rb
class NotificationPreference < ApplicationRecord
  belongs_to :user

  def should_notify?(channel, action)
    attribute = "#{channel}_#{action}"
    respond_to?(attribute) && send(attribute)
  end
end
```

## Checklist

- [ ] Action Mailer configurado
- [ ] Templates de email creados
- [ ] Modelo de notificaciones in-app
- [ ] Turbo Streams para tiempo real
- [ ] Push notifications (opcional)
- [ ] Preferencias de usuario
- [ ] Tests de mailers
- [ ] Emails enviados en background
