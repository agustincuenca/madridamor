# Comando: /learn

Sugiere mejoras al código actual y proporciona recursos de aprendizaje.

## Uso

```
/learn [área]
```

## Parámetros

- Sin argumentos: muestra áreas disponibles
- `architecture`: patrones arquitectónicos y organización de código
- `testing`: mejoras en estrategia de testing
- `security`: prácticas de seguridad
- `performance`: optimización de rendimiento
- `patterns`: patrones de diseño aplicables

## Flujo de trabajo

### Paso 1: Sin argumentos - Mostrar áreas disponibles

```markdown
# Áreas de Aprendizaje

¿Sobre qué área te gustaría aprender y mejorar?

1. **architecture** - Patrones arquitectónicos, organización de código, SOLID
2. **testing** - Estrategias de testing, TDD, cobertura
3. **security** - Seguridad web, OWASP, protección de datos
4. **performance** - Optimización, caching, queries eficientes
5. **patterns** - Patrones de diseño, refactoring

Escribe `/learn [área]` para comenzar.
```

### Paso 2: Analizar código actual

Para cada área, analizar el código del proyecto buscando:

1. **Oportunidades de mejora**
   - Código que podría beneficiarse de patrones
   - Áreas donde se repite lógica
   - Complejidad innecesaria

2. **Buenas prácticas existentes**
   - Reconocer lo que ya está bien hecho
   - Reforzar patrones positivos

3. **Prioridad de mejoras**
   - Alto impacto, bajo esfuerzo primero
   - Mejoras que desbloquean otras

### Paso 3: Generar reporte según área

## `/learn architecture`

### Análisis

```markdown
## Análisis de Arquitectura

### Estado Actual
- Estructura de directorios: [evaluación]
- Separación de responsabilidades: [evaluación]
- Dependencias entre módulos: [evaluación]

### Oportunidades de Mejora

#### 1. [Mejora identificada]
**Prioridad**: Alta/Media/Baja
**Esfuerzo**: Alto/Medio/Bajo

**Situación actual:**
```ruby
# Código actual
```

**Mejora sugerida:**
```ruby
# Código mejorado
```

**Por qué es mejor:**
- [Razón 1]
- [Razón 2]

### Recursos para Profundizar

📚 **Libros:**
- Clean Architecture (Robert C. Martin)
- Patterns of Enterprise Application Architecture (Martin Fowler)

🎓 **Cursos:**
- [Curso recomendado 1]
- [Curso recomendado 2]

📝 **Artículos:**
- [Artículo relevante 1]
- [Artículo relevante 2]
```

## `/learn testing`

### Análisis

```markdown
## Análisis de Testing

### Estado Actual
- Cobertura estimada: X%
- Tipos de tests: [unit/integration/system]
- Framework: RSpec

### Oportunidades de Mejora

#### 1. Áreas sin tests
- [Modelo/Controlador X] - Sin tests
- [Feature Y] - Tests incompletos

#### 2. Tests que podrían mejorar
**Antes:**
```ruby
it "works" do
  expect(user.save).to be true
end
```

**Después:**
```ruby
describe "#save" do
  context "with valid attributes" do
    it "persists the user" do
      user = build(:user)
      expect { user.save }.to change(User, :count).by(1)
    end
  end

  context "with invalid email" do
    it "returns false and adds error" do
      user = build(:user, email: "invalid")
      expect(user.save).to be false
      expect(user.errors[:email]).to include("is invalid")
    end
  end
end
```

### Recursos para Profundizar

📚 **Libros:**
- Effective Testing with RSpec 3
- Growing Object-Oriented Software, Guided by Tests
- The RSpec Book

🎓 **Cursos:**
- Test-Driven Development en Ruby (Upcase)
- Testing Rails Applications (GoRails)

📝 **Artículos:**
- Better Specs (betterspecs.org)
- Testing best practices de Thoughtbot
```

## `/learn security`

### Análisis

```markdown
## Análisis de Seguridad

### Estado Actual
- Autenticación: [implementación]
- Autorización: [implementación]
- Protección CSRF: [estado]
- Sanitización de entrada: [estado]

### Oportunidades de Mejora

#### 1. [Vulnerabilidad potencial]
**Riesgo**: Alto/Medio/Bajo
**Ubicación**: [archivo]

**Código actual:**
```ruby
# Código vulnerable
```

**Código seguro:**
```ruby
# Código corregido
```

### OWASP Top 10 - Checklist

- [ ] Injection
- [ ] Broken Authentication
- [ ] Sensitive Data Exposure
- [ ] XML External Entities
- [ ] Broken Access Control
- [ ] Security Misconfiguration
- [ ] Cross-Site Scripting (XSS)
- [ ] Insecure Deserialization
- [ ] Using Components with Known Vulnerabilities
- [ ] Insufficient Logging & Monitoring

### Recursos para Profundizar

📚 **Libros:**
- Web Application Security (Andrew Hoffman)
- The Web Application Hacker's Handbook

🎓 **Cursos:**
- OWASP Web Security Testing Guide
- Secure Rails Development

📝 **Artículos:**
- Rails Security Guide (oficial)
- Brakeman documentation
```

