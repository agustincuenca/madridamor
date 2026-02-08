# Skill: Security

## Purpose

Implementar y mantener la seguridad de aplicaciones Rails siguiendo las mejores prácticas y estándares de la industria (OWASP).

## Herramientas de auditoría

### Brakeman (análisis estático)

```bash
# Gemfile
group :development do
  gem "brakeman", require: false
end

# Ejecutar
bundle exec brakeman

# Generar reporte HTML
bundle exec brakeman -o tmp/brakeman-report.html

# Solo warnings altos
bundle exec brakeman -w2

# Ignorar falsos positivos (crear config/brakeman.ignore)
bundle exec brakeman -I
```

### Bundler Audit (dependencias vulnerables)

```bash
# Gemfile
group :development do
  gem "bundler-audit", require: false
end

# Actualizar DB de vulnerabilidades
bundle exec bundle-audit update

# Ejecutar auditoría
bundle exec bundle-audit check

# Actualizar gems vulnerables automáticamente
bundle exec bundle-audit check --update
```

### CI/CD Integration

```yaml
# .github/workflows/security.yml
name: Security

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true

      - name: Run Brakeman
        run: bundle exec brakeman -q -w2

      - name: Run Bundle Audit
        run: |
          bundle exec bundle-audit update
          bundle exec bundle-audit check
```

## OWASP Top 10 - Prevención

### A01: Broken Access Control

```ruby
# ✅ Usar Pundit para autorización
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    authorize @post  # Verificar permisos
  end

  def index
    @posts = policy_scope(Post)  # Solo posts autorizados
  end
end

# ✅ Verificar ownership
class PostPolicy < ApplicationPolicy
  def update?
    record.user == user || user.admin?
  end
end

# ✅ No exponer IDs secuenciales (usar UUIDs o tokens)
class Post < ApplicationRecord
  has_secure_token :public_id

  def to_param
    public_id
  end
end
```

### A02: Cryptographic Failures

```ruby
# ✅ Usar has_secure_password (bcrypt)
class User < ApplicationRecord
  has_secure_password

  # Password requirements
  validates :password, length: { minimum: 12 }, if: :password_required?
  validates :password, format: {
    with: /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
    message: "must include uppercase, lowercase, and number"
  }, if: :password_required?
end

# ✅ Encriptar datos sensibles con Active Record Encryption
class User < ApplicationRecord
  encrypts :ssn
  encrypts :api_key, deterministic: true  # Para búsquedas
end

# ✅ Configurar encryption
# config/credentials.yml.enc
active_record_encryption:
  primary_key: xxx
  deterministic_key: xxx
  key_derivation_salt: xxx
```

### A03: Injection

```ruby
# SQL Injection Prevention

# ❌ VULNERABLE
User.where("email = '#{params[:email]}'")
User.where("role IN (#{params[:roles].join(',')})")

# ✅ SEGURO
User.where(email: params[:email])
User.where("email = ?", params[:email])
User.where(role: params[:roles])

# XSS Prevention - Ya escapado por defecto en ERB
# Solo usar html_safe o raw cuando sea absolutamente necesario

# ✅ Sanitizar HTML permitido
<%= sanitize @post.body, tags: %w[p br strong em a], attributes: %w[href class] %>

# Command Injection Prevention
# ❌ VULNERABLE
system("convert #{params[:filename]} output.png")

# ✅ SEGURO
system("convert", params[:filename], "output.png")
# O usar Shellwords
system("convert #{Shellwords.escape(params[:filename])} output.png")
```

### A04: Insecure Design

```ruby
# ✅ Validar entrada temprano
class User < ApplicationRecord
  validates :email, presence: true,
                    format: { with: URI::MailTo::EMAIL_REGEXP },
                    length: { maximum: 255 }

  validates :name, presence: true,
                   length: { minimum: 2, maximum: 100 }
end

# ✅ Principio de mínimo privilegio
class ApplicationPolicy
  def initialize(user, record)
    @user = user
    @record = record
  end

  # Denegar todo por defecto
  def index? = false
  def show? = false
  def create? = false
  def update? = false
  def destroy? = false
end

# ✅ Defense in depth
class PostsController < ApplicationController
  before_action :authenticate_user!      # Capa 1: Autenticación
  before_action :set_post, only: [:show, :edit, :update, :destroy]
  after_action :verify_authorized        # Capa 2: Autorización

  def update
    authorize @post                      # Capa 3: Verificación explícita
    # ...
  end
end
```

