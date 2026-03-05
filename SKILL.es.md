name: Code Time Traveler
description: Analiza el historial de git para identificar cuándo se introdujeron errores y sugiere puntos de corrección óptimos
version: 1.0.0
author: OpenClaw Team
tags:
  - git
  - debugging
  - history-analysis
  - bug-tracking
  - blame
  - bisect
dependencies:
  - git >= 2.30
  - bash >= 4.0
  - jq (opcional, para formato JSON)
  - tig (opcional, para vista interactiva)
environment:
  GIT_DIR: ".git"
  GIT_PAGER: "cat"
  CTT_LOG_LEVEL: "INFO"
  CTT_MAX_COMMITS: "1000"
  CTT_BISECT_AUTO: "false"
---

# Code Time Traveler

Una habilidad especializada para analizar el historial de commits de git para detectar cuándo se introdujeron errores, rastrear la evolución del código y determinar los puntos óptimos para correcciones.

## Propósito

Casos de uso reales:
- "La API comenzó a devolver errores 500 después del despliegue el 3 de marzo - ¿qué commit causó esto?"
- "¿Cuándo se cambió por última vez la función `processPayment()` y quién la modificó?"
- "Encontrar el commit que introdujo la fuga de memoria que vemos en producción"
- "Mostrar el historial de cambios en `src/auth/middleware.ts` con contexto sobre cada cambio"
- "Tenemos un error de regresión - usar bisect para encontrar automáticamente el commit defectuoso"
- "¿Qué commits modificaron el esquema de base de datos entre v2.1.0 y v2.3.0?"
- "Rastrear la evolución de esta línea de código específica en todas las ramas"
- "Identificar todos los commits que tocaron el módulo `payment_processing` en el último trimestre"

## Alcance

### Comandos

**`analyze-bug [file] [line] [error-message]`**
Analiza el historial de git en torno a una ubicación de error específica para encontrar los commits culpables probables.

Parámetros:
- `file`: Ruta al archivo que contiene el error (requerido)
- `line`: Número de línea donde ocurre el error (requerido)
- `error-message`: Texto o patrón de error exacto (opcional pero recomendado)

Ejemplo: `analyze-bug src/utils/api.ts 127 "Connection timeout after 30s"`

**`bisect-start [good-commit] [bad-commit] [test-command]`**
Automatiza git bisect para encontrar el commit que introdujo un error.

Parámetros:
- `good-commit`: Commit válido conocido (hash/etiqueta/rama) (por defecto: HEAD~20)
- `bad-commit`: Commit defectuoso conocido (por defecto: HEAD)
- `test-command`: Comando de shell que devuelve 0 para bueno, no-0 para malo (requerido)

Ejemplo: `bisect-start v2.1.0 HEAD "npm test -- --grep 'payment flow'"`

**`blame-deep [file] [since]`**
Git blame mejorado con contexto de commit, mostrando mensajes de commit, autores y cambios adyacentes.

Parámetros:
- `file`: Archivo a analizar (requerido)
- `since`: Filtrar commits desde una fecha (ej: "2 weeks ago", "2025-01-01") (opcional)

Ejemplo: `blame-deep src/models/User.ts "1 month ago"`

**`timeline [path] [days]`**
Genera una línea de tiempo cronológica de cambios en una ruta con resumen estadístico.

Parámetros:
- `path`: Ruta de archivo o directorio (por defecto: directorio actual)
- `days`: Número de días a analizar (por defecto: 30)

Ejemplo: `timeline src/components/ 90`

**`find-introduction [pattern] [files]`**
Localiza cuándo se agregó por primera vez un patrón de código, función o importación específico.

Parámetros:
- `pattern`: Patrón regex para buscar (requerido)
- `files`: Patrón de glob de archivos (por defecto: "**/*.{js,ts,jsx,tsx}")

Ejemplo: `find-introduction "useReducer" "src/hooks/**/*.ts"`

**`compare-releases [tag1] [tag2] [filter]`**
Compara dos releases para identificar todos los cambios en un componente específico.

Parámetros:
- `tag1`: Primera etiqueta de release (requerido)
- `tag2`: Segunda etiqueta de release (requerido)
- `filter`: Filtrar por ruta, autor o mensaje (ej: "path:src/auth/", "author:john")

Ejemplo: `compare-releases v2.3.0 v2.4.0 "path:src/payment/"`

**`hotspot-analysis [days] [extensions]`**
Identifica archivos con mayor tasa de cambios (más frecuentemente modificados) en un período dado.

