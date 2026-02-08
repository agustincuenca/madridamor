# Skill: SEO

## Purpose

Optimizar aplicaciones Rails para motores de búsqueda, incluyendo meta tags, structured data, sitemaps, y performance.

## Meta Tags

### Helper de meta tags

```ruby
# app/helpers/meta_tags_helper.rb
module MetaTagsHelper
  def meta_title(title = nil)
    if title.present?
      content_for(:meta_title) { title }
    else
      content_for?(:meta_title) ? content_for(:meta_title) : t("app.name")
    end
  end

  def meta_description(description = nil)
    if description.present?
      content_for(:meta_description) { description }
    else
      content_for?(:meta_description) ? content_for(:meta_description) : t("app.description")
    end
  end

  def meta_image(image_url = nil)
    if image_url.present?
      content_for(:meta_image) { image_url }
    else
      content_for?(:meta_image) ? content_for(:meta_image) : asset_url("og-image.png")
    end
  end

  def canonical_url(url = nil)
    if url.present?
      content_for(:canonical_url) { url }
    else
      content_for?(:canonical_url) ? content_for(:canonical_url) : request.original_url.split("?").first
    end
  end

  def full_title
    base_title = t("app.name")
    if content_for?(:meta_title)
      "#{content_for(:meta_title)} | #{base_title}"
    else
      base_title
    end
  end
end
```

### Layout con meta tags

```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html lang="<%= I18n.locale %>">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <%# Título %>
  <title><%= full_title %></title>

  <%# Meta básicos %>
  <meta name="description" content="<%= meta_description %>">
  <meta name="robots" content="<%= content_for?(:robots) ? content_for(:robots) : 'index, follow' %>">

  <%# Canonical %>
  <link rel="canonical" href="<%= canonical_url %>">

  <%# Open Graph %>
  <meta property="og:type" content="<%= content_for?(:og_type) ? content_for(:og_type) : 'website' %>">
  <meta property="og:title" content="<%= meta_title %>">
  <meta property="og:description" content="<%= meta_description %>">
  <meta property="og:image" content="<%= meta_image %>">
  <meta property="og:url" content="<%= canonical_url %>">
  <meta property="og:site_name" content="<%= t('app.name') %>">
  <meta property="og:locale" content="<%= I18n.locale %>">

  <%# Twitter Card %>
  <meta name="twitter:card" content="summary_large_image">
  <meta name="twitter:title" content="<%= meta_title %>">
  <meta name="twitter:description" content="<%= meta_description %>">
  <meta name="twitter:image" content="<%= meta_image %>">
  <% if Rails.application.credentials.dig(:twitter, :handle) %>
    <meta name="twitter:site" content="@<%= Rails.application.credentials.dig(:twitter, :handle) %>">
  <% end %>

  <%# Otros %>
  <%= csrf_meta_tags %>
  <%= csp_meta_tag %>
  <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
  <%= javascript_importmap_tags %>

  <%# Contenido extra en head %>
  <%= yield :head %>
</head>
<body>
  <%= yield %>
</body>
</html>
```

### Uso en vistas

```erb
<%# app/views/posts/show.html.erb %>
<%
  meta_title @post.title
  meta_description @post.excerpt || truncate(strip_tags(@post.body), length: 160)
  meta_image url_for(@post.cover_image) if @post.cover_image.attached?
  canonical_url post_url(@post)
  content_for(:og_type) { "article" }
%>

<article>
  <h1><%= @post.title %></h1>
  <%# ... %>
</article>
```

## Structured Data (Schema.org)

### Helper para JSON-LD

