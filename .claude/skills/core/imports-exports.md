# Skill: Imports & Exports

## Purpose

Implementar funcionalidades de importación y exportación de datos en formatos CSV, Excel y JSON para aplicaciones Rails.

## CSV Export

### Export simple

```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController
  def index
    @users = User.all

    respond_to do |format|
      format.html
      format.csv { send_data @users.to_csv, filename: "users-#{Date.current}.csv" }
    end
  end
end

# app/models/user.rb
class User < ApplicationRecord
  def self.to_csv
    attributes = %w[id name email created_at]

    CSV.generate(headers: true) do |csv|
      csv << attributes

      all.find_each do |user|
        csv << attributes.map { |attr| user.send(attr) }
      end
    end
  end
end
```

### Export con headers personalizados

```ruby
# app/models/concerns/csv_exportable.rb
module CsvExportable
  extend ActiveSupport::Concern

  class_methods do
    def to_csv(columns: nil, headers: nil)
      columns ||= column_names
      headers ||= columns.map(&:humanize)

      CSV.generate(headers: true) do |csv|
        csv << headers

        find_each do |record|
          csv << columns.map { |col| format_csv_value(record, col) }
        end
      end
    end

    private

    def format_csv_value(record, column)
      value = record.send(column)

      case value
      when Time, DateTime
        value.strftime("%Y-%m-%d %H:%M:%S")
      when Date
        value.strftime("%Y-%m-%d")
      when true
        "Sí"
      when false
        "No"
      when nil
        ""
      else
        value.to_s
      end
    end
  end
end

# app/models/order.rb
class Order < ApplicationRecord
  include CsvExportable
end

# Uso
Order.where(status: "completed").to_csv(
  columns: %w[id user_name total status created_at],
  headers: ["ID", "Cliente", "Total", "Estado", "Fecha"]
)
```

### Export streaming (archivos grandes)

```ruby
# app/controllers/exports_controller.rb
class ExportsController < ApplicationController
  include ActionController::Live

  def orders
    response.headers["Content-Type"] = "text/csv"
    response.headers["Content-Disposition"] = "attachment; filename=orders-#{Date.current}.csv"

    response.stream.write CSV.generate_line(["ID", "Cliente", "Total", "Estado"])

    Order.find_each do |order|
      response.stream.write CSV.generate_line([
        order.id,
        order.user.name,
        order.total,
        order.status
      ])
    end
  ensure
    response.stream.close
  end
end
```

## CSV Import

### Import básico

```ruby
# app/services/csv_importer.rb
class CsvImporter
  def initialize(file, model_class)
    @file = file
    @model_class = model_class
  end

  def import
    results = { success: 0, errors: [] }

    CSV.foreach(@file.path, headers: true) do |row|
      record = @model_class.new(row.to_h)

      if record.save
        results[:success] += 1
      else
        results[:errors] << {
          row: $INPUT_LINE_NUMBER,
          errors: record.errors.full_messages
        }
      end
    end

    results
  end
end

# Uso en controller
class ImportsController < ApplicationController
  def create
    if params[:file].blank?
      redirect_to imports_path, alert: "Selecciona un archivo"
      return
    end

    result = CsvImporter.new(params[:file], User).import

    if result[:errors].empty?
      redirect_to users_path, notice: "#{result[:success]} usuarios importados"
    else
      flash.now[:alert] = "Errores en la importación"
      @errors = result[:errors]
      render :new
    end
  end
end
```

### Import con validación previa