Parámetros:
- `days`: Ventana de análisis (por defecto: 30)
- `extensions`: Extensiones de archivo a incluir (por defecto: ".js,.ts,.tsx,.jsx,.py,.go,.rs")

Ejemplo: `hotspot-analysis 60 ".js,.ts,.tsx"`

**`regression-search [date] [keywords]`**
Busca commits en o después de una fecha que puedan haber introducido regresiones.

Parámetros:
- `date`: Fecha para iniciar la búsqueda (por defecto: última fecha de despliegue exitoso desde .openclaw/deploy.log)
- `keywords`: Palabras clave que sugieren cambios riesgosos: "refactor", "perf", "security", "fix", "breaking"

Ejemplo: `regression-search "2025-03-01" "refactor,perf"`

## Proceso de Trabajo

### Analizando la Introducción de un Error

1. El usuario proporciona archivo, número de línea y mensaje de error
2. La habilidad ejecuta `git blame` en la línea específica para identificar el commit de última modificación
3. Ejecuta `git show <commit>` para examinar cambios en ese commit
4. Revisa el mensaje del commit para palabras clave como "fix", "refactor", "breaking" que sugieran cambios intencionales
5. Revisa commits adyacentes (±5) para entender el contexto y si el error fue introducido como efecto secundario
6. Consulta `git log -p --follow -L <line>:<file>` para ver la evolución completa de esa línea a través de renombres
7. Genera:
   - Hash de commit, autor, fecha
   - Diff completo del commit
   - Commits relacionados que tocaron la misma área
   - Punto de corrección sugerido (revertir, parchear o cherry-pick de una versión anterior)
   - Evaluación de riesgo: "Alta confianza: cambio directo en la línea con error" vs "Media: cambios adyacentes"

### Bisección Automatizada

1. El usuario proporciona comando de prueba que reproduce el error
2. La habilidad inicializa git bisect: `git bisect start`
3. Marca el HEAD actual como malo: `git bisect bad`
4. Marca el commit bueno proporcionado: `git bisect good <good-commit>`
5. Entra en el bucle de bisect automáticamente (a menos que CTT_BISECT_AUTO=false):
   - Para cada commit, ejecuta el comando de prueba
   - Registra resultado con información del commit
   - Llama a `git bisect good` o `git bisect bad`
6. Cuando bisect converge, genera:
   - Detalles del primer commit malo
   - Diff completo de ese commit
   - Commits entre el bueno y el malo para contexto
   - Recomendación: "Revertir este commit" o "Aplicar corrección mínima encima"
7. Reinicia bisect: `git bisect reset`

### Identificación de Puntos Calientes

1. La habilidad ejecuta: `git log --since="<days> days ago" --oneline --name-only | sort | uniq -c | sort -nr`
2. Analiza la salida para clasificar archivos por número de commits que los tocaron
3. Para los 10 principales hotspots, calcula:
   - Número de autores distintos
   - Ratio de commits de corrección de errores vs commits de características (usando análisis de mensajes)
   - Tiempo promedio entre commits (frecuencia de cambios)
4. Genera lista clasificada con métricas y puntuación de riesgo (alta rotación + muchos autores = mayor riesgo)

## Reglas de Oro

1. **Nunca sugerir una corrección sin verificar que el commit existe y es accesible** - Siempre ejecutar `git cat-file -e <commit>` antes de recomendar acciones
2. **Considerar merge commits por separado** - Usar `git log --no-merges` para análisis inicial, luego verificar fusiones explícitamente si es necesario
3. **Tener en cuenta los renombres de archivos** - Usar `git log --follow` al rastrear historial de archivos a través de renombres
4. **Bisect requiere árbol de trabajo limpio** - Verificar que `git status --porcelain` devuelva vacío antes de comenzar
5. **Verificar firma de commit** - Preferir commits verificados: `git log --show-signature -1 <commit>`
6. **El contexto es crítico** - Siempre mostrar al menos 3 commits antes y después del commit sospechoso
7. **Importa el rendimiento** - Limitar búsquedas de log con `--since` y globs de archivos; usar `git rev-list --count` en lugar de contar logs completos
8. **Las ramas deben especificarse explícitamente** - Al comparar releases, verificar que las etiquetas existan: `git tag -l | grep -E "^<tag>$"`
9. **Nunca modificar el historial** - Esta habilidad solo analiza; nunca enmendar, rebasar o force-push
10. **Documentar suposiciones** - Si se usan heurísticas (ej: palabra clave "fix"), indicar nivel de confianza

## Ejemplos

### Ejemplo 1: Encontrar introducción de error en un archivo específico

**Entrada de usuario:**
```
analyze-bug src/server/request.js 243 "Cannot read property 'headers' of undefined"
```

