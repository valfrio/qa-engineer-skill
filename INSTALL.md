# INSTALL — Para Claude (el ejecutor)

> Este archivo está escrito para **Claude leyéndolo durante la instalación**, no para humanos. Si eres humano, lee [`README.md`](README.md) en su lugar.

A ti (Claude) te han pedido instalar la skill `qa-engineer` desde este repositorio en el directorio de skills de Claude Code del usuario. Sigue estos pasos **en orden**. No saltes pasos de verificación.

---

## Paso 1 — Determina el destino de la instalación

La skill debe vivir en `~/.agents/skills/qa-engineer/`. Resuelve este path en el OS del usuario:

| OS | Path resuelto |
|----|---------------|
| Linux / macOS | `~/.agents/skills/qa-engineer/` |
| Windows (Git Bash) | `~/.agents/skills/qa-engineer/` (mapea a `C:/Users/<user>/.agents/skills/qa-engineer/`) |
| Windows (cmd / PowerShell) | `%USERPROFILE%\.agents\skills\qa-engineer\` |

**Prefiere siempre Git Bash en Windows** — el resto de esta guía asume sintaxis de shell Unix.

---

## Paso 2 — Comprueba si ya está instalada

```bash
ls ~/.agents/skills/qa-engineer/SKILL.md 2>/dev/null && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

- Si `ALREADY_INSTALLED` → pregunta al usuario si quiere actualizar (`git pull`) o reinstalar (borrar + clonar).
- Si `NOT_INSTALLED` → continúa al Paso 3.

---

## Paso 3 — Clona el repositorio

```bash
mkdir -p ~/.agents/skills
cd ~/.agents/skills
git clone https://github.com/AppsurDesarrollo/qa-engineer-skill qa-engineer
```

Si el usuario no tiene acceso a la red o la URL del repo falla, **detente** y pregunta al usuario cómo quiere proceder (descarga manual, mirror distinto, etc.). No te inventes URLs de fallback.

---

## Paso 4 — Verifica la estructura de archivos

```bash
ls ~/.agents/skills/qa-engineer/
```

Output esperado (debe contener al menos):

```
SKILL.md
README.md
INSTALL.md
CHANGELOG.md
LICENSE
references/
```

Y dentro de `references/`:

```
setup.md
qa-methodology.md
ci-cd.md
spec-template.md
example-qa-report.md
```

Si falta cualquier archivo requerido, **detente y reporta los archivos que faltan al usuario**. No intentes recrearlos — la instalación está rota.

---

## Paso 5 — Verifica que el frontmatter de SKILL.md parsea

Lee `~/.agents/skills/qa-engineer/SKILL.md` y confirma:

- El archivo empieza con un bloque de frontmatter `---`.
- El campo `name:` es `qa-engineer`.
- El campo `description:` no está vacío y menciona "adversarial".
- `metadata.version:` existe.

Si alguna de estas comprobaciones falla, el archivo está corrupto. Detente y reporta.

---

## Paso 6 — Detecta el contexto del proyecto (solo lectura)

La instalación **no** ejecuta ningún test. La skill funciona en dos modos: solo se activa el workflow completo cuando el usuario pide QA explícitamente más adelante. El Paso 6 es un pase silencioso de detección para que tu mensaje de confirmación le diga al usuario qué tiene.

```bash
# ¿Estamos dentro de algún proyecto?
test -f package.json && echo "NODE_PROJECT" || echo "NO_NODE_PROJECT"

# ¿Playwright ya está configurado?
test -f playwright.config.ts -o -f playwright.config.js && echo "PLAYWRIGHT_READY" || echo "PLAYWRIGHT_MISSING"

# ¿Hay tests existentes?
test -d tests -o -d e2e -o -d __tests__ && echo "TESTS_DIR_EXISTS" || echo "NO_TESTS_DIR"
```

Solo registra los resultados. **No** instales Playwright. **No** crees archivos. **No** ejecutes tests. Todo eso pasa más tarde, solo cuando el usuario pida QA sobre algo concreto.

---

## Paso 7 — Confirma al usuario

Después del Paso 6, reporta:

```
✅ Skill qa-engineer instalada en ~/.agents/skills/qa-engineer/
   Versión: <versión del frontmatter de SKILL.md>

Estado del proyecto: <uno de NO_NODE_PROJECT | PLAYWRIGHT_MISSING | PLAYWRIGHT_READY>
<Si TESTS_DIR_EXISTS:>  Tests existentes detectados — se usarán como baseline de regresión.
<Si PLAYWRIGHT_MISSING:>  Playwright no está configurado todavía. Lo configuraré la primera vez que aceptes una sugerencia de QA.

Cómo funciona esta skill (dos modos):

  📌 Modo SUGGEST (automático, ~50 tokens):
     Después de cualquier cambio behavioral que haga, añadiré una línea de
     sugerencia al final del mensaje ofreciendo un pase de QA adversarial.
     No se ejecutan tests. Si ignoras la línea, no pasa nada.

  ▶️  Modo EXECUTE (solo cuando tú lo digas, ~100K-300K tokens):
     Se dispara cuando dices "haz QA", "testea esto", "rompe esto", "run QA",
     o simplemente respondes "sí" / "yes" / "dale" a una línea SUGGEST.
     Ejecuta el workflow completo Planner → Generator → Healer y produce un report.

NUNCA ejecutaré un pase de QA completo sin tu consentimiento explícito.
```

---

## Paso 8 — Reglas duras

- **No** instales Playwright globalmente en la máquina del usuario. Playwright es per-proyecto. Instálalo dentro de los `node_modules` del proyecto solo cuando el usuario pida QA explícitamente Y no exista config de Playwright todavía.
- **No** modifiques ningún archivo dentro de `~/.agents/skills/qa-engineer/` después de instalar. Si el usuario quiere personalizar la skill, hace fork del repo y reinstala desde su fork.
- **No** modifiques el `.bashrc`, `.zshrc` ni ninguna config de shell del usuario. La skill la carga Claude Code, no la shell.
- **No** instales en `.claude/skills/` (project-local) por defecto. Esta skill está pensada para ser global. Si el usuario pide explícitamente instalación project-local, instálala en `<project>/.agents/skills/qa-engineer/` y avísale que solo aplicará a ese proyecto.
- **No** pushees tests, traces ni test-results a git como parte de la instalación. Commitear tests pasa más tarde, después de que el usuario revise el report de QA.

---

## Actualizar la skill

Si el usuario pide actualizarla:

```bash
cd ~/.agents/skills/qa-engineer
git pull
```

Después re-ejecuta el Paso 4 (verificación de archivos) y el Paso 5 (verificación del frontmatter). Reporta la versión nueva desde `SKILL.md`.

---

## Desinstalar la skill

```bash
rm -rf ~/.agents/skills/qa-engineer
```

Confirma con el usuario antes de ejecutar `rm -rf`. Reporta:

```
✅ Skill qa-engineer desinstalada. Los tests que generaste con ella siguen
en el directorio tests/ de tu proyecto y seguirán funcionando.
```
