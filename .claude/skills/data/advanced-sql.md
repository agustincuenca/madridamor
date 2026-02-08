# Skill: SQL Avanzado para Rails

## Purpose

Dominar tecnicas avanzadas de SQL para optimizar consultas, crear reportes complejos y manejar grandes volumenes de datos en aplicaciones Rails.

## CTEs (Common Table Expressions)

### Basico

```ruby
# SQL con WITH clause
class Report < ApplicationRecord
  def self.sales_by_category
    sql = <<-SQL
      WITH category_sales AS (
        SELECT
          categories.name AS category_name,
          SUM(order_items.quantity * order_items.unit_price) AS total_sales,
          COUNT(DISTINCT orders.id) AS order_count
        FROM order_items
        JOIN products ON products.id = order_items.product_id
        JOIN categories ON categories.id = products.category_id
        JOIN orders ON orders.id = order_items.order_id
        WHERE orders.created_at >= :start_date
        GROUP BY categories.id, categories.name
      )
      SELECT
        category_name,
        total_sales,
        order_count,
        total_sales / order_count AS avg_order_value
      FROM category_sales
      ORDER BY total_sales DESC
    SQL

    find_by_sql([sql, { start_date: 30.days.ago }])
  end
end
```

### CTEs multiples

```ruby
def self.customer_lifetime_analysis
  sql = <<-SQL
    WITH first_orders AS (
      SELECT
        user_id,
        MIN(created_at) AS first_order_date
      FROM orders
      GROUP BY user_id
    ),
    customer_totals AS (
      SELECT
        user_id,
        COUNT(*) AS total_orders,
        SUM(total_amount) AS lifetime_value,
        MAX(created_at) AS last_order_date
      FROM orders
      GROUP BY user_id
    ),
    customer_metrics AS (
      SELECT
        ct.user_id,
        ct.total_orders,
        ct.lifetime_value,
        fo.first_order_date,
        ct.last_order_date,
        EXTRACT(EPOCH FROM (ct.last_order_date - fo.first_order_date)) / 86400 AS customer_age_days
      FROM customer_totals ct
      JOIN first_orders fo ON fo.user_id = ct.user_id
    )
    SELECT
      u.id,
      u.email,
      cm.total_orders,
      cm.lifetime_value,
      cm.first_order_date,
      cm.last_order_date,
      cm.customer_age_days,
      CASE
        WHEN cm.total_orders >= 10 THEN 'vip'
        WHEN cm.total_orders >= 5 THEN 'loyal'
        WHEN cm.total_orders >= 2 THEN 'returning'
        ELSE 'new'
      END AS customer_tier
    FROM users u
    JOIN customer_metrics cm ON cm.user_id = u.id
    ORDER BY cm.lifetime_value DESC
  SQL

  find_by_sql(sql)
end
```

### CTEs recursivos

```ruby
# Arbol de categorias
def self.category_tree(root_id = nil)
  sql = <<-SQL
    WITH RECURSIVE category_tree AS (
      -- Caso base: categorias raiz
      SELECT
        id,
        name,
        parent_id,
        1 AS level,
        ARRAY[id] AS path,
        name AS full_path
      FROM categories
      WHERE parent_id #{root_id ? '= :root_id' : 'IS NULL'}

      UNION ALL

      -- Caso recursivo: hijos
      SELECT
        c.id,
        c.name,
        c.parent_id,
        ct.level + 1,
        ct.path || c.id,
        ct.full_path || ' > ' || c.name
      FROM categories c
      JOIN category_tree ct ON c.parent_id = ct.id
      WHERE ct.level < 10  -- Limite de profundidad
    )
    SELECT * FROM category_tree
    ORDER BY path
  SQL

  find_by_sql([sql, { root_id: root_id }])
end

# Estructura organizacional
def self.org_chart(manager_id)
  sql = <<-SQL
    WITH RECURSIVE subordinates AS (
      SELECT
        id,
        name,
        manager_id,
        1 AS depth,
        ARRAY[name] AS reporting_chain
      FROM employees
      WHERE manager_id = :manager_id

      UNION ALL

      SELECT
        e.id,
        e.name,
        e.manager_id,
        s.depth + 1,
        s.reporting_chain || e.name
      FROM employees e
      JOIN subordinates s ON e.manager_id = s.id
      WHERE s.depth < 20
    )
    SELECT
      id,
      name,
      depth,
      array_to_string(reporting_chain, ' -> ') AS chain
    FROM subordinates
    ORDER BY depth, name
  SQL

  find_by_sql([sql, { manager_id: manager_id }])
end
```

