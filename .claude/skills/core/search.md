# Skill: Search

## Purpose

Implementar funcionalidades de búsqueda en aplicaciones Rails, desde búsqueda simple hasta full-text search.

## Búsqueda simple con SQLite

### LIKE queries

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  scope :search, ->(query) {
    return all if query.blank?

    where("title LIKE ? OR body LIKE ?", "%#{query}%", "%#{query}%")
  }
end

# Uso
Post.search(params[:q])
```

### Búsqueda case-insensitive

```ruby
# SQLite
scope :search, ->(query) {
  return all if query.blank?

  where("LOWER(title) LIKE LOWER(?) OR LOWER(body) LIKE LOWER(?)",
        "%#{query}%", "%#{query}%")
}

# O con COLLATE NOCASE
scope :search, ->(query) {
  return all if query.blank?

  where("title LIKE ? COLLATE NOCASE OR body LIKE ? COLLATE NOCASE",
        "%#{query}%", "%#{query}%")
}
```

### Búsqueda en múltiples campos

```ruby
class Post < ApplicationRecord
  include Searchable

  searchable_fields :title, :body, :author_name

  def author_name
    author&.name
  end
end

# app/models/concerns/searchable.rb
module Searchable
  extend ActiveSupport::Concern

  class_methods do
    def searchable_fields(*fields)
      @searchable_fields = fields
    end

    def search(query)
      return all if query.blank?

      fields = @searchable_fields || [:name]
      conditions = fields.map { |f| "LOWER(#{f}) LIKE LOWER(?)" }.join(" OR ")
      values = fields.map { "%#{query}%" }

      where(conditions, *values)
    end
  end
end
```

## SQLite FTS5 (Full-Text Search)

### Setup

```ruby
# db/migrate/xxx_create_posts_fts.rb
class CreatePostsFts < ActiveRecord::Migration[8.0]
  def up
    execute <<-SQL
      CREATE VIRTUAL TABLE posts_fts USING fts5(
        title,
        body,
        content='posts',
        content_rowid='id'
      );

      -- Triggers para mantener sincronizado
      CREATE TRIGGER posts_ai AFTER INSERT ON posts BEGIN
        INSERT INTO posts_fts(rowid, title, body)
        VALUES (new.id, new.title, new.body);
      END;

      CREATE TRIGGER posts_ad AFTER DELETE ON posts BEGIN
        INSERT INTO posts_fts(posts_fts, rowid, title, body)
        VALUES ('delete', old.id, old.title, old.body);
      END;

      CREATE TRIGGER posts_au AFTER UPDATE ON posts BEGIN
        INSERT INTO posts_fts(posts_fts, rowid, title, body)
        VALUES ('delete', old.id, old.title, old.body);
        INSERT INTO posts_fts(rowid, title, body)
        VALUES (new.id, new.title, new.body);
      END;
    SQL

    # Poblar FTS con datos existentes
    execute <<-SQL
      INSERT INTO posts_fts(rowid, title, body)
      SELECT id, title, body FROM posts;
    SQL
  end

  def down
    execute "DROP TABLE IF EXISTS posts_fts"
  end
end
```

### Modelo con FTS

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  scope :full_text_search, ->(query) {
    return all if query.blank?

    # Sanitizar query
    sanitized = query.gsub(/[^\w\s]/, "")

    joins("INNER JOIN posts_fts ON posts_fts.rowid = posts.id")
      .where("posts_fts MATCH ?", sanitized)
      .select("posts.*, rank AS search_rank")
      .order("search_rank")
  }

  # Con highlighting
  scope :search_with_highlights, ->(query) {
    return all if query.blank?

    sanitized = query.gsub(/[^\w\s]/, "")

    joins("INNER JOIN posts_fts ON posts_fts.rowid = posts.id")
      .where("posts_fts MATCH ?", sanitized)
      .select(
        "posts.*",
        "highlight(posts_fts, 0, '<mark>', '</mark>') AS title_highlight",
        "snippet(posts_fts, 1, '<mark>', '</mark>', '...', 32) AS body_snippet"
      )
      .order("rank")
  }
end
```