### A05: Security Misconfiguration

```ruby
# config/environments/production.rb
Rails.application.configure do
  # ✅ Forzar SSL
  config.force_ssl = true

  # ✅ No mostrar errores detallados
  config.consider_all_requests_local = false

  # ✅ Logs seguros
  config.log_level = :info

  # ✅ Configurar hosts permitidos
  config.hosts << "myapp.com"
  config.hosts << /.*\.myapp\.com/
end

# config/initializers/content_security_policy.rb
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.default_src :self
    policy.font_src    :self, :data
    policy.img_src     :self, :data, :blob, "https:"
    policy.object_src  :none
    policy.script_src  :self
    policy.style_src   :self, :unsafe_inline
    policy.frame_ancestors :none
    policy.base_uri    :self
    policy.form_action :self
  end

  config.content_security_policy_nonce_generator = ->(request) {
    SecureRandom.base64(16)
  }
end

# config/initializers/permissions_policy.rb
Rails.application.config.permissions_policy do |policy|
  policy.camera      :none
  policy.microphone  :none
  policy.geolocation :none
  policy.usb         :none
end
```

### A06: Vulnerable Components

```ruby
# Gemfile - Mantener actualizadas
ruby "3.3.0"

gem "rails", "~> 8.0"

# Verificar regularmente
# bundle outdated
# bundle update --conservative

# Automatizar con Dependabot
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "bundler"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
```

### A07: Authentication Failures

```ruby
# ✅ Rate limiting para login
# config/initializers/rack_attack.rb
class Rack::Attack
  throttle("logins/ip", limit: 5, period: 60.seconds) do |req|
    req.ip if req.path == "/session" && req.post?
  end

  throttle("logins/email", limit: 5, period: 60.seconds) do |req|
    if req.path == "/session" && req.post?
      req.params.dig("session", "email")&.downcase
    end
  end
end

# ✅ Account lockout
class User < ApplicationRecord
  MAX_LOGIN_ATTEMPTS = 5
  LOCKOUT_DURATION = 15.minutes

  def locked?
    locked_at.present? && locked_at > LOCKOUT_DURATION.ago
  end

  def increment_failed_attempts!
    increment!(:failed_attempts)
    update!(locked_at: Time.current) if failed_attempts >= MAX_LOGIN_ATTEMPTS
  end

  def reset_failed_attempts!
    update!(failed_attempts: 0, locked_at: nil)
  end
end

# ✅ Secure session management
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store,
  key: "_myapp_session",
  secure: Rails.env.production?,
  httponly: true,
  same_site: :lax,
  expire_after: 12.hours
```

### A08: Data Integrity Failures

```ruby
# ✅ Verificar integridad de uploads
class Document < ApplicationRecord
  has_one_attached :file

  validates :file, content_type: {
    in: %w[application/pdf image/png image/jpeg],
    message: "must be a PDF or image"
  }

  validates :file, size: { less_than: 10.megabytes }

  # Verificar checksum
  after_attach :verify_checksum

  private

  def verify_checksum
    return unless file.attached?

    expected = file.checksum
    actual = Digest::MD5.base64digest(file.download)

    raise "Checksum mismatch" unless expected == actual
  end
end

# ✅ Signed URLs para downloads
<%= rails_blob_url(@document.file, disposition: "attachment", expires_in: 1.hour) %>
```

### A09: Logging & Monitoring

```ruby
# ✅ Log eventos de seguridad
class SecurityLogger
  def self.log_event(event_type, details = {})
    Rails.logger.info({
      type: "security_event",
      event: event_type,
      timestamp: Time.current.iso8601,
      ip: Current.request&.remote_ip,
      user_id: Current.user&.id,
      **details
    }.to_json)
  end
end

# Usar en controllers
class SessionsController < ApplicationController
  def create
    if user&.authenticate(params[:password])
      SecurityLogger.log_event("login_success", email: params[:email])
      # ...
    else
      SecurityLogger.log_event("login_failure", email: params[:email])
      # ...
    end
  end
end

# ✅ Filtrar parámetros sensibles
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [
  :password, :password_confirmation,
  :token, :secret, :api_key,
  :credit_card, :cvv, :ssn,
  :authorization
]
```

