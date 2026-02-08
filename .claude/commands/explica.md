# Comando: /explica

Explica cualquier concepto técnico en lenguaje sencillo.

## Flujo de trabajo

### Paso 1: Identificar qué explicar

El usuario puede preguntar sobre:
- Conceptos técnicos ("¿qué es una migración?")
- Código específico ("explícame este archivo")
- Arquitectura ("¿cómo funciona el sistema de usuarios?")
- Errores ("¿qué significa este error?")
- Decisiones ("¿por qué usamos SQLite?")

### Paso 2: Adaptar la explicación

Usar analogías del mundo real y evitar jerga técnica:

#### Ejemplo: Explicar una base de datos
```
❌ Técnico:
"SQLite es un sistema de gestión de bases de datos relacional
que implementa un motor SQL autónomo, sin servidor, de
configuración cero y transaccional."

✅ Sencillo:
"La base de datos es como un archivador muy organizado donde
guardamos toda la información de tu app.

Imagina que tienes:
- Una carpeta para usuarios
- Una carpeta para artículos
- Una carpeta para comentarios

Cada carpeta tiene fichas ordenadas, y podemos encontrar
cualquier información rápidamente. SQLite es el encargado
de mantener todo esto ordenado."
```

### Paso 3: Estructurar la respuesta

```markdown
## [Concepto]

### ¿Qué es?
[Explicación en una o dos frases simples]

### ¿Para qué sirve?
[Propósito práctico]

### Analogía
[Comparación con algo cotidiano]

### En tu proyecto
[Cómo se usa específicamente en este proyecto]

### Ejemplo práctico
[Código o demostración simple si aplica]
```

## Explicaciones comunes

### MVC (Model-View-Controller)
```markdown
## MVC - Cómo se organiza tu app

### ¿Qué es?
Es una forma de organizar el código en tres partes separadas.

### Analogía: Un restaurante
- **Modelo (Cocina)**: Donde se prepara la comida (los datos)
- **Vista (Mesa)**: Lo que ve el cliente (las páginas web)
- **Controlador (Mesero)**: Conecta pedidos con cocina (la lógica)

### En tu proyecto
```
app/
├── models/      ← Los datos (User, Article)
├── views/       ← Las páginas HTML
└── controllers/ ← La lógica que conecta todo
```

Cuando alguien visita /articles:
1. El controlador recibe la visita
2. Pide los artículos al modelo
3. Los envía a la vista para mostrarlos
```

### Migraciones
```markdown
## Migraciones - Cambios en la base de datos

### ¿Qué es?
Son instrucciones para modificar la estructura de tu base de datos.

### Analogía: Reformas en una casa
Imagina que tu base de datos es una casa:
- Añadir un campo = añadir un enchufe
- Crear una tabla = construir una habitación nueva
- Una migración = el plano de la reforma

Las migraciones guardan el historial de todos los cambios,
así que siempre puedes ver qué modificaciones se hicieron.

### Ejemplo
```ruby
# "Añadir columna 'avatar' a la tabla 'users'"
add_column :users, :avatar, :string
```
```

### Rutas
```markdown
## Rutas - Las direcciones de tu app

### ¿Qué es?
Las rutas conectan las URLs con el código que debe ejecutarse.

### Analogía: Recepción de un edificio
Cuando alguien llega y dice "quiero ir a contabilidad",
el recepcionista le indica el piso y oficina correctos.

Las rutas hacen lo mismo:
- Alguien visita `/articles`
- Rails lo envía al controlador de artículos
- Se muestra la lista de artículos

### En tu proyecto
```ruby
# config/routes.rb
resources :articles
# Esto crea automáticamente:
# GET /articles        → ver todos
# GET /articles/1      → ver uno
# GET /articles/new    → formulario nuevo
# POST /articles       → crear
# GET /articles/1/edit → formulario editar
# PATCH /articles/1    → actualizar
# DELETE /articles/1   → borrar
```
```

### Hotwire
```markdown
## Hotwire - Páginas rápidas sin JavaScript complejo

### ¿Qué es?
Una forma de hacer páginas web rápidas y dinámicas sin escribir
mucho JavaScript.

### Analogía: Remodelación vs Reconstrucción
Sin Hotwire: Cada vez que haces clic, se derriba toda la casa
y se construye de nuevo (la página se recarga completamente).

Con Hotwire: Solo se cambia la habitación que necesita cambios
(se actualiza solo una parte de la página).

### Componentes
- **Turbo Drive**: Hace clics más rápidos automáticamente
- **Turbo Frames**: Actualiza solo partes de la página
- **Turbo Streams**: Actualiza varias partes a la vez
- **Stimulus**: JavaScript simple para interacciones

### Ejemplo
Cuando añades un comentario, solo aparece el comentario nuevo
sin recargar toda la página.
```

## Variantes del comando

### `/explica [concepto]`
Explica un concepto técnico general.

### `/explica [archivo]`
Explica qué hace un archivo específico.

### `/explica [error]`
Explica qué significa un mensaje de error.

### `/explica arquitectura`
Explica cómo está organizado el proyecto.

## Formato de respuesta

Para errores:
```markdown
## Error: [mensaje de error]

### ¿Qué significa?
[Explicación en lenguaje sencillo]

### ¿Por qué ocurre?
[Causa común]

### ¿Cómo solucionarlo?
[Pasos para arreglarlo]
```

Para código:
```markdown
## Este código hace...

### Resumen
[Una frase explicando el propósito]

### Línea por línea
```ruby
# Esta línea busca al usuario por su email
user = User.find_by(email: params[:email])

# Esta comprueba si la contraseña es correcta
if user&.authenticate(params[:password])
```

### En contexto
[Cómo encaja en el resto de la aplicación]
```

## Notas importantes

- Siempre usar lenguaje sencillo
- Incluir analogías del mundo real
- Evitar acrónimos sin explicarlos
- Si algo es complejo, dividirlo en partes
- Usar ejemplos concretos del proyecto
- Preguntar si la explicación fue clara
