# Comando: /nuevo

Inicia el proceso de creación de un nuevo proyecto desde cero.

## Flujo de trabajo

### Paso 1: Descubrimiento inicial

Actúa como el **Product Owner** y realiza una entrevista amigable para entender la idea del usuario:

1. Saluda y pregunta por la idea general del proyecto
2. Haz preguntas clarificadoras:
   - ¿Quiénes van a usar la aplicación?
   - ¿Cuál es el problema principal que resuelve?
   - ¿Hay alguna app de referencia que te guste?
   - ¿Qué es lo más importante que debe poder hacer un usuario?

3. Resume lo entendido y confirma con el usuario antes de continuar

### Paso 1b: Ofrecer Flujo Estructurado (Opcional)

Después del discovery inicial, ofrecer al usuario la opción de usar el flujo estructurado:

```
Tu proyecto parece [simple/complejo]. Puedo proceder de dos formas:

1. **Empezar directamente** - Creo el proyecto y funcionalidades básicas (más rápido)
2. **Usar flujo estructurado** - Creo un feature con PRD, tareas y planes detallados (más trazabilidad)

¿Qué prefieres?
```

**Si elige flujo estructurado:**
1. Ejecutar `/feature "{descripción del proyecto}"`
2. Continuar con `/prd`, `/tasks`, `/plan`, `/code`
3. El proceso guiará paso a paso

**Si elige empezar directamente:**
Continuar con el Paso 2 tradicional

### Paso 2: Definición de requisitos

Una vez confirmada la idea:

1. Crea el archivo `.claude/contexts/requirements.md` con:
   - Nombre del proyecto
   - Descripción breve
   - Usuarios objetivo
   - Funcionalidades principales (MVP)
   - Funcionalidades futuras (nice to have)

2. Presenta el resumen al usuario para aprobación

### Paso 3: Arquitectura técnica

Actúa como el **Tech Lead**:

1. Define la estructura de modelos necesarios
2. Establece las relaciones entre entidades
3. Crea el archivo `.claude/contexts/architecture.md` con:
   - Diagrama de modelos
   - Relaciones
   - Decisiones técnicas clave

### Paso 4: Diseño visual

Actúa como el **Design Lead**:

1. Define el sistema de diseño básico
2. Actualiza `.claude/contexts/design-system.md` con:
   - Paleta de colores
   - Tipografía
   - Componentes principales

### Paso 5: Creación del proyecto Rails

Ejecuta los comandos necesarios:

```bash
# Crear proyecto Rails
rails new [nombre_proyecto] --database=sqlite3 --css=tailwind --skip-jbuilder

# Entrar al directorio
cd [nombre_proyecto]

# Instalar dependencias de testing
bundle add rspec-rails factory_bot_rails faker --group "development, test"
bundle add shoulda-matchers --group test

# Configurar RSpec
rails generate rspec:install

# Instalar autenticación
rails generate authentication

# Ejecutar migraciones
rails db:migrate
```

### Paso 6: Estructura inicial

1. Crea los modelos base identificados
2. Configura las rutas iniciales
3. Crea las vistas principales con layout base

### Paso 7: Resumen

Presenta al usuario:
- Lo que se ha creado
- Cómo ejecutar el proyecto (`bin/dev`)
- Próximos pasos sugeridos

## Notas importantes

- Siempre confirma con el usuario antes de ejecutar comandos destructivos
- Guarda el progreso en `.claude/history/changelog.md`
- Usa lenguaje sencillo, sin jerga técnica innecesaria
- Si algo no está claro, pregunta antes de asumir
