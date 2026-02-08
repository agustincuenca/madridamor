# Skill: Git Workflow

## Purpose

Gestionar control de versiones con Git siguiendo mejores prácticas de branching, commits convencionales y colaboración efectiva en equipos.

## Branching Strategies

### Git Flow

```
main (producción)
  │
  └── develop (integración)
        │
        ├── feature/nueva-funcionalidad
        ├── feature/otra-funcionalidad
        │
        └── release/1.0.0
              │
              └── hotfix/fix-critico → main + develop
```

```bash
# Crear feature branch
git checkout develop
git checkout -b feature/user-registration

# Trabajar en feature
git add .
git commit -m "feat: add user registration form"

# Finalizar feature
git checkout develop
git merge --no-ff feature/user-registration
git branch -d feature/user-registration

# Crear release
git checkout develop
git checkout -b release/1.0.0
# Hacer ajustes finales...
git checkout main
git merge --no-ff release/1.0.0
git tag -a v1.0.0 -m "Release 1.0.0"
git checkout develop
git merge --no-ff release/1.0.0

# Hotfix urgente
git checkout main
git checkout -b hotfix/security-patch
# Aplicar fix...
git checkout main
git merge --no-ff hotfix/security-patch
git tag -a v1.0.1 -m "Security patch"
git checkout develop
git merge --no-ff hotfix/security-patch
```

### GitHub Flow (Simplificado)

```bash
# Siempre desde main
git checkout main
git pull origin main
git checkout -b feature/add-comments

# Trabajar y commitear
git add .
git commit -m "feat: add comment system"
git push -u origin feature/add-comments

# Crear Pull Request en GitHub
# Después de review y merge, eliminar branch
git checkout main
git pull origin main
git branch -d feature/add-comments
```

### Trunk-Based Development

```bash
# Commits pequeños directo a main
git checkout main
git pull origin main

# Cambio pequeño y atómico
git add .
git commit -m "feat: add email validation"
git push origin main

# Para cambios grandes: feature flags
# config/features.yml
# new_checkout: false

# En código
if Feature.enabled?(:new_checkout)
  # nuevo código
else
  # código actual
end
```

## Conventional Commits

### Formato

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

### Tipos

| Tipo | Descripción | Ejemplo |
|------|-------------|---------|
| `feat` | Nueva funcionalidad | `feat: add user avatar upload` |
| `fix` | Corrección de bug | `fix: resolve login redirect loop` |
| `docs` | Documentación | `docs: update API endpoints` |
| `style` | Formato (no afecta código) | `style: fix indentation in user.rb` |
| `refactor` | Refactoring (sin cambio funcional) | `refactor: extract email service` |
| `perf` | Mejora de rendimiento | `perf: add database index for users` |
| `test` | Añadir/corregir tests | `test: add specs for payment flow` |
| `build` | Sistema de build | `build: update webpack config` |
| `ci` | Integración continua | `ci: add GitHub Actions workflow` |
| `chore` | Tareas de mantenimiento | `chore: update dependencies` |
| `revert` | Revertir commit | `revert: feat: add user avatar` |

### Ejemplos completos

```bash
# Feature simple
git commit -m "feat: add password strength indicator"

# Feature con scope
git commit -m "feat(auth): implement two-factor authentication"

# Fix con referencia a issue
git commit -m "fix(checkout): resolve cart total calculation

The total was not including shipping costs when
the user selected express delivery.

Fixes #123"

# Breaking change
git commit -m "feat(api)!: change response format to JSON:API

BREAKING CHANGE: API responses now follow JSON:API spec.
Update your client code accordingly."

# Múltiples párrafos
git commit -m "refactor(models): extract validation logic

- Move email validation to EmailValidator
- Move phone validation to PhoneValidator
- Add shared validation helpers

This makes validators reusable across models."
```

## Pull Request Best Practices

### Template de PR

```markdown
<!-- .github/pull_request_template.md -->
## Descripción
<!-- Qué cambia este PR y por qué -->

## Tipo de cambio
- [ ] Bug fix (cambio que soluciona un issue)
- [ ] Nueva feature (cambio que añade funcionalidad)
- [ ] Breaking change (fix o feature que rompe compatibilidad)
- [ ] Refactoring (mejora sin cambio funcional)

## Checklist
- [ ] Mi código sigue el estilo del proyecto
- [ ] He hecho self-review de mi código
- [ ] He comentado código complejo
- [ ] He actualizado la documentación
- [ ] Mis cambios no generan warnings
- [ ] He añadido tests que prueban mi fix/feature
- [ ] Tests nuevos y existentes pasan localmente

## Screenshots (si aplica)
<!-- Añadir capturas de UI changes -->

## Testing
<!-- Cómo probar estos cambios -->

## Issues relacionados
Closes #123
```

### Buenos títulos de PR

```
feat(users): add profile photo upload
fix(payments): handle failed webhook signatures
docs(api): document rate limiting headers
refactor(orders): extract shipping calculator
```

## Merge Strategies

### Merge Commit (--no-ff)

