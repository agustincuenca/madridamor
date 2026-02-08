# Skill: Accessibility

## Purpose
Ensure applications meet WCAG 2.1 AA standards and are usable by everyone.

## Core Principles (POUR)

1. **Perceivable** - Information must be presentable
2. **Operable** - Interface must be navigable
3. **Understandable** - Content must be readable
4. **Robust** - Compatible with assistive technologies

## Semantic HTML

### Document Structure
```erb
<!DOCTYPE html>
<html lang="<%= I18n.locale %>">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><%= content_for(:title) || "App Name" %></title>
</head>
<body>
  <a href="#main-content" class="sr-only focus:not-sr-only">
    Skip to main content
  </a>

  <header role="banner">
    <nav role="navigation" aria-label="Main navigation">
      <!-- Navigation -->
    </nav>
  </header>

  <main id="main-content" role="main">
    <%= yield %>
  </main>

  <footer role="contentinfo">
    <!-- Footer -->
  </footer>
</body>
</html>
```

### Headings Hierarchy
```erb
<%# Correct hierarchy - never skip levels %>
<h1>Page Title</h1>
  <h2>Section</h2>
    <h3>Subsection</h3>
    <h3>Subsection</h3>
  <h2>Section</h2>
    <h3>Subsection</h3>
```

### Landmarks
```erb
<header>     <%# role="banner" implied %>
<nav>        <%# role="navigation" implied %>
<main>       <%# role="main" implied %>
<aside>      <%# role="complementary" implied %>
<footer>     <%# role="contentinfo" implied %>
<section>    <%# role="region" when has accessible name %>
<article>    <%# role="article" implied %>
```

## Forms

### Accessible Form
```erb
<%= form_with model: @user, class: "space-y-4" do |f| %>
  <%# Error summary %>
  <% if @user.errors.any? %>
    <div role="alert" aria-labelledby="error-heading" class="bg-red-50 p-4 rounded">
      <h2 id="error-heading" class="text-red-800 font-bold">
        Please fix <%= pluralize(@user.errors.count, "error") %>:
      </h2>
      <ul class="list-disc list-inside text-red-700">
        <% @user.errors.full_messages.each do |error| %>
          <li><%= error %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <%# Text input with label %>
  <div>
    <%= f.label :name, class: "block font-medium" %>
    <%= f.text_field :name,
        required: true,
        aria: { describedby: "name-hint" },
        class: "mt-1 block w-full rounded border-gray-300" %>
    <p id="name-hint" class="mt-1 text-sm text-gray-500">
      Your full name as it appears on official documents.
    </p>
  </div>

  <%# Email with error %>
  <div>
    <%= f.label :email, class: "block font-medium" %>
    <%= f.email_field :email,
        required: true,
        aria: {
          invalid: @user.errors[:email].any?,
          describedby: @user.errors[:email].any? ? "email-error" : nil
        },
        class: "mt-1 block w-full rounded #{@user.errors[:email].any? ? 'border-red-500' : 'border-gray-300'}" %>
    <% if @user.errors[:email].any? %>
      <p id="email-error" class="mt-1 text-sm text-red-600" role="alert">
        <%= @user.errors[:email].first %>
      </p>
    <% end %>
  </div>

  <%# Checkbox group %>
  <fieldset>
    <legend class="font-medium">Notification preferences</legend>
    <div class="mt-2 space-y-2">
      <div class="flex items-center gap-2">
        <%= f.check_box :email_notifications, class: "rounded" %>
        <%= f.label :email_notifications, "Email notifications" %>
      </div>
      <div class="flex items-center gap-2">
        <%= f.check_box :sms_notifications, class: "rounded" %>
        <%= f.label :sms_notifications, "SMS notifications" %>
      </div>
    </div>
  </fieldset>

  <%= f.submit "Save", class: "bg-blue-600 text-white px-4 py-2 rounded" %>
<% end %>
```

### Required Fields
```erb
<%# Visual indicator %>
<label for="email">
  Email <span class="text-red-500" aria-hidden="true">*</span>
  <span class="sr-only">(required)</span>
</label>
<input type="email" id="email" required aria-required="true">
```

## Images

### Informative Images
```erb
<%# Meaningful image - describe it %>
<%= image_tag "chart.png", alt: "Sales increased 25% from January to March 2024" %>

<%# Decorative image - empty alt %>
<%= image_tag "decoration.png", alt: "", role: "presentation" %>

<%# Complex image %>
<figure>
  <%= image_tag "infographic.png", alt: "Company growth infographic", aria: { describedby: "infographic-desc" } %>
  <figcaption id="infographic-desc">
    Detailed description of the infographic...
  </figcaption>
</figure>
```

### Icons
```erb
<%# Icon with text - hide icon %>
<button>
  <svg aria-hidden="true"><!-- icon --></svg>
  <span>Delete</span>
</button>

<%# Icon only - add label %>
<button aria-label="Delete item">
  <svg aria-hidden="true"><!-- icon --></svg>
</button>

<%# Icon with visible text alternative %>
<button>
  <svg aria-hidden="true"><!-- icon --></svg>
  <span class="sr-only">Delete</span>
</button>
```

