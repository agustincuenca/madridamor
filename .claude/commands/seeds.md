# Comando: /seeds

Genera datos de prueba realistas para la aplicación.

## Flujo de trabajo

### Paso 1: Analizar modelos existentes

1. Leer todos los modelos en `app/models/`
2. Identificar relaciones entre modelos
3. Determinar validaciones y restricciones

### Paso 2: Preguntar preferencias

"¿Cuántos datos de prueba quieres generar?"
- Mínimo (5-10 registros por modelo)
- Medio (20-50 registros por modelo)
- Extenso (100+ registros por modelo)

"¿Quieres un usuario de prueba específico?"
- Email: test@example.com
- Contraseña: password123

### Paso 3: Generar archivo de seeds

```ruby
# db/seeds.rb

puts "🌱 Iniciando seed de datos..."

# Limpiar datos existentes (en orden inverso de dependencias)
puts "Limpiando datos existentes..."
# [Modelos dependientes primero]
Comment.destroy_all
Article.destroy_all
User.destroy_all

# Crear usuario de prueba
puts "Creando usuario de prueba..."
test_user = User.create!(
  email_address: "test@example.com",
  password: "password123",
  password_confirmation: "password123"
)
puts "  ✅ Usuario de prueba: test@example.com / password123"

# Crear usuarios adicionales
puts "Creando usuarios..."
users = 10.times.map do |i|
  User.create!(
    email_address: Faker::Internet.unique.email,
    password: "password123",
    password_confirmation: "password123"
  )
end
puts "  ✅ #{users.count} usuarios creados"

# Crear artículos
puts "Creando artículos..."
all_users = [test_user] + users
article_count = 0

all_users.each do |user|
  rand(3..8).times do
    Article.create!(
      title: Faker::Lorem.sentence(word_count: rand(4..8)),
      body: Faker::Lorem.paragraphs(number: rand(3..8)).join("\n\n"),
      published: [true, true, true, false].sample,
      user: user,
      created_at: Faker::Time.between(from: 6.months.ago, to: Time.current)
    )
    article_count += 1
  end
end
puts "  ✅ #{article_count} artículos creados"

# Crear comentarios
puts "Creando comentarios..."
comment_count = 0

Article.where(published: true).each do |article|
  rand(0..10).times do
    Comment.create!(
      body: Faker::Lorem.paragraph(sentence_count: rand(1..4)),
      user: all_users.sample,
      article: article,
      created_at: Faker::Time.between(from: article.created_at, to: Time.current)
    )
    comment_count += 1
  end
end
puts "  ✅ #{comment_count} comentarios creados"

# Resumen final
puts ""
puts "=" * 50
puts "🎉 Seed completado!"
puts "=" * 50
puts ""
puts "Resumen:"
puts "  - Usuarios: #{User.count}"
puts "  - Artículos: #{Article.count} (#{Article.where(published: true).count} publicados)"
puts "  - Comentarios: #{Comment.count}"
puts ""
puts "Usuario de prueba:"
puts "  📧 Email: test@example.com"
puts "  🔑 Contraseña: password123"
puts ""
```

### Paso 4: Ejecutar seeds

```bash
rails db:seed
```

### Paso 5: Verificar datos

```bash
rails runner "puts 'Users: ' + User.count.to_s; puts 'Articles: ' + Article.count.to_s"
```

### Paso 6: Mostrar resumen

```markdown
# Datos de Prueba Generados

## Resumen
| Modelo | Cantidad |
|--------|----------|
| Users | 11 |
| Articles | 55 |
| Comments | 234 |

## Usuario de prueba
- **Email**: test@example.com
- **Contraseña**: password123

## Datos generados
Los datos incluyen:
- Usuarios con emails realistas
- Artículos con títulos y contenido variado
- Mezcla de artículos publicados y borradores
- Comentarios en artículos publicados
- Fechas distribuidas en los últimos 6 meses

## Comandos útiles

```bash
# Regenerar datos
rails db:seed

# Limpiar y regenerar
rails db:reset  # Cuidado: borra todo

# Ver datos en consola
rails console
> User.all
> Article.published.count
```
```

## Variantes del comando

### `/seeds [modelo]`
Genera seeds solo para un modelo específico.

### `/seeds reset`
Limpia todos los datos y regenera.

### `/seeds add [n]`
Añade N registros más sin borrar existentes.

## Datos específicos por modelo

### Usuarios
```ruby
User.create!(
  email_address: Faker::Internet.unique.email,
  password: "password123",
  # Si hay campos adicionales:
  name: Faker::Name.name,
  bio: Faker::Lorem.paragraph
)
```

### Contenido con texto
```ruby
Article.create!(
  title: Faker::Book.title,  # O Faker::Lorem.sentence
  body: Faker::Markdown.sandwich(sentences: 6),
  summary: Faker::Lorem.paragraph
)
```

### Con fechas
```ruby
Event.create!(
  name: Faker::Company.catch_phrase,
  starts_at: Faker::Time.forward(days: 30),
  ends_at: Faker::Time.forward(days: 60)
)
```

### Con archivos (Active Storage)
```ruby
user = User.create!(...)
user.avatar.attach(
  io: URI.open(Faker::Avatar.image),
  filename: "avatar.jpg"
)
```

### Con enums
```ruby
Order.create!(
  status: Order::STATUSES.sample,
  # o
  status: [:pending, :processing, :completed].sample
)
```

## Notas importantes

- Los seeds deben ser idempotentes (seguros de ejecutar múltiples veces)
- Usar `destroy_all` con cuidado en producción
- Faker genera datos en inglés por defecto
- Para datos en español: `Faker::Config.locale = 'es'`
- Siempre incluir un usuario de prueba conocido
- Los datos de prueba no deben ir a producción