```bash
# Mantiene historial completo
git checkout main
git merge --no-ff feature/user-auth

# Resultado:
# *   Merge branch 'feature/user-auth'
# |\
# | * feat: add password reset
# | * feat: add login form
# |/
# * previous commit
```

### Squash Merge

```bash
# Combina todos los commits en uno
git checkout main
git merge --squash feature/user-auth
git commit -m "feat(auth): add complete user authentication"

# Resultado:
# * feat(auth): add complete user authentication
# * previous commit
```

### Rebase

```bash
# Reescribe historial para ser lineal
git checkout feature/user-auth
git rebase main
git checkout main
git merge feature/user-auth

# Resultado (lineal):
# * feat: add password reset
# * feat: add login form
# * previous commit
```

### Cuándo usar cada uno

| Estrategia | Cuándo usar |
|------------|-------------|
| Merge commit | Features grandes, mantener contexto |
| Squash | Features pequeñas, limpiar WIP commits |
| Rebase | Mantener historial limpio y lineal |

## Git Hooks

### Pre-commit (Rubocop + Format)

```bash
#!/bin/sh
# .git/hooks/pre-commit

echo "Running Rubocop..."
bundle exec rubocop --autocorrect-all

if [ $? -ne 0 ]; then
  echo "Rubocop failed. Please fix the issues before committing."
  exit 1
fi

echo "Running ERB Lint..."
bundle exec erblint --lint-all --autocorrect

# Re-add any auto-corrected files
git add -u

exit 0
```

### Pre-push (Tests)

```bash
#!/bin/sh
# .git/hooks/pre-push

echo "Running tests before push..."
bundle exec rspec --fail-fast

if [ $? -ne 0 ]; then
  echo "Tests failed. Push aborted."
  exit 1
fi

echo "Running Brakeman..."
bundle exec brakeman -q -w2

if [ $? -ne 0 ]; then
  echo "Security issues found. Push aborted."
  exit 1
fi

exit 0
```

### Commit-msg (Validar formato)

```bash
#!/bin/sh
# .git/hooks/commit-msg

commit_regex='^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(.+\))?: .{1,72}'

if ! grep -qE "$commit_regex" "$1"; then
  echo "Invalid commit message format."
  echo "Must match: type(scope): description"
  echo "Types: feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert"
  exit 1
fi
```

### Instalar hooks con Overcommit

```ruby
# Gemfile
group :development do
  gem "overcommit"
end
```

```yaml
# .overcommit.yml
PreCommit:
  RuboCop:
    enabled: true
    command: ['bundle', 'exec', 'rubocop']
    on_warn: fail

  ErbLint:
    enabled: true
    command: ['bundle', 'exec', 'erblint']

PrePush:
  RSpec:
    enabled: true
    command: ['bundle', 'exec', 'rspec', '--fail-fast']

  Brakeman:
    enabled: true
    command: ['bundle', 'exec', 'brakeman', '-q', '-w2']

CommitMsg:
  MessageFormat:
    enabled: true
    pattern: '^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(.+\))?: .+'
```

```bash
overcommit --install
```

## Comandos Útiles

### Stash

```bash
# Guardar cambios temporalmente
git stash
git stash push -m "WIP: user registration"

# Listar stashes
git stash list

# Aplicar último stash
git stash pop

# Aplicar stash específico
git stash apply stash@{2}

# Stash parcial (interactivo)
git stash push -p

# Stash incluyendo untracked
git stash -u
```

### Cherry-pick

```bash
# Aplicar commit específico a otra rama
git checkout main
git cherry-pick abc123

# Cherry-pick sin commitear
git cherry-pick --no-commit abc123

# Cherry-pick rango de commits
git cherry-pick abc123..def456

# Resolver conflictos y continuar
git cherry-pick --continue

# Abortar cherry-pick
git cherry-pick --abort
```

### Rebase Interactivo

```bash
# Editar últimos 3 commits
git rebase -i HEAD~3

# Comandos en editor:
# pick abc123 feat: first commit     (mantener)
# squash def456 fix: typo            (combinar con anterior)
# reword ghi789 docs: update         (cambiar mensaje)
# drop jkl012 WIP                    (eliminar)
# edit mno345 feat: needs split      (pausar para editar)

# Reordenar commits: simplemente mover líneas

# Después de edit:
git add .
git commit --amend
git rebase --continue
```

### Bisect (encontrar commit problemático)

```bash
# Iniciar bisect
git bisect start

# Marcar commit actual como malo
git bisect bad

# Marcar commit bueno conocido
git bisect good v1.0.0

# Git hace checkout de commit intermedio
# Probar si el bug existe...
git bisect good  # o git bisect bad

# Repetir hasta encontrar el commit culpable

# Finalizar
git bisect reset

# Bisect automático con script
git bisect start HEAD v1.0.0
git bisect run bundle exec rspec spec/models/user_spec.rb
```

### Reflog (recuperar commits perdidos)