## Window Functions

### ROW_NUMBER, RANK, DENSE_RANK

```ruby
# Ranking de productos por ventas
def self.products_ranked_by_sales
  sql = <<-SQL
    SELECT
      p.id,
      p.name,
      c.name AS category,
      SUM(oi.quantity) AS units_sold,
      SUM(oi.quantity * oi.unit_price) AS revenue,
      ROW_NUMBER() OVER (ORDER BY SUM(oi.quantity * oi.unit_price) DESC) AS overall_rank,
      RANK() OVER (PARTITION BY c.id ORDER BY SUM(oi.quantity * oi.unit_price) DESC) AS category_rank,
      DENSE_RANK() OVER (ORDER BY SUM(oi.quantity) DESC) AS units_rank
    FROM products p
    JOIN categories c ON c.id = p.category_id
    JOIN order_items oi ON oi.product_id = p.id
    JOIN orders o ON o.id = oi.order_id
    WHERE o.created_at >= :start_date
    GROUP BY p.id, p.name, c.id, c.name
    ORDER BY revenue DESC
  SQL

  find_by_sql([sql, { start_date: 30.days.ago }])
end
```

### LAG y LEAD

```ruby
# Comparacion mes a mes
def self.monthly_comparison
  sql = <<-SQL
    WITH monthly_sales AS (
      SELECT
        DATE_TRUNC('month', created_at) AS month,
        SUM(total_amount) AS revenue,
        COUNT(*) AS order_count
      FROM orders
      WHERE created_at >= :start_date
      GROUP BY DATE_TRUNC('month', created_at)
    )
    SELECT
      month,
      revenue,
      order_count,
      LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
      LEAD(revenue) OVER (ORDER BY month) AS next_month_revenue,
      revenue - LAG(revenue) OVER (ORDER BY month) AS revenue_change,
      ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month))::numeric /
        NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100,
        2
      ) AS revenue_change_pct
    FROM monthly_sales
    ORDER BY month
  SQL

  find_by_sql([sql, { start_date: 1.year.ago }])
end
```

### SUM OVER (Running totals)

```ruby
# Totales acumulados
def self.cumulative_sales
  sql = <<-SQL
    SELECT
      DATE(created_at) AS sale_date,
      SUM(total_amount) AS daily_revenue,
      SUM(SUM(total_amount)) OVER (
        ORDER BY DATE(created_at)
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
      ) AS cumulative_revenue,
      AVG(SUM(total_amount)) OVER (
        ORDER BY DATE(created_at)
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
      ) AS rolling_7day_avg
    FROM orders
    WHERE created_at >= :start_date
    GROUP BY DATE(created_at)
    ORDER BY sale_date
  SQL

  find_by_sql([sql, { start_date: 30.days.ago }])
end
```

### NTILE y percentiles

```ruby
# Segmentacion de clientes por valor
def self.customer_segments
  sql = <<-SQL
    WITH customer_values AS (
      SELECT
        user_id,
        SUM(total_amount) AS total_spent,
        COUNT(*) AS order_count
      FROM orders
      WHERE created_at >= :start_date
      GROUP BY user_id
    )
    SELECT
      u.id,
      u.email,
      cv.total_spent,
      cv.order_count,
      NTILE(4) OVER (ORDER BY cv.total_spent) AS spending_quartile,
      PERCENT_RANK() OVER (ORDER BY cv.total_spent) AS percentile_rank,
      CASE NTILE(4) OVER (ORDER BY cv.total_spent)
        WHEN 4 THEN 'Premium'
        WHEN 3 THEN 'High Value'
        WHEN 2 THEN 'Medium Value'
        ELSE 'Low Value'
      END AS segment
    FROM users u
    JOIN customer_values cv ON cv.user_id = u.id
    ORDER BY cv.total_spent DESC
  SQL

  find_by_sql([sql, { start_date: 1.year.ago }])
end
```

