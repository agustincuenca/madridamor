# Comando: /estado

Muestra un resumen del estado actual del proyecto.

## Flujo de trabajo

### Paso 1: Recopilar información

Leer archivos de contexto:
- `.claude/contexts/requirements.md`
- `.claude/contexts/architecture.md`
- `.claude/history/changelog.md`
- `.claude/history/active/`

### Paso 1b: Verificar Features (si existen)

Buscar en `features/*/feature.json`:

```bash
ls features/*/feature.json 2>/dev/null
```

Si hay features, incluir en el resumen:

```markdown
## Features Activos

| Feature | Estado | Progreso | Siguiente Acción |
|---------|--------|----------|------------------|
| {título} | {status} | {X}% | /plan {path} |

### Detalle de Feature en Progreso
{Para cada feature con status in_progress}
- Tareas completadas: X/Y
- Tarea actual: {nombre}
- Próxima acción: {comando sugerido}
```

### Paso 2: Analizar el proyecto

```bash
# Contar modelos
ls -la app/models/*.rb 2>/dev/null | wc -l

# Contar controladores
ls -la app/controllers/*.rb 2>/dev/null | wc -l

# Contar vistas
find app/views -name "*.erb" 2>/dev/null | wc -l

# Contar tests
find spec -name "*_spec.rb" 2>/dev/null | wc -l

# Verificar estado de tests
bundle exec rspec --format progress 2>/dev/null || echo "Tests no configurados"
```

### Paso 3: Generar resumen

```markdown
# Estado del Proyecto: [Nombre]

## Información General
- **Nombre**: [nombre del proyecto]
- **Creado**: [fecha de creación]
- **Última modificación**: [fecha]
- **Stack**: Rails 8.1 + SQLite3 + Tailwind + Hotwire

## Estructura Actual

### Modelos (X)
| Modelo | Campos principales | Relaciones |
|--------|-------------------|------------|
| User | email, name | has_many :articles |
| Article | title, body, published | belongs_to :user |

### Controladores (X)
- `ApplicationController`
- `SessionsController`
- `ArticlesController`
- ...

### Vistas (X páginas)
- Home
- Artículos (index, show, new, edit)
- Autenticación (login, registro)

## Funcionalidades

### ✅ Implementadas
- [x] Autenticación de usuarios
- [x] CRUD de artículos
- [x] Sistema de comentarios

### 🔄 En progreso
- [ ] [Funcionalidad en desarrollo]

### 📋 Pendientes
- [ ] [Funcionalidad planificada]
- [ ] [Otra funcionalidad]

## Tests

| Tipo | Total | Pasando | Fallando |
|------|-------|---------|----------|
| Modelos | X | X | 0 |
| Controladores | X | X | 0 |
| Features | X | X | 0 |

**Cobertura estimada**: X%

## Últimos Cambios

### [Fecha más reciente]
- [Cambio 1]
- [Cambio 2]

### [Fecha anterior]
- [Cambio 3]

## Métricas del Proyecto

| Métrica | Valor |
|---------|-------|
| Archivos Ruby | X |
| Líneas de código | ~X |
| Migraciones | X |
| Rutas | X |

## Dependencias Principales

```ruby
# Gemfile highlights
rails (~> 8.1)
sqlite3
tailwindcss-rails
turbo-rails
stimulus-rails
pundit
```

## Próximos Pasos Sugeridos

1. [Sugerencia basada en el estado actual]
2. [Otra sugerencia]
3. [Tercera sugerencia]

## Comandos Útiles

```bash
# Iniciar servidor
bin/dev

# Correr tests
bundle exec rspec

# Consola de Rails
rails console

# Ver rutas
rails routes
```
```

### Paso 4: Mostrar al usuario

Presentar el resumen de forma amigable:

1. Estado general (todo bien / hay problemas)
2. Lo que está funcionando
3. Lo que falta por hacer
4. Recomendaciones

## Variantes del comando

### `/estado breve`
Muestra solo un resumen corto:
```
📊 Proyecto: [Nombre]
✅ 5 modelos, 8 controladores, 15 vistas
🧪 45 tests pasando
📝 Última modificación: hace 2 horas
```

### `/estado modelos`
Muestra detalle de los modelos y sus relaciones.

### `/estado tests`
Muestra estado detallado de los tests.

### `/estado cambios`
Muestra los últimos cambios realizados.

## Información adicional

### Si hay problemas
```
⚠️ Se detectaron algunos problemas:
- 3 tests fallando
- 1 migración pendiente

¿Quieres que los arregle?
```

### Si todo está bien
```
✅ Todo en orden
- Tests pasando
- Sin migraciones pendientes
- Sin errores detectados
```

## Notas importantes

- Mantener actualizado el archivo de requirements
- Actualizar changelog con cada cambio
- Si falta información, sugerir crearla
- Usar lenguaje sencillo y visual (emojis para estado)