## Ransack (búsqueda avanzada)

### Setup

```ruby
# Gemfile
gem "ransack"
```

### Controller

```ruby
class PostsController < ApplicationController
  def index
    @q = Post.ransack(params[:q])
    @posts = @q.result(distinct: true)
               .includes(:author)
               .page(params[:page])
  end
end
```

### Vista con formulario de búsqueda

```erb
<%# app/views/posts/index.html.erb %>
<%= search_form_for @q, url: posts_path, method: :get do |f| %>
  <div class="flex gap-4">
    <%# Búsqueda de texto %>
    <%= f.search_field :title_or_body_cont,
                       placeholder: t(".search_placeholder"),
                       class: "input" %>

    <%# Filtro por estado %>
    <%= f.select :status_eq,
                 Post.statuses.keys.map { |s| [s.humanize, s] },
                 { include_blank: t(".all_statuses") },
                 class: "select" %>

    <%# Rango de fechas %>
    <%= f.date_field :created_at_gteq, class: "input" %>
    <%= f.date_field :created_at_lteq, class: "input" %>

    <%# Ordenamiento %>
    <%= f.select :s,
                 [
                   [t(".newest"), "created_at desc"],
                   [t(".oldest"), "created_at asc"],
                   [t(".title_az"), "title asc"]
                 ],
                 {},
                 class: "select" %>

    <%= f.submit t(".search"), class: "btn btn-primary" %>
  </div>
<% end %>
```

### Predicados de Ransack

```ruby
# Predicados disponibles
title_eq        # título igual a
title_cont      # título contiene
title_start     # título empieza con
title_end       # título termina con
title_present   # título está presente
title_blank     # título está vacío
created_at_gt   # fecha mayor que
created_at_lt   # fecha menor que
created_at_gteq # fecha mayor o igual
created_at_lteq # fecha menor o igual
id_in           # id en array
status_not_eq   # status diferente de

# Combinaciones
title_or_body_cont  # título O cuerpo contiene
```

### Custom ransackers

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  # Búsqueda personalizada
  ransacker :published_year do
    Arel.sql("strftime('%Y', published_at)")
  end

  # Búsqueda en asociación
  ransacker :author_name do
    Arel.sql("(SELECT name FROM users WHERE users.id = posts.author_id)")
  end
end

# Uso: params[:q][:published_year_eq] = "2024"
```

## Meilisearch (motor de búsqueda externo)

### Setup

```ruby
# Gemfile
gem "meilisearch-rails"
```

```ruby
# config/initializers/meilisearch.rb
MeiliSearch::Rails.configuration = {
  meilisearch_url: ENV.fetch("MEILISEARCH_URL", "http://localhost:7700"),
  meilisearch_api_key: Rails.application.credentials.dig(:meilisearch, :api_key)
}
```

### Modelo

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include MeiliSearch::Rails

  meilisearch do
    # Atributos a indexar
    attribute :title, :body, :status
    attribute :author_name do
      author&.name
    end
    attribute :created_at_timestamp do
      created_at.to_i
    end

    # Atributos buscables
    searchable_attributes [:title, :body, :author_name]

    # Atributos filtrables
    filterable_attributes [:status, :author_id, :created_at_timestamp]

    # Atributos ordenables
    sortable_attributes [:created_at_timestamp, :title]

    # Ranking personalizado
    ranking_rules [
      "words",
      "typo",
      "proximity",
      "attribute",
      "sort",
      "exactness"
    ]
  end

  # Solo indexar posts publicados
  def self.meilisearch_settings
    {
      if: -> (post) { post.published? }
    }
  end
end
```

### Controller

```ruby
class SearchController < ApplicationController
  def index
    if params[:q].present?
      @results = Post.search(
        params[:q],
        {
          filters: build_filters,
          sort: [params[:sort] || "created_at_timestamp:desc"],
          limit: 20,
          offset: (params[:page].to_i - 1) * 20,
          attributes_to_highlight: ["title", "body"],
          highlight_pre_tag: "<mark>",
          highlight_post_tag: "</mark>"
        }
      )
    else
      @results = []
    end
  end

  private

  def build_filters
    filters = []
    filters << "status = published"
    filters << "author_id = #{params[:author_id]}" if params[:author_id].present?
    filters.join(" AND ")
  end
end
```