## Vistas

### Crear vista en migracion

```ruby
# db/migrate/xxx_create_order_summary_view.rb
class CreateOrderSummaryView < ActiveRecord::Migration[8.0]
  def up
    execute <<-SQL
      CREATE VIEW order_summaries AS
      SELECT
        o.id AS order_id,
        o.user_id,
        u.email AS user_email,
        o.created_at AS order_date,
        o.status,
        COUNT(oi.id) AS item_count,
        SUM(oi.quantity) AS total_quantity,
        SUM(oi.quantity * oi.unit_price) AS subtotal,
        o.shipping_cost,
        o.tax_amount,
        o.total_amount
      FROM orders o
      JOIN users u ON u.id = o.user_id
      LEFT JOIN order_items oi ON oi.order_id = o.id
      GROUP BY o.id, u.id
    SQL
  end

  def down
    execute "DROP VIEW IF EXISTS order_summaries"
  end
end

# app/models/order_summary.rb
class OrderSummary < ApplicationRecord
  self.primary_key = :order_id

  # Solo lectura
  def readonly?
    true
  end

  belongs_to :user
  belongs_to :order, foreign_key: :order_id

  scope :recent, -> { where("order_date >= ?", 30.days.ago) }
  scope :by_status, ->(status) { where(status: status) }
end
```

## Vistas Materializadas

### Con Scenic gem

```ruby
# Gemfile
gem "scenic"

# Generar vista materializada
# rails generate scenic:view product_sales_stats --materialized

# db/views/product_sales_stats_v01.sql
SELECT
  p.id AS product_id,
  p.name AS product_name,
  p.sku,
  c.name AS category_name,
  COUNT(DISTINCT o.id) AS order_count,
  SUM(oi.quantity) AS units_sold,
  SUM(oi.quantity * oi.unit_price) AS total_revenue,
  AVG(oi.unit_price) AS avg_selling_price,
  MIN(o.created_at) AS first_sale_date,
  MAX(o.created_at) AS last_sale_date
FROM products p
JOIN categories c ON c.id = p.category_id
LEFT JOIN order_items oi ON oi.product_id = p.id
LEFT JOIN orders o ON o.id = oi.order_id AND o.status = 'completed'
GROUP BY p.id, p.name, p.sku, c.name

# db/migrate/xxx_create_product_sales_stats.rb
class CreateProductSalesStats < ActiveRecord::Migration[8.0]
  def change
    create_view :product_sales_stats, materialized: true
  end
end
```

### Indice para vista materializada

```ruby
# db/migrate/xxx_add_index_to_product_sales_stats.rb
class AddIndexToProductSalesStats < ActiveRecord::Migration[8.0]
  def change
    # Indice unico requerido para REFRESH CONCURRENTLY
    add_index :product_sales_stats, :product_id, unique: true

    # Indices adicionales para queries comunes
    add_index :product_sales_stats, :total_revenue
    add_index :product_sales_stats, :units_sold
    add_index :product_sales_stats, :category_name
  end
end
```

### Refrescar vista materializada

```ruby
# app/models/product_sales_stat.rb
class ProductSalesStat < ApplicationRecord
  self.primary_key = :product_id

  def readonly?
    true
  end

  def self.refresh
    Scenic.database.refresh_materialized_view(
      table_name,
      concurrently: true,
      cascade: false
    )
  end
end

# app/jobs/refresh_materialized_views_job.rb
class RefreshMaterializedViewsJob < ApplicationJob
  queue_as :low

  def perform
    ProductSalesStat.refresh
    CustomerMetric.refresh
    # Otras vistas...
  end
end

# Programar en recurring.yml
# refresh_views:
#   class: RefreshMaterializedViewsJob
#   schedule: every 1 hour
```

## Funciones y Procedimientos

### Crear funcion

