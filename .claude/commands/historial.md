# Comando: /historial

Muestra el historial de cambios realizados en el proyecto.

## Flujo de trabajo

### Paso 1: Leer el historial

Leer `.claude/history/changelog.md` y archivos en `.claude/history/active/`

### Paso 2: Formatear la salida

```markdown
# Historial de Cambios

## Resumen
- Total de cambios registrados: X
- Primer cambio: [fecha]
- Último cambio: [fecha]

---

## Cambios Recientes

### [Fecha más reciente] - [Título]
**Tipo**: Añadido/Modificado/Eliminado
**Descripción**: [Descripción del cambio]
**Archivos afectados**:
- `path/to/file1.rb`
- `path/to/file2.erb`

---

### [Fecha anterior] - [Título]
**Tipo**: Añadido/Modificado/Eliminado
**Descripción**: [Descripción del cambio]
**Archivos afectados**:
- `path/to/file.rb`

---

[... más cambios ...]

## Estadísticas

| Mes | Cambios | Tipo más común |
|-----|---------|----------------|
| [Mes actual] | X | Añadidos |
| [Mes anterior] | Y | Modificaciones |
```

### Paso 3: Mostrar opciones

"¿Qué te gustaría hacer?"
- Ver más detalles de algún cambio
- Deshacer algún cambio
- Filtrar por fecha o tipo

## Variantes del comando

### `/historial [n]`
Muestra los últimos N cambios.
```
/historial 5  → Muestra los últimos 5 cambios
```

### `/historial [fecha]`
Muestra cambios de una fecha específica.
```
/historial 2024-01-15  → Cambios del 15 de enero
/historial enero       → Cambios de enero
/historial hoy         → Cambios de hoy
/historial ayer        → Cambios de ayer
/historial semana      → Cambios de esta semana
```

### `/historial [tipo]`
Filtra por tipo de cambio.
```
/historial añadidos     → Solo funcionalidades añadidas
/historial modificados  → Solo modificaciones
/historial eliminados   → Solo eliminaciones
```

### `/historial [archivo]`
Muestra historial de un archivo específico.
```
/historial app/models/user.rb  → Cambios en User model
```

## Formato del changelog

El archivo `.claude/history/changelog.md` debe seguir este formato:

```markdown
# Changelog

## [2024-01-15] - Añadido sistema de comentarios

### Tipo
Añadido

### Descripción
Se implementó un sistema de comentarios para los artículos. Los usuarios
autenticados pueden dejar comentarios en cualquier artículo publicado.

### Cambios
- Nuevo modelo: `Comment`
- Nueva migración: `create_comments`
- Nuevas vistas: `comments/_form`, `comments/_comment`
- Modificado: `Article` model (has_many :comments)
- Nuevos tests: `spec/models/comment_spec.rb`

### Archivos creados
- `app/models/comment.rb`
- `app/controllers/comments_controller.rb`
- `app/views/comments/_form.html.erb`
- `app/views/comments/_comment.html.erb`
- `db/migrate/XXXX_create_comments.rb`
- `spec/models/comment_spec.rb`

### Archivos modificados
- `app/models/article.rb`
- `app/views/articles/show.html.erb`
- `config/routes.rb`

---

## [2024-01-14] - Modificado estilo del header

### Tipo
Modificado

### Descripción
Se cambió el color del header de azul a verde según solicitud del usuario.

### Cambios
- Modificado: `app/views/layouts/application.html.erb`

### Estado anterior
```erb
<header class="bg-blue-600">
```

### Estado nuevo
```erb
<header class="bg-green-600">
```
```

## Estructura de archivos de historial

```
.claude/history/
├── changelog.md           # Log principal de cambios
├── active/               # Estado actual documentado
│   ├── models.md         # Modelos actuales
│   ├── controllers.md    # Controladores actuales
│   ├── views.md          # Vistas actuales
│   ├── features.md       # Funcionalidades actuales
│   └── architecture.md   # Arquitectura actual
└── legacy/               # Backups de estados anteriores
    ├── 20240115_pre_comments/
    │   └── ...
    └── 20240114_pre_header_change/
        └── ...
```

## Ejemplo de salida

```
📜 Historial del Proyecto

Últimos 5 cambios:

1. 🟢 Hace 2 horas - Añadido sistema de comentarios
   Archivos: +5 nuevos, 3 modificados

2. 🟡 Ayer - Modificado estilo del header
   Archivos: 1 modificado

3. 🟢 Hace 3 días - Añadida paginación
   Archivos: +2 nuevos, 4 modificados

4. 🔴 Hace 1 semana - Eliminado campo obsoleto
   Archivos: 2 modificados

5. 🟢 Hace 1 semana - Añadido perfil de usuario
   Archivos: +4 nuevos, 2 modificados

---
¿Quieres ver más detalles de algún cambio? (escribe el número)
¿O deshacer alguno? (escribe "deshacer [número]")
```

## Notas importantes

- Mantener el changelog actualizado con cada cambio
- Incluir suficiente detalle para poder deshacer si es necesario
- Guardar backups de cambios importantes en `legacy/`
- Usar fechas ISO (YYYY-MM-DD) para consistencia
- Incluir el "antes y después" en modificaciones importantes