```ruby
# app/services/csv_import_service.rb
class CsvImportService
  Result = Struct.new(:success?, :imported, :errors, keyword_init: true)

  def initialize(file, model_class, column_mapping: {})
    @file = file
    @model_class = model_class
    @column_mapping = column_mapping
  end

  def preview(limit: 5)
    rows = []
    CSV.foreach(@file.path, headers: true).with_index do |row, index|
      break if index >= limit
      rows << transform_row(row)
    end
    rows
  end

  def validate
    errors = []
    line = 2 # Línea 1 son headers

    CSV.foreach(@file.path, headers: true) do |row|
      record = @model_class.new(transform_row(row))
      unless record.valid?
        errors << { line: line, errors: record.errors.full_messages }
      end
      line += 1
    end

    errors
  end

  def import
    imported = 0
    errors = []

    ActiveRecord::Base.transaction do
      CSV.foreach(@file.path, headers: true).with_index(2) do |row, line|
        record = @model_class.new(transform_row(row))

        if record.save
          imported += 1
        else
          errors << { line: line, errors: record.errors.full_messages }
        end
      end

      # Rollback si hay muchos errores
      if errors.size > imported * 0.1 # Más del 10% de errores
        raise ActiveRecord::Rollback
      end
    end

    Result.new(success?: errors.empty?, imported: imported, errors: errors)
  end

  private

  def transform_row(row)
    if @column_mapping.present?
      @column_mapping.transform_values { |csv_col| row[csv_col] }
    else
      row.to_h
    end
  end
end

# Uso
service = CsvImportService.new(
  params[:file],
  Product,
  column_mapping: {
    name: "Nombre",
    price: "Precio",
    sku: "Código"
  }
)

# Preview
@preview = service.preview(limit: 10)

# Validar antes de importar
@validation_errors = service.validate
return render :new if @validation_errors.any?

# Importar
result = service.import
```

### Import con job en background

```ruby
# app/jobs/import_job.rb
class ImportJob < ApplicationJob
  queue_as :imports

  def perform(import_id)
    import = Import.find(import_id)
    import.update!(status: "processing")

    file_path = ActiveStorage::Blob.service.path_for(import.file.key)
    result = CsvImportService.new(
      File.open(file_path),
      import.model_class.constantize
    ).import

    import.update!(
      status: result.success? ? "completed" : "completed_with_errors",
      imported_count: result.imported,
      error_details: result.errors
    )

    # Notificar al usuario
    ImportMailer.completed(import).deliver_later
  rescue StandardError => e
    import.update!(status: "failed", error_details: [{ error: e.message }])
    raise
  end
end

# app/models/import.rb
class Import < ApplicationRecord
  belongs_to :user
  has_one_attached :file

  enum :status, {
    pending: "pending",
    processing: "processing",
    completed: "completed",
    completed_with_errors: "completed_with_errors",
    failed: "failed"
  }

  validates :file, presence: true
  validate :file_format

  after_create_commit :process_later

  private

  def file_format
    return unless file.attached?
    unless file.content_type.in?(%w[text/csv application/vnd.ms-excel])
      errors.add(:file, "debe ser un archivo CSV")
    end
  end

  def process_later
    ImportJob.perform_later(id)
  end
end
```

## Excel Export (XLSX)

### Setup

```ruby
# Gemfile
gem "caxlsx"
gem "caxlsx_rails"
```

### Export básico

```ruby
# app/views/orders/index.xlsx.axlsx
wb = xlsx_package.workbook

wb.add_worksheet(name: "Pedidos") do |sheet|
  # Estilos
  header_style = sheet.styles.add_style(
    bg_color: "4472C4",
    fg_color: "FFFFFF",
    b: true,
    alignment: { horizontal: :center }
  )

  money_style = sheet.styles.add_style(
    num_fmt: "€#,##0.00",
    alignment: { horizontal: :right }
  )

  date_style = sheet.styles.add_style(
    num_fmt: "dd/mm/yyyy",
    alignment: { horizontal: :center }
  )

  # Headers
  sheet.add_row(
    ["ID", "Cliente", "Email", "Total", "Estado", "Fecha"],
    style: header_style
  )

  # Datos
  @orders.each do |order|
    sheet.add_row(
      [
        order.id,
        order.user.name,
        order.user.email,
        order.total,
        order.status.humanize,
        order.created_at
      ],
      style: [nil, nil, nil, money_style, nil, date_style]
    )
  end

  # Auto-ajustar columnas
  sheet.column_widths 10, 25, 30, 15, 15, 15
end

# Controller
class OrdersController < ApplicationController
  def index
    @orders = Order.includes(:user).recent

    respond_to do |format|
      format.html
      format.xlsx {
        response.headers["Content-Disposition"] = "attachment; filename=pedidos-#{Date.current}.xlsx"
      }
    end
  end
end
```

### Export con múltiples hojas

