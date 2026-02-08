# Skill: WebSockets (Action Cable)

## Purpose

Implementar funcionalidades en tiempo real usando Action Cable, el framework de WebSockets integrado en Rails.

## Setup

### Configuración básica

```ruby
# config/cable.yml
development:
  adapter: async

test:
  adapter: test

production:
  adapter: solid_cable  # Rails 8
  # O Redis:
  # adapter: redis
  # url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  # channel_prefix: myapp_production
```

```ruby
# config/environments/production.rb
config.action_cable.url = "wss://myapp.com/cable"
config.action_cable.allowed_request_origins = [
  "https://myapp.com",
  /https:\/\/.*\.myapp\.com/
]
```

### JavaScript setup

```javascript
// app/javascript/application.js
import "@hotwired/turbo-rails"
import "./channels"
```

```javascript
// app/javascript/channels/index.js
import { createConsumer } from "@hotwired/actioncable"
window.App = { cable: createConsumer() }
```

## Canales básicos

### Canal de chat

```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def subscribed
    @room = Room.find(params[:room_id])

    # Verificar autorización
    reject unless can_access_room?

    stream_for @room
  end

  def unsubscribed
    # Limpiar cuando el usuario se desconecta
  end

  def speak(data)
    message = @room.messages.create!(
      user: current_user,
      body: data["body"]
    )

    # Broadcast a todos los suscriptores
    ChatChannel.broadcast_to(@room, {
      message: render_message(message),
      user_id: current_user.id
    })
  end

  def typing
    ChatChannel.broadcast_to(@room, {
      type: "typing",
      user_id: current_user.id,
      user_name: current_user.name
    })
  end

  private

  def can_access_room?
    @room.users.include?(current_user)
  end

  def render_message(message)
    ApplicationController.render(
      partial: "messages/message",
      locals: { message: message }
    )
  end
end
```

```javascript
// app/javascript/channels/chat_channel.js
import consumer from "./consumer"

document.addEventListener("turbo:load", () => {
  const chatRoom = document.getElementById("chat-room")
  if (!chatRoom) return

  const roomId = chatRoom.dataset.roomId

  consumer.subscriptions.create(
    { channel: "ChatChannel", room_id: roomId },
    {
      connected() {
        console.log("Connected to chat")
      },

      disconnected() {
        console.log("Disconnected from chat")
      },

      received(data) {
        if (data.type === "typing") {
          this.showTypingIndicator(data.user_name)
        } else {
          this.appendMessage(data.message)
        }
      },

      speak(body) {
        this.perform("speak", { body: body })
      },

      typing() {
        this.perform("typing")
      },

      appendMessage(html) {
        const messages = document.getElementById("messages")
        messages.insertAdjacentHTML("beforeend", html)
        messages.scrollTop = messages.scrollHeight
      },

      showTypingIndicator(userName) {
        const indicator = document.getElementById("typing-indicator")
        indicator.textContent = `${userName} is typing...`
        indicator.classList.remove("hidden")

        clearTimeout(this.typingTimeout)
        this.typingTimeout = setTimeout(() => {
          indicator.classList.add("hidden")
        }, 2000)
      }
    }
  )
})
```

### Canal de notificaciones

```ruby
# app/channels/notifications_channel.rb
class NotificationsChannel < ApplicationCable::Channel
  def subscribed
    stream_for current_user
  end

  def mark_as_read(data)
    notification = current_user.notifications.find(data["id"])
    notification.mark_as_read!

    NotificationsChannel.broadcast_to(current_user, {
      type: "read",
      id: notification.id
    })
  end
end
```

### Canal de presencia (quién está online)

