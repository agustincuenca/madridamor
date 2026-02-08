# Skill: Caching

## Purpose

Implementar estrategias de caché en Rails para mejorar el rendimiento de aplicaciones usando Solid Cache (Rails 8) y otras técnicas.

## Setup

### Solid Cache (Rails 8)

```ruby
# Gemfile (ya incluido en Rails 8)
gem "solid_cache"
```

```bash
# Generar migraciones
bin/rails solid_cache:install

# Ejecutar migraciones
bin/rails db:migrate
```

```ruby
# config/environments/production.rb
config.cache_store = :solid_cache_store

# config/environments/development.rb
# Habilitar caché en desarrollo (toggle)
# rails dev:cache
if Rails.root.join("tmp/caching-dev.txt").exist?
  config.cache_store = :solid_cache_store
  config.public_file_server.headers = {
    "Cache-Control" => "public, max-age=#{2.days.to_i}"
  }
else
  config.cache_store = :null_store
end
```

### Redis Cache (alternativa)

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url: ENV.fetch("REDIS_URL") { "redis://localhost:6379/0" },
  expires_in: 1.day,
  namespace: "cache"
}
```

## Fragment Caching

### Cache básico

```erb
<%# Caché de un bloque %>
<% cache @post do %>
  <article class="post">
    <h2><%= @post.title %></h2>
    <div><%= @post.body %></div>
    <p>By <%= @post.author.name %></p>
  </article>
<% end %>
```

### Cache con dependencias

```erb
<%# La caché se invalida cuando cambia el post O sus comentarios %>
<% cache [@post, @post.comments.maximum(:updated_at)] do %>
  <article>
    <h2><%= @post.title %></h2>
    <%= render @post.comments %>
  </article>
<% end %>
```

### Cache de colecciones

```erb
<%# Caché individual por item %>
<%= render partial: "posts/post", collection: @posts, cached: true %>

<%# El partial necesita estar preparado %>
<%# app/views/posts/_post.html.erb %>
<% cache post do %>
  <article id="<%= dom_id(post) %>">
    <%= post.title %>
  </article>
<% end %>
```

### Cache con versión

```erb
<%# Útil para invalidar manualmente %>
<% cache ["v2", @post] do %>
  <%= render @post %>
<% end %>
```

## Russian Doll Caching

```erb
<%# Cache anidada - se invalida de adentro hacia afuera %>
<% cache @post do %>
  <article>
    <h2><%= @post.title %></h2>

    <% cache [@post, "comments"] do %>
      <div class="comments">
        <% @post.comments.each do |comment| %>
          <% cache comment do %>
            <%= render comment %>
          <% end %>
        <% end %>
      </div>
    <% end %>
  </article>
<% end %>
```

## Low-Level Caching

```ruby
# Caché manual de valores
class Post < ApplicationRecord
  def comment_count
    Rails.cache.fetch("#{cache_key_with_version}/comment_count") do
      comments.count
    end
  end

  def expensive_calculation
    Rails.cache.fetch("#{cache_key_with_version}/calculation", expires_in: 1.hour) do
      # Cálculo costoso...
      heavy_computation
    end
  end
end

# Con bloque y opciones
def self.popular_posts
  Rails.cache.fetch("posts/popular", expires_in: 30.minutes) do
    where("views_count > ?", 1000)
      .order(views_count: :desc)
      .limit(10)
      .to_a # Importante: materializar la query
  end
end

# Forzar recalcular
Rails.cache.fetch("key", force: true) do
  # Siempre ejecuta el bloque
end

# Escribir y leer directamente
Rails.cache.write("key", value, expires_in: 1.hour)
Rails.cache.read("key")
Rails.cache.delete("key")
Rails.cache.exist?("key")

# Incrementar/decrementar
Rails.cache.increment("visits/#{post.id}")
Rails.cache.decrement("stock/#{product.id}")

# Multi-get
Rails.cache.read_multi("key1", "key2", "key3")
Rails.cache.write_multi({ "key1" => "value1", "key2" => "value2" })
```

## Query Caching

```ruby
# Rails cachea queries idénticas dentro de una request
# Esto es automático, pero puedes forzarlo:

ActiveRecord::Base.cache do
  # Todas las queries aquí se cachean
  User.find(1)
  User.find(1) # No hace query, usa cache
end

# Para queries específicas
def expensive_query
  Rails.cache.fetch("expensive_query", expires_in: 5.minutes) do
    User.includes(:posts, :comments)
        .where(active: true)
        .order(created_at: :desc)
        .to_a # Materializar para cachear
  end
end
```

## Counter Caching

```ruby
# db/migrate/xxx_add_comments_count_to_posts.rb
class AddCommentsCountToPosts < ActiveRecord::Migration[8.0]
  def change
    add_column :posts, :comments_count, :integer, default: 0, null: false

    # Actualizar valores existentes
    reversible do |dir|
      dir.up do
        Post.find_each do |post|
          Post.reset_counters(post.id, :comments)
        end
      end
    end
  end
end

# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end

