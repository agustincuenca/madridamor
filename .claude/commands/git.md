# Comando: /git

Gestión avanzada de Git con buenas prácticas y convenciones.

## Uso

```
/git [acción]
```

## Acciones disponibles

| Acción | Descripción |
|--------|-------------|
| `status` | Estado detallado con sugerencias |
| `branch [nombre]` | Crea rama con convención de nombres |
| `commit` | Genera mensaje con Conventional Commits |
| `pr` | Crea Pull Request con template |
| `merge` | Merge con estrategia recomendada |
| `sync` | Sincroniza con main/develop |

## Flujo de trabajo

### `/git status`

Muestra estado detallado con análisis y sugerencias.

```bash
# Comandos ejecutados
git status
git log --oneline -5
git branch -vv
```

**Salida esperada:**

```markdown
## Estado del Repositorio

📍 **Rama actual**: feature/user-auth
🔗 **Tracking**: origin/feature/user-auth (up to date)

### Cambios

#### Staged (listos para commit)
- ✅ `app/models/user.rb` (modified)
- ✅ `app/controllers/sessions_controller.rb` (new file)

#### Not staged (pendientes)
- 📝 `app/views/sessions/new.html.erb` (modified)

#### Untracked (sin seguimiento)
- ❓ `app/helpers/sessions_helper.rb`

### Commits recientes
1. `a1b2c3d` - feat: add login form
2. `d4e5f6g` - feat: create User model
3. `h7i8j9k` - chore: initial setup

### Sugerencias
- 💡 Tienes archivos modificados sin añadir al stage
- 💡 Hay 1 archivo nuevo sin seguimiento
- 💡 Considera hacer commit de los cambios staged
```

### `/git branch [nombre]`

Crea una rama siguiendo convenciones.

**Convención de nombres:**

| Tipo | Prefijo | Ejemplo |
|------|---------|---------|
| Feature | `feature/` | `feature/user-authentication` |
| Bug fix | `fix/` | `fix/login-redirect` |
| Hotfix | `hotfix/` | `hotfix/security-patch` |
| Refactor | `refactor/` | `refactor/user-model` |
| Docs | `docs/` | `docs/api-readme` |
| Chore | `chore/` | `chore/update-deps` |

**Flujo:**

1. Preguntar tipo si no está claro
2. Sanitizar nombre (kebab-case)
3. Crear rama desde main/develop

```bash
# Sincronizar con origin
git fetch origin

# Crear rama desde main
git checkout -b feature/nombre-feature origin/main
```

**Salida:**

```markdown
✅ Rama creada: `feature/user-authentication`
   Base: `main` (commit a1b2c3d)

📝 Próximos pasos:
1. Implementa los cambios
2. Haz commits frecuentes con `/git commit`
3. Cuando termines, usa `/git pr` para crear el PR
```

### `/git commit`

Genera mensaje de commit siguiendo Conventional Commits.

**Formato:**

```
<tipo>(<scope>): <descripción>

[cuerpo opcional]

[footer opcional]
```

**Tipos permitidos:**

| Tipo | Descripción |
|------|-------------|
| `feat` | Nueva funcionalidad |
| `fix` | Corrección de bug |
| `docs` | Documentación |
| `style` | Formato (no afecta código) |
| `refactor` | Refactoring |
| `perf` | Mejora de rendimiento |
| `test` | Añadir/modificar tests |
| `chore` | Tareas de mantenimiento |

**Flujo:**

1. Analizar archivos cambiados
2. Detectar tipo de cambio
3. Generar mensaje sugerido
4. Confirmar con usuario

```bash
# Ver cambios staged
git diff --staged --stat
git diff --staged
```

**Ejemplo de interacción:**

```markdown
## Análisis de Cambios

### Archivos modificados
- `app/models/user.rb` - Añadido método `full_name`
- `spec/models/user_spec.rb` - Tests para `full_name`

### Mensaje sugerido

```
feat(user): add full_name method

- Combines first_name and last_name
- Returns "Unknown" if both are blank
- Includes unit tests
```

¿Confirmas este mensaje? (s/n/editar)
```

**Comando final:**

```bash
git commit -m "feat(user): add full_name method

- Combines first_name and last_name
- Returns \"Unknown\" if both are blank
- Includes unit tests"
```

### `/git pr`

Crea Pull Request con template estructurado.

**Flujo:**

1. Verificar que hay commits para PR
2. Push de la rama si no está en origin
3. Analizar commits para generar descripción
4. Crear PR con gh CLI

```bash
# Verificar estado
git log origin/main..HEAD --oneline

# Push si es necesario
git push -u origin HEAD

# Crear PR
gh pr create --title "..." --body "..."
```

