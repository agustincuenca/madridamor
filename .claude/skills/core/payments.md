# Skill: Payments (Stripe)

## Purpose

Integrar pagos con Stripe en aplicaciones Rails, incluyendo pagos únicos, suscripciones, y manejo de webhooks.

## Setup

### Instalación

```ruby
# Gemfile
gem "stripe"
gem "pay", "~> 7.0"  # Opcional: abstracción sobre Stripe
```

```bash
bundle install
```

### Configuración

```bash
# Agregar a credentials
rails credentials:edit
```

```yaml
# config/credentials.yml.enc
stripe:
  publishable_key: pk_test_xxx
  secret_key: sk_test_xxx
  webhook_secret: whsec_xxx
```

```ruby
# config/initializers/stripe.rb
Stripe.api_key = Rails.application.credentials.dig(:stripe, :secret_key)
```

## Pagos únicos (Checkout Session)

### Controller

```ruby
# app/controllers/checkouts_controller.rb
class CheckoutsController < ApplicationController
  before_action :authenticate_user!

  def create
    session = Stripe::Checkout::Session.create(
      customer_email: current_user.email,
      payment_method_types: ["card"],
      line_items: [{
        price_data: {
          currency: "eur",
          product_data: {
            name: params[:product_name],
            description: params[:description]
          },
          unit_amount: params[:amount].to_i * 100  # En centavos
        },
        quantity: 1
      }],
      mode: "payment",
      success_url: success_checkout_url + "?session_id={CHECKOUT_SESSION_ID}",
      cancel_url: cancel_checkout_url,
      metadata: {
        user_id: current_user.id,
        product_id: params[:product_id]
      }
    )

    redirect_to session.url, allow_other_host: true
  end

  def success
    @session = Stripe::Checkout::Session.retrieve(params[:session_id])
  end

  def cancel
    flash[:alert] = t(".cancelled")
    redirect_to root_path
  end
end
```

### Routes

```ruby
# config/routes.rb
resources :checkouts, only: [:create] do
  collection do
    get :success
    get :cancel
  end
end
```

### Vista de checkout

```erb
<%# app/views/products/show.html.erb %>
<%= button_to "Comprar por #{number_to_currency(@product.price)}",
              checkouts_path(
                product_name: @product.name,
                amount: @product.price,
                product_id: @product.id
              ),
              class: "btn btn-primary",
              data: { turbo: false } %>
```

## Suscripciones

### Modelos

```ruby
# db/migrate/xxx_add_stripe_fields_to_users.rb
class AddStripeFieldsToUsers < ActiveRecord::Migration[8.0]
  def change
    add_column :users, :stripe_customer_id, :string
    add_column :users, :stripe_subscription_id, :string
    add_column :users, :subscription_status, :string, default: "inactive"
    add_column :users, :subscription_ends_at, :datetime

    add_index :users, :stripe_customer_id, unique: true
  end
end

# app/models/user.rb
class User < ApplicationRecord
  def create_or_get_stripe_customer
    return stripe_customer_id if stripe_customer_id.present?

    customer = Stripe::Customer.create(
      email: email,
      name: name,
      metadata: { user_id: id }
    )

    update!(stripe_customer_id: customer.id)
    customer.id
  end

  def active_subscription?
    subscription_status == "active" &&
      (subscription_ends_at.nil? || subscription_ends_at > Time.current)
  end

  def subscribed?
    %w[active trialing].include?(subscription_status)
  end
end
```

### Controller de suscripciones

```ruby
# app/controllers/subscriptions_controller.rb
class SubscriptionsController < ApplicationController
  before_action :authenticate_user!

  PLANS = {
    "basic" => "price_xxx",
    "pro" => "price_yyy",
    "enterprise" => "price_zzz"
  }.freeze

  def new
    @plans = PLANS
  end

  def create
    price_id = PLANS[params[:plan]]
    return redirect_to new_subscription_path, alert: t(".invalid_plan") unless price_id

    session = Stripe::Checkout::Session.create(
      customer: current_user.create_or_get_stripe_customer,
      payment_method_types: ["card"],
      line_items: [{
        price: price_id,
        quantity: 1
      }],
      mode: "subscription",
      success_url: subscription_success_url + "?session_id={CHECKOUT_SESSION_ID}",
      cancel_url: subscription_cancel_url,
      metadata: {
        user_id: current_user.id
      }
    )

    redirect_to session.url, allow_other_host: true
  end

  def success
    @session = Stripe::Checkout::Session.retrieve(
      params[:session_id],
      expand: ["subscription"]
    )

    flash[:notice] = t(".success")
  end

  def cancel
    flash[:alert] = t(".cancelled")
    redirect_to pricing_path
  end

  def portal
    # Portal de cliente de Stripe para gestionar suscripción
    session = Stripe::BillingPortal::Session.create(
      customer: current_user.stripe_customer_id,
      return_url: dashboard_url
    )

    redirect_to session.url, allow_other_host: true
  end
end
```