```ruby
# db/migrate/xxx_create_calculate_discount_function.rb
class CreateCalculateDiscountFunction < ActiveRecord::Migration[8.0]
  def up
    execute <<-SQL
      CREATE OR REPLACE FUNCTION calculate_discount(
        subtotal NUMERIC,
        customer_tier VARCHAR,
        promo_code VARCHAR DEFAULT NULL
      )
      RETURNS NUMERIC AS $$
      DECLARE
        discount NUMERIC := 0;
        tier_discount NUMERIC := 0;
        promo_discount NUMERIC := 0;
      BEGIN
        -- Descuento por tier
        tier_discount := CASE customer_tier
          WHEN 'vip' THEN subtotal * 0.15
          WHEN 'premium' THEN subtotal * 0.10
          WHEN 'loyal' THEN subtotal * 0.05
          ELSE 0
        END;

        -- Descuento por promocion
        IF promo_code IS NOT NULL THEN
          SELECT COALESCE(
            CASE p.discount_type
              WHEN 'percentage' THEN subtotal * p.discount_value / 100
              WHEN 'fixed' THEN p.discount_value
            END,
            0
          )
          INTO promo_discount
          FROM promotions p
          WHERE p.code = promo_code
            AND p.active = true
            AND p.valid_until >= CURRENT_DATE;
        END IF;

        -- Usar el mayor descuento
        discount := GREATEST(tier_discount, promo_discount);

        -- Maximo 50% de descuento
        RETURN LEAST(discount, subtotal * 0.5);
      END;
      $$ LANGUAGE plpgsql STABLE;
    SQL
  end

  def down
    execute "DROP FUNCTION IF EXISTS calculate_discount"
  end
end

# Uso en Rails
Order.select(
  "*, calculate_discount(subtotal, customer_tier, promo_code) AS discount"
)
```

### Triggers

```ruby
# db/migrate/xxx_create_audit_trigger.rb
class CreateAuditTrigger < ActiveRecord::Migration[8.0]
  def up
    # Tabla de auditoria
    create_table :audit_logs do |t|
      t.string :table_name, null: false
      t.bigint :record_id, null: false
      t.string :action, null: false
      t.jsonb :old_data
      t.jsonb :new_data
      t.bigint :user_id
      t.timestamps
    end

    add_index :audit_logs, [:table_name, :record_id]
    add_index :audit_logs, :user_id

    # Funcion de auditoria
    execute <<-SQL
      CREATE OR REPLACE FUNCTION audit_trigger_function()
      RETURNS TRIGGER AS $$
      BEGIN
        IF TG_OP = 'INSERT' THEN
          INSERT INTO audit_logs (table_name, record_id, action, new_data, created_at, updated_at)
          VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', to_jsonb(NEW), NOW(), NOW());
          RETURN NEW;
        ELSIF TG_OP = 'UPDATE' THEN
          INSERT INTO audit_logs (table_name, record_id, action, old_data, new_data, created_at, updated_at)
          VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), NOW(), NOW());
          RETURN NEW;
        ELSIF TG_OP = 'DELETE' THEN
          INSERT INTO audit_logs (table_name, record_id, action, old_data, created_at, updated_at)
          VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', to_jsonb(OLD), NOW(), NOW());
          RETURN OLD;
        END IF;
        RETURN NULL;
      END;
      $$ LANGUAGE plpgsql;
    SQL

    # Aplicar trigger a tablas sensibles
    %w[orders payments users].each do |table|
      execute <<-SQL
        CREATE TRIGGER #{table}_audit_trigger
        AFTER INSERT OR UPDATE OR DELETE ON #{table}
        FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();
      SQL
    end
  end

  def down
    %w[orders payments users].each do |table|
      execute "DROP TRIGGER IF EXISTS #{table}_audit_trigger ON #{table}"
    end
    execute "DROP FUNCTION IF EXISTS audit_trigger_function"
    drop_table :audit_logs
  end
end
```

## Indices Avanzados

### Indices parciales

```ruby
# db/migrate/xxx_add_partial_indexes.rb
class AddPartialIndexes < ActiveRecord::Migration[8.0]
  def change
    # Solo indexar pedidos pendientes (los mas consultados)
    add_index :orders, :created_at,
      where: "status = 'pending'",
      name: "index_orders_on_created_at_pending"

    # Solo indexar usuarios activos
    add_index :users, :email,
      where: "deleted_at IS NULL",
      name: "index_users_on_email_active"

    # Solo productos en stock
    add_index :products, [:category_id, :price],
      where: "stock_quantity > 0",
      name: "index_products_available"
  end
end
```

### Indices de expresion