## Interactive Elements

### Buttons vs Links
```erb
<%# Link - navigates somewhere %>
<%= link_to "View article", article_path(@article) %>

<%# Button - performs action %>
<button type="button" onclick="toggleMenu()">Menu</button>

<%# Button that looks like link %>
<button type="button" class="text-blue-600 underline">
  Show more
</button>
```

### Custom Controls
```erb
<%# Custom checkbox %>
<div role="checkbox"
     aria-checked="false"
     tabindex="0"
     data-controller="checkbox"
     data-action="click->checkbox#toggle keydown->checkbox#handleKey">
  <!-- Custom styling -->
</div>

<%# Disclosure/accordion %>
<div data-controller="disclosure">
  <button type="button"
          aria-expanded="false"
          aria-controls="panel-1"
          data-action="disclosure#toggle">
    Section Title
  </button>
  <div id="panel-1" hidden data-disclosure-target="panel">
    Content here...
  </div>
</div>
```

### Modal Dialog
```erb
<div role="dialog"
     aria-modal="true"
     aria-labelledby="modal-title"
     data-controller="modal"
     tabindex="-1">
  <h2 id="modal-title">Confirm Action</h2>
  <p>Are you sure you want to proceed?</p>
  <button type="button" data-action="modal#close">Cancel</button>
  <button type="button" data-action="modal#confirm">Confirm</button>
</div>
```

## Keyboard Navigation

### Focus Management
```javascript
// controllers/modal_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["dialog", "firstFocusable"]

  open() {
    this.previousFocus = document.activeElement
    this.dialogTarget.hidden = false
    this.firstFocusableTarget.focus()
    this.trapFocus()
  }

  close() {
    this.dialogTarget.hidden = true
    this.previousFocus?.focus()
  }

  trapFocus() {
    const focusable = this.dialogTarget.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    )
    const first = focusable[0]
    const last = focusable[focusable.length - 1]

    this.dialogTarget.addEventListener('keydown', (e) => {
      if (e.key === 'Tab') {
        if (e.shiftKey && document.activeElement === first) {
          e.preventDefault()
          last.focus()
        } else if (!e.shiftKey && document.activeElement === last) {
          e.preventDefault()
          first.focus()
        }
      }
      if (e.key === 'Escape') {
        this.close()
      }
    })
  }
}
```

### Skip Links
```erb
<body>
  <a href="#main-content"
     class="sr-only focus:not-sr-only focus:absolute focus:top-4 focus:left-4
            bg-white p-2 rounded shadow">
    Skip to main content
  </a>

  <a href="#navigation"
     class="sr-only focus:not-sr-only focus:absolute focus:top-4 focus:left-40
            bg-white p-2 rounded shadow">
    Skip to navigation
  </a>
</body>
```

## ARIA Live Regions

```erb
<%# Polite announcements %>
<div aria-live="polite" aria-atomic="true" class="sr-only">
  <%= flash[:notice] %>
</div>

<%# Urgent announcements %>
<div role="alert" aria-live="assertive">
  <%= flash[:alert] %>
</div>

<%# Status updates %>
<div role="status" aria-live="polite">
  Loading... <%= @progress %>% complete
</div>
```

## Color and Contrast

### Tailwind Accessible Colors
```erb
<%# Good contrast ratios %>
<p class="text-gray-900 bg-white">High contrast (21:1)</p>
<p class="text-gray-700 bg-white">Good contrast (8.5:1)</p>
<p class="text-gray-600 bg-white">Minimum for large text (5.7:1)</p>

<%# Don't rely on color alone %>
<span class="text-red-600">
  <svg aria-hidden="true"><!-- error icon --></svg>
  Error: Invalid input
</span>
```

## Testing Accessibility

### Automated Testing
```ruby
# Gemfile
gem "axe-core-rspec"

# spec/system/accessibility_spec.rb
require "rails_helper"

RSpec.describe "Accessibility", type: :system do
  it "homepage is accessible" do
    visit root_path
    expect(page).to be_axe_clean
  end

  it "article page is accessible" do
    article = create(:article, :published)
    visit article_path(article)
    expect(page).to be_axe_clean
  end
end
```

## Screen Reader Only Class

```css
/* Tailwind's sr-only */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border-width: 0;
}

.focus\:not-sr-only:focus {
  position: static;
  width: auto;
  height: auto;
  padding: inherit;
  margin: inherit;
  overflow: visible;
  clip: auto;
  white-space: normal;
}
```

## Checklist

- [ ] Page has single `<h1>`
- [ ] Heading hierarchy is correct
- [ ] All images have appropriate alt text
- [ ] Forms have labels and error messages
- [ ] Color contrast meets 4.5:1 minimum
- [ ] Interactive elements are keyboard accessible
- [ ] Focus is visible and logical
- [ ] Page works at 200% zoom
- [ ] No content relies on color alone
- [ ] Animations respect prefers-reduced-motion
