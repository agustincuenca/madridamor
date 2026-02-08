# Skill: Bases de Datos NoSQL

## Purpose

Implementar y utilizar bases de datos NoSQL en aplicaciones Rails para casos de uso especificos donde SQL tradicional no es la mejor opcion.

## Tipos de Bases NoSQL

```markdown
| Tipo | Ejemplos | Casos de Uso |
|------|----------|--------------|
| Document | MongoDB, CouchDB | Datos semi-estructurados, CMS, catalogos |
| Key-Value | Redis, Memcached | Cache, sessions, colas |
| Column-Family | Cassandra, HBase | Time-series, analytics a gran escala |
| Graph | Neo4j, Amazon Neptune | Redes sociales, recomendaciones |
| Search | Elasticsearch, Solr | Full-text search, logs |
```

## MongoDB

### Configuracion con Mongoid

```ruby
# Gemfile
gem "mongoid"

# Generar configuracion
# rails g mongoid:config

# config/mongoid.yml
development:
  clients:
    default:
      database: myapp_development
      hosts:
        - localhost:27017
      options:
        server_selection_timeout: 5
        max_pool_size: 5

production:
  clients:
    default:
      uri: <%= ENV['MONGODB_URI'] %>
      options:
        ssl: true
        ssl_verify: true
        max_pool_size: 50
```

### Modelos con Mongoid

```ruby
# app/models/product.rb
class Product
  include Mongoid::Document
  include Mongoid::Timestamps

  # Campos con tipos
  field :name, type: String
  field :sku, type: String
  field :price, type: BigDecimal
  field :stock_quantity, type: Integer, default: 0
  field :active, type: Boolean, default: true
  field :tags, type: Array, default: []
  field :release_date, type: Date

  # Documento embebido (almacenado dentro del producto)
  embeds_many :variants
  embeds_one :dimensions

  # Referencia a otro documento (como foreign key)
  belongs_to :category
  has_many :reviews

  # Indices
  index({ sku: 1 }, { unique: true })
  index({ name: "text", description: "text" })
  index({ category_id: 1, active: 1 })
  index({ tags: 1 })

  # Validaciones
  validates :name, presence: true
  validates :sku, presence: true, uniqueness: true
  validates :price, numericality: { greater_than: 0 }

  # Scopes
  scope :active, -> { where(active: true) }
  scope :in_stock, -> { where(:stock_quantity.gt => 0) }
  scope :by_category, ->(cat_id) { where(category_id: cat_id) }
  scope :with_tag, ->(tag) { where(:tags.in => [tag]) }

  # Callbacks
  before_save :normalize_sku

  private

  def normalize_sku
    self.sku = sku.upcase.strip if sku.present?
  end
end

# app/models/variant.rb (documento embebido)
class Variant
  include Mongoid::Document

  embedded_in :product

  field :name, type: String
  field :sku_suffix, type: String
  field :price_modifier, type: BigDecimal, default: 0
  field :stock_quantity, type: Integer, default: 0
  field :attributes, type: Hash, default: {}

  validates :name, presence: true
end

# app/models/dimensions.rb
class Dimensions
  include Mongoid::Document

  embedded_in :product

  field :width, type: Float
  field :height, type: Float
  field :depth, type: Float
  field :weight, type: Float
  field :unit, type: String, default: "cm"
end
```

### Queries en MongoDB