```ruby
# db/migrate/xxx_add_expression_indexes.rb
class AddExpressionIndexes < ActiveRecord::Migration[8.0]
  def change
    # Busqueda case-insensitive
    add_index :users, "LOWER(email)",
      name: "index_users_on_lower_email",
      unique: true

    # Extraer ano de fecha
    add_index :orders, "DATE_TRUNC('month', created_at)",
      name: "index_orders_on_month"

    # Campo JSONB
    add_index :products, "(metadata->>'brand')",
      name: "index_products_on_brand"
  end
end
```

### Indices GIN y GiST

```ruby
# db/migrate/xxx_add_gin_gist_indexes.rb
class AddGinGistIndexes < ActiveRecord::Migration[8.0]
  def change
    # GIN para JSONB
    add_index :products, :metadata, using: :gin

    # GIN para arrays
    add_index :articles, :tags, using: :gin

    # GIN para full-text search
    execute <<-SQL
      CREATE INDEX articles_search_idx ON articles
      USING gin(to_tsvector('spanish', title || ' ' || body))
    SQL

    # GiST para rangos y geometria
    add_index :events, :duration, using: :gist

    # GiST para busqueda por proximidad
    execute <<-SQL
      CREATE INDEX locations_coordinates_idx ON locations
      USING gist(ll_to_earth(latitude, longitude))
    SQL
  end
end
```

## EXPLAIN ANALYZE

### Analizar queries

```ruby
# app/models/concerns/queryable.rb
module Queryable
  extend ActiveSupport::Concern

  class_methods do
    def explain_query(relation = all)
      result = connection.execute("EXPLAIN ANALYZE #{relation.to_sql}")
      result.values.map(&:first).join("\n")
    end

    def analyze_slow_query(relation = all)
      plan = connection.execute(
        "EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) #{relation.to_sql}"
      )

      JSON.parse(plan.first["QUERY PLAN"])
    end
  end
end

# Uso
puts Order.where(status: "pending").explain_query

# Resultado:
# Seq Scan on orders  (cost=0.00..1234.00 rows=50 width=100) (actual time=0.015..2.345 rows=48 loops=1)
#   Filter: (status = 'pending')
#   Rows Removed by Filter: 9952
# Planning Time: 0.123 ms
# Execution Time: 2.456 ms
```

### Interpretar planes

```ruby
# app/services/query_analyzer.rb
class QueryAnalyzer
  WARNINGS = {
    seq_scan: "Sequential scan detected - consider adding an index",
    high_rows_removed: "Many rows removed by filter - index may help",
    nested_loop: "Nested loop with high cost - consider JOIN optimization",
    sort: "Sort operation - consider adding index for ORDER BY"
  }.freeze

  def initialize(plan)
    @plan = plan.is_a?(String) ? JSON.parse(plan) : plan
  end

  def analyze
    warnings = []
    @plan.each do |node|
      plan_node = node["Plan"]
      warnings.concat(analyze_node(plan_node))
    end
    warnings
  end

  private

  def analyze_node(node, depth = 0)
    warnings = []

    # Detectar problemas comunes
    if node["Node Type"] == "Seq Scan" && node["Actual Rows"].to_i > 1000
      warnings << { type: :seq_scan, node: node["Relation Name"], rows: node["Actual Rows"] }
    end

    if node["Rows Removed by Filter"].to_i > node["Actual Rows"].to_i * 10
      warnings << { type: :high_rows_removed, removed: node["Rows Removed by Filter"] }
    end

    # Analizar nodos hijos
    (node["Plans"] || []).each do |child|
      warnings.concat(analyze_node(child, depth + 1))
    end

    warnings
  end
end
```

## Batch Operations

### INSERT...SELECT

```ruby
# Copiar datos entre tablas
def self.archive_old_orders
  execute <<-SQL
    INSERT INTO archived_orders (
      id, user_id, total_amount, status, created_at, archived_at
    )
    SELECT
      id, user_id, total_amount, status, created_at, NOW()
    FROM orders
    WHERE created_at < :cutoff_date
      AND status IN ('completed', 'cancelled')
    ON CONFLICT (id) DO NOTHING
  SQL
end
```

### UPDATE...FROM

