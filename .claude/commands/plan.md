# Comando: /plan

Genera un plan detallado para implementar una tarea específica siguiendo el **Principio de Responsabilidad Única**.

## Argumentos
- `task_path` (requerido): Path de la tarea (ej. features/2025-12-19-143052/tasks/001-crear-comentario)

## Principio: Responsabilidad Única

Cada tarea planificada debe cumplir:

| Aspecto | Requisito |
|---------|-----------|
| **Una Funcionalidad** | Una capacidad por tarea - no agrupar |
| **Auto-contenido** | Sin dependencias de código no commiteado |
| **Trazabilidad** | Nombres claros para vincular a código/tests |
| **Testeable** | Validación pass/fail posible |
| **Reversible** | Puede revertirse sin romper otros componentes |

### Detección de Violaciones de Diseño

**CRÍTICO**: Al analizar el código existente, detectar activamente:

1. **Código Duplicado**: Misma lógica implementada en múltiples lugares
2. **Scope Creep**: Componente que hace más de lo que debería
3. **Ejemplos Redundantes**: Mismos patrones usados inconsistentemente
4. **Conflictos de Responsabilidad**: Múltiples componentes modificando lo mismo

**Si se encuentran violaciones, el plan DEBE incluir tareas de refactorización.**

## Flujo de trabajo

### Fase 1: Localizar y Validar la Tarea

1. Verificar que existe `{task_path}/user-story.md`
2. Si no existe, mostrar error con el path correcto
3. Extraer el feature_id del path (segundo segmento)
4. Cargar `features/{feature_id}/feature.json` para contexto
5. Cargar `features/{feature_id}/prd.md` para requisitos

### Fase 2: Análisis del Feature

1. **Leer `CLAUDE.md`**: Convenciones del proyecto
2. **User Story**: Cargar `{task_path}/user-story.md`
3. **Criterios de Aceptación**: Identificar todos los escenarios
4. **PRD del Feature**: Entender el contexto completo

### Fase 3: Análisis de Código Existente (OBLIGATORIO)

1. **Buscar implementaciones similares** en el codebase
2. **Identificar patrones** que se deben seguir
3. **Detectar código que necesita modificación** vs creación

**Crear una matriz de impacto:**
```
| Componente | Archivo Existente | Líneas | Impacto |
|------------|-------------------|--------|---------|
| Model | app/models/x.rb | 45-67 | MODIFICAR |
| Controller | app/controllers/y.rb | 12-30 | MODIFICAR |
| View | app/views/z/index.html.erb | N/A | CREAR |
| Test | spec/models/x_spec.rb | 80-95 | EXTENDER |
```

### Fase 4: Detección de Conflictos (OBLIGATORIO)

Verificar conflictos potenciales:

1. **Conflictos de Archivo**: Otras tareas pendientes modificando los mismos archivos
2. **Conflictos de Schema**: Migraciones que podrían conflicturar
3. **Conflictos de Rutas**: Patrones de URL que se solapan
4. **Conflictos de Tests**: Fixtures o datos de test compartidos

**Crear matriz de conflictos:**
```
| Tipo | Recurso | Otra Tarea | Resolución |
|------|---------|------------|------------|
| Archivo | users_controller.rb | task-002 | Secuenciar |
| Schema | add_column :users | task-003 | Combinar migración |
```

### Fase 5: Evaluación de Atomicidad

Determinar si esta tarea debe ser:
- Ejecutada como unidad única
- Dividida en sub-tareas más pequeñas
- Combinada con otra tarea relacionada

### Fase 6: Generar el Plan

Escribir el plan en `{task_path}/plan.md` usando el template de `.claude/templates/task-plan.md`

### Fase 7: Actualizar Estado de la Tarea

1. Cargar `features/{feature_id}/feature.json`
2. Encontrar la tarea en el array `tasks`
3. Cambiar `status` a `"planned"`
4. Actualizar `updated_at` del feature
5. Guardar el archivo

### Fase 8: Validar el Plan Creado

```bash
# Verificar que el plan existe
test -f {task_path}/plan.md

# Verificar secciones obligatorias
grep -q "## Metadata" {task_path}/plan.md
grep -q "## Matriz de Impacto" {task_path}/plan.md
grep -q "## Pasos de Implementación" {task_path}/plan.md
grep -q "## Criterios de Aceptación" {task_path}/plan.md
```

## Report

```
Plan creado: {task_path}/plan.md

Resumen:
- Archivos a crear: {N}
- Archivos a modificar: {M}
- Conflictos detectados: {C}

Siguiente paso:
/code {task_path}
```

## Consideraciones

- El plan debe ser lo suficientemente detallado para ejecutarse sin ambigüedad
- Seguir las convenciones y patrones existentes en el proyecto
- Incluir tests desde el principio (TDD cuando sea apropiado)
- Los comandos de validación deben ejecutarse sin errores al finalizar
- Documentar cualquier conflicto o dependencia encontrada
- NO duplicar lógica existente - reutilizar siempre que sea posible