```ruby
# Busquedas basicas
Product.where(active: true)
Product.where(price: 10..100)
Product.where(:stock_quantity.gt => 0)
Product.where(:tags.in => ["electronics", "gadgets"])

# Busqueda en campos anidados
Product.where("dimensions.weight" => { "$lt" => 5 })
Product.where("variants.name" => "Large")

# Busqueda de texto
Product.text_search("wireless headphones")

# Operadores de comparacion
Product.where(:price.gt => 50)
Product.where(:price.gte => 50)
Product.where(:price.lt => 100)
Product.where(:price.lte => 100)
Product.where(:price.ne => 0)

# Operadores logicos
Product.or({ active: true }, { :stock_quantity.gt => 0 })
Product.and({ active: true }, { :price.lt => 100 })
Product.not.where(active: false)

# Operadores de array
Product.where(:tags.all => ["featured", "sale"])
Product.where(:tags.size => 3)
Product.where(:tags.elem_match => { "$regex" => /^tech/ })

# Proyeccion (seleccionar campos)
Product.only(:name, :price)
Product.without(:description, :metadata)

# Ordenamiento y paginacion
Product.order_by(price: :asc).skip(20).limit(10)
Product.order_by(created_at: :desc).first

# Aggregation
Product.collection.aggregate([
  { "$match" => { active: true } },
  { "$group" => {
    "_id" => "$category_id",
    "count" => { "$sum" => 1 },
    "avg_price" => { "$avg" => "$price" }
  }},
  { "$sort" => { "count" => -1 } }
])
```

### Aggregation Pipeline

```ruby
# app/services/product_analytics.rb
class ProductAnalytics
  def self.sales_by_category(start_date:, end_date:)
    Order.collection.aggregate([
      # Filtrar por fecha
      { "$match" => {
        "created_at" => {
          "$gte" => start_date,
          "$lte" => end_date
        }
      }},

      # Descomponer array de items
      { "$unwind" => "$items" },

      # Lookup para traer producto
      { "$lookup" => {
        "from" => "products",
        "localField" => "items.product_id",
        "foreignField" => "_id",
        "as" => "product"
      }},

      # Descomponer resultado del lookup
      { "$unwind" => "$product" },

      # Agrupar por categoria
      { "$group" => {
        "_id" => "$product.category_id",
        "total_revenue" => {
          "$sum" => { "$multiply" => ["$items.quantity", "$items.price"] }
        },
        "total_units" => { "$sum" => "$items.quantity" },
        "order_count" => { "$sum" => 1 }
      }},

      # Lookup para nombre de categoria
      { "$lookup" => {
        "from" => "categories",
        "localField" => "_id",
        "foreignField" => "_id",
        "as" => "category"
      }},

      { "$unwind" => "$category" },

      # Proyectar resultado final
      { "$project" => {
        "_id" => 0,
        "category_name" => "$category.name",
        "total_revenue" => 1,
        "total_units" => 1,
        "order_count" => 1,
        "avg_order_value" => { "$divide" => ["$total_revenue", "$order_count"] }
      }},

      # Ordenar
      { "$sort" => { "total_revenue" => -1 } }
    ]).to_a
  end
end
```

## Redis

### Configuracion

```ruby
# Gemfile
gem "redis"
gem "hiredis"  # Driver nativo mas rapido
gem "connection_pool"

# config/initializers/redis.rb
require "connection_pool"

REDIS_POOL = ConnectionPool.new(size: 10, timeout: 5) do
  Redis.new(
    url: ENV.fetch("REDIS_URL") { "redis://localhost:6379/0" },
    driver: :hiredis
  )
end

# Helper para acceso
def redis
  REDIS_POOL.with { |conn| yield conn }
end

# O crear cliente global
REDIS = Redis.new(url: ENV.fetch("REDIS_URL") { "redis://localhost:6379/0" })
```

### Strings

```ruby
# Set/Get basico
redis { |r| r.set("user:1:name", "John") }
redis { |r| r.get("user:1:name") }  # => "John"

# Con expiracion
redis { |r| r.setex("session:abc123", 3600, user_data.to_json) }  # Expira en 1 hora

# Solo si no existe
redis { |r| r.setnx("lock:order:123", "processing") }

# Incrementar/Decrementar
redis { |r| r.incr("visits:page:home") }
redis { |r| r.incrby("user:1:points", 10) }
redis { |r| r.decr("inventory:product:456") }

# Multiples operaciones
redis { |r|
  r.mset("key1", "value1", "key2", "value2")
  r.mget("key1", "key2")  # => ["value1", "value2"]
}
```

### Hashes

