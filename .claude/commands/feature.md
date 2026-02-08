# Comando: /feature

Gestiona features del proyecto: crea nuevos, lista existentes, o detecta automáticamente el trabajo pendiente.

## Argumentos
- `description` (opcional): Descripción del feature a crear

## Comportamiento

### Sin argumentos: Listar y Recomendar

1. Buscar `features/*/feature.json`
2. Mostrar cada feature con: ID, título, estado, progreso, lista de tareas
3. **Detectar siguiente acción** para features en progreso:

| Estado de Tarea | Siguiente Comando |
|-----------------|-------------------|
| `defined` | `/plan {task_path}` |
| `planned` | `/code {task_path}` |
| `in_progress` | `/code {task_path}` |

Buscar la primera tarea no completada (ordenadas por priority) y recomendar el comando apropiado.

**Ejemplo de salida:**
```
FEATURES DEL PROYECTO
=====================

ID: 2025-12-19-175523-crm-rails
Título: CRM con Rails y Tailwind
Estado: in_progress
Progreso: [███████░░░░░░░░░] 43% (3/7 tareas)

  Tareas:
  ✓ 001 - Setup del Proyecto
  ✓ 002 - CRUD de Empresas
  ✓ 003 - CRUD de Contactos
  ○ 004 - CRUD de Oportunidades (planned)    ← PRIMERA PENDIENTE
  ○ 005 - Dashboard con Métricas (defined)
  ○ 006 - Búsqueda (defined)
  ○ 007 - Vista Pipeline (defined)

═══════════════════════════════════════════════════════════════
SIGUIENTE ACCIÓN:

/code features/.../tasks/004-crud-oportunidades
═══════════════════════════════════════════════════════════════
```

**Si no hay features:**
```
No hay features creados todavía.

Para crear uno nuevo:
/feature "Descripción de lo que quieres construir..."
```

### Con descripción: Crear Feature

1. **Verificar solapamiento** con features existentes (revisar títulos, descripciones, PRDs)
   - Si hay solapamiento potencial, alertar y pedir confirmación
2. **Generar ID**: `{YYYY-MM-DD-hhmmss}-{slug}`
   - Slug: palabras clave, lowercase, sin acentos, guiones, max 30 chars
3. **Crear estructura**:
   ```
   features/{id}/
   features/{id}/feature.json
   ```
4. **Crear feature.json** usando template de `.claude/templates/feature.json`:
   ```json
   {
     "version": "1.0",
     "id": "{id}",
     "title": "{título inferido}",
     "description": "{descripción}",
     "original_request": "{texto exacto del usuario}",
     "created_at": "{ISO timestamp}",
     "updated_at": "{ISO timestamp}",
     "status": "created",
     "progress": 0,
     "current_phase": "initial",
     "tasks": []
   }
   ```

   **IMPORTANTE:** El campo `original_request` debe contener el texto EXACTO que el usuario proporcionó, sin modificaciones.

5. **Validar** que se creó correctamente
6. **Actualizar** `.claude/history/changelog.md`

## Report de Creación

```
Feature creado exitosamente!

## Metadata
id: `{id}`
created_at: `{timestamp}`

## Detalles
Título: {título}
Ubicación: features/{id}/

## Request Original
> {original_request}

## Análisis de Solapamiento
Features revisados: {N}
Solapamientos detectados: {M}
{lista de solapamientos si hay}

## Siguiente Paso
/prd {id}
```

## Notas importantes

- Siempre mostrar el siguiente paso recomendado
- Mantener el historial actualizado
- Detectar y alertar sobre solapamientos antes de crear