# Ahora post.comments.count usa el counter
# En lugar de COUNT(*) en la BD
```

## HTTP Caching

### ETag y Last-Modified

```ruby
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])

    # Usar ETag
    if stale?(@post)
      respond_to do |format|
        format.html
        format.json { render json: @post }
      end
    end
  end

  def index
    @posts = Post.published.includes(:author)

    # Con última modificación
    if stale?(last_modified: @posts.maximum(:updated_at))
      respond_to do |format|
        format.html
        format.json { render json: @posts }
      end
    end
  end
end
```

### Cache-Control Headers

```ruby
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])

    # Cache público (CDN puede cachear)
    expires_in 1.hour, public: true

    # Cache privado (solo browser)
    expires_in 30.minutes, public: false

    # No cachear
    expires_now
  end
end
```

## Memoization

```ruby
class User < ApplicationRecord
  # Memoization simple
  def full_name
    @full_name ||= "#{first_name} #{last_name}"
  end

  # Con argumentos (usar hash)
  def posts_in_year(year)
    @posts_by_year ||= {}
    @posts_by_year[year] ||= posts.where("YEAR(created_at) = ?", year)
  end

  # Resetear memoization
  def reload
    @full_name = nil
    @posts_by_year = nil
    super
  end
end

# Para memoization más compleja
class ExpensiveService
  include ActiveSupport::Memoizable

  def expensive_call
    # ...cálculo costoso
  end
  memoize :expensive_call
end
```

## Cache Invalidation

### Touch para propagar cambios

```ruby
class Comment < ApplicationRecord
  belongs_to :post, touch: true
  # Cuando se actualiza un comentario, post.updated_at también se actualiza
end

class Post < ApplicationRecord
  belongs_to :author, class_name: "User", touch: true
  has_many :comments
end
```

### Invalidación manual

```ruby
# Borrar una clave
Rails.cache.delete("posts/popular")

# Borrar por patrón (solo Redis)
Rails.cache.delete_matched("posts/*")

# Versionar para invalidar
class Post < ApplicationRecord
  after_commit :bump_cache_version

  def cache_version
    Rails.cache.read("posts/version") || 1
  end

  private

  def bump_cache_version
    Rails.cache.increment("posts/version")
  end
end
```

### Callbacks de cache

```ruby
class Post < ApplicationRecord
  after_commit :clear_cache

  private

  def clear_cache
    Rails.cache.delete("posts/popular")
    Rails.cache.delete("posts/recent")
    Rails.cache.delete("homepage/stats")
  end
end
```

## Caché de vistas parciales con conditional GET

```ruby
class PostsController < ApplicationController
  def index
    @posts = Post.published.recent

    fresh_when(
      etag: @posts,
      last_modified: @posts.maximum(:updated_at),
      public: true
    )
  end
end
```

## Action Caching (páginas completas)

```ruby
# Para páginas que no varían por usuario
class PagesController < ApplicationController
  caches_action :about, expires_in: 1.day

  def about
    # Esta acción se cachea completa
  end
end

# Con condiciones
class PostsController < ApplicationController
  caches_action :show,
    expires_in: 30.minutes,
    if: -> { !user_signed_in? }  # Solo para visitantes

  def show
    @post = Post.find(params[:id])
  end
end
```

## Monitoring y debugging

```ruby
# config/environments/development.rb
# Ver información de cache en logs
config.action_controller.enable_fragment_cache_logging = true

# En console
Rails.cache.stats  # Solo Redis

# Ver hits/misses
ActiveSupport::Notifications.subscribe("cache_read.active_support") do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  Rails.logger.debug "Cache #{event.payload[:hit] ? 'HIT' : 'MISS'}: #{event.payload[:key]}"
end
```

## Estrategias por tipo de contenido

### Contenido estático (rara vez cambia)

```ruby
# Cache largo, versionar manualmente
Rails.cache.fetch("site/footer/v1", expires_in: 1.week) do
  render_to_string(partial: "shared/footer")
end
```

### Contenido de usuario (cambia frecuentemente)

```erb
<%# Cache corto, con key específico del usuario %>
<% cache [current_user, "dashboard", Date.current] do %>
  <%= render "dashboard/stats" %>
<% end %>
```

### Datos agregados (conteos, estadísticas)

```ruby
# Precalcular y cachear
class StatsJob < ApplicationJob
  def perform
    stats = {
      total_users: User.count,
      total_posts: Post.count,
      posts_today: Post.where("created_at > ?", Date.current.beginning_of_day).count
    }

    Rails.cache.write("site/stats", stats, expires_in: 5.minutes)
  end
end

# Ejecutar periódicamente con Solid Queue
# O en un scheduler
```

## Checklist

- [ ] Solid Cache o Redis configurado
- [ ] Fragment caching en vistas pesadas
- [ ] Russian doll caching para anidados
- [ ] Counter cache para conteos frecuentes
- [ ] Touch en belongs_to para invalidación
- [ ] HTTP caching con ETags
- [ ] Memoization en modelos
- [ ] Monitoring de hit/miss rate
- [ ] Cache warming en deploy (opcional)