```ruby
# Almacenar objeto como hash
redis { |r|
  r.hset("user:1", {
    "name" => "John",
    "email" => "john@example.com",
    "age" => 30
  })
}

# Obtener campos
redis { |r| r.hget("user:1", "name") }  # => "John"
redis { |r| r.hgetall("user:1") }  # => {"name" => "John", ...}
redis { |r| r.hmget("user:1", "name", "email") }

# Incrementar campo numerico
redis { |r| r.hincrby("user:1", "login_count", 1) }

# Verificar existencia
redis { |r| r.hexists("user:1", "email") }  # => true

# Patron para cache de objetos
class UserCache
  def self.get(user_id)
    data = REDIS.hgetall("user:#{user_id}")
    return nil if data.empty?

    User.new(data.symbolize_keys)
  end

  def self.set(user)
    REDIS.hset("user:#{user.id}", user.cache_attributes)
    REDIS.expire("user:#{user.id}", 1.hour.to_i)
  end

  def self.invalidate(user_id)
    REDIS.del("user:#{user_id}")
  end
end
```

### Lists

```ruby
# Cola FIFO
redis { |r| r.lpush("queue:emails", email_data.to_json) }  # Agregar al inicio
redis { |r| r.rpop("queue:emails") }  # Sacar del final

# Cola LIFO (stack)
redis { |r| r.lpush("stack:undo", action.to_json) }
redis { |r| r.lpop("stack:undo") }

# Obtener rango
redis { |r| r.lrange("recent:posts", 0, 9) }  # Primeros 10

# Bloquear hasta que haya elementos (para workers)
redis { |r| r.brpop("queue:jobs", timeout: 30) }

# Limitar tamano de lista
redis { |r|
  r.lpush("user:1:notifications", notification.to_json)
  r.ltrim("user:1:notifications", 0, 99)  # Mantener solo 100
}

# Timeline/Feed
class UserFeed
  def self.add_post(user_id, post_id)
    followers = User.find(user_id).follower_ids

    REDIS.pipelined do |pipe|
      followers.each do |follower_id|
        pipe.lpush("feed:#{follower_id}", post_id)
        pipe.ltrim("feed:#{follower_id}", 0, 999)
      end
    end
  end

  def self.get_feed(user_id, page: 1, per_page: 20)
    start = (page - 1) * per_page
    stop = start + per_page - 1

    post_ids = REDIS.lrange("feed:#{user_id}", start, stop)
    Post.where(id: post_ids).order_by_ids(post_ids)
  end
end
```

### Sets

```ruby
# Agregar a set
redis { |r| r.sadd("tags:product:1", ["electronics", "gadgets", "tech"]) }

# Verificar pertenencia
redis { |r| r.sismember("tags:product:1", "electronics") }  # => true

# Obtener todos los miembros
redis { |r| r.smembers("tags:product:1") }

# Operaciones de conjuntos
redis { |r| r.sinter("tags:product:1", "tags:product:2") }  # Interseccion
redis { |r| r.sunion("tags:product:1", "tags:product:2") }  # Union
redis { |r| r.sdiff("tags:product:1", "tags:product:2") }   # Diferencia

# Contar miembros
redis { |r| r.scard("online:users") }

# Usuarios online
class OnlineTracker
  def self.mark_online(user_id)
    REDIS.sadd("online:users", user_id)
    REDIS.expire("online:users", 5.minutes.to_i)
  end

  def self.online_count
    REDIS.scard("online:users")
  end

  def self.is_online?(user_id)
    REDIS.sismember("online:users", user_id)
  end

  def self.online_friends(user_id)
    friend_ids = User.find(user_id).friend_ids
    REDIS.sinter("online:users", friend_ids)
  end
end
```

### Sorted Sets