```ruby
# app/helpers/structured_data_helper.rb
module StructuredDataHelper
  def json_ld(data)
    content_tag(:script, data.to_json.html_safe, type: "application/ld+json")
  end

  def organization_schema
    {
      "@context": "https://schema.org",
      "@type": "Organization",
      name: t("app.name"),
      url: root_url,
      logo: asset_url("logo.png"),
      sameAs: [
        "https://twitter.com/myapp",
        "https://www.linkedin.com/company/myapp"
      ]
    }
  end

  def website_schema
    {
      "@context": "https://schema.org",
      "@type": "WebSite",
      name: t("app.name"),
      url: root_url,
      potentialAction: {
        "@type": "SearchAction",
        target: {
          "@type": "EntryPoint",
          urlTemplate: "#{search_url}?q={search_term_string}"
        },
        "query-input": "required name=search_term_string"
      }
    }
  end

  def article_schema(post)
    {
      "@context": "https://schema.org",
      "@type": "Article",
      headline: post.title,
      description: post.excerpt,
      image: post.cover_image.attached? ? url_for(post.cover_image) : nil,
      datePublished: post.published_at&.iso8601,
      dateModified: post.updated_at.iso8601,
      author: {
        "@type": "Person",
        name: post.author.name,
        url: user_url(post.author)
      },
      publisher: {
        "@type": "Organization",
        name: t("app.name"),
        logo: {
          "@type": "ImageObject",
          url: asset_url("logo.png")
        }
      },
      mainEntityOfPage: {
        "@type": "WebPage",
        "@id": post_url(post)
      }
    }.compact
  end

  def breadcrumb_schema(items)
    {
      "@context": "https://schema.org",
      "@type": "BreadcrumbList",
      itemListElement: items.map.with_index(1) do |item, index|
        {
          "@type": "ListItem",
          position: index,
          name: item[:name],
          item: item[:url]
        }
      end
    }
  end

  def product_schema(product)
    {
      "@context": "https://schema.org",
      "@type": "Product",
      name: product.name,
      description: product.description,
      image: product.images.attached? ? product.images.map { |i| url_for(i) } : nil,
      brand: {
        "@type": "Brand",
        name: product.brand
      },
      offers: {
        "@type": "Offer",
        price: product.price,
        priceCurrency: "EUR",
        availability: product.in_stock? ? "https://schema.org/InStock" : "https://schema.org/OutOfStock",
        url: product_url(product)
      }
    }.compact
  end

  def faq_schema(faqs)
    {
      "@context": "https://schema.org",
      "@type": "FAQPage",
      mainEntity: faqs.map do |faq|
        {
          "@type": "Question",
          name: faq[:question],
          acceptedAnswer: {
            "@type": "Answer",
            text: faq[:answer]
          }
        }
      end
    }
  end
end
```

### Uso en layout

```erb
<%# app/views/layouts/application.html.erb %>
<head>
  <%# ... otros meta tags ... %>

  <%# Schema.org global %>
  <%= json_ld(organization_schema) %>
  <%= json_ld(website_schema) %>

  <%# Schema.org específico de la página %>
  <%= yield :structured_data %>
</head>
```

### Uso en vistas

```erb
<%# app/views/posts/show.html.erb %>
<% content_for :structured_data do %>
  <%= json_ld(article_schema(@post)) %>
  <%= json_ld(breadcrumb_schema([
    { name: "Home", url: root_url },
    { name: "Blog", url: posts_url },
    { name: @post.title, url: post_url(@post) }
  ])) %>
<% end %>
```

## Sitemap

### Gem sitemap_generator

```ruby
# Gemfile
gem "sitemap_generator"
```

```ruby
# config/sitemap.rb
SitemapGenerator::Sitemap.default_host = "https://myapp.com"
SitemapGenerator::Sitemap.compress = false  # Para debug
SitemapGenerator::Sitemap.create do
  # Páginas estáticas
  add root_path, changefreq: "daily", priority: 1.0
  add about_path, changefreq: "monthly", priority: 0.7
  add contact_path, changefreq: "monthly", priority: 0.7

  # Posts
  Post.published.find_each do |post|
    add post_path(post),
        lastmod: post.updated_at,
        changefreq: "weekly",
        priority: 0.8,
        images: post.images.map { |img|
          { loc: url_for(img), title: post.title }
        }
  end

  # Categorías
  Category.find_each do |category|
    add category_path(category),
        lastmod: category.posts.maximum(:updated_at),
        changefreq: "weekly",
        priority: 0.6
  end

  # Usuarios/perfiles públicos
  User.with_public_profile.find_each do |user|
    add user_path(user),
        lastmod: user.updated_at,
        changefreq: "weekly",
        priority: 0.5
  end
end
```

```bash
# Generar sitemap
rails sitemap:refresh

# Notificar a motores de búsqueda
rails sitemap:refresh:no_ping  # Sin notificar
```

### Tarea programada

```ruby
# config/schedule.rb (whenever gem) o Solid Queue
every 1.day, at: "4:00 am" do
  rake "sitemap:refresh"
end
```

## Robots.txt

