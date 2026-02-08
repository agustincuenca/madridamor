# Skill: Hotwire

## Purpose
Implement dynamic, SPA-like interactions using Turbo (Drive, Frames, Streams) and Stimulus without writing custom JavaScript.

## Turbo Drive

### Automatic Enhancement
Turbo Drive is enabled by default. All links and forms are enhanced automatically.

### Disabling Turbo
```erb
<%# Disable for specific link %>
<%= link_to "External", "https://example.com", data: { turbo: false } %>

<%# Disable for form %>
<%= form_with model: @article, data: { turbo: false } do |f| %>
<% end %>
```

### Progress Bar
```css
/* Customize in application.css */
.turbo-progress-bar {
  height: 3px;
  background-color: #3b82f6;
}
```

## Turbo Frames

### Basic Frame
```erb
<%# index.html.erb %>
<%= turbo_frame_tag "articles" do %>
  <%= render @articles %>
<% end %>

<%# _article.html.erb %>
<%= turbo_frame_tag dom_id(article) do %>
  <div class="article-card">
    <%= article.title %>
    <%= link_to "Edit", edit_article_path(article) %>
  </div>
<% end %>

<%# edit.html.erb %>
<%= turbo_frame_tag dom_id(@article) do %>
  <%= render "form", article: @article %>
<% end %>
```

### Lazy Loading
```erb
<%= turbo_frame_tag "comments",
    src: article_comments_path(@article),
    loading: :lazy do %>
  <p>Loading comments...</p>
<% end %>
```

### Breaking Out of Frames
```erb
<%# Target entire page %>
<%= link_to "View All", articles_path, data: { turbo_frame: "_top" } %>

<%# Target specific frame %>
<%= link_to "Details", article_path(article), data: { turbo_frame: "main_content" } %>
```

### Frame Targeting
```erb
<%# Form targets different frame %>
<%= form_with model: @comment, data: { turbo_frame: "comments" } do |f| %>
  <%= f.text_area :body %>
  <%= f.submit %>
<% end %>
```

## Turbo Streams

### Stream Actions
```erb
<%# append - Add to end %>
<%= turbo_stream.append "articles", @article %>

<%# prepend - Add to beginning %>
<%= turbo_stream.prepend "articles", @article %>

<%# replace - Replace entire element %>
<%= turbo_stream.replace @article %>

<%# update - Replace contents only %>
<%= turbo_stream.update "counter", "42" %>

<%# remove - Delete element %>
<%= turbo_stream.remove @article %>

<%# before/after - Insert adjacent %>
<%= turbo_stream.before @article, partial: "divider" %>
```

### Controller Response
```ruby
def create
  @article = current_user.articles.build(article_params)

  if @article.save
    respond_to do |format|
      format.html { redirect_to @article }
      format.turbo_stream
    end
  else
    render :new, status: :unprocessable_entity
  end
end
```

### Stream Template
```erb
<%# app/views/articles/create.turbo_stream.erb %>
<%= turbo_stream.prepend "articles", @article %>
<%= turbo_stream.update "articles_count", Article.count %>
<%= turbo_stream.replace "new_article_form" do %>
  <%= render "form", article: Article.new %>
<% end %>
```

### Inline Streams
```ruby
def destroy
  @article.destroy

  render turbo_stream: turbo_stream.remove(@article)
end

# Multiple streams
render turbo_stream: [
  turbo_stream.remove(@article),
  turbo_stream.update("count", Article.count)
]
```

## Stimulus Controllers

### Basic Controller
```javascript
// app/javascript/controllers/hello_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["name", "output"]
  static values = { greeting: { type: String, default: "Hello" } }

  greet() {
    this.outputTarget.textContent = `${this.greetingValue}, ${this.nameTarget.value}!`
  }
}
```

### HTML Usage
```erb
<div data-controller="hello" data-hello-greeting-value="Hi">
  <input data-hello-target="name" type="text">
  <button data-action="click->hello#greet">Greet</button>
  <span data-hello-target="output"></span>
</div>
```

### Common Controllers

#### Toggle
```javascript
// toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["content"]
  static classes = ["hidden"]

  toggle() {
    this.contentTarget.classList.toggle(this.hiddenClass)
  }
}
```

#### Flash Auto-Dismiss
```javascript
// flash_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    setTimeout(() => this.dismiss(), 5000)
  }

  dismiss() {
    this.element.remove()
  }
}
```

#### Form Validation
```javascript
// form_validation_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["submit"]

  validate(event) {
    const form = event.target.closest("form")
    this.submitTarget.disabled = !form.checkValidity()
  }
}
```

#### Dropdown
```javascript
// dropdown_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu"]

  toggle() {
    this.menuTarget.classList.toggle("hidden")
  }

  close(event) {
    if (!this.element.contains(event.target)) {
      this.menuTarget.classList.add("hidden")
    }
  }

  connect() {
    document.addEventListener("click", this.close.bind(this))
  }

  disconnect() {
    document.removeEventListener("click", this.close.bind(this))
  }
}
```

## Real-Time with Turbo Streams

### Broadcast from Model
```ruby
class Comment < ApplicationRecord
  belongs_to :article

  after_create_commit -> {
    broadcast_prepend_to article, target: "comments"
  }

  after_destroy_commit -> {
    broadcast_remove_to article
  }
end
```

### View Subscription
```erb
<%= turbo_stream_from @article %>

<div id="comments">
  <%= render @article.comments %>
</div>
```

## Best Practices

1. **Use frames for in-place editing** - Edit forms replace show content
2. **Use streams for lists** - Append/prepend without full reload
3. **Keep Stimulus simple** - Small, focused controllers
4. **Leverage morphing** - For complex updates
5. **Progressive enhancement** - Works without JS first