```ruby
# Agregar con score
redis { |r| r.zadd("leaderboard", 1000, "user:1") }
redis { |r| r.zadd("leaderboard", 850, "user:2") }

# Incrementar score
redis { |r| r.zincrby("leaderboard", 50, "user:1") }

# Ranking (mayor a menor)
redis { |r| r.zrevrange("leaderboard", 0, 9, with_scores: true) }

# Obtener rank de usuario
redis { |r| r.zrevrank("leaderboard", "user:1") }  # 0-based

# Rango por score
redis { |r| r.zrangebyscore("leaderboard", 500, 1000) }

# Leaderboard completo
class Leaderboard
  def initialize(name)
    @name = "leaderboard:#{name}"
  end

  def add_score(user_id, score)
    REDIS.zincrby(@name, score, user_id)
  end

  def top(count = 10)
    REDIS.zrevrange(@name, 0, count - 1, with_scores: true).map do |user_id, score|
      { user_id: user_id, score: score.to_i }
    end
  end

  def rank_for(user_id)
    rank = REDIS.zrevrank(@name, user_id)
    rank ? rank + 1 : nil
  end

  def score_for(user_id)
    REDIS.zscore(@name, user_id)&.to_i
  end

  def around_user(user_id, range: 5)
    rank = REDIS.zrevrank(@name, user_id)
    return [] unless rank

    start = [0, rank - range].max
    stop = rank + range

    REDIS.zrevrange(@name, start, stop, with_scores: true).map.with_index do |(uid, score), idx|
      { rank: start + idx + 1, user_id: uid, score: score.to_i }
    end
  end
end
```

### Pub/Sub

```ruby
# Publicador
class EventPublisher
  def self.publish(channel, event)
    REDIS.publish("events:#{channel}", event.to_json)
  end
end

# Suscriptor (en proceso separado)
class EventSubscriber
  def self.subscribe(*channels)
    redis = Redis.new(url: ENV["REDIS_URL"])

    redis.subscribe(*channels.map { |c| "events:#{c}" }) do |on|
      on.subscribe do |channel, subscriptions|
        puts "Subscribed to #{channel} (#{subscriptions} subscriptions)"
      end

      on.message do |channel, message|
        event = JSON.parse(message)
        handle_event(channel, event)
      end
    end
  end

  def self.handle_event(channel, event)
    case channel
    when "events:orders"
      OrderEventHandler.process(event)
    when "events:users"
      UserEventHandler.process(event)
    end
  end
end

# Real-time notifications con Action Cable
class NotificationChannel < ApplicationCable::Channel
  def subscribed
    # Suscribirse a Redis channel
    redis_subscribe("notifications:#{current_user.id}")
  end

  private

  def redis_subscribe(channel)
    Thread.new do
      Redis.new.subscribe(channel) do |on|
        on.message do |_channel, message|
          transmit(JSON.parse(message))
        end
      end
    end
  end
end
```

## Neo4j

### Configuracion

```ruby
# Gemfile
gem "neo4j"
gem "neo4j-core"

# config/neo4j.yml
development:
  type: bolt
  url: bolt://localhost:7687
  username: neo4j
  password: password

production:
  type: bolt
  url: <%= ENV['NEO4J_URL'] %>
  username: <%= ENV['NEO4J_USERNAME'] %>
  password: <%= ENV['NEO4J_PASSWORD'] %>
```

### Modelos

```ruby
# app/models/person.rb
class Person
  include Neo4j::ActiveNode

  property :name, type: String
  property :email, type: String
  property :born, type: Integer

  has_many :out, :friends, rel_class: :Friendship, model_class: :Person
  has_many :in, :followers, rel_class: :Follows, model_class: :Person
  has_many :out, :following, rel_class: :Follows, model_class: :Person
  has_many :out, :posts, type: :AUTHORED
  has_many :out, :liked_posts, rel_class: :Likes, model_class: :Post

  validates :name, presence: true
  validates :email, presence: true, uniqueness: true
end

# app/models/post.rb
class Post
  include Neo4j::ActiveNode

  property :title, type: String
  property :body, type: String
  property :created_at, type: DateTime

  has_one :in, :author, type: :AUTHORED, model_class: :Person
  has_many :in, :likers, rel_class: :Likes, model_class: :Person
  has_many :out, :tags, type: :TAGGED_WITH
end

# app/models/friendship.rb (relacion con propiedades)
class Friendship
  include Neo4j::ActiveRel

  from_class :Person
  to_class :Person
  type :FRIENDS_WITH

  property :since, type: Date
  property :closeness, type: Integer  # 1-10

  validates :since, presence: true
end
```

