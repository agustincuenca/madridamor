# Comando: /debug

Analiza errores y ayuda a depurar problemas en la aplicación.

## Uso

```
/debug [descripción del error | "logs" | "performance"]
```

## Parámetros

- Sin argumentos: pregunta qué tipo de problema hay
- `descripción`: analiza el stack trace o error específico
- `logs`: revisa logs recientes e identifica patrones de error
- `performance`: ejecuta profiling e identifica problemas de rendimiento

## Flujo de trabajo

### Paso 1: Identificar el tipo de problema

Sin argumentos, preguntar:

"¿Qué tipo de problema estás experimentando?"

Opciones:
- Un error específico (pega el mensaje o stack trace)
- La app va lenta
- Algo no funciona como esperaba
- Quiero revisar los logs

### Paso 2: Según el tipo de problema

#### Para errores específicos

1. **Analizar el stack trace**
   ```bash
   # Extraer información clave
   # - Archivo y línea donde ocurre
   # - Tipo de excepción
   # - Mensaje de error
   # - Contexto (controlador, modelo, vista)
   ```

2. **Identificar la causa probable**
   - Error de sintaxis
   - Nil reference (NoMethodError on nil)
   - Base de datos (ActiveRecord errors)
   - Routing
   - Validación
   - Autorización

3. **Buscar el código relacionado**
   ```bash
   # Leer el archivo mencionado en el stack trace
   # Verificar el contexto alrededor de la línea
   ```

4. **Proponer solución**
   - Explicar qué está pasando en lenguaje simple
   - Mostrar el código problemático
   - Proponer corrección con ejemplo

#### Para `/debug logs`

1. **Revisar logs recientes**
   ```bash
   # Ver últimas líneas del log de desarrollo
   tail -100 log/development.log

   # Buscar errores
   grep -i "error\|exception\|fail" log/development.log | tail -50

   # Ver requests fallidos
   grep "500\|422\|404" log/development.log | tail -20
   ```

2. **Identificar patrones**
   - Errores repetidos
   - Queries lentas
   - Excepciones no manejadas
   - Problemas de memoria

3. **Generar reporte**
   ```markdown
   ## Análisis de Logs

   ### Errores encontrados
   | Error | Frecuencia | Última vez |
   |-------|------------|------------|
   | [Error 1] | X veces | [timestamp] |

   ### Warnings
   - [Warning 1]

   ### Recomendaciones
   1. [Acción 1]
   2. [Acción 2]
   ```

#### Para `/debug performance`

1. **Ejecutar análisis**
   ```bash
   # Identificar N+1 queries en desarrollo
   # El bullet gem detecta esto automáticamente

   # Ver queries lentas
   grep "ms)" log/development.log | sort -t'(' -k2 -rn | head -20
   ```

2. **Analizar código**
   - Buscar `each` sin `includes`
   - Identificar queries en loops
   - Verificar índices en la base de datos
   - Revisar uso de memoria

3. **Generar reporte de rendimiento**
   ```markdown
   ## Análisis de Rendimiento

   ### N+1 Queries Detectadas
   1. **[Ubicación]**
      - Problema: [Descripción]
      - Solución: `includes(:asociacion)`

   ### Queries Lentas
   | Query | Tiempo | Ubicación |
   |-------|--------|-----------|
   | [Query] | Xms | [archivo:línea] |

   ### Índices Faltantes
   - Tabla `X`, columna `Y`

   ### Recomendaciones
   1. [Mejora 1]
   2. [Mejora 2]
   ```

### Paso 3: Proporcionar solución

Para cada problema identificado:

1. **Explicar el problema**
   - Qué está pasando
   - Por qué está pasando
   - Impacto (crítico/medio/bajo)

2. **Mostrar la solución**
   ```ruby
   # ❌ Antes (código problemático)
   [código actual]

   # ✅ Después (código corregido)
   [código sugerido]
   ```

3. **Ofrecer implementar**
   "¿Quieres que aplique esta corrección?"

## Errores comunes y soluciones

### NoMethodError: undefined method for nil:NilClass