**Salida de la habilidad:**
```
Analyzing bug location: src/server/request.js:243
==============================================

1. BLAME ANALYSIS (last modification of line 243):
   Commit:  7a3f9c2d
   Author:  jane.doe@company.com
   Date:    2025-02-28 14:32:18 -0500
   Message: feat: add request validation middleware

2. COMMIT DIFF:
   commit 7a3f9c2d8f4e2a1b...
   Author: Jane Doe <jane.doe@company.com>
   Date:   Fri Feb 28 14:32:18 2025 -0500

       feat: add request validation middleware

       - Add validation for incoming requests
       - Reject requests missing Content-Type header
       - Log validation failures

   diff --git a/src/server/request.js b/src/server/request.js
   index 5a8b2c1..d4e5f6a 100644
   --- a/src/server/request.js
   +++ b/src/server/request.js
   @@ -240,7 +240,12 @@ function processRequest(req, res) {
     // Validate request
   - if (!req.headers) {
   -   throw new Error('Missing headers');
   - }
   + const validationErrors = validateRequest(req);
   + if (validationErrors.length > 0) {
   +   logger.warn('Validation failed:', validationErrors);
   +   return res.status(400).json({ error: validationErrors });
   + }

   - // Process request (line 243 was shifted)
     await processHandler(req, res);

3. ADJACENT COMMITS:
   - 7a3f9c2d (above) - current
   - b8c4d9e1 - refactor: simplify error handling (2 days before)
   - 3f2a1b4c - fix: handle null headers gracefully (5 days before)

4. ANALYSIS:
   The line 243 ("await processHandler(req, res);") existed before commit 7a3f9c2d.
   That commit added validation logic ABOVE line 243, but did NOT modify line 243 itself.
   However, the validation now returns early for invalid requests, meaning line 243
   is only executed when validation passes. The error "Cannot read property 'headers'
   of undefined" suggests that `req` itself is undefined, not just `req.headers`.

   RECOMMENDATION: Medium confidence that commit 7a3f9c2d introduced the bug as a
   side effect. The validation assumes `req` exists, but upstream code may now
   call processRequest() with undefined in some edge cases. Check commit 3f2a1b4c
   which explicitly handled null headers - this may have been removed in 7a3f9c2d.

5. SUGGESTED FIX POINT:
   - Option A (revert validation): Revert portions of 7a3f9c2d to restore null check
   - Option B (defensive coding): Add `if (!req) throw new Error('Request is undefined')`
     at the start of processRequest()
   - Option C (bisect): Run automated bisect to confirm if 7a3f9c2d is truly first bad

Confidence: 65% | Evidence: Line 243 unchanged, but surrounding logic altered
```

### Ejemplo 2: Bisección automatizada para encontrar regresión

**Entrada de usuario:**
```
bisect-start v2.5.0 HEAD "curl -f http://localhost:3000/api/health || exit 1"
```

**Salida de la habilidad:**
```
Starting bisect to find regression between v2.5.0 (good) and HEAD (bad)
===============================================================

Verifying repository state:
  Working tree: clean
  Current branch: feature/auth-overhaul
  HEAD: abc1234de

Initializing git bisect...
  git bisect start
  git bisect bad  # HEAD is bad
  git bisect good v2.5.0  # Tag v2.5.0 is good

Testing commit [10/156]: d9e8f7a6 (3 days ago)
  Command: curl -f http://localhost:3000/api/health || exit 1
  Result: SUCCESS (0)
  Marking as: GOOD

Testing commit [9/156]: b4c3d2e1 (4 days ago)
  Command: curl -f http://localhost:3000/api/health || exit 1
  Result: FAILURE (22)
  Error: curl: (22) The requested URL returned error: 500
  Marking as: BAD

Testing commit [8/155]: a1b2c3d4 (5 days ago)
  Command: curl -f http://localhost:3000/api/health || exit 1
  Result: SUCCESS (0)
  Marking as: GOOD

[... skipping intermediate commits for brevity ...]

Testing commit [1/1]: f1e2d3c4 (11 days ago)
  Command: curl -f http://localhost:3000/api/health || exit 1
  Result: SUCCESS (0)
  Marking as: GOOD

BISECT COMPLETE
===============

First bad commit: b4c3d2e1
  Hash:        b4c3d2e1a5f6e7d8c9b0a1
  Author:      mike.smith@company.com
  Date:        2025-02-24 09:15:33 -0500
  Message:     fix: cache database connections

Full diff:
  commit b4c3d2e1a5f6e7d8c9b0a1
  Author: Mike Smith <mike.smith@company.com>
  Date:   Mon Feb 24 09:15:33 2025 -0500

      fix: cache database connections

      - Add Redis-based connection pooling
      - Cache DB connections for 5 minutes
      - Reduces connection overhead

  diff --git a/src/db/connection.js b/src/db/connection.js
  index 8a9b0c1..2d3e4f5 100644
  --- a/src/db/connection.js
  +++ b/src/db/connection.js
  @@ -45,6 +45,12 @@ function getConnection() {
       return connectionPool.get();
     }

  +  // Cache connections in Redis
  +  const cached = await redis.get('db:connection');
  +  if (cached) {
  +    return deserialize(cached);
  +  }
  +
     const conn = await createNewConnection(config);
  +  await redis.setex('db:connection', 300, serialize(conn));
     return conn;
   }

Context: This is the only commit between v2.5.0 (good) and the bad state. The commit
introduces Redis caching for database connections. The health endpoint likely depends
on a database query that now fails when Redis is unavailable or when serialization
fails.

VERIFICATION:
  - Check Redis connectivity: `redis-cli ping`
  - Review logs: `tail -100 logs/error.log | grep -i redis`
  - Test health endpoint directly after commit: `git checkout b4c3d2e1 && npm start`

RECOMMENDATION:
  Option 1 (revert): git revert b4c3d2e1
  Option 2 (fix): Add Redis availability check and fallback to direct connection
  Option 3 (config): Ensure Redis is deployed with the new version

Run `git bisect reset` to exit bisect mode.
```