```ruby
# Actualizar basado en otra tabla
def self.update_product_stats
  execute <<-SQL
    UPDATE products
    SET
      total_sold = stats.units_sold,
      last_sale_at = stats.last_sale,
      updated_at = NOW()
    FROM (
      SELECT
        product_id,
        SUM(quantity) AS units_sold,
        MAX(orders.created_at) AS last_sale
      FROM order_items
      JOIN orders ON orders.id = order_items.order_id
      WHERE orders.status = 'completed'
      GROUP BY product_id
    ) AS stats
    WHERE products.id = stats.product_id
  SQL
end
```

### UPSERT (INSERT ON CONFLICT)

```ruby
# db/migrate/xxx_add_unique_constraint_for_upsert.rb
class AddUniqueConstraintForUpsert < ActiveRecord::Migration[8.0]
  def change
    add_index :daily_stats, [:date, :product_id], unique: true
  end
end

# Upsert en Rails
def self.update_daily_stats(date, product_id, sales_data)
  execute <<-SQL
    INSERT INTO daily_stats (date, product_id, units_sold, revenue, created_at, updated_at)
    VALUES (:date, :product_id, :units_sold, :revenue, NOW(), NOW())
    ON CONFLICT (date, product_id)
    DO UPDATE SET
      units_sold = daily_stats.units_sold + EXCLUDED.units_sold,
      revenue = daily_stats.revenue + EXCLUDED.revenue,
      updated_at = NOW()
  SQL
end

# Con ActiveRecord (Rails 7+)
DailyStat.upsert(
  { date: Date.current, product_id: 1, units_sold: 10, revenue: 100 },
  unique_by: [:date, :product_id],
  on_duplicate: Arel.sql("units_sold = daily_stats.units_sold + EXCLUDED.units_sold")
)

# Upsert multiple
DailyStat.upsert_all(
  stats_array,
  unique_by: [:date, :product_id]
)
```

## Transacciones

### Isolation Levels

```ruby
# app/services/inventory_service.rb
class InventoryService
  def reserve_stock(order)
    # SERIALIZABLE para prevenir race conditions
    Order.transaction(isolation: :serializable) do
      order.line_items.each do |item|
        product = Product.lock.find(item.product_id)

        if product.stock_quantity < item.quantity
          raise InsufficientStockError, "Not enough stock for #{product.name}"
        end

        product.decrement!(:stock_quantity, item.quantity)
      end

      order.update!(status: "reserved")
    end
  rescue ActiveRecord::SerializationFailure
    # Retry on serialization failure
    retry
  end

  def process_concurrent_updates
    # READ COMMITTED (default) - cada query ve datos commiteados
    Order.transaction(isolation: :read_committed) do
      # ...
    end

    # REPEATABLE READ - snapshot al inicio de transaccion
    Order.transaction(isolation: :repeatable_read) do
      # ...
    end

    # SERIALIZABLE - transacciones parecen secuenciales
    Order.transaction(isolation: :serializable) do
      # ...
    end
  end
end
```

### Locks

```ruby
# Pessimistic locking
def update_balance(user_id, amount)
  User.transaction do
    user = User.lock("FOR UPDATE").find(user_id)
    user.update!(balance: user.balance + amount)
  end
end

# Skip locked (para job queues)
def claim_next_job
  Job.transaction do
    job = Job.lock("FOR UPDATE SKIP LOCKED")
             .where(status: "pending")
             .order(:created_at)
             .first

    return nil unless job

    job.update!(status: "processing", worker_id: worker_id)
    job
  end
end

# Advisory locks
def with_named_lock(name, timeout: 5)
  lock_id = Zlib.crc32(name)

  result = ActiveRecord::Base.connection.execute(
    "SELECT pg_try_advisory_lock(#{lock_id})"
  ).first["pg_try_advisory_lock"]

  unless result
    raise LockNotAcquiredError, "Could not acquire lock: #{name}"
  end

  yield
ensure
  ActiveRecord::Base.connection.execute(
    "SELECT pg_advisory_unlock(#{lock_id})"
  )
end
```

### Deadlock Prevention

