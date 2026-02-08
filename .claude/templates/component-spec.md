# Component Specification Template

## Información General

| Campo | Valor |
|-------|-------|
| Nombre | [Nombre del componente] |
| Tipo | UI / Funcional / Layout |
| Ubicación | `app/views/shared/_component.html.erb` |
| Estado | Diseño / Desarrollo / Completado |

## Descripción

[Breve descripción del propósito del componente]

## Vista Previa

```
┌─────────────────────────────────────┐
│  [Representación ASCII del diseño]  │
│                                     │
│  ┌───────────────────────────────┐ │
│  │        Contenido               │ │
│  └───────────────────────────────┘ │
│                                     │
│  [ Botón ]                         │
│                                     │
└─────────────────────────────────────┘
```

## Variantes

### Variante Default
[Descripción y cuándo usar]

### Variante [Nombre]
[Descripción y cuándo usar]

### Variante [Nombre]
[Descripción y cuándo usar]

## Props / Parámetros

| Nombre | Tipo | Requerido | Default | Descripción |
|--------|------|-----------|---------|-------------|
| `title` | String | Sí | - | Título principal |
| `description` | String | No | nil | Texto descriptivo |
| `variant` | Symbol | No | :default | Estilo visual |
| `data` | Hash | No | {} | Data attributes adicionales |

## Uso

### Básico
```erb
<%= render "shared/component", title: "Mi Título" %>
```

### Con todas las opciones
```erb
<%= render "shared/component",
           title: "Mi Título",
           description: "Descripción opcional",
           variant: :outlined,
           data: { controller: "my-controller" } %>
```

### Con bloque
```erb
<%= render "shared/component", title: "Mi Título" do %>
  <p>Contenido personalizado</p>
<% end %>
```

## Implementación

### ERB Template
```erb
<%# app/views/shared/_component.html.erb %>
<%# locals: (title:, description: nil, variant: :default, data: {}) %>
<%
  base_classes = "rounded-lg shadow"
  variant_classes = case variant
                    when :outlined then "border-2 border-gray-200"
                    when :filled then "bg-gray-100"
                    else "bg-white"
                    end
  classes = "#{base_classes} #{variant_classes}"
%>

<div class="<%= classes %>"
     <%= tag.attributes(data) %>>
  <h3 class="text-lg font-semibold text-gray-900">
    <%= title %>
  </h3>

  <% if description.present? %>
    <p class="mt-2 text-gray-600">
      <%= description %>
    </p>
  <% end %>

  <% if block_given? %>
    <div class="mt-4">
      <%= yield %>
    </div>
  <% end %>
</div>
```

### Stimulus Controller (si aplica)
```javascript
// app/javascript/controllers/component_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["content"]
  static values = {
    expanded: { type: Boolean, default: false }
  }
  static classes = ["hidden"]

  toggle() {
    this.expandedValue = !this.expandedValue
  }

  expandedValueChanged() {
    this.contentTarget.classList.toggle(this.hiddenClass, !this.expandedValue)
  }
}
```

## Estilos

### Clases de Tailwind utilizadas

| Clase | Propósito |
|-------|-----------|
| `rounded-lg` | Bordes redondeados |
| `shadow` | Sombra sutil |
| `p-4` | Padding interno |
| `text-lg` | Tamaño de texto título |
| `font-semibold` | Peso de fuente título |

### Customización
```erb
<%# Añadir clases adicionales %>
<%= render "shared/component",
           title: "Título",
           class: "my-custom-class" %>
```

## Estados

### Default
[Descripción del estado normal]

### Hover
[Comportamiento al pasar el mouse]

### Focus
[Comportamiento al recibir foco]

### Disabled
[Comportamiento cuando está deshabilitado]

### Loading
[Comportamiento durante carga]

## Accesibilidad

| Aspecto | Implementación |
|---------|---------------|
| Rol ARIA | `role="[rol]"` |
| Labels | `aria-label="[descripción]"` |
| Focus | Orden lógico, visible |
| Keyboard | [Teclas soportadas] |

### Ejemplo accesible
```erb
<div role="region"
     aria-labelledby="component-title-<%= id %>">
  <h3 id="component-title-<%= id %>">
    <%= title %>
  </h3>
</div>
```

## Responsive

| Breakpoint | Comportamiento |
|------------|----------------|
| Mobile (< 640px) | [Descripción] |
| Tablet (640-1024px) | [Descripción] |
| Desktop (> 1024px) | [Descripción] |

## Tests

```ruby
# spec/views/shared/_component.html.erb_spec.rb
require "rails_helper"

RSpec.describe "shared/_component", type: :view do
  it "renders title" do
    render partial: "shared/component", locals: { title: "Test" }
    expect(rendered).to have_text("Test")
  end

  it "renders description when provided" do
    render partial: "shared/component",
           locals: { title: "Test", description: "Desc" }
    expect(rendered).to have_text("Desc")
  end

  it "renders block content" do
    render partial: "shared/component", locals: { title: "Test" } do
      "Custom content"
    end
    expect(rendered).to have_text("Custom content")
  end
end
```

## Ejemplos de Uso Real

### En página de artículos
```erb
<%= render "shared/component",
           title: @article.title,
           description: @article.excerpt %>
```

### En dashboard
```erb
<%= render "shared/component", title: "Estadísticas" do %>
  <%= render "dashboard/stats_chart" %>
<% end %>
```

## Notas de Implementación

- [Nota importante 1]
- [Nota importante 2]

## Changelog

| Versión | Fecha | Cambios |
|---------|-------|---------|
| 1.0 | [Fecha] | Versión inicial |