### Cypher Queries

```ruby
# Encontrar amigos de amigos
def friends_of_friends(person)
  Neo4j::ActiveBase.new_query
    .match("(p:Person)-[:FRIENDS_WITH]->(friend)-[:FRIENDS_WITH]->(fof:Person)")
    .where(p: { uuid: person.uuid })
    .where_not("(p)-[:FRIENDS_WITH]->(fof)")
    .where_not("p = fof")
    .return("DISTINCT fof")
    .to_a
end

# Camino mas corto entre dos personas
def shortest_path(person1, person2)
  Neo4j::ActiveBase.new_query
    .match("path = shortestPath((p1:Person)-[:FRIENDS_WITH*]-(p2:Person))")
    .where(p1: { uuid: person1.uuid })
    .where(p2: { uuid: person2.uuid })
    .return("path, length(path) as distance")
    .first
end

# Recomendaciones basadas en grafo
def recommend_friends(person, limit: 10)
  Neo4j::ActiveBase.new_query
    .match("(p:Person)-[:FRIENDS_WITH]->(friend)-[:FRIENDS_WITH]->(recommendation:Person)")
    .where(p: { uuid: person.uuid })
    .where_not("(p)-[:FRIENDS_WITH]->(recommendation)")
    .where_not("p = recommendation")
    .return("recommendation, COUNT(friend) as mutual_friends")
    .order("mutual_friends DESC")
    .limit(limit)
    .to_a
end

# Posts que podrian gustarle al usuario
def recommend_posts(person, limit: 20)
  Neo4j::ActiveBase.new_query
    .match("(p:Person)-[:FRIENDS_WITH]->(friend)-[:LIKES]->(post:Post)")
    .where(p: { uuid: person.uuid })
    .where_not("(p)-[:LIKES]->(post)")
    .with("post, COUNT(friend) as friend_likes")
    .order("friend_likes DESC")
    .limit(limit)
    .return("post, friend_likes")
    .to_a
end

# Influencers (personas con mas conexiones)
def top_influencers(limit: 10)
  Neo4j::ActiveBase.new_query
    .match("(p:Person)<-[:FOLLOWS]-(follower:Person)")
    .with("p, COUNT(follower) as followers")
    .order("followers DESC")
    .limit(limit)
    .return("p, followers")
    .to_a
end
```

## Elasticsearch

### Configuracion

```ruby
# Gemfile
gem "elasticsearch-model"
gem "elasticsearch-rails"

# config/initializers/elasticsearch.rb
Elasticsearch::Model.client = Elasticsearch::Client.new(
  url: ENV.fetch("ELASTICSEARCH_URL") { "http://localhost:9200" },
  log: Rails.env.development?
)
```

### Modelo con Elasticsearch

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  include Elasticsearch::Model
  include Elasticsearch::Model::Callbacks  # Sync automatico

  # Configurar indice
  settings index: {
    number_of_shards: 1,
    analysis: {
      analyzer: {
        spanish_analyzer: {
          type: "spanish",
          stopwords: "_spanish_"
        }
      }
    }
  } do
    mappings dynamic: "false" do
      indexes :title, type: "text", analyzer: "spanish_analyzer", boost: 2
      indexes :body, type: "text", analyzer: "spanish_analyzer"
      indexes :tags, type: "keyword"
      indexes :author_name, type: "text"
      indexes :category, type: "keyword"
      indexes :published_at, type: "date"
      indexes :views_count, type: "integer"
      indexes :location, type: "geo_point"
    end
  end

  # Definir que datos indexar
  def as_indexed_json(options = {})
    {
      title: title,
      body: body,
      tags: tags,
      author_name: author.name,
      category: category.name,
      published_at: published_at,
      views_count: views_count,
      location: { lat: latitude, lon: longitude }
    }
  end

  # Metodos de busqueda
  def self.search_query(query, filters = {})
    search_definition = {
      query: {
        bool: {
          must: [
            {
              multi_match: {
                query: query,
                fields: ["title^2", "body", "author_name"],
                type: "best_fields",
                fuzziness: "AUTO"
              }
            }
          ],
          filter: build_filters(filters)
        }
      },
      highlight: {
        fields: {
          title: {},
          body: { fragment_size: 150 }
        }
      },
      aggs: {
        categories: { terms: { field: "category" } },
        tags: { terms: { field: "tags", size: 20 } }
      }
    }

    __elasticsearch__.search(search_definition)
  end

  private

  def self.build_filters(filters)
    result = []

    if filters[:category].present?
      result << { term: { category: filters[:category] } }
    end

    if filters[:tags].present?
      result << { terms: { tags: Array(filters[:tags]) } }
    end

    if filters[:date_range].present?
      result << {
        range: {
          published_at: {
            gte: filters[:date_range][:from],
            lte: filters[:date_range][:to]
          }
        }
      }
    end

    result
  end