```ruby
# Siempre adquirir locks en orden consistente
def transfer_funds(from_account_id, to_account_id, amount)
  # Ordenar IDs para evitar deadlock
  ids = [from_account_id, to_account_id].sort

  Account.transaction do
    accounts = Account.lock.where(id: ids).order(:id).to_a
    from_account = accounts.find { |a| a.id == from_account_id }
    to_account = accounts.find { |a| a.id == to_account_id }

    from_account.decrement!(:balance, amount)
    to_account.increment!(:balance, amount)
  end
end
```

## JSON/JSONB

### Queries

```ruby
# Buscar en campo JSONB
Product.where("metadata @> ?", { color: "red" }.to_json)
Product.where("metadata->>'brand' = ?", "Apple")
Product.where("metadata->'specs'->>'size' = ?", "large")

# Verificar si key existe
Product.where("metadata ? 'warranty'")
Product.where("metadata ?& ARRAY['color', 'size']")  # tiene todas
Product.where("metadata ?| ARRAY['color', 'size']")  # tiene alguna

# Buscar en array dentro de JSONB
Product.where("metadata->'tags' @> ?", ["electronics"].to_json)

# Actualizar campo JSONB
Product.where(id: 1).update_all(
  "metadata = jsonb_set(metadata, '{specs,weight}', '\"2kg\"')"
)

# Concatenar JSONB
Product.where(id: 1).update_all(
  "metadata = metadata || '{\"new_field\": \"value\"}'::jsonb"
)

# Remover key
Product.where(id: 1).update_all(
  "metadata = metadata - 'old_field'"
)
```

### Indices para JSONB

```ruby
# db/migrate/xxx_add_jsonb_indexes.rb
class AddJsonbIndexes < ActiveRecord::Migration[8.0]
  def change
    # Indice general GIN (para @>, ?, ?&, ?|)
    add_index :products, :metadata, using: :gin

    # Indice para path especifico
    add_index :products, "(metadata->>'brand')",
      name: "index_products_on_metadata_brand"

    # Indice GIN para path especifico (para @> en ese path)
    execute <<-SQL
      CREATE INDEX index_products_on_metadata_tags
      ON products USING gin ((metadata->'tags'))
    SQL
  end
end
```

## Full-Text Search

### Configuracion basica

```ruby
# db/migrate/xxx_add_full_text_search.rb
class AddFullTextSearch < ActiveRecord::Migration[8.0]
  def up
    # Agregar columna tsvector
    add_column :articles, :search_vector, :tsvector

    # Indice GIN
    add_index :articles, :search_vector, using: :gin

    # Trigger para mantener actualizado
    execute <<-SQL
      CREATE OR REPLACE FUNCTION articles_search_trigger()
      RETURNS trigger AS $$
      BEGIN
        NEW.search_vector :=
          setweight(to_tsvector('spanish', COALESCE(NEW.title, '')), 'A') ||
          setweight(to_tsvector('spanish', COALESCE(NEW.excerpt, '')), 'B') ||
          setweight(to_tsvector('spanish', COALESCE(NEW.body, '')), 'C');
        RETURN NEW;
      END
      $$ LANGUAGE plpgsql;

      CREATE TRIGGER articles_search_update
      BEFORE INSERT OR UPDATE ON articles
      FOR EACH ROW EXECUTE FUNCTION articles_search_trigger();
    SQL

    # Actualizar registros existentes
    execute "UPDATE articles SET search_vector = search_vector"
  end

  def down
    execute "DROP TRIGGER IF EXISTS articles_search_update ON articles"
    execute "DROP FUNCTION IF EXISTS articles_search_trigger"
    remove_column :articles, :search_vector
  end
end

# app/models/article.rb
class Article < ApplicationRecord
  scope :search, ->(query) {
    where("search_vector @@ plainto_tsquery('spanish', ?)", query)
      .order(Arel.sql("ts_rank(search_vector, plainto_tsquery('spanish', '#{sanitize_sql_like(query)}')) DESC"))
  }

  scope :search_with_highlights, ->(query) {
    select(
      "articles.*",
      "ts_headline('spanish', title, plainto_tsquery('spanish', #{connection.quote(query)})) AS title_highlight",
      "ts_headline('spanish', body, plainto_tsquery('spanish', #{connection.quote(query)}), 'MaxWords=50') AS body_highlight"
    ).search(query)
  }
end
```

### Con pg_search gem