## Webhooks

### Controller de webhooks

```ruby
# app/controllers/webhooks/stripe_controller.rb
module Webhooks
  class StripeController < ApplicationController
    skip_before_action :verify_authenticity_token
    skip_before_action :authenticate_user!

    def create
      payload = request.body.read
      sig_header = request.env["HTTP_STRIPE_SIGNATURE"]
      webhook_secret = Rails.application.credentials.dig(:stripe, :webhook_secret)

      begin
        event = Stripe::Webhook.construct_event(
          payload, sig_header, webhook_secret
        )
      rescue JSON::ParserError
        render json: { error: "Invalid payload" }, status: :bad_request
        return
      rescue Stripe::SignatureVerificationError
        render json: { error: "Invalid signature" }, status: :bad_request
        return
      end

      handle_event(event)

      render json: { received: true }
    end

    private

    def handle_event(event)
      case event.type
      when "checkout.session.completed"
        handle_checkout_completed(event.data.object)

      when "customer.subscription.created"
        handle_subscription_created(event.data.object)

      when "customer.subscription.updated"
        handle_subscription_updated(event.data.object)

      when "customer.subscription.deleted"
        handle_subscription_deleted(event.data.object)

      when "invoice.paid"
        handle_invoice_paid(event.data.object)

      when "invoice.payment_failed"
        handle_payment_failed(event.data.object)

      else
        Rails.logger.info "Unhandled Stripe event: #{event.type}"
      end
    end

    def handle_checkout_completed(session)
      return unless session.mode == "payment"

      user = User.find(session.metadata.user_id)
      product_id = session.metadata.product_id

      # Crear orden/compra
      Order.create!(
        user: user,
        product_id: product_id,
        stripe_session_id: session.id,
        amount: session.amount_total / 100.0,
        status: "completed"
      )

      # Enviar email de confirmación
      OrderMailer.confirmation(user, product_id).deliver_later
    end

    def handle_subscription_created(subscription)
      user = User.find_by(stripe_customer_id: subscription.customer)
      return unless user

      user.update!(
        stripe_subscription_id: subscription.id,
        subscription_status: subscription.status,
        subscription_ends_at: Time.at(subscription.current_period_end)
      )
    end

    def handle_subscription_updated(subscription)
      user = User.find_by(stripe_customer_id: subscription.customer)
      return unless user

      user.update!(
        subscription_status: subscription.status,
        subscription_ends_at: subscription.cancel_at_period_end ?
          Time.at(subscription.current_period_end) : nil
      )
    end

    def handle_subscription_deleted(subscription)
      user = User.find_by(stripe_customer_id: subscription.customer)
      return unless user

      user.update!(
        stripe_subscription_id: nil,
        subscription_status: "cancelled",
        subscription_ends_at: Time.current
      )

      SubscriptionMailer.cancelled(user).deliver_later
    end

    def handle_invoice_paid(invoice)
      user = User.find_by(stripe_customer_id: invoice.customer)
      return unless user

      # Registrar pago
      Payment.create!(
        user: user,
        stripe_invoice_id: invoice.id,
        amount: invoice.amount_paid / 100.0,
        status: "paid"
      )
    end

    def handle_payment_failed(invoice)
      user = User.find_by(stripe_customer_id: invoice.customer)
      return unless user

      # Notificar al usuario
      PaymentMailer.failed(user, invoice.id).deliver_later
    end
  end
end
```

### Routes para webhooks

```ruby
# config/routes.rb
namespace :webhooks do
  post "stripe", to: "stripe#create"
end
```

## Stripe Elements (formulario embebido)

### JavaScript

```javascript
// app/javascript/controllers/stripe_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["card", "errors", "submit"]
  static values = {
    publishableKey: String,
    clientSecret: String
  }

  connect() {
    this.stripe = Stripe(this.publishableKeyValue)
    this.elements = this.stripe.elements()

    this.card = this.elements.create("card", {
      style: {
        base: {
          fontSize: "16px",
          color: "#32325d",
          fontFamily: "-apple-system, BlinkMacSystemFont, sans-serif"
        }
      }
    })

    this.card.mount(this.cardTarget)

    this.card.on("change", (event) => {
      this.errorsTarget.textContent = event.error ? event.error.message : ""
    })
  }

  async submit(event) {
    event.preventDefault()
    this.submitTarget.disabled = true

    const { error, paymentIntent } = await this.stripe.confirmCardPayment(
      this.clientSecretValue,
      {
        payment_method: {
          card: this.card
        }
      }
    )

    if (error) {
      this.errorsTarget.textContent = error.message
      this.submitTarget.disabled = false
    } else {
      // Redirigir a success
      window.location.href = `/payments/success?payment_intent=${paymentIntent.id}`
    }
  }

  disconnect() {
    this.card.destroy()
  }
}
```

