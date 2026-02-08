# Convenciones del Proyecto

## Codigo

### Nomenclatura
- **Clases**: PascalCase (`UserAccount`)
- **Metodos**: snake_case (`find_by_email`)
- **Variables**: snake_case (`current_user`)
- **Constantes**: SCREAMING_SNAKE_CASE (`MAX_RETRIES`)

### Idioma
- Codigo en **ingles**
- Comentarios en **espanol** (cuando sean necesarios)
- Commits en **espanol**

## Git

### Ramas
- `main` - Produccion
- `develop` - Desarrollo
- `feature/xxx` - Nuevas funcionalidades
- `fix/xxx` - Correcciones

### Commits
Formato: `tipo: descripcion breve`

Tipos:
- `feat`: Nueva funcionalidad
- `fix`: Correccion de bug
- `refactor`: Refactorizacion
- `docs`: Documentacion
- `test`: Tests
- `chore`: Tareas de mantenimiento

## Testing

- Minimo 80% de cobertura
- Tests unitarios para modelos y servicios
- Tests de integracion para flujos criticos
- Tests E2E para happy paths

## Documentacion

- README actualizado
- Documentar decisiones arquitectonicas
- API documentada con ejemplos
