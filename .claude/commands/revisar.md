# Comando: /revisar

Analiza el código del proyecto para detectar problemas de calidad, seguridad y rendimiento.

## Flujo de trabajo

### Paso 1: Preguntar alcance

"¿Quieres que revise todo el proyecto o algo específico?"

Opciones:
- Todo el proyecto
- Un archivo específico
- Una funcionalidad
- Solo seguridad
- Solo rendimiento

### Paso 2: Ejecutar análisis automatizados

```bash
# Tests
bundle exec rspec --format documentation

# Seguridad (si está instalado brakeman)
bundle exec brakeman -q

# Auditoría de dependencias
bundle audit check --update
```

### Paso 3: Revisión manual de código

Usar el skill de Code Review para analizar:

#### Seguridad
- [ ] SQL injection
- [ ] XSS vulnerabilities
- [ ] CSRF protection
- [ ] Mass assignment
- [ ] Authentication/Authorization
- [ ] Secrets hardcoded

#### Rendimiento
- [ ] N+1 queries
- [ ] Missing indexes
- [ ] Inefficient queries
- [ ] Memory usage
- [ ] Caching opportunities

#### Calidad de código
- [ ] Convenciones de Rails
- [ ] DRY (Don't Repeat Yourself)
- [ ] Single Responsibility
- [ ] Naming clarity
- [ ] Code complexity

#### Tests
- [ ] Cobertura adecuada
- [ ] Tests significativos
- [ ] Edge cases cubiertos

### Paso 4: Generar reporte

```markdown
# Reporte de Revisión de Código
Fecha: [fecha]
Alcance: [todo el proyecto / archivo específico / etc.]

## Resumen Ejecutivo

| Área | Estado | Problemas |
|------|--------|-----------|
| Seguridad | 🟢/🟡/🔴 | X |
| Rendimiento | 🟢/🟡/🔴 | X |
| Calidad | 🟢/🟡/🔴 | X |
| Tests | 🟢/🟡/🔴 | X |

## Hallazgos de Seguridad

### 🔴 Críticos
[Ninguno / Lista de problemas]

### 🟡 Advertencias
[Ninguna / Lista de advertencias]

### ✅ Buenas prácticas encontradas
[Lista de cosas bien hechas]

## Hallazgos de Rendimiento

### Problemas encontrados
1. **[Título del problema]**
   - Ubicación: `archivo.rb:línea`
   - Problema: [Descripción]
   - Impacto: [Alto/Medio/Bajo]
   - Solución sugerida: [Cómo arreglarlo]

### Oportunidades de mejora
- [Sugerencia 1]
- [Sugerencia 2]

## Calidad de Código

### Problemas
- [Lista de problemas de calidad]

### Sugerencias
- [Lista de mejoras sugeridas]

## Cobertura de Tests

- Modelos: X/Y testeados
- Controladores: X/Y testeados
- Features: X/Y testeados

### Tests faltantes
- [Lista de áreas sin tests]

## Recomendaciones Prioritarias

1. **[Más urgente]**: [Descripción y cómo resolverlo]
2. **[Segundo]**: [Descripción y cómo resolverlo]
3. **[Tercero]**: [Descripción y cómo resolverlo]

## Acciones Sugeridas

- [ ] [Acción 1]
- [ ] [Acción 2]
- [ ] [Acción 3]
```

### Paso 5: Presentar resultados

Explicar al usuario:
1. Estado general del código
2. Problemas críticos (si los hay)
3. Mejoras recomendadas
4. Ofrecer implementar las correcciones

### Paso 6: Ofrecer correcciones

"¿Quieres que arregle alguno de estos problemas?"

Si el usuario acepta:
1. Priorizar por criticidad
2. Implementar correcciones una por una
3. Verificar que no se rompe nada
4. Actualizar el reporte

## Tipos de revisión

### `/revisar seguridad`
Enfocarse solo en vulnerabilidades de seguridad.

### `/revisar rendimiento`
Enfocarse solo en problemas de rendimiento.

### `/revisar [archivo]`
Revisar un archivo específico en detalle.

### `/revisar tests`
Analizar la cobertura y calidad de tests.

## Checklist rápido de seguridad

```ruby
# ❌ Malo - SQL Injection
User.where("name = '#{params[:name]}'")

# ✅ Bueno
User.where(name: params[:name])

# ❌ Malo - XSS
<%= raw user_input %>

# ✅ Bueno
<%= user_input %>

# ❌ Malo - Mass Assignment
params.permit!

# ✅ Bueno
params.require(:user).permit(:name, :email)
```

## Checklist rápido de rendimiento

```ruby
# ❌ Malo - N+1
Article.all.each { |a| puts a.user.name }

# ✅ Bueno
Article.includes(:user).each { |a| puts a.user.name }

# ❌ Malo - Carga innecesaria
User.all.count

# ✅ Bueno
User.count
```

## Notas importantes

- No alarmar innecesariamente al usuario
- Priorizar problemas por severidad
- Explicar los problemas en lenguaje sencillo
- Ofrecer soluciones concretas
- Celebrar las buenas prácticas encontradas
