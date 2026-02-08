# Model Specification Template

## Información General

| Campo | Valor |
|-------|-------|
| Nombre | [NombreModelo] |
| Tabla | [nombre_modelos] |
| Archivo | `app/models/nombre_modelo.rb` |
| Estado | Diseño / Implementado |

## Descripción

[Descripción del propósito del modelo y su rol en el dominio]

## Diagrama de Relaciones

```
                    ┌─────────────┐
                    │    User     │
                    └──────┬──────┘
                           │
                           │ has_many
                           ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Category  │◄────│   Article   │────►│   Comment   │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           │ has_many_attached
                           ▼
                    ┌─────────────┐
                    │   Images    │
                    └─────────────┘
```

## Atributos

| Nombre | Tipo | Null | Default | Descripción |
|--------|------|------|---------|-------------|
| `id` | bigint | No | auto | Primary key |
| `field_name` | string | No | - | Descripción del campo |
| `description` | text | Sí | nil | Descripción larga |
| `status` | string | No | 'draft' | Estado del registro |
| `published` | boolean | No | false | Si está publicado |
| `amount` | decimal | Sí | nil | Valor monetario |
| `user_id` | bigint | No | - | FK a users |
| `created_at` | datetime | No | auto | Fecha creación |
| `updated_at` | datetime | No | auto | Fecha actualización |

## Migración

```ruby
class CreateModelNames < ActiveRecord::Migration[8.0]
  def change
    create_table :model_names do |t|
      t.string :field_name, null: false
      t.text :description
      t.string :status, null: false, default: 'draft'
      t.boolean :published, null: false, default: false
      t.decimal :amount, precision: 10, scale: 2
      t.references :user, null: false, foreign_key: true

      t.timestamps
    end

    add_index :model_names, :status
    add_index :model_names, :published
    add_index :model_names, [:user_id, :status]
  end
end
```

## Asociaciones

| Tipo | Modelo | Opciones | Descripción |
|------|--------|----------|-------------|
| `belongs_to` | `:user` | - | Usuario propietario |
| `has_many` | `:comments` | `dependent: :destroy` | Comentarios |
| `has_many` | `:tags` | `through: :taggings` | Tags asociadas |
| `has_one_attached` | `:image` | - | Imagen principal |
| `has_many_attached` | `:documents` | - | Documentos adjuntos |

## Validaciones

| Campo | Validación | Opciones | Mensaje |
|-------|------------|----------|---------|
| `field_name` | `presence` | - | "no puede estar vacío" |
| `field_name` | `length` | `maximum: 255` | "máximo 255 caracteres" |
| `field_name` | `uniqueness` | `scope: :user_id` | "ya existe" |
| `status` | `inclusion` | `in: STATUSES` | "estado inválido" |
| `amount` | `numericality` | `greater_than: 0` | "debe ser mayor a 0" |

## Callbacks

| Tipo | Método | Descripción |
|------|--------|-------------|
| `before_validation` | `:normalize_data` | Limpia y normaliza datos |
| `before_save` | `:generate_slug` | Genera slug del título |
| `after_create` | `:send_notification` | Envía notificación |
| `after_commit` | `:update_search_index` | Actualiza índice (on create/update) |

## Scopes

| Nombre | Query | Descripción |
|--------|-------|-------------|
| `published` | `where(published: true)` | Solo publicados |
| `draft` | `where(published: false)` | Solo borradores |
| `recent` | `order(created_at: :desc)` | Ordenados por fecha |
| `by_status` | `where(status: status)` | Filtrar por status |
| `search` | `where("title LIKE ?", "%#{q}%")` | Búsqueda por título |

## Métodos de Clase

### `self.search(query)`
```ruby
def self.search(query)
  return all if query.blank?
  where("field_name ILIKE ?", "%#{query}%")
end
```

### `self.statistics`
```ruby
def self.statistics
  {
    total: count,
    published: published.count,
    draft: draft.count
  }
end
```

## Métodos de Instancia