### Vista con highlights

```erb
<% @results.each do |result| %>
  <article>
    <h2>
      <%# Usar highlighted si está disponible %>
      <%= raw result._formatted&.dig("title") || result.title %>
    </h2>
    <p>
      <%= raw result._formatted&.dig("body")&.truncate(200) || result.body.truncate(200) %>
    </p>
  </article>
<% end %>
```

## Elasticsearch (alternativa enterprise)

### Setup básico

```ruby
# Gemfile
gem "elasticsearch-model"
gem "elasticsearch-rails"
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include Elasticsearch::Model
  include Elasticsearch::Model::Callbacks

  settings do
    mappings dynamic: "false" do
      indexes :title, type: "text", analyzer: "spanish"
      indexes :body, type: "text", analyzer: "spanish"
      indexes :status, type: "keyword"
      indexes :created_at, type: "date"
    end
  end

  def as_indexed_json(options = {})
    as_json(only: [:title, :body, :status, :created_at])
  end
end
```

## Autocompletado

### Con Stimulus

```javascript
// app/javascript/controllers/autocomplete_controller.js
import { Controller } from "@hotwired/stimulus"
import { debounce } from "lodash-es"

export default class extends Controller {
  static targets = ["input", "results"]
  static values = { url: String, minLength: { type: Number, default: 2 } }

  connect() {
    this.search = debounce(this.search.bind(this), 300)
  }

  async search() {
    const query = this.inputTarget.value

    if (query.length < this.minLengthValue) {
      this.hideResults()
      return
    }

    const response = await fetch(`${this.urlValue}?q=${encodeURIComponent(query)}`)
    const html = await response.text()

    this.resultsTarget.innerHTML = html
    this.showResults()
  }

  select(event) {
    this.inputTarget.value = event.currentTarget.dataset.value
    this.hideResults()
  }

  showResults() {
    this.resultsTarget.classList.remove("hidden")
  }

  hideResults() {
    this.resultsTarget.classList.add("hidden")
  }

  // Cerrar al hacer click fuera
  clickOutside(event) {
    if (!this.element.contains(event.target)) {
      this.hideResults()
    }
  }
}
```

### Controller de autocompletado

```ruby
class AutocompleteController < ApplicationController
  def posts
    @posts = Post.search(params[:q]).limit(10)
    render partial: "autocomplete/posts", locals: { posts: @posts }
  end
end
```

### Vista

```erb
<%# Formulario con autocompletado %>
<div data-controller="autocomplete"
     data-autocomplete-url-value="<%= autocomplete_posts_path %>"
     data-action="click@window->autocomplete#clickOutside">

  <%= text_field_tag :q,
                     params[:q],
                     placeholder: "Buscar...",
                     data: {
                       autocomplete_target: "input",
                       action: "input->autocomplete#search"
                     },
                     class: "input w-full" %>

  <div data-autocomplete-target="results"
       class="hidden absolute bg-white border rounded-lg shadow-lg mt-1 w-full z-50">
    <%# Los resultados se cargan aquí %>
  </div>
</div>

<%# app/views/autocomplete/_posts.html.erb %>
<ul class="divide-y">
  <% posts.each do |post| %>
    <li>
      <button type="button"
              data-action="autocomplete#select"
              data-value="<%= post.title %>"
              class="w-full text-left px-4 py-2 hover:bg-gray-100">
        <%= post.title %>
        <span class="text-gray-500 text-sm block"><%= truncate(post.body, length: 50) %></span>
      </button>
    </li>
  <% end %>
</ul>
```

## Checklist

- [ ] Índices creados en columnas buscadas
- [ ] Queries sanitizadas (no SQL injection)
- [ ] Paginación implementada
- [ ] Búsqueda case-insensitive
- [ ] Resultados destacados (highlights)
- [ ] Debounce en autocompletado
- [ ] Manejo de queries vacías
- [ ] Performance testeada con datos reales