### A10: Server-Side Request Forgery (SSRF)

```ruby
# ✅ Validar URLs antes de hacer requests
class UrlValidator
  BLOCKED_HOSTS = %w[localhost 127.0.0.1 0.0.0.0 ::1]
  BLOCKED_NETWORKS = [
    IPAddr.new("10.0.0.0/8"),
    IPAddr.new("172.16.0.0/12"),
    IPAddr.new("192.168.0.0/16"),
    IPAddr.new("169.254.0.0/16")
  ]

  def self.safe?(url)
    uri = URI.parse(url)
    return false unless %w[http https].include?(uri.scheme)
    return false if BLOCKED_HOSTS.include?(uri.host)

    ip = IPAddr.new(Resolv.getaddress(uri.host))
    BLOCKED_NETWORKS.none? { |net| net.include?(ip) }
  rescue StandardError
    false
  end
end

# Usar antes de hacer requests externos
def fetch_external_resource(url)
  raise "Unsafe URL" unless UrlValidator.safe?(url)

  HTTP.get(url)
end
```

## Headers de seguridad

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :set_security_headers

  private

  def set_security_headers
    # Prevenir clickjacking
    response.headers["X-Frame-Options"] = "DENY"

    # Prevenir MIME sniffing
    response.headers["X-Content-Type-Options"] = "nosniff"

    # XSS filter (legacy browsers)
    response.headers["X-XSS-Protection"] = "1; mode=block"

    # Referrer policy
    response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"

    # Permissions policy
    response.headers["Permissions-Policy"] = "geolocation=(), microphone=(), camera=()"
  end
end
```

## Manejo de secretos

```bash
# Generar master key
rails credentials:edit

# Estructura recomendada
# config/credentials.yml.enc
secret_key_base: generado_automaticamente

# Base de datos
database:
  password: xxx

# Servicios externos
stripe:
  secret_key: sk_xxx
  publishable_key: pk_xxx
  webhook_secret: whsec_xxx

aws:
  access_key_id: xxx
  secret_access_key: xxx
  bucket: myapp-production

# Email
smtp:
  user_name: xxx
  password: xxx
```

```ruby
# Acceder a credentials
Rails.application.credentials.stripe[:secret_key]

# En config
config.stripe_key = Rails.application.credentials.dig(:stripe, :secret_key)
```

## Testing de seguridad

```ruby
# spec/requests/security_spec.rb
RSpec.describe "Security", type: :request do
  describe "headers" do
    before { get root_path }

    it "sets X-Frame-Options" do
      expect(response.headers["X-Frame-Options"]).to eq("DENY")
    end

    it "sets X-Content-Type-Options" do
      expect(response.headers["X-Content-Type-Options"]).to eq("nosniff")
    end

    it "sets CSP header" do
      expect(response.headers["Content-Security-Policy"]).to be_present
    end
  end

  describe "authentication" do
    it "requires login for protected resources" do
      get dashboard_path
      expect(response).to redirect_to(new_session_path)
    end
  end

  describe "authorization" do
    let(:user) { create(:user) }
    let(:other_user) { create(:user) }
    let(:post) { create(:post, user: other_user) }

    it "prevents unauthorized access" do
      sign_in user
      patch post_path(post), params: { post: { title: "Hacked" } }
      expect(response).to have_http_status(:forbidden)
    end
  end

  describe "rate limiting" do
    it "blocks excessive login attempts" do
      6.times do
        post session_path, params: { email: "test@test.com", password: "wrong" }
      end

      expect(response).to have_http_status(:too_many_requests)
    end
  end
end
```

## Checklist de seguridad

### Antes de deploy

- [ ] Brakeman sin warnings críticos/altos
- [ ] bundler-audit sin vulnerabilidades
- [ ] Secrets en credentials, no en código
- [ ] HTTPS forzado
- [ ] Headers de seguridad configurados
- [ ] CSP implementado
- [ ] Rate limiting activo
- [ ] Logs filtrados (sin passwords/tokens)
- [ ] Backups configurados
- [ ] Monitoreo de errores activo
