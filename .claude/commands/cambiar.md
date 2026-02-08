# Comando: /cambiar

Modifica algo existente en el proyecto.

## Flujo de trabajo

### Paso 1: Entender el cambio

1. Escucha qué quiere cambiar el usuario
2. Identifica el tipo de cambio:
   - **Visual**: Colores, tamaños, disposición
   - **Funcional**: Comportamiento, lógica
   - **Estructural**: Modelos, relaciones
   - **Contenido**: Textos, etiquetas

3. Pregunta para clarificar:
   - ¿Qué está mal con cómo funciona ahora?
   - ¿Cómo debería funcionar/verse en su lugar?
   - ¿Esto afecta a otras partes de la app?

### Paso 2: Localizar el código

1. Busca los archivos afectados
2. Lee el código actual para entender cómo funciona
3. Identifica todas las dependencias y efectos secundarios

### Paso 3: Planificar el cambio

1. Explica al usuario qué archivos se modificarán
2. Describe el impacto del cambio
3. Si hay riesgos, adviértelos
4. Confirma antes de proceder

### Paso 4: Hacer backup (si es cambio mayor)

Si el cambio es significativo:

1. Guarda el estado actual en `.claude/history/legacy/`:
   ```
   .claude/history/legacy/[timestamp]_[descripción]/
   ├── archivos_modificados/
   └── descripción.md
   ```

### Paso 5: Implementar el cambio

1. Realiza las modificaciones necesarias
2. Mantén los tests actualizados
3. Verifica que nada más se rompa

### Paso 6: Verificar

1. Ejecuta los tests: `bundle exec rspec`
2. Revisa visualmente si es un cambio de UI
3. Prueba flujos relacionados

### Paso 7: Documentar

Actualiza `.claude/history/changelog.md`:
```markdown
## [Fecha] - Modificado: [Qué se cambió]

### Motivo
[Por qué se hizo el cambio]

### Cambios realizados
- Archivo: [ruta] - [descripción del cambio]

### Estado anterior
[Breve descripción de cómo era antes]
```

### Paso 8: Confirmar

Muestra al usuario:
- Qué se cambió
- Antes vs después (si aplica)
- Cómo verificar el cambio

## Tipos comunes de cambios

### Cambios visuales
- Colores: Modificar variables en Tailwind o clases
- Tamaños: Ajustar clases de Tailwind
- Layout: Reorganizar estructura HTML

### Cambios funcionales
- Validaciones: Modificar modelos
- Comportamiento: Ajustar controladores o Stimulus
- Flujos: Cambiar redirecciones o lógica

### Cambios de datos
- Campos: Crear migración para añadir/modificar/eliminar
- Relaciones: Actualizar modelos y migraciones

## Ejemplos de uso

- "El botón debería ser verde, no azul"
- "Quiero que el título sea más grande"
- "El formulario debería tener un campo más"
- "Los artículos deberían ordenarse por fecha, no alfabéticamente"
- "Cambia el texto de 'Enviar' a 'Publicar'"

## Notas importantes

- Siempre lee el código antes de modificarlo
- Haz backup de cambios importantes
- Actualiza los tests si cambia el comportamiento
- Confirma cambios grandes antes de implementarlos
- Si el cambio es muy complejo, sugiere dividirlo en pasos