### Ejemplo 3: Blame profundo con contexto de commit

**Entrada de usuario:**
```
blame-deep src/auth/jwt.js "2 weeks ago"
```

**Salida de la habilidad:**
```
Deep blame analysis: src/auth/jwt.js
===================================

Line-by-line with commit context:
─────────────────────────────────────────────────────────────────────────────
  1  const jwt = require('jsonwebtoken');                   │ 3f2a1b4c │ Alice │ 2025-02-10
     └─ fix: add null check for headers (5 days ago)
  2  const config = require('../config');                  │ abc1234d │ Bob   │ 2025-02-15
     └─ chore: update config import path
  3                                                                             
  4  function verifyToken(token) {                         │ 7a3f9c2d │ Jane  │ 2025-02-28
     └─ feat: add request validation middleware (3 weeks ago)
  5    if (!token) {                                       │ 7a3f9c2d │ Jane  │ 2025-02-28
  6      throw new Error('Token required');                │ 7a3f9c2d │ Jane  │ 2025-02-28
  7    }                                                   │ 7a3f9c2d │ Jane  │ 2025-02-28
  8    return jwt.verify(token, config.secret);           │ 3f2a1b4c │ Alice │ 2025-02-10
     └─ fix: add null check for headers (5 days ago)
  9  }                                                     │ 3f2a1b4c │ Alice │ 2025-02-10
 10                                                                            
 11  function generateToken(user) {                        │ def56789 │ Carol │ 2025-01-20
     └─ feat: implement token generation (6 weeks ago)
 12    const payload = {                                   │ def56789 │ Carol │ 2025-01-20
 13      userId: user.id,                                  │ def56789 │ Carol │ 2025-01-20
 14      role: user.role,                                  │ def56789 │ Carol │ 2025-01-20
 15      iat: Math.floor(Date.now() / 1000)               │ def56789 │ Carol │ 2025-01-20
 16    };                                                  │ def56789 │ Carol │ 2025-01-20
 17    return jwt.sign(payload, config.secret);           │ def56789 │ Carol │ 2025-01-20
 18  }                                                     │ def56789 │ Carol │ 2025-01-20

SUMMARY:
  Total commits touching this file (last 2 weeks): 4
  Lines unchanged since 2025-02-10: 8 (lines 1, 5-7, 9, 12-16, 18)
  Most recent change: 7a3f9c2d by Jane Doe on 2025-02-28
  Authors: Alice (2 commits), Jane (1), Bob (1), Carol (1)

RISK INDICATORS:
  - Line 8 (jwt.verify call) was last modified 5 days ago in a "fix" commit
  - Lines 1-2 (imports) have 3 different authors in 2 weeks - high churn
  - No changes in last 3 days - stable period

RECOMMENDATION:
  If you're debugging JWT issues, start with commit 3f2a1b4c (last fix to verifyToken)
  and review 7a3f9c2d (most recent change to the file structure).
```

## Comandos de Reversión

**Revertir un commit único:**
```bash
git revert <commit-hash>
# Crea un nuevo commit que deshace los cambios
# Seguro para ramas compartidas
```

