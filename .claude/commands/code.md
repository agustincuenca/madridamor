# Comando: /code

Ejecuta el plan de implementación de una tarea.

## Argumentos
- `task_path` (requerido): Path de la tarea (ej. features/2025-12-19-143052/tasks/001-crear-comentario)

## Flujo de trabajo

### Paso 1: Leer el Plan

1. Cargar `{task_path}/plan.md`
2. Si no existe, mostrar error y sugerir `/plan {task_path}` primero
3. Extraer el feature_id del path

### Paso 2: Verificar Dependencias

1. Cargar `features/{feature_id}/feature.json`
2. Verificar que las tareas de las que depende están completadas
3. Si hay dependencias no completadas, alertar y preguntar si continuar

### Paso 3: Implementar

1. Seguir los pasos del plan en orden
2. Para cada paso:
   - Mostrar qué se está haciendo
   - Ejecutar la implementación
   - Validar que funcionó
3. Invocar sub-agentes según necesidad:
   - `rails-dev` para modelos/controllers/migraciones
   - `frontend-dev` para vistas/Stimulus/Tailwind
   - `qa` para tests

### Paso 4: Ejecutar Validaciones

Ejecutar los comandos de validación del plan:

```bash
# Tests
bundle exec rspec

# Linting
bundle exec standard

# ERB linting (si existe)
bundle exec erblint --lint-all 2>/dev/null || true

# Compilar assets (si existe yarn)
yarn build 2>/dev/null || true
```

### Paso 5: Actualizar Estado

1. Cargar `features/{feature_id}/feature.json`
2. Encontrar la tarea en el array `tasks`
3. Cambiar `status` a `"completed"`
4. Recalcular `progress` del feature:
   ```
   progress = (tareas_completadas / total_tareas) * 100
   ```
5. Si todas las tareas están completadas:
   - Cambiar `status` del feature a `"completed"`
   - Cambiar `current_phase` a `"done"`
6. De lo contrario:
   - Cambiar `status` del feature a `"in_progress"`
7. Actualizar `updated_at`
8. Guardar el archivo

### Paso 6: Actualizar Historial

Agregar entrada a `.claude/history/changelog.md`:

```markdown
### {fecha} - Tarea Completada

**Feature**: {feature_title}
**Tarea**: {task_id} - {task_title}

Cambios realizados:
- {lista de archivos creados/modificados}

Progreso del feature: {X}% ({N}/{M} tareas)
```

## Report

```
Tarea completada: {task_title}

Resumen del trabajo:
- {bullet point 1}
- {bullet point 2}
- {bullet point 3}

Archivos modificados:
{git diff --stat output}

Validaciones:
- Tests: {PASS/FAIL}
- Linting: {PASS/FAIL}

Progreso del feature: {X}% ({N}/{M} tareas)

{SI hay más tareas pendientes}
Siguiente paso:
/plan features/{feature_id}/tasks/{siguiente_tarea}

{SI el feature está completo}
Feature completado! Todas las tareas han sido implementadas.
```

## Manejo de Errores

### Si algo falla durante la implementación

1. No marcar la tarea como completada
2. Mostrar el error claramente
3. Sugerir opciones:
   - Reintentar el paso
   - Modificar el plan
   - Pedir ayuda

### Si los tests fallan

1. Mostrar qué tests fallaron
2. Intentar arreglar automáticamente si es posible
3. Si no se puede arreglar, dejar la tarea como `in_progress`

## Integración con el Sistema

- Usa los mismos sub-agentes que `/añadir`
- Actualiza el mismo changelog que el resto del sistema
- Compatible con `/deshacer` para revertir cambios
- El progreso se refleja en `/estado`

## Consideraciones

- Siempre seguir el plan paso a paso
- No saltarse validaciones
- Preguntar si algo no está claro en el plan
- Mantener commits atómicos si es posible
- Documentar cualquier desviación del plan