**Template de PR:**

```markdown
## Descripción

[Resumen de los cambios basado en los commits]

## Tipo de cambio

- [ ] Nueva funcionalidad (feat)
- [ ] Corrección de bug (fix)
- [ ] Refactoring
- [ ] Documentación
- [ ] Otro: ___

## Cambios realizados

- [Lista de cambios principales]

## Checklist

- [ ] Tests añadidos/actualizados
- [ ] Documentación actualizada
- [ ] Sin errores de linting
- [ ] Revisé mi propio código

## Screenshots (si aplica)

[Adjuntar capturas si hay cambios visuales]

## Notas para el reviewer

[Cualquier contexto adicional]
```

**Salida:**

```markdown
✅ Pull Request creado

📎 URL: https://github.com/user/repo/pull/123

📋 Detalles:
- Título: feat(auth): implement user login
- Base: main
- Commits: 3
- Archivos: 7

🔍 Próximos pasos:
1. Espera la revisión del código
2. Resuelve los comentarios si los hay
3. Cuando esté aprobado, usa `/git merge`
```

### `/git merge`

Merge con la estrategia recomendada.

**Estrategias:**

| Situación | Estrategia |
|-----------|------------|
| Feature pequeña | Squash merge |
| Feature grande | Merge commit |
| Hotfix | Fast-forward o merge |

**Flujo:**

1. Verificar que PR está aprobado
2. Verificar que CI pasa
3. Sugerir estrategia
4. Ejecutar merge

```bash
# Verificar estado del PR
gh pr status

# Merge con squash (recomendado para features)
gh pr merge --squash

# O merge commit (para features grandes)
gh pr merge --merge
```

**Salida:**

```markdown
## Merge de PR #123

### Pre-checks
- ✅ PR aprobado
- ✅ CI pasando
- ✅ Sin conflictos

### Estrategia recomendada: Squash merge

Esta es una feature pequeña (3 commits).
Squash merge mantiene el historial limpio.

¿Proceder con squash merge? (s/n)

---

✅ PR #123 mergeado exitosamente

📋 Resumen:
- Método: Squash merge
- Commit: a1b2c3d
- Rama eliminada: feature/user-auth

🔄 Sincroniza tu local:
git checkout main && git pull
```

### `/git sync`

Sincroniza la rama actual con main/develop.

**Flujo:**

1. Detectar rama base (main o develop)
2. Fetch de cambios remotos
3. Rebase o merge según preferencia
4. Resolver conflictos si los hay

```bash
# Fetch cambios
git fetch origin

# Rebase sobre main (preferido)
git rebase origin/main

# O merge si hay conflictos complejos
git merge origin/main
```

**Salida:**

```markdown
## Sincronización

### Estado inicial
- Rama actual: `feature/user-auth`
- Rama base: `main`
- Tu rama está 2 commits detrás de main

### Proceso

```
git fetch origin
git rebase origin/main
```

### Resultado
✅ Sincronización exitosa

Tu rama ahora incluye los últimos cambios de main.
No hubo conflictos.

### Si hubiera conflictos

```markdown
⚠️ Conflictos detectados en:
- `app/models/user.rb`
- `config/routes.rb`

Para resolver:
1. Edita los archivos marcados
2. `git add <archivo>`
3. `git rebase --continue`

¿Quieres que te ayude a resolver los conflictos?
```
```

## Buenas prácticas de Git

### Commits

- Commits pequeños y frecuentes
- Un commit = un cambio lógico
- Mensajes descriptivos en presente
- Referenciar issues: `fix: resolve login bug (#123)`

### Branches

- Vida corta (máximo 1-2 semanas)
- Sincronizar frecuentemente con main
- Eliminar después de merge

### Pull Requests

- PRs pequeños y enfocados
- Descripción clara del "qué" y "por qué"
- Screenshots para cambios visuales
- Responder a comentarios rápidamente

## Comandos útiles adicionales

```bash
# Ver historial visual
git log --oneline --graph --all

# Deshacer último commit (mantener cambios)
git reset --soft HEAD~1

# Stash de cambios
git stash
git stash pop

# Cherry-pick de un commit
git cherry-pick <commit-hash>

# Ver quién cambió cada línea
git blame <archivo>

# Buscar en el historial
git log -S "texto a buscar"
```

## Notas importantes

- NUNCA hacer force push a main/develop
- NUNCA commitear secretos o credenciales
- Siempre revisar `git diff` antes de commit
- Usar `.gitignore` para archivos locales
- Hacer backup antes de operaciones destructivas