```ruby
# app/channels/presence_channel.rb
class PresenceChannel < ApplicationCable::Channel
  def subscribed
    @room = Room.find(params[:room_id])
    stream_for @room

    # Registrar usuario como online
    add_user_to_presence

    # Notificar a otros
    broadcast_presence
  end

  def unsubscribed
    remove_user_from_presence
    broadcast_presence
  end

  private

  def presence_key
    "room:#{@room.id}:presence"
  end

  def add_user_to_presence
    Rails.cache.write(
      "#{presence_key}:#{current_user.id}",
      { id: current_user.id, name: current_user.name },
      expires_in: 5.minutes
    )
  end

  def remove_user_from_presence
    Rails.cache.delete("#{presence_key}:#{current_user.id}")
  end

  def online_users
    keys = Rails.cache.redis.keys("#{presence_key}:*")
    keys.map { |key| Rails.cache.read(key) }.compact
  end

  def broadcast_presence
    PresenceChannel.broadcast_to(@room, {
      type: "presence",
      users: online_users
    })
  end
end
```

## Integración con Turbo Streams

### Broadcast desde modelos

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  belongs_to :room
  belongs_to :user

  # Broadcast automático con Turbo
  broadcasts_to :room

  # O personalizado:
  after_create_commit -> {
    broadcast_append_to room,
      target: "messages",
      partial: "messages/message",
      locals: { message: self }
  }

  after_update_commit -> {
    broadcast_replace_to room,
      target: dom_id(self),
      partial: "messages/message",
      locals: { message: self }
  }

  after_destroy_commit -> {
    broadcast_remove_to room, target: dom_id(self)
  }
end
```

### Vista con Turbo Streams

```erb
<%# app/views/rooms/show.html.erb %>
<div id="chat-room" data-room-id="<%= @room.id %>">
  <%# Suscribirse a updates de Turbo Stream %>
  <%= turbo_stream_from @room %>

  <div id="messages" class="h-96 overflow-y-auto">
    <%= render @room.messages.includes(:user).order(created_at: :asc) %>
  </div>

  <div id="typing-indicator" class="hidden text-sm text-gray-500 italic"></div>

  <%= form_with url: room_messages_path(@room),
                data: { controller: "chat", action: "submit->chat#send" } do |f| %>
    <%= f.text_field :body,
                     placeholder: t(".type_message"),
                     autocomplete: "off",
                     data: { chat_target: "input", action: "input->chat#typing" },
                     class: "w-full px-4 py-2 border rounded-lg" %>
  <% end %>
</div>
```

## Autenticación en canales

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    private

    def find_verified_user
      # Opción 1: Session-based (para apps web)
      if verified_user = User.find_by(id: session["user_id"])
        verified_user
      # Opción 2: Token-based (para móvil/API)
      elsif verified_user = User.find_by(auth_token: request.params[:token])
        verified_user
      else
        reject_unauthorized_connection
      end
    end

    def session
      @session ||= cookies.encrypted[Rails.application.config.session_options[:key]]
    end
  end
end
```

## Patrones avanzados

### Rate limiting

```ruby
# app/channels/concerns/rate_limitable.rb
module RateLimitable
  extend ActiveSupport::Concern

  class_methods do
    def rate_limit(method_name, limit:, period:)
      original_method = instance_method(method_name)

      define_method(method_name) do |data = {}|
        key = "rate_limit:#{self.class.name}:#{method_name}:#{current_user.id}"
        count = Rails.cache.increment(key, 1, expires_in: period)

        if count > limit
          transmit(error: "Rate limit exceeded. Try again later.")
          return
        end

        original_method.bind(self).call(data)
      end
    end
  end
end

# Uso en canal
class ChatChannel < ApplicationCable::Channel
  include RateLimitable

  rate_limit :speak, limit: 10, period: 1.minute
end
```

### Heartbeat para presencia

```ruby
# app/channels/presence_channel.rb
class PresenceChannel < ApplicationCable::Channel
  periodically :heartbeat, every: 30.seconds

  def heartbeat
    update_presence
    broadcast_presence
  end
end
```

### Broadcast condicional

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  after_create_commit :broadcast_to_relevant_users

  private

  def broadcast_to_relevant_users
    # Solo broadcast a usuarios que pueden ver el comentario
    post.authorized_viewers.each do |user|
      Turbo::StreamsChannel.broadcast_append_to(
        [user, "notifications"],
        target: "notifications",
        partial: "notifications/new_comment",
        locals: { comment: self }
      )
    end
  end