end
```

### Queries avanzadas

```ruby
# app/services/article_search.rb
class ArticleSearch
  def initialize(params)
    @query = params[:q]
    @page = params[:page] || 1
    @per_page = params[:per_page] || 20
    @filters = params.slice(:category, :tags, :author)
    @sort = params[:sort]
  end

  def call
    Article.__elasticsearch__.search(search_body)
  end

  private

  def search_body
    {
      query: build_query,
      from: (@page - 1) * @per_page,
      size: @per_page,
      sort: build_sort,
      aggs: aggregations,
      highlight: highlights,
      suggest: suggestions
    }
  end

  def build_query
    {
      bool: {
        must: must_clauses,
        filter: filter_clauses,
        should: boost_clauses
      }
    }
  end

  def must_clauses
    return [{ match_all: {} }] if @query.blank?

    [
      {
        multi_match: {
          query: @query,
          fields: ["title^3", "body", "tags^2"],
          type: "cross_fields",
          operator: "and"
        }
      }
    ]
  end

  def filter_clauses
    filters = []

    filters << { term: { category: @filters[:category] } } if @filters[:category]
    filters << { terms: { tags: @filters[:tags] } } if @filters[:tags]
    filters << { term: { "author_name.keyword": @filters[:author] } } if @filters[:author]

    filters
  end

  def boost_clauses
    [
      { range: { published_at: { gte: "now-7d", boost: 2 } } },  # Articulos recientes
      { range: { views_count: { gte: 1000, boost: 1.5 } } }     # Populares
    ]
  end

  def build_sort
    case @sort
    when "recent"
      [{ published_at: "desc" }]
    when "popular"
      [{ views_count: "desc" }]
    else
      ["_score", { published_at: "desc" }]
    end
  end

  def aggregations
    {
      categories: {
        terms: { field: "category", size: 10 }
      },
      popular_tags: {
        terms: { field: "tags", size: 20 }
      },
      date_histogram: {
        date_histogram: {
          field: "published_at",
          calendar_interval: "month"
        }
      }
    }
  end

  def highlights
    {
      pre_tags: ["<mark>"],
      post_tags: ["</mark>"],
      fields: {
        title: { number_of_fragments: 0 },
        body: { fragment_size: 200, number_of_fragments: 3 }
      }
    }
  end

  def suggestions
    return {} if @query.blank?

    {
      title_suggestion: {
        text: @query,
        term: { field: "title" }
      }
    }
  end
end
```

## DynamoDB

### Configuracion

```ruby
# Gemfile
gem "aws-sdk-dynamodb"
gem "dynamoid"  # ORM para DynamoDB

# config/initializers/dynamodb.rb
Dynamoid.configure do |config|
  config.namespace = "myapp_#{Rails.env}"
  config.region = ENV.fetch("AWS_REGION") { "us-east-1" }
  config.access_key = ENV["AWS_ACCESS_KEY_ID"]
  config.secret_key = ENV["AWS_SECRET_ACCESS_KEY"]
end
```

### Modelos con Dynamoid

```ruby
# app/models/session.rb
class Session
  include Dynamoid::Document

  table name: :sessions, key: :session_id

  field :session_id, :string
  field :user_id, :string
  field :data, :serialized
  field :expires_at, :datetime

  global_secondary_index hash_key: :user_id, projected_attributes: :all

  # TTL para auto-expiracion
  field :ttl, :integer

  before_create :set_ttl

  private

  def set_ttl
    self.ttl = (expires_at || 1.day.from_now).to_i
  end
