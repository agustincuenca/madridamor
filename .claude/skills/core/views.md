# Skill: Views

## Purpose
Create Rails views with ERB, Tailwind CSS, and proper structure following mobile-first design principles.

## View Structure

```
app/views/
├── layouts/
│   ├── application.html.erb
│   └── _navigation.html.erb
├── shared/
│   ├── _flash.html.erb
│   ├── _footer.html.erb
│   └── _pagination.html.erb
└── articles/
    ├── index.html.erb
    ├── show.html.erb
    ├── new.html.erb
    ├── edit.html.erb
    ├── _article.html.erb
    └── _form.html.erb
```

## Layout Template

```erb
<!DOCTYPE html>
<html lang="<%= I18n.locale %>" class="h-full">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><%= content_for(:title) || "App Name" %></title>
  <%= csrf_meta_tags %>
  <%= csp_meta_tag %>
  <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
  <%= javascript_importmap_tags %>
</head>
<body class="h-full bg-gray-50">
  <%= render "shared/navigation" %>

  <main class="container mx-auto px-4 py-8">
    <%= render "shared/flash" %>
    <%= yield %>
  </main>

  <%= render "shared/footer" %>
</body>
</html>
```

## Flash Messages

```erb
<%# app/views/shared/_flash.html.erb %>
<div id="flash" class="fixed top-4 right-4 z-50 space-y-2">
  <% flash.each do |type, message| %>
    <div class="<%= flash_class(type) %> px-4 py-3 rounded-lg shadow-lg flex items-center gap-2"
         data-controller="flash"
         data-flash-target="message">
      <span><%= message %></span>
      <button type="button" data-action="flash#dismiss" class="ml-auto">
        <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 20 20">
          <path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z" clip-rule="evenodd"/>
        </svg>
      </button>
    </div>
  <% end %>
</div>
```

## Index View

```erb
<%# app/views/articles/index.html.erb %>
<% content_for :title, "Articles" %>

<div class="flex justify-between items-center mb-6">
  <h1 class="text-2xl font-bold text-gray-900">Articles</h1>
  <%= link_to "New Article", new_article_path,
      class: "bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-lg" %>
</div>

<%= turbo_frame_tag "articles" do %>
  <div class="space-y-4">
    <%= render @articles %>
  </div>

  <% if @articles.empty? %>
    <div class="text-center py-12 text-gray-500">
      <p>No articles yet.</p>
      <%= link_to "Create your first article", new_article_path, class: "text-blue-600 hover:underline" %>
    </div>
  <% end %>
<% end %>

<%= render "shared/pagination", pagy: @pagy %>
```

## Show View

```erb
<%# app/views/articles/show.html.erb %>
<% content_for :title, @article.title %>

<article class="max-w-3xl mx-auto">
  <header class="mb-8">
    <h1 class="text-3xl font-bold text-gray-900 mb-2"><%= @article.title %></h1>
    <div class="flex items-center gap-4 text-sm text-gray-500">
      <span>By <%= @article.user.name %></span>
      <time datetime="<%= @article.created_at.iso8601 %>">
        <%= @article.created_at.strftime("%B %d, %Y") %>
      </time>
    </div>
  </header>

  <div class="prose prose-lg max-w-none">
    <%= simple_format(@article.body) %>
  </div>

  <footer class="mt-8 pt-8 border-t flex gap-4">
    <%= link_to "Edit", edit_article_path(@article), class: "text-blue-600 hover:underline" %>
    <%= button_to "Delete", @article, method: :delete,
        data: { turbo_confirm: "Are you sure?" },
        class: "text-red-600 hover:underline" %>
    <%= link_to "Back", articles_path, class: "text-gray-600 hover:underline" %>
  </footer>
</article>
```

## Form Partial

```erb
<%# app/views/articles/_form.html.erb %>
<%= form_with model: article, class: "space-y-6" do |f| %>
  <% if article.errors.any? %>
    <div class="bg-red-50 border border-red-200 rounded-lg p-4">
      <h3 class="text-red-800 font-medium mb-2">Please fix the following errors:</h3>
      <ul class="list-disc list-inside text-red-700 text-sm">
        <% article.errors.full_messages.each do |error| %>
          <li><%= error %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div>
    <%= f.label :title, class: "block text-sm font-medium text-gray-700 mb-1" %>
    <%= f.text_field :title,
        class: "w-full rounded-lg border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500",
        required: true %>
  </div>

  <div>
    <%= f.label :body, class: "block text-sm font-medium text-gray-700 mb-1" %>
    <%= f.text_area :body,
        rows: 10,
        class: "w-full rounded-lg border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500" %>
  </div>

  <div class="flex items-center gap-2">
    <%= f.check_box :published, class: "rounded border-gray-300 text-blue-600 focus:ring-blue-500" %>
    <%= f.label :published, "Publish immediately", class: "text-sm text-gray-700" %>
  </div>

  <div class="flex gap-4">
    <%= f.submit class: "bg-blue-600 hover:bg-blue-700 text-white px-6 py-2 rounded-lg cursor-pointer" %>
    <%= link_to "Cancel", articles_path, class: "text-gray-600 hover:underline py-2" %>
  </div>
<% end %>
```

## Card Partial

```erb
<%# app/views/articles/_article.html.erb %>
<%= turbo_frame_tag dom_id(article) do %>
  <article class="bg-white rounded-lg shadow p-6 hover:shadow-md transition-shadow">
    <h2 class="text-xl font-semibold mb-2">
      <%= link_to article.title, article, class: "text-gray-900 hover:text-blue-600" %>
    </h2>

    <p class="text-gray-600 mb-4 line-clamp-2"><%= truncate(article.body, length: 150) %></p>

    <div class="flex items-center justify-between text-sm text-gray-500">
      <span><%= article.user.name %></span>
      <time><%= time_ago_in_words(article.created_at) %> ago</time>
    </div>
  </article>
<% end %>
```

## Helper Methods

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  def flash_class(type)
    case type.to_sym
    when :notice, :success
      "bg-green-100 text-green-800 border border-green-200"
    when :alert, :error
      "bg-red-100 text-red-800 border border-red-200"
    when :warning
      "bg-yellow-100 text-yellow-800 border border-yellow-200"
    else
      "bg-blue-100 text-blue-800 border border-blue-200"
    end
  end

  def active_link_class(path)
    current_page?(path) ? "text-blue-600 font-medium" : "text-gray-600 hover:text-gray-900"
  end
end
```

## Responsive Design

```erb
<%# Mobile-first grid %>
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
  <%= render @articles %>
</div>

<%# Responsive navigation %>
<nav class="flex flex-col md:flex-row md:items-center gap-4">
  <%= link_to "Home", root_path %>
  <%= link_to "Articles", articles_path %>
</nav>

<%# Hidden on mobile %>
<div class="hidden md:block">Desktop only</div>

<%# Visible on mobile %>
<div class="md:hidden">Mobile only</div>
```

## Turbo Frames

```erb
<%# Lazy loading %>
<%= turbo_frame_tag "comments", src: article_comments_path(@article), loading: :lazy do %>
  <p class="text-gray-500">Loading comments...</p>
<% end %>

<%# Modal frame %>
<%= turbo_frame_tag "modal" %>

<%# Form that updates frame %>
<%= turbo_frame_tag "search_results" do %>
  <%= form_with url: search_path, method: :get, data: { turbo_frame: "search_results" } do |f| %>
    <%= f.search_field :q, placeholder: "Search..." %>
  <% end %>

  <div id="results">
    <%= render @results %>
  </div>
<% end %>
```