end
```

## Stimulus Controller para WebSockets

```javascript
// app/javascript/controllers/chat_controller.js
import { Controller } from "@hotwired/stimulus"
import consumer from "../channels/consumer"

export default class extends Controller {
  static targets = ["input", "messages", "typing"]
  static values = { roomId: Number }

  connect() {
    this.subscription = consumer.subscriptions.create(
      { channel: "ChatChannel", room_id: this.roomIdValue },
      {
        connected: () => this.connected(),
        disconnected: () => this.disconnected(),
        received: (data) => this.received(data)
      }
    )
  }

  disconnect() {
    this.subscription?.unsubscribe()
  }

  connected() {
    this.element.classList.remove("opacity-50")
    console.log("Chat connected")
  }

  disconnected() {
    this.element.classList.add("opacity-50")
    console.log("Chat disconnected")
  }

  received(data) {
    switch (data.type) {
      case "typing":
        this.showTyping(data.user_name)
        break
      case "message":
        this.appendMessage(data.html)
        break
    }
  }

  send(event) {
    event.preventDefault()
    const body = this.inputTarget.value.trim()
    if (!body) return

    this.subscription.perform("speak", { body })
    this.inputTarget.value = ""
  }

  typing() {
    this.subscription.perform("typing")
  }

  appendMessage(html) {
    this.messagesTarget.insertAdjacentHTML("beforeend", html)
    this.scrollToBottom()
  }

  showTyping(userName) {
    this.typingTarget.textContent = `${userName} está escribiendo...`
    this.typingTarget.classList.remove("hidden")

    clearTimeout(this.typingTimeout)
    this.typingTimeout = setTimeout(() => {
      this.typingTarget.classList.add("hidden")
    }, 2000)
  }

  scrollToBottom() {
    this.messagesTarget.scrollTop = this.messagesTarget.scrollHeight
  }
}
```

## Testing

```ruby
# spec/channels/chat_channel_spec.rb
require "rails_helper"

RSpec.describe ChatChannel, type: :channel do
  let(:user) { create(:user) }
  let(:room) { create(:room, users: [user]) }

  before do
    stub_connection current_user: user
  end

  describe "#subscribed" do
    it "subscribes to the room stream" do
      subscribe(room_id: room.id)
      expect(subscription).to be_confirmed
      expect(subscription).to have_stream_for(room)
    end

    it "rejects unauthorized users" do
      other_room = create(:room)
      subscribe(room_id: other_room.id)
      expect(subscription).to be_rejected
    end
  end

  describe "#speak" do
    before { subscribe(room_id: room.id) }

    it "creates a message" do
      expect {
        perform :speak, body: "Hello!"
      }.to change(Message, :count).by(1)
    end

    it "broadcasts the message" do
      expect {
        perform :speak, body: "Hello!"
      }.to have_broadcasted_to(room).with(
        hash_including(user_id: user.id)
      )
    end
  end
end

# spec/support/action_cable.rb
RSpec.configure do |config|
  config.include ActionCable::TestHelper
end
```

## Debugging

```ruby
# config/environments/development.rb
# Habilitar logs de Action Cable
config.action_cable.log_tags = [:action_cable]
config.log_level = :debug

# Para ver mensajes en consola del navegador
# En app/channels/application_cable/channel.rb
class Channel < ActionCable::Channel::Base
  rescue_from Exception do |exception|
    Rails.logger.error "Channel error: #{exception.message}"
    transmit(error: exception.message)
  end
end
```

## Checklist

- [ ] cable.yml configurado para cada environment
- [ ] CORS configurado en producción
- [ ] Autenticación en Connection
- [ ] Autorización en cada Canal
- [ ] Turbo Streams integrado donde aplica
- [ ] Rate limiting implementado
- [ ] Tests de canales escritos
- [ ] Manejo de errores y reconexión
