# Comando: /deshacer

Revierte el último cambio realizado en el proyecto.

## Flujo de trabajo

### Paso 1: Identificar el último cambio

1. Lee `.claude/history/changelog.md`
2. Identifica la entrada más reciente
3. Muestra al usuario qué se va a deshacer:
   ```
   El último cambio fue:
   - Fecha: [fecha]
   - Descripción: [qué se hizo]
   - Archivos afectados: [lista]

   ¿Quieres deshacer este cambio?
   ```

### Paso 2: Confirmar con el usuario

**Importante**: Siempre pedir confirmación antes de deshacer.

Si el usuario confirma, continuar. Si no, preguntar qué cambio específico quiere deshacer.

### Paso 3: Verificar disponibilidad de backup

1. Buscar en `.claude/history/legacy/` si existe backup del estado anterior
2. Si existe, usarlo para restaurar
3. Si no existe, intentar revertir manualmente

### Paso 4: Revertir el cambio

#### Si hay backup:
```bash
# Restaurar archivos desde backup
cp -r .claude/history/legacy/[timestamp]/* [destino]
```

#### Si es un cambio de código simple:
1. Identificar los archivos modificados
2. Revertir cada archivo a su estado anterior

#### Si es una migración de base de datos:
```bash
# Revertir última migración
rails db:rollback

# Si son varias migraciones
rails db:rollback STEP=n
```

### Paso 5: Verificar

1. Ejecutar tests: `bundle exec rspec`
2. Verificar que la app funciona
3. Confirmar visualmente si aplica

### Paso 6: Actualizar historial

Añadir entrada en `.claude/history/changelog.md`:
```markdown
## [Fecha] - Revertido: [Descripción del cambio deshecho]

### Motivo
[Por qué se deshizo - si el usuario lo especificó]

### Estado restaurado
La aplicación ha vuelto al estado anterior a [descripción del cambio].
```

### Paso 7: Confirmar al usuario

```
✅ Cambio deshecho exitosamente.

Se ha revertido: [descripción]
La aplicación está ahora en el estado de [fecha/descripción del estado anterior].
```

## Casos especiales

### Cambios que no se pueden deshacer fácilmente

Algunos cambios son difíciles de revertir:
- Eliminación de datos de la base de datos
- Cambios en archivos que luego fueron modificados de nuevo
- Múltiples cambios interdependientes

En estos casos:
1. Explicar la situación al usuario
2. Ofrecer alternativas:
   - Crear una nueva versión que "arregle" el problema
   - Restaurar desde un backup de base de datos si existe
   - Recrear manualmente el estado anterior

### Deshacer varios cambios

Si el usuario quiere deshacer más de un cambio:

1. Mostrar lista de cambios recientes
2. Preguntar cuántos quiere deshacer
3. Advertir sobre posibles conflictos
4. Proceder paso a paso, confirmando cada uno

## Ejemplos de uso

- "Deshaz el último cambio"
- "Vuelve atrás"
- "Eso no me gusta, déjalo como estaba"
- "Revierte los cambios de hoy"

## Notas importantes

- **Siempre confirmar** antes de deshacer
- Mantener el historial actualizado
- Si no hay backup, ser honesto sobre las limitaciones
- Para cambios de base de datos, verificar que no haya pérdida de datos
- Si el cambio afectó múltiples archivos, revertir todos juntos
