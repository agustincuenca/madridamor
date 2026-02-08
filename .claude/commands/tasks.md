# Comando: /tasks

Crea tareas (historias de usuario) a partir del PRD del feature siguiendo el **Principio de Independencia**.

## Argumentos
- `feature_id` (requerido): ID del feature (ej. 2025-12-19-143052-mi-feature)

## Principio: Independencia de Tareas

Cada tarea debe cumplir:

| Aspecto | Requisito |
|---------|-----------|
| **Independiente** | Puede implementarse sin depender de tareas no completadas |
| **Atómica** | Una sola funcionalidad por tarea |
| **Verificable** | Criterios de aceptación claros y testeables |
| **Estimable** | Tamaño razonable para una sesión de trabajo |
| **Ordenada** | Prioridad clara basada en dependencias y valor |

### Detección de Violaciones

**CRÍTICO**: Al generar tareas, detectar activamente:

1. **Dependencias Circulares**: Tarea A depende de B, B depende de A
2. **Tareas Muy Grandes**: Más de 5 criterios de aceptación
3. **Tareas Duplicadas**: Misma funcionalidad en diferentes tareas
4. **Conflictos con Otros Features**: Tareas que modifican los mismos archivos

**Si se encuentran violaciones, dividir o reorganizar las tareas.**

## Flujo de trabajo

### Fase 1: Localizar y Validar el Feature

1. Buscar en `features/{feature_id}/`
2. Verificar que existe `feature.json` y `prd.md`
3. Si falta el PRD, mostrar error y sugerir `/prd {feature_id}` primero

### Fase 2: Análisis de Tareas Existentes (OBLIGATORIO)

1. Listar todas las tareas en `features/*/tasks/*/`
2. Identificar tareas con status `defined`, `planned`, `in_progress`
3. Extraer archivos que planean modificar (de sus plan.md si existen)

**Crear matriz de conflictos potenciales:**
```
| Archivo/Recurso | Este Feature | Otro Feature | Conflicto? |
|-----------------|--------------|--------------|------------|
| users_controller.rb | Tarea 001 | feature-abc/002 | REVISAR |
```

### Fase 3: Leer y Analizar el PRD

1. Cargar `features/{feature_id}/prd.md`
2. Identificar los requisitos funcionales (RF-XX)
3. Entender el flujo de usuario
4. Identificar dependencias entre funcionalidades
5. Revisar el alcance (incluido vs excluido)

### Fase 4: Crear Carpeta de Tareas

```bash
mkdir -p features/{feature_id}/tasks/
```

### Fase 5: Generar Tareas

Aplicar criterios:

1. **Una tarea por requisito funcional principal**
2. **Ordenar por dependencias** (las base primero)
3. **IDs secuenciales**: 001, 002, 003...
4. **Slugs descriptivos**: `crear-comentario`, `responder-comentario`

### Fase 6: Crear Estructura de Cada Tarea

Para cada tarea, crear:
```
features/{feature_id}/tasks/{id}-{slug}/
features/{feature_id}/tasks/{id}-{slug}/user-story.md
```

Usar el template de `.claude/templates/task-user-story.md`

### Fase 7: Actualizar feature.json

1. Agregar array `tasks` con cada tarea:
   ```json
   {
     "id": "001",
     "slug": "crear-comentario",
     "title": "Crear comentario en artículo",
     "status": "defined",
     "priority": 1,
     "requisito": "RF-01",
     "depends_on": []
   }
   ```
2. Cambiar `status` a `"tasks_created"`
3. Cambiar `current_phase` a `"tasks"`
4. Actualizar `updated_at`

### Fase 8: Validar Tareas Creadas

```bash
# Verificar que existe al menos una tarea
ls features/{feature_id}/tasks/*/user-story.md | wc -l

# Verificar tamaño de tareas (máximo 5 criterios por tarea)
for story in features/{feature_id}/tasks/*/user-story.md; do
  count=$(grep -c "^\- \[ \]" "$story")
  if [ "$count" -gt 5 ]; then
    echo "WARNING: $story tiene $count criterios (máximo recomendado: 5)"
  fi
done
```

## Criterios de Priorización

| Prioridad | Descripción | Ejemplo |
|-----------|-------------|---------|
| **1** | Funcionalidad base necesaria para otras tareas | Modelo de datos |
| **2** | Funcionalidad de alto valor para el usuario | CRUD principal |
| **3** | Funcionalidades complementarias o de mejora | Validaciones extra |
| **4** | Funcionalidades opcionales o de bajo impacto | Mejoras de UX |

## Report

```
Tareas generadas exitosamente!

Feature: {título}
Total: {N} tareas creadas

TAREAS:
001 - {título tarea 1} [defined] (RF-01) P1
     Depende de: ninguna
002 - {título tarea 2} [defined] (RF-02) P1
     Depende de: 001
003 - {título tarea 3} [defined] (RF-03) P2
     Depende de: 001, 002

Conflictos detectados: {M}
- {descripción del conflicto si hay}

Siguiente paso:
/plan features/{feature_id}/tasks/001-{slug}
```

## Consideraciones

- Las tareas deben ser independientes cuando sea posible
- Si hay dependencias, la tarea dependiente debe tener prioridad mayor (número mayor)
- Cada tarea debe ser lo suficientemente pequeña para completarse en una sesión
- Los criterios de aceptación deben ser verificables objetivamente
- Documentar conflictos con tareas de otros features
- Máximo 5 criterios de aceptación por tarea (dividir si hay más)