```ruby
# app/views/reports/monthly.xlsx.axlsx
wb = xlsx_package.workbook

# Hoja de resumen
wb.add_worksheet(name: "Resumen") do |sheet|
  sheet.add_row ["Reporte Mensual - #{@month.strftime('%B %Y')}"]
  sheet.add_row []
  sheet.add_row ["Total Pedidos", @stats[:total_orders]]
  sheet.add_row ["Ingresos", @stats[:revenue]]
  sheet.add_row ["Ticket Medio", @stats[:average_ticket]]
end

# Hoja de pedidos
wb.add_worksheet(name: "Pedidos") do |sheet|
  sheet.add_row ["ID", "Cliente", "Total", "Fecha"]
  @orders.each do |order|
    sheet.add_row [order.id, order.user.name, order.total, order.created_at]
  end
end

# Hoja de productos más vendidos
wb.add_worksheet(name: "Top Productos") do |sheet|
  sheet.add_row ["Producto", "Unidades", "Ingresos"]
  @top_products.each do |product|
    sheet.add_row [product.name, product.units_sold, product.revenue]
  end
end
```

## Excel Import

### Con roo gem

```ruby
# Gemfile
gem "roo"
```

```ruby
# app/services/excel_import_service.rb
class ExcelImportService
  def initialize(file)
    @spreadsheet = open_spreadsheet(file)
  end

  def import(model_class, sheet: 0, header_row: 1)
    sheet = @spreadsheet.sheet(sheet)
    headers = sheet.row(header_row).map(&:to_s).map(&:strip)

    results = { success: 0, errors: [] }

    ((header_row + 1)..sheet.last_row).each do |i|
      row = Hash[headers.zip(sheet.row(i))]

      record = model_class.new(row)
      if record.save
        results[:success] += 1
      else
        results[:errors] << { row: i, errors: record.errors.full_messages }
      end
    end

    results
  end

  def preview(sheet: 0, header_row: 1, limit: 5)
    sheet = @spreadsheet.sheet(sheet)
    headers = sheet.row(header_row)

    rows = []
    ((header_row + 1)..[header_row + limit, sheet.last_row].min).each do |i|
      rows << Hash[headers.zip(sheet.row(i))]
    end

    { headers: headers, rows: rows }
  end

  def sheets
    @spreadsheet.sheets
  end

  private

  def open_spreadsheet(file)
    case File.extname(file.original_filename)
    when ".csv"
      Roo::CSV.new(file.path)
    when ".xls"
      Roo::Excel.new(file.path)
    when ".xlsx"
      Roo::Excelx.new(file.path)
    else
      raise "Formato no soportado: #{file.original_filename}"
    end
  end
end
```

## JSON Export/Import

### Export

```ruby
# app/controllers/api/exports_controller.rb
module Api
  class ExportsController < ApplicationController
    def users
      @users = User.includes(:posts, :comments)

      send_data(
        @users.to_json(include: [:posts, :comments]),
        filename: "users-export-#{Time.current.to_i}.json",
        type: :json
      )
    end
  end
end

# Export personalizado con JBuilder
# app/views/api/exports/users.json.jbuilder
json.exported_at Time.current.iso8601
json.count @users.size

json.users @users do |user|
  json.extract! user, :id, :name, :email, :created_at
  json.posts_count user.posts.size

  json.posts user.posts do |post|
    json.extract! post, :id, :title, :published_at
  end
end
```

### Import

```ruby
# app/services/json_import_service.rb
class JsonImportService
  def initialize(file)
    @data = JSON.parse(file.read)
  end

  def import_users
    results = { success: 0, errors: [] }

    @data["users"].each_with_index do |user_data, index|
      user = User.new(user_data.except("posts"))

      if user.save
        import_posts(user, user_data["posts"]) if user_data["posts"]
        results[:success] += 1
      else
        results[:errors] << { index: index, errors: user.errors.full_messages }
      end
    end

    results
  end

  private

  def import_posts(user, posts_data)
    posts_data.each do |post_data|
      user.posts.create!(post_data)
    end
  end
end
```

## Vista de importación con Stimulus