```ruby
# ❌ Problema
user.profile.name  # user.profile es nil

# ✅ Solución 1: Safe navigation
user.profile&.name

# ✅ Solución 2: Verificación
user.profile.name if user.profile.present?

# ✅ Solución 3: Default
user.profile&.name || "Sin nombre"
```

### ActiveRecord::RecordNotFound

```ruby
# ❌ Problema
@article = Article.find(params[:id])  # Lanza excepción si no existe

# ✅ Solución 1: find_by (retorna nil)
@article = Article.find_by(id: params[:id])
return head :not_found unless @article

# ✅ Solución 2: Manejo de excepción
rescue ActiveRecord::RecordNotFound
  redirect_to articles_path, alert: "Artículo no encontrado"
```

### ActionController::ParameterMissing

```ruby
# ❌ Problema
params.require(:article)  # :article no está en params

# ✅ Solución
# Verificar que el formulario envía correctamente
# o hacer el parámetro opcional
params.fetch(:article, {})
```

### N+1 Query

```ruby
# ❌ Problema
Article.all.each do |article|
  puts article.user.name  # Una query por cada artículo
end

# ✅ Solución
Article.includes(:user).each do |article|
  puts article.user.name  # Una sola query
end
```

### Rollback en validaciones

```ruby
# ❌ Problema
# Transaction rolled back - pero no sabes por qué

# ✅ Solución: Ver errores
article = Article.new(params)
unless article.save
  puts article.errors.full_messages
end

# O usar save! para ver la excepción
article.save!  # Lanza ActiveRecord::RecordInvalid con detalles
```

## Herramientas de debugging

### Rails Console

```ruby
# Iniciar consola
bin/rails console

# Recargar código
reload!

# Ver SQL generado
Article.where(published: true).to_sql

# Debugear objeto
article.inspect
article.attributes

# Ver errores de validación
article.valid?
article.errors.full_messages
```

### Byebug/Debug

```ruby
# Añadir breakpoint en el código
debugger  # Rails 7+
byebug    # Gem byebug

# Comandos en el debugger
n         # next (siguiente línea)
s         # step (entrar en método)
c         # continue
p variable # print variable
```

### Logs mejorados

```ruby
# En cualquier parte del código
Rails.logger.debug "Variable: #{variable.inspect}"
Rails.logger.info "Llegamos aquí"
Rails.logger.error "Algo salió mal: #{error.message}"
```

## Checklist de debugging

- [ ] ¿Leí el mensaje de error completo?
- [ ] ¿Identifiqué el archivo y línea del error?
- [ ] ¿Revisé el contexto alrededor del error?
- [ ] ¿Busqué el error en Google/Stack Overflow?
- [ ] ¿Reproduje el error localmente?
- [ ] ¿Aislé el problema (comentando código)?
- [ ] ¿Usé breakpoints para inspeccionar el estado?
- [ ] ¿Verifiqué los datos de entrada?
- [ ] ¿Revisé los logs?
- [ ] ¿Probé la solución con tests?

## Ejemplo de salida

```
🔍 Analizando el error...

📍 Ubicación: app/controllers/articles_controller.rb:25

🐛 Error: NoMethodError: undefined method 'name' for nil:NilClass

📋 Contexto:
   El error ocurre cuando intentas acceder a `@article.user.name`
   pero el artículo no tiene un usuario asociado (user_id es nil).

💡 Causa probable:
   - El artículo se creó sin asociar un usuario
   - El usuario fue eliminado pero el artículo permanece

🔧 Solución sugerida:

   # En app/controllers/articles_controller.rb línea 25
   # Cambiar:
   @article.user.name

   # Por:
   @article.user&.name || "Usuario desconocido"

   # O mejor, arreglar el modelo:
   # app/models/article.rb
   validates :user, presence: true

¿Quieres que aplique esta corrección?
```

## Notas importantes

- No modificar código de producción sin entender el problema
- Siempre reproducir el error antes de corregir
- Los tests deben pasar después de la corrección
- Documentar bugs significativos para evitar repetirlos
- Explicar en lenguaje simple qué pasó y cómo se resolvió