```ruby
# config/routes.rb
get "robots.txt", to: "pages#robots", defaults: { format: :text }

# app/controllers/pages_controller.rb
class PagesController < ApplicationController
  def robots
    render plain: <<~ROBOTS
      User-agent: *
      Disallow: /admin/
      Disallow: /api/
      Disallow: /users/sign_in
      Disallow: /users/sign_up
      Disallow: /search?

      Sitemap: #{sitemap_url}
    ROBOTS
  end

  private

  def sitemap_url
    "#{request.protocol}#{request.host_with_port}/sitemap.xml"
  end
end
```

## URLs amigables

### Slugs con friendly_id

```ruby
# Gemfile
gem "friendly_id"
```

```ruby
# db/migrate/xxx_add_slug_to_posts.rb
class AddSlugToPosts < ActiveRecord::Migration[8.0]
  def change
    add_column :posts, :slug, :string
    add_index :posts, :slug, unique: true
  end
end

# app/models/post.rb
class Post < ApplicationRecord
  extend FriendlyId
  friendly_id :title, use: [:slugged, :history]

  # Regenerar slug solo si título cambia significativamente
  def should_generate_new_friendly_id?
    title_changed? || slug.blank?
  end
end

# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def show
    @post = Post.friendly.find(params[:id])

    # Redirect a URL canónica si slug cambió
    if request.path != post_path(@post)
      redirect_to @post, status: :moved_permanently
    end
  end
end
```

### URLs localizadas

```ruby
# config/routes.rb
scope "(:locale)", locale: /es|en/ do
  resources :posts, path_names: {
    new: I18n.t("routes.new"),
    edit: I18n.t("routes.edit")
  }
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  around_action :switch_locale

  def default_url_options
    { locale: I18n.locale }
  end

  private

  def switch_locale(&action)
    locale = params[:locale] || I18n.default_locale
    I18n.with_locale(locale, &action)
  end
end
```

## Performance SEO

### Lazy loading de imágenes

```erb
<%= image_tag post.cover_image,
              loading: "lazy",
              decoding: "async",
              alt: post.title,
              class: "w-full h-auto" %>
```

### Preload de recursos críticos

```erb
<head>
  <%# Preload fuentes %>
  <link rel="preload" href="<%= asset_path('Inter.woff2') %>" as="font" type="font/woff2" crossorigin>

  <%# Preload imagen LCP %>
  <% if content_for?(:preload_image) %>
    <link rel="preload" href="<%= content_for(:preload_image) %>" as="image">
  <% end %>

  <%# Preconnect a servicios externos %>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://www.google-analytics.com">
</head>
```

### Core Web Vitals

```erb
<%# Evitar CLS (Cumulative Layout Shift) %>
<%= image_tag post.image,
              width: 800,
              height: 600,
              class: "aspect-[4/3] object-cover" %>

<%# Skeleton para contenido dinámico %>
<div class="min-h-[200px]">  <!-- Reservar espacio -->
  <%= turbo_frame_tag "comments", src: post_comments_path(@post) do %>
    <div class="animate-pulse bg-gray-200 h-32 rounded"></div>
  <% end %>
</div>
```

## Hreflang para múltiples idiomas

```erb
<head>
  <% I18n.available_locales.each do |locale| %>
    <link rel="alternate"
          hreflang="<%= locale %>"
          href="<%= url_for(locale: locale) %>">
  <% end %>
  <link rel="alternate" hreflang="x-default" href="<%= url_for(locale: I18n.default_locale) %>">
</head>
```

## Checklist SEO

### Técnico
- [ ] Meta title único por página (50-60 chars)
- [ ] Meta description única (150-160 chars)
- [ ] URLs amigables con slugs
- [ ] Canonical URLs configuradas
- [ ] Sitemap.xml generado y actualizado
- [ ] Robots.txt configurado
- [ ] HTTPS habilitado
- [ ] Mobile-friendly (responsive)

### Contenido
- [ ] H1 único por página
- [ ] Jerarquía de headings (H1 > H2 > H3)
- [ ] Alt text en imágenes
- [ ] Internal linking

### Structured Data
- [ ] Organization schema
- [ ] Article schema para posts
- [ ] BreadcrumbList schema
- [ ] Product schema (si aplica)
- [ ] FAQ schema (si aplica)

### Social
- [ ] Open Graph tags
- [ ] Twitter Card tags
- [ ] Imagen OG (1200x630px)

### Performance
- [ ] Core Web Vitals optimizados
- [ ] Imágenes optimizadas (WebP, lazy loading)
- [ ] CSS/JS minificados
- [ ] Preload de recursos críticos