### `#publish!`
```ruby
def publish!
  update!(published: true, published_at: Time.current)
end
```
Publica el registro y guarda la fecha.

### `#owned_by?(user)`
```ruby
def owned_by?(user)
  self.user == user
end
```
Verifica si el usuario es propietario.

### `#status_label`
```ruby
def status_label
  I18n.t("model_name.status.#{status}")
end
```
Devuelve etiqueta traducida del status.

## Enums / Constantes

```ruby
STATUSES = %w[draft pending published archived].freeze

# O usando enum de Rails (si aplica)
enum :status, {
  draft: 'draft',
  pending: 'pending',
  published: 'published',
  archived: 'archived'
}, default: :draft
```

## Implementación Completa

```ruby
# app/models/model_name.rb
class ModelName < ApplicationRecord
  # Constants
  STATUSES = %w[draft pending published archived].freeze

  # Associations
  belongs_to :user
  has_many :comments, dependent: :destroy
  has_many :taggings, dependent: :destroy
  has_many :tags, through: :taggings
  has_one_attached :image

  # Validations
  validates :field_name, presence: true,
                         length: { maximum: 255 },
                         uniqueness: { scope: :user_id }
  validates :status, inclusion: { in: STATUSES }
  validates :amount, numericality: { greater_than: 0 }, allow_nil: true

  # Callbacks
  before_validation :normalize_data
  before_save :generate_slug

  # Scopes
  scope :published, -> { where(published: true) }
  scope :draft, -> { where(published: false) }
  scope :recent, -> { order(created_at: :desc) }
  scope :by_status, ->(status) { where(status: status) }

  # Class methods
  def self.search(query)
    return all if query.blank?
    where("field_name ILIKE ?", "%#{query}%")
  end

  # Instance methods
  def publish!
    update!(published: true, published_at: Time.current)
  end

  def owned_by?(user)
    self.user == user
  end

  private

  def normalize_data
    self.field_name = field_name&.strip
  end

  def generate_slug
    self.slug = field_name.parameterize if slug.blank?
  end
end
```

## Factory

```ruby
# spec/factories/model_names.rb
FactoryBot.define do
  factory :model_name do
    user
    field_name { Faker::Lorem.sentence(word_count: 3) }
    description { Faker::Lorem.paragraph }
    status { ModelName::STATUSES.sample }
    published { false }

    trait :published do
      published { true }
      published_at { Time.current }
    end

    trait :with_image do
      after(:build) do |record|
        record.image.attach(
          io: File.open(Rails.root.join("spec/fixtures/files/test.jpg")),
          filename: "test.jpg"
        )
      end
    end
  end
end
```

## Tests

```ruby
# spec/models/model_name_spec.rb
require "rails_helper"

RSpec.describe ModelName, type: :model do
  describe "validations" do
    it { should validate_presence_of(:field_name) }
    it { should validate_length_of(:field_name).is_at_most(255) }
    it { should validate_inclusion_of(:status).in_array(ModelName::STATUSES) }
  end

  describe "associations" do
    it { should belong_to(:user) }
    it { should have_many(:comments).dependent(:destroy) }
  end

  describe "scopes" do
    describe ".published" do
      it "returns only published records" do
        published = create(:model_name, :published)
        draft = create(:model_name, published: false)

        expect(ModelName.published).to include(published)
        expect(ModelName.published).not_to include(draft)
      end
    end
  end

  describe "#publish!" do
    it "marks record as published" do
      record = create(:model_name, published: false)

      record.publish!

      expect(record.reload).to be_published
      expect(record.published_at).to be_present
    end
  end
end
```

## Consideraciones

### Seguridad
- [ ] Autorización implementada en controlador/policy
- [ ] Strong parameters definidos
- [ ] Validaciones de usuario

### Rendimiento
- [ ] Índices en columnas frecuentemente consultadas
- [ ] Eager loading configurado donde sea necesario
- [ ] Counter caches si hay conteos frecuentes

### Notas
- [Nota adicional 1]
- [Nota adicional 2]
