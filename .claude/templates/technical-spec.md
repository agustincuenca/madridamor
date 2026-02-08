# Technical Specification Template

## Resumen

| Campo | Valor |
|-------|-------|
| Título | [Nombre de la especificación] |
| Autor | [Nombre] |
| Fecha | [Fecha] |
| Estado | Borrador / En revisión / Aprobado |
| Versión | 1.0 |

## Contexto

### Problema
[Descripción del problema que se está resolviendo]

### Objetivos
1. [Objetivo 1]
2. [Objetivo 2]
3. [Objetivo 3]

### Alcance
**Incluido:**
- [Lo que está incluido]

**Excluido:**
- [Lo que NO está incluido]

## Diseño Propuesto

### Vista General

```
┌─────────────────────────────────────────────┐
│           Diagrama de arquitectura           │
│                                             │
│  ┌─────────┐     ┌─────────┐     ┌───────┐ │
│  │Component│ ──▶ │Component│ ──▶ │ Data  │ │
│  │    A    │     │    B    │     │ Store │ │
│  └─────────┘     └─────────┘     └───────┘ │
│                                             │
└─────────────────────────────────────────────┘
```

### Modelos de Datos

#### Nuevo Modelo: `ModelName`

```ruby
# app/models/model_name.rb
class ModelName < ApplicationRecord
  # Associations
  belongs_to :user
  has_many :related_models

  # Validations
  validates :field, presence: true

  # Scopes
  scope :active, -> { where(active: true) }
end
```

**Migración:**
```ruby
class CreateModelNames < ActiveRecord::Migration[8.0]
  def change
    create_table :model_names do |t|
      t.references :user, null: false, foreign_key: true
      t.string :field, null: false
      t.boolean :active, default: true
      t.timestamps
    end

    add_index :model_names, :field
  end
end
```

### API / Endpoints

#### `POST /api/v1/resource`

**Request:**
```json
{
  "resource": {
    "field": "value",
    "other_field": "other_value"
  }
}
```

**Response (201 Created):**
```json
{
  "resource": {
    "id": 1,
    "field": "value",
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

**Errors:**
| Código | Descripción |
|--------|-------------|
| 400 | Parámetros inválidos |
| 401 | No autenticado |
| 403 | No autorizado |
| 422 | Error de validación |

### Flujo de Usuario

```
1. Usuario accede a /feature
   │
2. Sistema muestra formulario
   │
3. Usuario completa y envía
   │
4. Sistema valida datos
   │
   ├── Si válido:
   │   │
   │   5. Guarda en BD
   │   │
   │   6. Envía email (async)
   │   │
   │   7. Redirige a confirmación
   │
   └── Si inválido:
       │
       5. Muestra errores
       │
       6. Usuario corrige
```

### Componentes de UI

#### Nuevo Componente: [Nombre]

```erb
<%# app/views/shared/_component_name.html.erb %>
<%# locals: (param1:, param2: nil) %>
<div class="..." data-controller="component-name">
  <!-- Contenido -->
</div>
```

**Stimulus Controller:**
```javascript
// app/javascript/controllers/component_name_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["element"]
  static values = { config: Object }

  connect() {
    // Inicialización
  }

  action() {
    // Lógica
  }
}
```

## Consideraciones

### Seguridad
- [ ] Autenticación requerida
- [ ] Autorización implementada (Pundit)
- [ ] Validación de inputs
- [ ] Protección CSRF
- [ ] Rate limiting (si aplica)

### Rendimiento
- [ ] Queries optimizadas
- [ ] Índices de BD necesarios
- [ ] Caching implementado (si aplica)
- [ ] Jobs para operaciones lentas

### Accesibilidad
- [ ] Navegable por teclado
- [ ] Labels en formularios
- [ ] ARIA attributes donde sea necesario
- [ ] Contraste adecuado

### Testing

**Tests necesarios:**
```ruby
# spec/models/model_name_spec.rb
RSpec.describe ModelName, type: :model do
  describe "validations" do
    it { should validate_presence_of(:field) }
  end
end

# spec/requests/resource_spec.rb
RSpec.describe "Resource API", type: :request do
  describe "POST /api/v1/resource" do
    it "creates resource" do
      # ...
    end
  end
end
```

## Alternativas Consideradas

### Alternativa 1: [Nombre]
**Descripción:** [Breve descripción]
**Pros:** [Ventajas]
**Cons:** [Desventajas]
**Razón de descarte:** [Por qué no se eligió]

### Alternativa 2: [Nombre]
**Descripción:** [Breve descripción]
**Pros:** [Ventajas]
**Cons:** [Desventajas]
**Razón de descarte:** [Por qué no se eligió]

## Plan de Implementación

### Fase 1: [Nombre]
- [ ] Tarea 1
- [ ] Tarea 2

### Fase 2: [Nombre]
- [ ] Tarea 3
- [ ] Tarea 4

## Riesgos y Mitigación

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|--------------|---------|------------|
| [Riesgo 1] | Alta/Media/Baja | Alto/Medio/Bajo | [Cómo mitigar] |

## Métricas de Éxito

- [ ] [Métrica 1]: [Valor objetivo]
- [ ] [Métrica 2]: [Valor objetivo]

## Referencias

- [Enlace a documentación relevante]
- [Enlace a diseños]
- [Enlace a otras specs relacionadas]

---

## Historial de Revisiones

| Versión | Fecha | Autor | Cambios |
|---------|-------|-------|---------|
| 1.0 | [Fecha] | [Autor] | Versión inicial |