```erb
<%# app/views/imports/new.html.erb %>
<div data-controller="import"
     data-import-preview-url-value="<%= preview_imports_path %>">

  <h1>Importar datos</h1>

  <%= form_with url: imports_path, method: :post, local: true, multipart: true do |f| %>
    <div class="mb-4">
      <label class="block text-sm font-medium text-gray-700">
        Archivo CSV o Excel
      </label>
      <%= f.file_field :file,
                       accept: ".csv,.xlsx,.xls",
                       class: "mt-1 block w-full",
                       data: { action: "change->import#preview" } %>
    </div>

    <%# Preview %>
    <div data-import-target="preview" class="hidden mb-4">
      <h3 class="font-medium mb-2">Vista previa</h3>
      <div class="overflow-x-auto">
        <table class="min-w-full divide-y divide-gray-200">
          <thead data-import-target="headers"></thead>
          <tbody data-import-target="rows"></tbody>
        </table>
      </div>
    </div>

    <%# Errores de validación %>
    <div data-import-target="errors" class="hidden mb-4 bg-red-50 p-4 rounded">
      <h3 class="font-medium text-red-800 mb-2">Errores encontrados</h3>
      <ul data-import-target="errorList" class="list-disc pl-5 text-red-700"></ul>
    </div>

    <div class="flex gap-4">
      <%= f.submit "Importar",
                   class: "btn btn-primary",
                   data: { import_target: "submit" } %>

      <button type="button"
              class="btn btn-secondary"
              data-action="import#validate"
              data-import-target="validateBtn">
        Validar primero
      </button>
    </div>
  <% end %>
</div>
```

```javascript
// app/javascript/controllers/import_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["preview", "headers", "rows", "errors", "errorList", "submit", "validateBtn"]
  static values = { previewUrl: String }

  async preview(event) {
    const file = event.target.files[0]
    if (!file) return

    const formData = new FormData()
    formData.append("file", file)

    const response = await fetch(this.previewUrlValue, {
      method: "POST",
      body: formData,
      headers: {
        "X-CSRF-Token": document.querySelector("[name='csrf-token']").content
      }
    })

    const data = await response.json()
    this.showPreview(data)
  }

  showPreview(data) {
    // Headers
    this.headersTarget.innerHTML = `
      <tr>
        ${data.headers.map(h => `<th class="px-4 py-2 bg-gray-50">${h}</th>`).join("")}
      </tr>
    `

    // Rows
    this.rowsTarget.innerHTML = data.rows.map(row => `
      <tr>
        ${data.headers.map(h => `<td class="px-4 py-2 border-t">${row[h] || ""}</td>`).join("")}
      </tr>
    `).join("")

    this.previewTarget.classList.remove("hidden")
  }

  async validate() {
    // Implementar validación AJAX
  }
}
```

## Export en background con notificación

```ruby
# app/jobs/export_job.rb
class ExportJob < ApplicationJob
  queue_as :exports

  def perform(export_id)
    export = Export.find(export_id)
    export.update!(status: "processing")

    # Generar archivo
    data = generate_export(export)

    # Guardar archivo
    export.file.attach(
      io: StringIO.new(data),
      filename: "#{export.export_type}-#{Date.current}.csv",
      content_type: "text/csv"
    )

    export.update!(status: "completed")

    # Notificar vía Turbo Stream
    Turbo::StreamsChannel.broadcast_update_to(
      "exports_#{export.user_id}",
      target: "export_#{export.id}",
      partial: "exports/export",
      locals: { export: export }
    )

    # También por email
    ExportMailer.ready(export).deliver_later
  end

  private

  def generate_export(export)
    case export.export_type
    when "users"
      User.to_csv
    when "orders"
      Order.where(created_at: export.date_range).to_csv
    end
  end
end
```

## Checklist

### Export
- [ ] Streaming para archivos grandes
- [ ] Formato de fechas consistente
- [ ] Encoding UTF-8 con BOM para Excel
- [ ] Headers descriptivos
- [ ] Nombre de archivo con fecha

### Import
- [ ] Validación de formato de archivo
- [ ] Preview antes de importar
- [ ] Validación de datos antes de guardar
- [ ] Manejo de errores por fila
- [ ] Transacción para rollback
- [ ] Background job para archivos grandes
- [ ] Notificación al completar
- [ ] Log de importaciones
