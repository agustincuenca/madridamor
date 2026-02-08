# Comando: /añadir

Añade una nueva funcionalidad al proyecto existente.

## Flujo de trabajo

### Paso 1: Entender la solicitud

Actúa como el **Product Owner**:

1. Escucha la funcionalidad que el usuario quiere añadir
2. Haz preguntas clarificadoras si es necesario:
   - ¿Quién usará esta funcionalidad?
   - ¿Qué problema resuelve?
   - ¿Cómo debería funcionar desde la perspectiva del usuario?
   - ¿Hay algún ejemplo o referencia?

3. Resume la funcionalidad en términos sencillos y confirma

### Paso 2: Análisis técnico

Actúa como el **Tech Lead**:

1. Lee los archivos de contexto existentes:
   - `.claude/contexts/requirements.md`
   - `.claude/contexts/architecture.md`
   - `.claude/history/active/models.md`

2. Determina:
   - ¿Se necesitan nuevos modelos?
   - ¿Se modifican modelos existentes?
   - ¿Qué controladores y vistas se requieren?
   - ¿Se necesitan jobs en background?

3. Presenta el plan técnico de forma sencilla al usuario

### Paso 3: Diseño

Actúa como el **Design Lead**:

1. Define cómo se verá la nueva funcionalidad
2. Asegura consistencia con el diseño existente
3. Considera la experiencia móvil

### Paso 4: Implementación

Coordina a los sub-agentes necesarios:

1. **Rails Dev**: Modelos, migraciones, controladores
2. **Frontend Dev**: Vistas, Hotwire, estilos
3. **QA**: Tests para la nueva funcionalidad

### Paso 5: Pruebas

1. Ejecuta los tests: `bundle exec rspec`
2. Verifica que todo funciona correctamente
3. Prueba la funcionalidad manualmente si es necesario

### Paso 6: Documentación

1. Actualiza `.claude/history/changelog.md` con:
   ```markdown
   ## [Fecha] - Añadido: [Nombre de funcionalidad]

   ### Descripción
   [Qué se añadió y por qué]

   ### Cambios
   - Nuevo modelo: [nombre]
   - Nuevas vistas: [rutas]
   - Nuevos tests: [archivos]
   ```

2. Actualiza los archivos de contexto si hubo cambios estructurales

### Paso 7: Resumen

Presenta al usuario:
- Lo que se implementó
- Cómo usar la nueva funcionalidad
- Tests añadidos
- Próximos pasos sugeridos (si aplica)

## Ejemplos de uso

El usuario puede decir cosas como:
- "Quiero añadir comentarios a los artículos"
- "Me gustaría que los usuarios pudieran subir fotos"
- "Necesito un sistema de notificaciones"
- "Añade la opción de marcar favoritos"

## Notas importantes

- Lee siempre el contexto existente antes de hacer cambios
- Mantén consistencia con el código existente
- Añade tests para toda funcionalidad nueva
- Si el cambio es grande, divídelo en pasos más pequeños
- Pregunta si no estás seguro de algo