end

# app/models/event.rb
class Event
  include Dynamoid::Document

  table name: :events, key: :event_id, range_key: :timestamp

  field :event_id, :string
  field :timestamp, :datetime
  field :event_type, :string
  field :user_id, :string
  field :data, :map
  field :metadata, :map

  global_secondary_index hash_key: :user_id,
                         range_key: :timestamp,
                         projected_attributes: :all

  global_secondary_index hash_key: :event_type,
                         range_key: :timestamp,
                         projected_attributes: :keys_only
end
```

### Queries en DynamoDB

```ruby
# Buscar por primary key
session = Session.find("session-123")

# Query por GSI
events = Event.where(user_id: "user-123")
              .where("timestamp.gt": 1.day.ago)

# Scan con filtro (costoso, evitar en produccion)
sessions = Session.scan_filter(
  :user_id => { eq: "user-123" },
  :expires_at => { gt: Time.current }
)

# Batch operations
Session.import([session1, session2, session3])
Session.find_all(["id1", "id2", "id3"])

# Operaciones condicionales
session.update_attributes(
  { data: new_data },
  conditions: { if: { version: current_version } }
)
```

## Cuando Usar NoSQL vs SQL

```markdown
## Usar SQL cuando:
- Necesitas transacciones ACID complejas
- Datos altamente relacionados
- Queries ad-hoc complejos
- Reporting y analytics
- Schema estricto es importante

## Usar MongoDB cuando:
- Schema flexible/evolutivo
- Documentos anidados complejos
- Datos semi-estructurados
- Prototipado rapido
- Catalogos de productos

## Usar Redis cuando:
- Cache de alto rendimiento
- Sessions
- Colas y pub/sub
- Leaderboards
- Rate limiting
- Datos efimeros

## Usar Elasticsearch cuando:
- Full-text search
- Log analytics
- Busqueda facetada
- Autocompletado
- Geo-search

## Usar Neo4j cuando:
- Redes sociales
- Sistemas de recomendacion
- Deteccion de fraude
- Knowledge graphs
- Path finding

## Usar DynamoDB cuando:
- Escala masiva (millones de requests/segundo)
- Latencia consistente
- Serverless
- Key-value simple con indices
- Gaming/IoT
```

## Patrones Hibridos

```ruby
# Usar SQL como fuente de verdad + Redis para cache
class Product < ApplicationRecord
  after_commit :invalidate_cache

  def self.featured
    Rails.cache.fetch("products:featured", expires_in: 1.hour) do
      where(featured: true).limit(10).to_a
    end
  end

  private

  def invalidate_cache
    Rails.cache.delete("products:featured")
  end
end

# Usar SQL + Elasticsearch para busqueda
class Product < ApplicationRecord
  include Elasticsearch::Model

  # SQL para transacciones y relaciones
  # Elasticsearch para busqueda

  def self.search(query)
    if query.present?
      __elasticsearch__.search(query).records
    else
      all
    end
  end
end

# Usar SQL + MongoDB para diferentes partes
class Order < ApplicationRecord
  # Datos transaccionales en PostgreSQL
  has_many :line_items

  # Logs/eventos en MongoDB
  def log_event(event_type, data = {})
    OrderEvent.create!(
      order_id: id.to_s,
      event_type: event_type,
      data: data,
      timestamp: Time.current
    )
  end
end
```

## Checklist

- [ ] Evaluar si NoSQL es apropiado para el caso de uso
- [ ] Considerar patron hibrido SQL + NoSQL
- [ ] Configurar conexiones con pooling
- [ ] Implementar manejo de errores y reintentos
- [ ] Configurar indices apropiados
- [ ] Establecer TTLs para datos temporales
- [ ] Monitorear rendimiento y uso de memoria
- [ ] Planificar estrategia de backup
- [ ] Documentar decisiones de modelado