## `/learn performance`

### Análisis

```markdown
## Análisis de Rendimiento

### Estado Actual
- Tiempo de respuesta promedio: [estimación]
- Queries por página: [estimación]
- Uso de caching: [estado]

### Oportunidades de Mejora

#### 1. N+1 Queries
**Ubicación**: [archivo:línea]
**Impacto**: Alto

**Antes:**
```ruby
@articles = Article.all
# En la vista:
<% @articles.each do |article| %>
  <%= article.user.name %>
<% end %>
# Genera: 1 query para articles + N queries para users
```

**Después:**
```ruby
@articles = Article.includes(:user)
# Genera: 2 queries total
```

#### 2. Oportunidades de Caching
- Fragment caching para [componente]
- Russian doll caching para [lista]

#### 3. Índices Faltantes
```ruby
# Migration sugerida
add_index :articles, :user_id
add_index :articles, [:published, :created_at]
```

### Recursos para Profundizar

📚 **Libros:**
- High Performance Ruby (Charles Nutter)
- Ruby Performance Optimization (Alexander Dymo)

🎓 **Cursos:**
- Scaling Rails Applications
- Performance Optimization (GoRails)

📝 **Artículos:**
- Rails Performance Guide
- Bullet gem documentation
```

## `/learn patterns`

### Análisis

```markdown
## Análisis de Patrones de Diseño

### Patrones Aplicables al Proyecto

#### 1. Service Objects
**Cuándo usarlo**: Lógica de negocio compleja en controladores

**Antes:**
```ruby
class OrdersController < ApplicationController
  def create
    @order = Order.new(order_params)
    @order.calculate_total
    @order.apply_discount(current_user)
    @order.reserve_inventory
    if @order.save
      OrderMailer.confirmation(@order).deliver_later
      redirect_to @order
    end
  end
end
```

**Después:**
```ruby
# app/services/create_order_service.rb
class CreateOrderService
  def initialize(user:, order_params:)
    @user = user
    @order_params = order_params
  end

  def call
    order = Order.new(@order_params)
    order.calculate_total
    order.apply_discount(@user)
    order.reserve_inventory

    if order.save
      OrderMailer.confirmation(order).deliver_later
      Result.success(order)
    else
      Result.failure(order.errors)
    end
  end
end

# En el controlador
class OrdersController < ApplicationController
  def create
    result = CreateOrderService.new(
      user: current_user,
      order_params: order_params
    ).call

    if result.success?
      redirect_to result.value
    else
      render :new
    end
  end
end
```

#### 2. Form Objects
**Cuándo usarlo**: Formularios que afectan múltiples modelos

#### 3. Query Objects
**Cuándo usarlo**: Queries complejas reutilizables

#### 4. Presenter/Decorator
**Cuándo usarlo**: Lógica de presentación compleja

### Recursos para Profundizar

📚 **Libros:**
- Design Patterns in Ruby (Russ Olsen)
- Practical Object-Oriented Design in Ruby (Sandi Metz)
- Refactoring: Ruby Edition (Jay Fields)

🎓 **Cursos:**
- Design Patterns in Ruby (Upcase)
- Refactoring Rails (GoRails)

📝 **Artículos:**
- 7 Patterns to Refactor Fat ActiveRecord Models
- Service Objects in Rails
```

## Recursos Generales

### Libros Esenciales

| Libro | Área | Nivel |
|-------|------|-------|
| Clean Code | General | Intermedio |
| POODR | OOP/Ruby | Intermedio |
| The Rails Way | Rails | Intermedio |
| Refactoring | Patterns | Avanzado |
| Domain-Driven Design | Architecture | Avanzado |

### Comunidades

- Ruby on Rails Discord
- Reddit r/rails
- Ruby Weekly newsletter
- Thoughtbot Blog

### Práctica

- Exercism.io (Ruby track)
- Codewars
- Contribuir a open source

## Notas importantes

- Las mejoras deben ser incrementales
- No refactorizar todo a la vez
- Siempre tener tests antes de refactorizar
- Priorizar por impacto en el proyecto
- El objetivo es aprender, no complicar el código