```bash
# Ver historial de HEAD
git reflog

# Ver reflog de rama específica
git reflog show feature/user

# Recuperar commit después de reset --hard
git reflog
# abc123 HEAD@{2}: commit: feat: important change
git checkout abc123
# o
git branch recovered-branch abc123

# Recuperar rama eliminada
git reflog
# def456 HEAD@{5}: checkout: moving from deleted-branch to main
git checkout -b deleted-branch def456
```

### Otros comandos útiles

```bash
# Ver diferencias de un archivo específico
git diff HEAD~3..HEAD -- app/models/user.rb

# Blame con ignore de whitespace
git blame -w app/models/user.rb

# Log de un archivo
git log --follow -p -- app/models/user.rb

# Buscar en historial
git log -S "método_buscado" --oneline

# Ver branches mergeadas
git branch --merged main

# Limpiar branches mergeadas
git branch --merged main | grep -v main | xargs git branch -d

# Verificar qué se va a pushear
git log origin/main..HEAD

# Reset archivo a versión específica
git checkout abc123 -- app/models/user.rb

# Unstage archivos
git reset HEAD app/models/user.rb

# Descartar cambios locales
git checkout -- app/models/user.rb

# Amend sin cambiar mensaje
git commit --amend --no-edit
```

## Resolución de Conflictos

### Estrategias

```bash
# Ver archivos en conflicto
git status

# Abrir herramienta de merge
git mergetool

# Aceptar versión nuestra
git checkout --ours app/models/user.rb

# Aceptar versión de ellos
git checkout --theirs app/models/user.rb

# Marcar como resuelto
git add app/models/user.rb
git commit
```

### Anatomía de un conflicto

```ruby
# app/models/user.rb
<<<<<<< HEAD
def full_name
  "#{first_name} #{last_name}"
end
=======
def full_name
  [first_name, middle_name, last_name].compact.join(" ")
end
>>>>>>> feature/add-middle-name
```

### Resolver manualmente

```ruby
# Elegir/combinar el código correcto:
def full_name
  [first_name, middle_name, last_name].compact.join(" ")
end
```

```bash
git add app/models/user.rb
git commit -m "fix: merge conflict in user full_name"
```

## .gitignore para Rails

```gitignore
# .gitignore

# Bundler
/.bundle
/vendor/bundle

# SQLite
/storage/*.sqlite3
/storage/*.sqlite3-*

# Environment
.env
.env.local
.env.*.local

# Logs
/log/*
!/log/.keep

# Temp files
/tmp/*
!/tmp/.keep
!/tmp/pids
!/tmp/pids/.keep

# Node modules
/node_modules

# Assets
/public/assets
/public/packs
/public/packs-test

# Coverage
/coverage

# OS files
.DS_Store
Thumbs.db

# IDE
.idea/
.vscode/
*.swp
*.swo
*~

# Master key (IMPORTANTE)
/config/master.key
/config/credentials/*.key

# Byebug
.byebug_history

# Spring
/tmp/spring/

# Brakeman
/tmp/brakeman-report.*
```

## Semantic Versioning

### Formato: MAJOR.MINOR.PATCH

```
v2.1.3
│ │ └── PATCH: bug fixes (compatible)
│ └──── MINOR: new features (compatible)
└────── MAJOR: breaking changes (incompatible)
```

### Reglas

| Incrementar | Cuándo |
|-------------|--------|
| PATCH | Bug fixes sin cambios de API |
| MINOR | Nueva funcionalidad compatible hacia atrás |
| MAJOR | Cambios que rompen compatibilidad |

### Ejemplos

```bash
# Bug fix
v1.2.3 → v1.2.4
git tag -a v1.2.4 -m "Fix: resolve payment processing error"

# Nueva feature
v1.2.4 → v1.3.0
git tag -a v1.3.0 -m "Feature: add subscription management"

# Breaking change
v1.3.0 → v2.0.0
git tag -a v2.0.0 -m "BREAKING: new API response format"

# Pre-release
v2.0.0-alpha.1
v2.0.0-beta.1
v2.0.0-rc.1
```

### Tags en Git

```bash
# Crear tag anotado
git tag -a v1.0.0 -m "Release version 1.0.0"

# Listar tags
git tag -l "v1.*"

# Pushear tags
git push origin v1.0.0
git push origin --tags

# Eliminar tag
git tag -d v1.0.0
git push origin --delete v1.0.0

# Checkout a tag
git checkout v1.0.0
```

## Automatización con GitHub Actions

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Create Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          body: |
            Changes in this release:
            - Feature X
            - Fix Y
          draft: false
          prerelease: false
```

## Workflow diario recomendado

```bash
# Empezar el día
git checkout main
git pull origin main

# Crear branch para trabajo
git checkout -b feature/mi-tarea

# Hacer commits frecuentes y pequeños
git add -p  # Añadir por hunks
git commit -m "feat: add user validation"

# Sincronizar con main regularmente
git fetch origin
git rebase origin/main

# Al terminar
git push -u origin feature/mi-tarea
# Crear PR en GitHub

# Después del merge
git checkout main
git pull origin main
git branch -d feature/mi-tarea
```
