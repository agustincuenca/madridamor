# Plan: {Título de la tarea}

## Metadata
task_path: `{task_path}`
feature_id: `{feature_id}`
created_at: `{timestamp}`
status: `planned`

## Análisis de Código Existente

### Búsqueda Realizada
{Lista de archivos y patrones encontrados relacionados con esta tarea}

### Matriz de Impacto (OBLIGATORIO)

| Componente | Archivo Existente | Líneas | Impacto |
|------------|-------------------|--------|---------|
| {componente 1} | {archivo} | {líneas} | CREAR/MODIFICAR/EXTENDER |
| {componente 2} | {archivo} | {líneas} | CREAR/MODIFICAR/EXTENDER |

**Archivos Nuevos Requeridos**: {count}
**Archivos a Modificar**: {count}

### Evaluación de Patrones

{Describir patrones encontrados y cómo se seguirán:
- Convenciones de naming
- Estructura de archivos
- Patrones de diseño en uso}

### Matriz de Conflictos

| Tipo | Recurso | Otra Tarea | Resolución |
|------|---------|------------|------------|
| {tipo} | {recurso} | {tarea} | {resolución} |

**Conflictos Encontrados**: {count} (Si > 0, documentar resolución)

## Resumen
{Descripción clara de qué se va a implementar y cómo}

## Historia de Usuario
**Como** {tipo de usuario}
**Quiero** {acción}
**Para** {beneficio}

## Archivos a Modificar
- `{ruta/archivo1.rb}` - {razón de la modificación}
- `{ruta/archivo2.rb}` - {razón de la modificación}

## Archivos a Crear
- `{ruta/nuevo1.rb}` - {propósito del archivo}
- `{ruta/nuevo2.rb}` - {propósito del archivo}

## Plan de Implementación

### Fase 1: Fundamentos
{Trabajo base necesario antes de la funcionalidad principal}

### Fase 2: Implementación Principal
{El núcleo de la funcionalidad}

### Fase 3: Integración
{Cómo se conecta con el resto del sistema}

## Pasos de Implementación

IMPORTANTE: Ejecutar cada paso en orden.

### 0. Refactorización Previa (SI SE ENCONTRARON VIOLACIONES)

**Saltar esta sección si no se encontraron violaciones de diseño**

Para cada violación detectada:

#### Violación 1: {descripción}
- **Archivo fuente**: {archivo}:{líneas} - Acción: {MANTENER|ELIMINAR|MODIFICAR}
- **Cambios específicos**:
  - [ ] Eliminar líneas X-Y de {archivo}
  - [ ] Mover lógica a {nuevo_archivo}
  - [ ] Actualizar referencias

### 1. {Nombre del paso}
- {Subtarea detallada}
- {Subtarea detallada}
- {Comando o acción específica si aplica}

### 2. {Nombre del paso}
- {Subtarea detallada}
- {Subtarea detallada}

### 3. {Nombre del paso - Tests}
- Crear tests para {funcionalidad}
- Verificar criterios de aceptación

### 4. Validación Final
- Ejecutar comandos de validación
- Verificar que todos los tests pasan

## Criterios de Aceptación
- [ ] {Criterio 1 del user-story}
- [ ] {Criterio 2 del user-story}
- [ ] {Criterio 3 del user-story}

## Comandos de Validación

```bash
# Ejecutar tests
bundle exec rspec

# Verificar formato
bundle exec standard

# Verificar templates ERB
bundle exec erblint --lint-all

# Compilar assets
yarn build
```

## Notas
{Consideraciones adicionales, advertencias, o decisiones de diseño}
