# API Specification Template

## Información General

| Campo | Valor |
|-------|-------|
| Recurso | [Nombre del recurso] |
| Base URL | `/api/v1/[recursos]` |
| Versión | v1 |
| Estado | Diseño / Implementado |

## Autenticación

```
Authorization: Bearer <token>
```

Todos los endpoints requieren autenticación excepto los marcados como públicos.

## Endpoints

### Listar [Recursos]

```
GET /api/v1/recursos
```

**Autenticación:** Requerida

**Parámetros de Query:**

| Nombre | Tipo | Requerido | Default | Descripción |
|--------|------|-----------|---------|-------------|
| `page` | integer | No | 1 | Número de página |
| `per_page` | integer | No | 20 | Elementos por página (max 100) |
| `sort` | string | No | created_at | Campo de ordenamiento |
| `order` | string | No | desc | Dirección (asc/desc) |
| `status` | string | No | - | Filtrar por estado |
| `q` | string | No | - | Búsqueda por texto |

**Response (200 OK):**

```json
{
  "recursos": [
    {
      "id": 1,
      "field_name": "valor",
      "status": "active",
      "created_at": "2024-01-15T10:30:00Z",
      "updated_at": "2024-01-15T10:30:00Z",
      "user": {
        "id": 5,
        "name": "Usuario"
      }
    }
  ],
  "meta": {
    "current_page": 1,
    "total_pages": 5,
    "total_count": 100,
    "per_page": 20
  }
}
```

---

### Obtener [Recurso]

```
GET /api/v1/recursos/:id
```

**Autenticación:** Requerida

**Parámetros de URL:**

| Nombre | Tipo | Descripción |
|--------|------|-------------|
| `id` | integer | ID del recurso |

**Response (200 OK):**

```json
{
  "recurso": {
    "id": 1,
    "field_name": "valor",
    "description": "Descripción completa",
    "status": "active",
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-15T10:30:00Z",
    "user": {
      "id": 5,
      "name": "Usuario",
      "email": "user@example.com"
    },
    "related_items": [
      {
        "id": 10,
        "name": "Item relacionado"
      }
    ]
  }
}
```

**Response (404 Not Found):**

```json
{
  "error": "Recurso no encontrado"
}
```

---

### Crear [Recurso]

```
POST /api/v1/recursos
```

**Autenticación:** Requerida

**Request Body:**

```json
{
  "recurso": {
    "field_name": "valor",
    "description": "Descripción opcional",
    "status": "draft"
  }
}
```

**Campos del Request:**

| Nombre | Tipo | Requerido | Descripción |
|--------|------|-----------|-------------|
| `field_name` | string | Sí | Nombre del campo |
| `description` | string | No | Descripción |
| `status` | string | No | Estado inicial (default: draft) |

**Response (201 Created):**

```json
{
  "recurso": {
    "id": 2,
    "field_name": "valor",
    "description": "Descripción opcional",
    "status": "draft",
    "created_at": "2024-01-15T11:00:00Z",
    "updated_at": "2024-01-15T11:00:00Z"
  }
}
```

**Response (422 Unprocessable Entity):**

```json
{
  "error": "Validation failed",
  "details": [
    "Field name can't be blank",
    "Field name is too short (minimum is 3 characters)"
  ]
}
```

---

### Actualizar [Recurso]

```
PATCH /api/v1/recursos/:id
```

**Autenticación:** Requerida (propietario o admin)

**Request Body:**

```json
{
  "recurso": {
    "field_name": "nuevo valor",
    "status": "published"
  }
}
```

**Response (200 OK):**

```json
{
  "recurso": {
    "id": 1,
    "field_name": "nuevo valor",
    "status": "published",
    "updated_at": "2024-01-15T12:00:00Z"
  }
}
```

**Response (403 Forbidden):**

```json
{
  "error": "No autorizado para modificar este recurso"
}
```

---

### Eliminar [Recurso]

```
DELETE /api/v1/recursos/:id
```

**Autenticación:** Requerida (propietario o admin)

**Response (204 No Content):**

Sin cuerpo de respuesta.

**Response (403 Forbidden):**

```json
{
  "error": "No autorizado para eliminar este recurso"
}
```

---

### Acción Custom: Publicar

```
POST /api/v1/recursos/:id/publish
```

**Autenticación:** Requerida (propietario o admin)

**Response (200 OK):**

```json
{
  "recurso": {
    "id": 1,
    "status": "published",
    "published_at": "2024-01-15T12:30:00Z"
  }
}
```

---

## Códigos de Error

| Código | Descripción | Ejemplo de Response |
|--------|-------------|---------------------|
| 400 | Bad Request - Parámetros malformados | `{"error": "Invalid JSON"}` |
| 401 | Unauthorized - Token inválido o ausente | `{"error": "Token inválido"}` |
| 403 | Forbidden - Sin permisos | `{"error": "No autorizado"}` |
| 404 | Not Found - Recurso no existe | `{"error": "No encontrado"}` |
| 422 | Unprocessable Entity - Validación fallida | `{"error": "...", "details": [...]}` |
| 429 | Too Many Requests - Rate limit | `{"error": "Rate limit", "retry_after": 60}` |
| 500 | Server Error | `{"error": "Error interno"}` |

