# Comando: /seguridad

Ejecuta una auditoría de seguridad completa de la aplicación.

## Descripción

Este comando analiza la aplicación en busca de vulnerabilidades de seguridad, revisa la configuración, y genera un reporte detallado con recomendaciones.

## Proceso

### 1. Análisis estático con Brakeman

```bash
# Verificar si Brakeman está instalado
bundle show brakeman || bundle add brakeman --group development

# Ejecutar análisis
bundle exec brakeman -q -o tmp/brakeman-report.json -f json
bundle exec brakeman -o tmp/brakeman-report.html
```

### 2. Auditoría de dependencias

```bash
# Verificar si bundler-audit está instalado
bundle show bundler-audit || bundle add bundler-audit --group development

# Actualizar base de datos de vulnerabilidades
bundle exec bundle-audit update

# Ejecutar auditoría
bundle exec bundle-audit check
```

### 3. Verificar configuración de seguridad

Revisar los siguientes archivos:

```ruby
# config/environments/production.rb
- [ ] config.force_ssl = true
- [ ] config.consider_all_requests_local = false
- [ ] config.hosts configurado

# config/initializers/content_security_policy.rb
- [ ] CSP configurado

# config/initializers/filter_parameter_logging.rb
- [ ] Parámetros sensibles filtrados

# config/initializers/session_store.rb
- [ ] Cookies seguras (secure, httponly, same_site)
```

### 4. Revisar autenticación

```ruby
# Verificar:
- [ ] has_secure_password implementado
- [ ] Validaciones de password (longitud, complejidad)
- [ ] Rate limiting en login
- [ ] Account lockout tras intentos fallidos
```

### 5. Revisar autorización

```ruby
# Verificar:
- [ ] Pundit implementado
- [ ] after_action :verify_authorized en controllers
- [ ] Políticas restrictivas por defecto
- [ ] verify_policy_scoped para index actions
```

### 6. Buscar patrones vulnerables

```bash
# SQL Injection potencial
grep -rn "where.*\#{" app/ --include="*.rb"
grep -rn "\.find_by_sql" app/ --include="*.rb"

# XSS potencial
grep -rn "\.html_safe" app/ --include="*.rb"
grep -rn "raw(" app/ --include="*.erb"
grep -rn "<%= raw" app/ --include="*.erb"

# Mass assignment
grep -rn "permit!" app/ --include="*.rb"

# Hardcoded secrets
grep -rn "password\s*=" app/ --include="*.rb" | grep -v "password_digest"
grep -rn "api_key\s*=" app/ --include="*.rb"
grep -rn "secret" app/ --include="*.rb" | grep -v credentials
```

### 7. Verificar headers de seguridad

```ruby
# Hacer request y verificar headers
curl -I https://localhost:3000 2>/dev/null | grep -E "X-Frame|X-Content|X-XSS|Content-Security|Strict-Transport"

# Headers esperados:
# X-Frame-Options: DENY o SAMEORIGIN
# X-Content-Type-Options: nosniff
# X-XSS-Protection: 1; mode=block
# Content-Security-Policy: ...
# Strict-Transport-Security: max-age=...
```

## Output

### Reporte de seguridad

```markdown
# Auditoría de Seguridad
Fecha: [fecha actual]
Proyecto: [nombre del proyecto]

## Resumen Ejecutivo

| Categoría | Estado |
|-----------|--------|
| Brakeman | ✅ / ⚠️ / ❌ |
| Dependencias | ✅ / ⚠️ / ❌ |
| Configuración | ✅ / ⚠️ / ❌ |
| Autenticación | ✅ / ⚠️ / ❌ |
| Autorización | ✅ / ⚠️ / ❌ |
| Headers | ✅ / ⚠️ / ❌ |

## Hallazgos por Severidad

### Críticos (requieren acción inmediata)
[Lista de hallazgos críticos]

### Altos (resolver pronto)
[Lista de hallazgos altos]

### Medios (planificar corrección)
[Lista de hallazgos medios]

### Bajos (mejoras recomendadas)
[Lista de hallazgos bajos]

## Detalles

### Brakeman
[Resumen de warnings encontrados]

### Dependencias vulnerables
[Lista de gems con vulnerabilidades conocidas]

### Configuración
[Problemas de configuración encontrados]

## Recomendaciones

1. [Recomendación prioritaria con pasos]
2. [Siguiente recomendación]
...

## Próximos pasos

- [ ] Corregir hallazgos críticos
- [ ] Actualizar dependencias vulnerables
- [ ] Implementar mejoras de configuración
- [ ] Programar siguiente auditoría
```

## Ejemplos de uso

```bash
# Usuario escribe:
/seguridad

# O en lenguaje natural:
"Revisa la seguridad de la app"
"Ejecuta una auditoría de seguridad"
"Busca vulnerabilidades"
"¿Es segura mi aplicación?"
```

## Automatización

### GitHub Action para CI

```yaml
# .github/workflows/security.yml
name: Security Audit

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 1'  # Lunes a medianoche

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true

      - name: Run Brakeman
        run: |
          gem install brakeman
          brakeman -q -w2 --no-pager

      - name: Run Bundle Audit
        run: |
          gem install bundler-audit
          bundle-audit check --update
```

## Notas

- Ejecutar regularmente (al menos semanalmente)
- Revisar antes de cada deploy a producción
- Mantener dependencias actualizadas
- Documentar excepciones de seguridad aceptadas