### Vista con Stripe Elements

```erb
<%# app/views/payments/new.html.erb %>
<div data-controller="stripe"
     data-stripe-publishable-key-value="<%= Rails.application.credentials.dig(:stripe, :publishable_key) %>"
     data-stripe-client-secret-value="<%= @client_secret %>">

  <%= form_with url: payments_path, data: { action: "submit->stripe#submit" } do |f| %>
    <div class="mb-4">
      <label class="block text-sm font-medium text-gray-700 mb-2">
        <%= t(".card_details") %>
      </label>
      <div data-stripe-target="card"
           class="p-3 border border-gray-300 rounded-lg"></div>
      <p data-stripe-target="errors"
         class="mt-1 text-sm text-red-600"></p>
    </div>

    <%= f.submit t(".pay"),
                 data: { stripe_target: "submit" },
                 class: "w-full bg-blue-600 text-white py-2 px-4 rounded-lg hover:bg-blue-700" %>
  <% end %>
</div>

<%# Incluir Stripe.js %>
<script src="https://js.stripe.com/v3/"></script>
```

## Testing

### Tarjetas de prueba

| Número | Resultado |
|--------|-----------|
| 4242 4242 4242 4242 | Éxito |
| 4000 0000 0000 0002 | Rechazada |
| 4000 0000 0000 3220 | Requiere 3D Secure |
| 4000 0027 6000 3184 | Requiere autenticación |

### Specs

```ruby
# spec/requests/webhooks/stripe_spec.rb
RSpec.describe "Webhooks::Stripe", type: :request do
  let(:webhook_secret) { "whsec_test" }
  let(:user) { create(:user, stripe_customer_id: "cus_xxx") }

  before do
    allow(Rails.application.credentials).to receive(:dig)
      .with(:stripe, :webhook_secret)
      .and_return(webhook_secret)
  end

  def generate_signature(payload)
    timestamp = Time.current.to_i
    signed_payload = "#{timestamp}.#{payload}"
    signature = OpenSSL::HMAC.hexdigest(
      "SHA256",
      webhook_secret,
      signed_payload
    )
    "t=#{timestamp},v1=#{signature}"
  end

  describe "subscription events" do
    it "handles subscription created" do
      payload = {
        type: "customer.subscription.created",
        data: {
          object: {
            id: "sub_xxx",
            customer: user.stripe_customer_id,
            status: "active",
            current_period_end: 1.month.from_now.to_i
          }
        }
      }.to_json

      post webhooks_stripe_path,
           params: payload,
           headers: {
             "Content-Type" => "application/json",
             "Stripe-Signature" => generate_signature(payload)
           }

      expect(response).to have_http_status(:ok)
      expect(user.reload.subscription_status).to eq("active")
    end
  end
end
```

## Manejo de errores

```ruby
# app/services/stripe_service.rb
class StripeService
  class << self
    def create_checkout_session(params)
      Stripe::Checkout::Session.create(params)
    rescue Stripe::CardError => e
      handle_card_error(e)
    rescue Stripe::RateLimitError
      Rails.logger.error "Stripe rate limit exceeded"
      raise
    rescue Stripe::InvalidRequestError => e
      Rails.logger.error "Invalid Stripe request: #{e.message}"
      raise
    rescue Stripe::AuthenticationError
      Rails.logger.error "Stripe authentication failed"
      raise
    rescue Stripe::APIConnectionError
      Rails.logger.error "Stripe API connection error"
      raise
    rescue Stripe::StripeError => e
      Rails.logger.error "Stripe error: #{e.message}"
      raise
    end

    private

    def handle_card_error(error)
      case error.code
      when "card_declined"
        { error: I18n.t("stripe.errors.card_declined") }
      when "expired_card"
        { error: I18n.t("stripe.errors.expired_card") }
      when "incorrect_cvc"
        { error: I18n.t("stripe.errors.incorrect_cvc") }
      else
        { error: error.message }
      end
    end
  end
end
```

## Checklist

- [ ] Stripe gems instaladas
- [ ] Credentials configuradas (publishable, secret, webhook_secret)
- [ ] Webhook endpoint configurado en Stripe Dashboard
- [ ] Eventos de webhook manejados
- [ ] Tarjetas de prueba funcionando
- [ ] Customer portal configurado (para suscripciones)
- [ ] Emails transaccionales configurados
- [ ] Manejo de errores implementado
- [ ] Tests escritos