## Rate Limiting

| Endpoint | Límite | Ventana |
|----------|--------|---------|
| General | 100 requests | 1 minuto |
| Creación | 10 requests | 1 minuto |
| Login | 5 requests | 15 minutos |

Headers de respuesta:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1705319400
```

## Paginación

Todos los endpoints de listado usan paginación basada en offset.

**Parámetros:**
- `page`: Número de página (empezando en 1)
- `per_page`: Elementos por página (default: 20, max: 100)

**Meta de respuesta:**
```json
{
  "meta": {
    "current_page": 2,
    "total_pages": 10,
    "total_count": 195,
    "per_page": 20
  }
}
```

## Filtrado y Ordenamiento

### Filtrado
```
GET /api/v1/recursos?status=active&user_id=5
```

### Ordenamiento
```
GET /api/v1/recursos?sort=created_at&order=desc
```

### Búsqueda
```
GET /api/v1/recursos?q=texto+de+busqueda
```

## Implementación

### Controlador

```ruby
# app/controllers/api/v1/recursos_controller.rb
module Api
  module V1
    class RecursosController < BaseController
      before_action :set_recurso, only: [:show, :update, :destroy, :publish]

      def index
        @recursos = policy_scope(Recurso)
                    .includes(:user)
                    .filter_by(filter_params)
                    .order_by(sort_params)

        @pagy, @recursos = pagy(@recursos, items: per_page)

        render json: {
          recursos: @recursos.map { |r| recurso_json(r) },
          meta: pagination_meta(@pagy)
        }
      end

      def show
        authorize @recurso
        render json: { recurso: recurso_json(@recurso, full: true) }
      end

      def create
        @recurso = current_api_user.recursos.build(recurso_params)
        authorize @recurso

        if @recurso.save
          render json: { recurso: recurso_json(@recurso) }, status: :created
        else
          render_validation_error(@recurso)
        end
      end

      def update
        authorize @recurso

        if @recurso.update(recurso_params)
          render json: { recurso: recurso_json(@recurso) }
        else
          render_validation_error(@recurso)
        end
      end

      def destroy
        authorize @recurso
        @recurso.destroy
        head :no_content
      end

      def publish
        authorize @recurso
        @recurso.publish!
        render json: { recurso: recurso_json(@recurso) }
      end

      private

      def set_recurso
        @recurso = Recurso.find(params[:id])
      end

      def recurso_params
        params.require(:recurso).permit(:field_name, :description, :status)
      end

      def filter_params
        params.permit(:status, :user_id, :q)
      end

      def sort_params
        {
          sort: params[:sort] || 'created_at',
          order: params[:order] || 'desc'
        }
      end

      def per_page
        [params[:per_page]&.to_i || 20, 100].min
      end

      def recurso_json(recurso, full: false)
        # Implementación del serializer
      end
    end
  end
end
```

### Rutas

```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :recursos, only: [:index, :show, :create, :update, :destroy] do
      member do
        post :publish
      end
    end
  end
end
```

## Tests

```ruby
# spec/requests/api/v1/recursos_spec.rb
require "rails_helper"

RSpec.describe "API V1 Recursos", type: :request do
  let(:user) { create(:user) }
  let(:headers) { auth_headers(user) }

  describe "GET /api/v1/recursos" do
    let!(:recursos) { create_list(:recurso, 3, user: user) }

    it "returns paginated list" do
      get "/api/v1/recursos", headers: headers

      expect(response).to have_http_status(:ok)
      expect(json_response["recursos"].size).to eq(3)
      expect(json_response["meta"]).to include("total_count" => 3)
    end

    it "filters by status" do
      create(:recurso, status: "active")
      create(:recurso, status: "draft")

      get "/api/v1/recursos?status=active", headers: headers

      expect(json_response["recursos"].size).to eq(1)
    end
  end

  describe "POST /api/v1/recursos" do
    let(:valid_params) { { recurso: { field_name: "Test" } } }

    it "creates recurso" do
      expect {
        post "/api/v1/recursos", params: valid_params, headers: headers
      }.to change(Recurso, :count).by(1)

      expect(response).to have_http_status(:created)
    end

    it "returns errors for invalid params" do
      post "/api/v1/recursos",
           params: { recurso: { field_name: "" } },
           headers: headers

      expect(response).to have_http_status(:unprocessable_entity)
      expect(json_response["details"]).to include(/blank/)
    end
  end
end
```

## Notas

- [Nota adicional 1]
- [Nota adicional 2]

## Changelog

| Versión | Fecha | Cambios |
|---------|-------|---------|
| v1 | [Fecha] | Versión inicial |
