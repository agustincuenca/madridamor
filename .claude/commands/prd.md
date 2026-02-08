# Comando: /prd

Genera un documento PRD (Product Requirements Document) estructurado a partir de la descripción del feature siguiendo el **Principio de Fuente Única de Verdad**.

## Argumentos
- `feature_id` (requerido): ID del feature (ej. 2025-12-19-143052-mi-feature)

## Principio: Fuente Única de Verdad

Cada PRD debe cumplir:

| Aspecto | Requisito |
|---------|-----------|
| **Único** | No duplicar requisitos de otros PRDs existentes |
| **Completo** | Toda la información del feature en un solo lugar |
| **Trazable** | Cada requisito con ID único para referenciar |
| **Verificable** | Requisitos específicos que se pueden probar |
| **Delimitado** | Alcance claro de qué incluye y qué excluye |

### Detección de Violaciones

**CRÍTICO**: Al analizar PRDs existentes, detectar activamente:

1. **Requisitos Duplicados**: Mismo requisito en múltiples PRDs
2. **Scope Overlap**: Funcionalidades que se solapan con otros features
3. **Dependencias Ocultas**: Requisitos que dependen de otros PRDs sin documentar
4. **Conflictos**: Requisitos que contradicen otros PRDs

**Si se encuentran violaciones, el PRD DEBE documentarlas y proponer resolución.**

## Flujo de trabajo

### Fase 1: Localizar y Validar el Feature

1. Buscar en `features/{feature_id}/`
2. Verificar que existe `feature.json`
3. Si no existe, mostrar error y sugerir `/feature` primero
4. Cargar `feature.json` y extraer `title` y `description`

### Fase 2: Análisis de PRDs Existentes (OBLIGATORIO)

1. Listar todos los `features/*/prd.md` existentes
2. Para cada PRD, extraer:
   - Requisitos funcionales (RF-XX)
   - Alcance (incluido/excluido)
   - Flujos de usuario

**Crear matriz de solapamiento:**
```
| Aspecto | Este PRD | PRD Existente | Conflicto? |
|---------|----------|---------------|------------|
| Login flow | RF-01 | feature-abc/RF-03 | REVISAR |
| User model | RF-02 | feature-xyz/RF-01 | NO |
```

### Fase 3: Investigar el Proyecto

1. **Leer `CLAUDE.md`**: Convenciones del proyecto
2. **Revisar estructura**: Modelos, controladores, vistas existentes
3. **Identificar patrones**: Cómo se implementan features similares
4. **Detectar constraints**: Limitaciones técnicas o de negocio

### Fase 4: Generar el PRD

Escribir el PRD en `features/{feature_id}/prd.md` usando el template de `.claude/templates/prd-template.md`

### Fase 5: Actualizar Estado del Feature

1. Cargar `features/{feature_id}/feature.json`
2. Cambiar `status` a `"prd_created"`
3. Cambiar `current_phase` a `"prd"`
4. Actualizar `updated_at`
5. Guardar el archivo

### Fase 6: Validar el PRD Creado

```bash
# Verificar que el PRD existe
test -f features/{feature_id}/prd.md

# Verificar secciones obligatorias
grep -q "## Metadata" features/{feature_id}/prd.md
grep -q "## Requisitos" features/{feature_id}/prd.md
grep -q "### Funcionales" features/{feature_id}/prd.md
grep -q "## Alcance" features/{feature_id}/prd.md
```

## Report

```
PRD creado exitosamente!

Feature: {título}
Archivo: features/{feature_id}/prd.md

Resumen del PRD:
- {N} requisitos funcionales identificados
- {M} requisitos no funcionales
- Alcance definido con {P} funcionalidades incluidas
- {S} solapamientos detectados con otros PRDs

Siguiente paso:
/tasks {feature_id}
```

## Consideraciones

- El PRD debe ser comprensible para personas no técnicas
- Usar lenguaje claro y evitar jerga técnica innecesaria
- Los requisitos deben ser específicos y verificables
- El alcance debe ser realista y bien delimitado
- NO duplicar requisitos de otros PRDs - referenciar en su lugar
- Documentar todas las dependencias con otros features