**Restablecer a un estado anterior (DESTRUCTIVO - solo local):**
```bash
git reset --hard <commit-hash>
# ADVERTENCIA: descarta todos los commits después de <commit-hash>
# Usar solo en ramas no compartidas
```

**Cherry-pick de una corrección desde otra rama:**
```bash
git checkout feature/fix-branch
git cherry-pick <good-commit-hash>
```

**Rebase interactivo para editar/aplastar commits:**
```bash
git rebase -i <commit-hash>^
# Marcar commit como "edit" para modificar, o "squash" para fusionar
# Usar solo en commits privados/no enviados
```

**Revertir manualmente líneas específicas:**
```bash
# Restaurar versión específica de archivo desde commit anterior
git checkout <good-commit-hash> -- path/to/file.js
# Luego commit la versión restaurada
git commit -m "revert: restaurar versión funcional de file.js desde <good-commit>"
```

**Bisect reset (limpieza):**
```bash
git bisect reset
# Retorna la rama al HEAD original y desactiva bisect
```

**Revertir parcial (solo hunks específicos):**
```bash
git revert -n <commit-hash>
# Preparar solo cambios seleccionados con git add -p
git commit -m "revert: reversión parcial de <commit-hash>"
```

## Pasos de Verificación

Después de identificar un commit sospechoso:
1. **Prueba de checkout**: `git checkout <commit-hash>` y ejecutar suite de pruebas para confirmar que el error aparece
2. **Verificación de rango**: `git log --oneline <good-commit>..<bad-commit>` para ver todos los commits en el rango
3. **Revisión de diff**: `git diff <good-commit> <bad-commit> -- path/to/file` para aislar cambios
4. **Cruce de blame**: `git blame <bad-commit> -- path/to/file` para ver propiedad de líneas en estado roto
5. **Verificación de build**: `git checkout <commit-hash> && npm test` (o equivalente) para reproducir

**Un análisis exitoso requiere:**
- Repositorio git con historial suficiente (mínimo 20 commits recomendados)
- Historial lineal (commits de merge pueden complicar bisect; usar `--first-parent` si es necesario)
- Comando de prueba que devuelva confiablemente 0 para bueno, no-0 para malo
- Árbol de trabajo limpio antes de operaciones bisect

## Solución de Problemas

**"fatal: ambiguous argument 'HEAD' - unknown revision or path not in the working tree"**
- Asegurarse de estar dentro de un repositorio git: `git rev-parse --is-inside-work-tree`
- Verificar que la rama/etiqueta existe: `git tag -l` o `git branch -a`

**"bisect start... you need to start by 'git bisect start'..."**
- Ya hay una sesión bisect activa. Ejecutar `git bisect reset` primero.

**"some test runs with no changes" durante bisect**
- El comando de prueba no es determinístico. Añadir semilla explícita: `npm test -- --seed=12345`
- O usar `CTT_BISECT_AUTO=false` para verificar manualmente cada commit

**Demasiados commits para analizar (rendimiento)**
- Establecer variable de entorno `CTT_MAX_COMMITS` para limitar alcance del análisis
- Añadir flags `--since`: `timeline --since="1 month ago"`
- Usar globs de archivos específicos: `hotspot-analysis 30 "src/components/*.tsx"`

**Blame muestra "Not Committed Yet"**
- El archivo tiene cambios sin commit. Ejecutar `git status` y commit o stash cambios primero.

**Cannot find introduction of pattern**
- El patrón puede haber sido renombrado o movido. Usar `git log --all -S'<pattern>'` para buscar cambios de contenido
- O usar `find-introduction` con patrón más amplio o verificar entre ramas: `git log --all --oneline --grep='<pattern>'`

**Conflicting bisect results**
- El comando de prueba es inestable. Añadir reintentos: `for i in {1..3}; do <test> && break || sleep 1; done`
- O usar `CTT_BISECT_AUTO=false` para verificar manualmente cada paso

**"error: pathspec '<file>' did not match any files"**
- El archivo fue eliminado en la rama actual. Usar `--follow` con commit explícito: `git log --follow -- <file>`
- O verificar si el archivo existe en otra rama: `git checkout <branch> -- <file>`

## Requisitos

- Repositorio git con historial suficiente (mínimo 20 commits recomendados)
- Historial lineal (los commits de merge pueden complicar bisect; usar `--first-parent` si es necesario)
- Comando de prueba que devuelva confiablemente 0 para bueno, no-0 para malo
- Árbol de trabajo limpio antes de operaciones bisect (verificación con `git status --porcelain`)