```ruby
# Gemfile
gem "pg_search"

# app/models/article.rb
class Article < ApplicationRecord
  include PgSearch::Model

  pg_search_scope :search,
    against: {
      title: 'A',
      excerpt: 'B',
      body: 'C'
    },
    using: {
      tsearch: {
        dictionary: "spanish",
        tsvector_column: "search_vector",
        prefix: true
      },
      trigram: {
        threshold: 0.3
      }
    }

  pg_search_scope :search_by_category,
    against: :title,
    associated_against: {
      category: :name,
      tags: :name
    }
end
```

## Partitioning

### Range Partitioning

```ruby
# db/migrate/xxx_create_partitioned_orders.rb
class CreatePartitionedOrders < ActiveRecord::Migration[8.0]
  def up
    execute <<-SQL
      CREATE TABLE orders (
        id BIGSERIAL,
        user_id BIGINT NOT NULL,
        total_amount DECIMAL(10,2),
        status VARCHAR(20),
        created_at TIMESTAMP NOT NULL,
        PRIMARY KEY (id, created_at)
      ) PARTITION BY RANGE (created_at);
    SQL

    # Crear particiones por mes
    12.times do |i|
      month_start = Date.current.beginning_of_year + i.months
      month_end = month_start + 1.month

      execute <<-SQL
        CREATE TABLE orders_#{month_start.strftime('%Y_%m')}
        PARTITION OF orders
        FOR VALUES FROM ('#{month_start}') TO ('#{month_end}');
      SQL
    end

    # Particion default para datos futuros
    execute <<-SQL
      CREATE TABLE orders_default PARTITION OF orders DEFAULT;
    SQL
  end

  def down
    execute "DROP TABLE orders CASCADE"
  end
end
```

### Mantenimiento de particiones

```ruby
# app/jobs/partition_maintenance_job.rb
class PartitionMaintenanceJob < ApplicationJob
  queue_as :low

  def perform
    create_future_partitions
    archive_old_partitions
  end

  private

  def create_future_partitions
    # Crear particiones para los proximos 3 meses
    3.times do |i|
      month = Date.current.beginning_of_month + (i + 1).months
      partition_name = "orders_#{month.strftime('%Y_%m')}"

      unless partition_exists?(partition_name)
        ActiveRecord::Base.connection.execute <<-SQL
          CREATE TABLE IF NOT EXISTS #{partition_name}
          PARTITION OF orders
          FOR VALUES FROM ('#{month}') TO ('#{month + 1.month}');
        SQL
      end
    end
  end

  def archive_old_partitions
    # Detach particiones mayores a 2 anos
    cutoff = Date.current - 2.years

    old_partitions = ActiveRecord::Base.connection.execute(<<-SQL
      SELECT tablename FROM pg_tables
      WHERE tablename LIKE 'orders_%'
        AND tablename ~ 'orders_[0-9]{4}_[0-9]{2}'
    SQL
    ).map { |r| r["tablename"] }

    old_partitions.each do |partition|
      match = partition.match(/orders_(\d{4})_(\d{2})/)
      next unless match

      partition_date = Date.new(match[1].to_i, match[2].to_i, 1)

      if partition_date < cutoff
        # Mover a tabla de archivo
        ActiveRecord::Base.connection.execute <<-SQL
          ALTER TABLE orders DETACH PARTITION #{partition};
          ALTER TABLE #{partition} RENAME TO archived_#{partition};
        SQL
      end
    end
  end

  def partition_exists?(name)
    ActiveRecord::Base.connection.execute(<<-SQL
      SELECT 1 FROM pg_tables WHERE tablename = '#{name}'
    SQL
    ).any?
  end
end
```

## Checklist

- [ ] CTEs para queries complejos y legibles
- [ ] Window functions para rankings y comparaciones
- [ ] Vistas para abstraer queries frecuentes
- [ ] Vistas materializadas para reportes pesados
- [ ] Indices parciales para filtros comunes
- [ ] Indices GIN para JSONB y full-text
- [ ] EXPLAIN ANALYZE para optimizar queries lentos
- [ ] Batch operations para actualizaciones masivas
- [ ] Isolation levels apropiados para transacciones
- [ ] Full-text search configurado
- [ ] Partitioning para tablas grandes
