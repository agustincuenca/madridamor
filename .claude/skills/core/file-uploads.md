# Skill: File Uploads

## Purpose
Handle file uploads using Active Storage with local disk storage.

## Setup

### Install Active Storage
```bash
rails active_storage:install
rails db:migrate
```

### Configure Storage
```yaml
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>
```

```ruby
# config/environments/development.rb
config.active_storage.service = :local

# config/environments/production.rb
config.active_storage.service = :local  # Or cloud service
```

## Model Attachments

### Single Attachment
```ruby
class User < ApplicationRecord
  has_one_attached :avatar

  # With validation
  validates :avatar, content_type: ['image/png', 'image/jpeg'],
                     size: { less_than: 5.megabytes }
end
```

### Multiple Attachments
```ruby
class Article < ApplicationRecord
  has_many_attached :images

  validates :images, content_type: ['image/png', 'image/jpeg', 'image/webp'],
                     size: { less_than: 10.megabytes },
                     limit: { max: 10 }
end
```

### With Variants (Image Processing)
```ruby
# Gemfile
gem "image_processing", "~> 1.2"
```

```ruby
class User < ApplicationRecord
  has_one_attached :avatar do |attachable|
    attachable.variant :thumb, resize_to_limit: [100, 100]
    attachable.variant :medium, resize_to_limit: [300, 300]
    attachable.variant :large, resize_to_limit: [800, 800]
  end
end
```

## Controller Handling

### Single File Upload
```ruby
class UsersController < ApplicationController
  def update
    if current_user.update(user_params)
      redirect_to current_user, notice: "Profile updated."
    else
      render :edit, status: :unprocessable_entity
    end
  end

  private

  def user_params
    params.require(:user).permit(:name, :email, :avatar)
  end
end
```

### Multiple File Upload
```ruby
class ArticlesController < ApplicationController
  def create
    @article = current_user.articles.build(article_params)

    if @article.save
      redirect_to @article
    else
      render :new, status: :unprocessable_entity
    end
  end

  def update
    @article = Article.find(params[:id])

    if @article.update(article_params)
      redirect_to @article
    else
      render :edit, status: :unprocessable_entity
    end
  end

  private

  def article_params
    params.require(:article).permit(:title, :body, images: [])
  end
end
```

### Removing Attachments
```ruby
class UsersController < ApplicationController
  def remove_avatar
    current_user.avatar.purge
    redirect_to edit_user_path(current_user), notice: "Avatar removed."
  end
end

# Routes
resources :users do
  member do
    delete :remove_avatar
  end
end
```

## Form Helpers

### Single File
```erb
<%= form_with model: @user do |f| %>
  <div>
    <%= f.label :avatar %>
    <%= f.file_field :avatar,
        accept: "image/png,image/jpeg",
        class: "block w-full text-sm text-gray-500
               file:mr-4 file:py-2 file:px-4
               file:rounded file:border-0
               file:text-sm file:font-semibold
               file:bg-blue-50 file:text-blue-700
               hover:file:bg-blue-100" %>
  </div>

  <% if @user.avatar.attached? %>
    <div class="mt-2">
      <%= image_tag @user.avatar.variant(:thumb), class: "rounded" %>
      <%= link_to "Remove", remove_avatar_user_path(@user),
          method: :delete,
          data: { turbo_confirm: "Remove avatar?" },
          class: "text-red-600 text-sm" %>
    </div>
  <% end %>

  <%= f.submit "Save" %>
<% end %>
```

### Multiple Files
```erb
<%= form_with model: @article do |f| %>
  <div>
    <%= f.label :images %>
    <%= f.file_field :images,
        multiple: true,
        accept: "image/*",
        class: "block w-full" %>
  </div>

  <% if @article.images.attached? %>
    <div class="grid grid-cols-3 gap-4 mt-4">
      <% @article.images.each do |image| %>
        <div class="relative">
          <%= image_tag image.variant(resize_to_limit: [200, 200]), class: "rounded" %>
          <%= button_to "×",
              purge_image_article_path(@article, image_id: image.id),
              method: :delete,
              class: "absolute top-1 right-1 bg-red-500 text-white rounded-full w-6 h-6" %>
        </div>
      <% end %>
    </div>
  <% end %>

  <%= f.submit "Save" %>
<% end %>
```

