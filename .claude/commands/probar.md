# Comando: /probar

Prueba la aplicación como si fueras un usuario real.

## Flujo de trabajo

### Paso 1: Verificar que la app está corriendo

```bash
# Verificar si hay un servidor corriendo
curl -s http://localhost:3000/up || echo "El servidor no está corriendo"
```

Si no está corriendo, sugerir:
```bash
bin/dev
```

### Paso 2: Identificar qué probar

Preguntar al usuario:
- "¿Quieres que pruebe algo específico o hago una prueba general?"

#### Prueba general
Probar los flujos principales de la aplicación.

#### Prueba específica
Probar la funcionalidad que el usuario indique.

### Paso 3: Ejecutar pruebas automatizadas

```bash
# Correr todos los tests
bundle exec rspec

# Si hay tests específicos
bundle exec rspec spec/features/
bundle exec rspec spec/system/
```

### Paso 4: Pruebas manuales (CUA)

Usar el skill de Computer Use Agent para navegar la app visualmente:

#### Prueba de registro/login
1. Ir a la página de registro
2. Completar el formulario
3. Verificar que el registro funciona
4. Cerrar sesión
5. Iniciar sesión con las credenciales

#### Prueba de funcionalidades principales
1. Navegar por las páginas principales
2. Crear contenido (si aplica)
3. Editar contenido
4. Eliminar contenido
5. Verificar que todo funciona

### Paso 5: Pruebas de responsive

1. Probar en viewport de escritorio (1920px)
2. Probar en viewport de tablet (768px)
3. Probar en viewport móvil (375px)

### Paso 6: Pruebas de accesibilidad

1. Verificar navegación con teclado
2. Verificar contraste de colores
3. Verificar que los formularios tienen labels

### Paso 7: Generar reporte

```markdown
# Reporte de Pruebas
Fecha: [fecha]

## Resumen
- Tests automatizados: ✅ X pasaron / ❌ Y fallaron
- Tests manuales: ✅ X OK / ❌ Y problemas encontrados

## Tests Automatizados
[Salida de RSpec]

## Tests Manuales

### Flujo de autenticación
- [✅/❌] Registro de usuario
- [✅/❌] Inicio de sesión
- [✅/❌] Cierre de sesión
- [✅/❌] Recuperación de contraseña

### Funcionalidades principales
- [✅/❌] [Funcionalidad 1]
- [✅/❌] [Funcionalidad 2]

### Responsive
- [✅/❌] Desktop
- [✅/❌] Tablet
- [✅/❌] Móvil

### Accesibilidad
- [✅/❌] Navegación con teclado
- [✅/❌] Contraste de colores
- [✅/❌] Labels en formularios

## Problemas Encontrados

### Problema 1: [Título]
- **Severidad**: Alta/Media/Baja
- **Descripción**: [Qué pasa]
- **Pasos para reproducir**: [Cómo reproducirlo]
- **Comportamiento esperado**: [Qué debería pasar]

## Recomendaciones
- [Recomendación 1]
- [Recomendación 2]
```

### Paso 8: Presentar resultados

Mostrar al usuario:
1. Resumen de resultados
2. Problemas encontrados (si los hay)
3. Recomendaciones de mejora
4. Ofrecer arreglar los problemas encontrados

## Tipos de pruebas

### Por funcionalidad
- `/probar login` - Prueba solo autenticación
- `/probar formularios` - Prueba todos los formularios
- `/probar navegación` - Prueba menús y enlaces

### Por página
- `/probar /articles` - Prueba la página de artículos
- `/probar /users/1` - Prueba el perfil de usuario

### Por dispositivo
- `/probar móvil` - Prueba responsive en móvil
- `/probar tablet` - Prueba responsive en tablet

## Notas importantes

- Asegurarse de que el servidor está corriendo
- Usar datos de prueba, no datos reales
- Documentar todos los problemas encontrados
- Ofrecer soluciones para los problemas
- Mantener las pruebas actualizadas con los cambios