### Direct Upload (JavaScript)
```erb
<%= form_with model: @article, data: { controller: "upload" } do |f| %>
  <%= f.file_field :images,
      multiple: true,
      direct_upload: true,
      data: {
        upload_target: "input",
        action: "change->upload#upload"
      } %>

  <div data-upload-target="progress" class="hidden">
    <div class="bg-blue-600 h-2 rounded" data-upload-target="bar" style="width: 0%"></div>
  </div>
<% end %>
```

```javascript
// app/javascript/controllers/upload_controller.js
import { Controller } from "@hotwired/stimulus"
import { DirectUpload } from "@rails/activestorage"

export default class extends Controller {
  static targets = ["input", "progress", "bar"]

  upload() {
    Array.from(this.inputTarget.files).forEach(file => {
      const upload = new DirectUpload(file, this.inputTarget.dataset.directUploadUrl, this)

      upload.create((error, blob) => {
        if (error) {
          console.error(error)
        } else {
          // Append hidden field with signed_id
          const hiddenField = document.createElement("input")
          hiddenField.type = "hidden"
          hiddenField.name = this.inputTarget.name
          hiddenField.value = blob.signed_id
          this.element.appendChild(hiddenField)
        }
      })
    })
  }

  directUploadWillStoreFileWithXHR(request) {
    this.progressTarget.classList.remove("hidden")
    request.upload.addEventListener("progress", event => {
      const progress = (event.loaded / event.total) * 100
      this.barTarget.style.width = `${progress}%`
    })
  }
}
```

## Displaying Attachments

### Images
```erb
<%# Original %>
<%= image_tag @user.avatar %>

<%# With variant %>
<%= image_tag @user.avatar.variant(:thumb) %>

<%# With fallback %>
<% if @user.avatar.attached? %>
  <%= image_tag @user.avatar.variant(:medium), class: "rounded-full" %>
<% else %>
  <%= image_tag "default-avatar.png", class: "rounded-full" %>
<% end %>
```

### Files/Documents
```erb
<% if @document.file.attached? %>
  <div class="flex items-center gap-2">
    <span><%= @document.file.filename %></span>
    <span class="text-gray-500">(<%= number_to_human_size(@document.file.byte_size) %>)</span>
    <%= link_to "Download", rails_blob_path(@document.file, disposition: "attachment"),
        class: "text-blue-600" %>
  </div>
<% end %>
```

### PDF Preview
```erb
<% if @document.file.content_type == "application/pdf" %>
  <iframe src="<%= rails_blob_path(@document.file) %>"
          class="w-full h-96 border rounded"></iframe>
<% end %>
```

## Validations

### Custom Validator
```ruby
# app/validators/content_type_validator.rb
class ContentTypeValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return unless value.attached?

    allowed = Array(options[:in] || options[:with])

    files = value.is_a?(ActiveStorage::Attached::Many) ? value : [value]

    files.each do |file|
      unless allowed.include?(file.content_type)
        record.errors.add(attribute, :invalid_content_type)
      end
    end
  end
end
```

### Usage
```ruby
class Article < ApplicationRecord
  has_many_attached :images

  validates :images, content_type: { in: %w[image/png image/jpeg image/webp] }
  validates :images, size: { less_than: 5.megabytes }
end
```

## Background Processing

### Analyze on Upload
```ruby
# Active Storage analyzes files automatically in background
# Configure in config/application.rb
config.active_storage.queues.analysis = :default
config.active_storage.queues.purge = :default
```

### Custom Processing Job
```ruby
class ProcessImageJob < ApplicationJob
  queue_as :default

  def perform(image_id)
    image = ActiveStorage::Blob.find(image_id)
    # Custom processing
  end
end
```

## Best Practices

1. **Validate content types** - Prevent malicious uploads
2. **Set size limits** - Protect storage and bandwidth
3. **Use variants** - Don't serve original large images
4. **Enable direct uploads** - Better UX for large files
5. **Purge unused files** - Clean up orphaned attachments
