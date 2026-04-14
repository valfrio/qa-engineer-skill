# qa-engineer — QA Adversarial para Claude Code

> **No estás testeando para confirmar que las features funcionan — estás testeando para romperlas.**

Convierte a Claude en un QA engineer adversarial. Construido sobre los [Playwright Test Agents](https://playwright.dev/docs/test-agents) (Planner → Generator → Healer), estructurado en torno a **8 ángulos adversariales obligatorios**.

Se puede instalar de **dos formas** según tu flujo de trabajo: como **skill** (este repo) o como **comando `/qa`** (vive en [AppsurDesarrollo/claude-commands](https://github.com/AppsurDesarrollo/claude-commands)). Las dos comparten la metodología; cambia la forma de disparar el pase de QA.

---

## Skill vs Comando — ¿cuál instalar?

| | **Skill** (este repo) | **Comando `/qa`** |
|---|---|---|
| **Activación** | Automática (SUGGEST) + manual (EXECUTE) | Solo manual |
| **Tras un cambio behavioral** | Claude añade una línea de sugerencia al final del mensaje (~50 tokens): *"💡 Este cambio toca X. ¿Lanzo QA? → di `haz QA de X`"* | No pasa nada — tú decides cuándo invocar |
| **Cómo se dispara el pase** | Dices "haz QA de X", "sí" a una línea SUGGEST, o `/qa` | Escribes `/qa` |
| **Coste en reposo** | ~50 tokens por cada mensaje de implementación (la línea SUGGEST) | Cero |
| **Coste al ejecutar** | 100K–300K tokens por pase | 100K–300K tokens por pase |
| **Ventaja principal** | **Descubribilidad**: Claude te recuerda que QA está disponible tras cada cambio — no se te olvida ejecutarlo cuando toca | **Control total**: cero ruido, cero sugerencias. Tú decides 100% cuándo se hace QA |
| **Mejor para** | Devs que trabajan continuamente en un proyecto y quieren disciplina de QA por defecto | Equipos que prefieren QA como paso explícito (p.ej. pre-merge, pre-deploy) o que no quieren líneas extra en las respuestas de Claude |
| **Instalación** | Clonar este repo en `~/.agents/skills/qa-engineer/` | `/actualizar-comandos` desde AppsurDesarrollo/claude-commands |

**¿Puedo tener ambos?** Sí. La skill y el comando son complementarios — la skill añade la línea SUGGEST tras los cambios, y el comando `/qa` queda como atajo manual. Si tienes los dos instalados y dices `/qa`, se ejecuta el comando; si dices "haz QA", se ejecuta la skill en modo EXECUTE. El pase final es equivalente.

**Regla práctica:** empieza con el comando. Si notas que olvidas invocar QA cuando tocaría, instala también la skill para que te lo recuerde.

---

## Qué hace

- **Dos modos de activación — SUGGEST (gratis) y EXECUTE (bajo demanda).** Tras cualquier cambio behavioral, Claude añade una línea de sugerencia de QA al final de su mensaje (~50 tokens, funcionalmente gratis). El pase de QA completo solo se ejecuta cuando dices que sí — sin ejecución automática de tests, sin gasto sorpresa de tokens.
- **Fuerza una mentalidad adversarial.** Cada plan de test debe cubrir 8 ángulos: inputs vacíos, datos inválidos, valores límite, caracteres especiales/inyección, doble click/submit rápido, edges de navegación, **regresión en features cercanas**, y edges de auth/permisos.
- **Genera tests reales de Playwright** vía el loop de agentes Planner → Generator → Healer. Los tests se commitean y pasan a formar parte de una suite de regresión permanente.
- **Captura vídeo, traces y screenshots** en cada fallo. El trace viewer (`npx playwright show-trace`) te da un timeline paso a paso con snapshots del DOM, network log, console — superior al vídeo plano.
- **Se integra con CI/CD.** Incluye workflows de GitHub Actions, config de Docker y un script post-deploy. Opcional — la mayoría de equipos corren los tests en local contra el servidor desplegado.

---

## ¿Por qué "adversarial-first"?

La mayoría del QA falla porque intenta *confirmar* que la feature funciona. Un test que rellena un formulario con datos válidos y hace click en submit caza casi cero bugs reales — esos bugs nunca iban a llegar a producción de todos modos.

Los bugs que llegan a producción son los que nadie pensó en probar: submits vacíos, doble clicks, el botón atrás, el campo de email con un emoji, la página de al lado que compartía un componente con la que acabas de cambiar.

Esta skill codifica la disciplina de **intentar romper el sistema** en un workflow estructurado, repetible y orquestado por agentes. Los 8 ángulos adversariales son el suelo, no el techo. El Ángulo 7 (regresión en features cercanas) es el más saltado y el más valioso — esta skill lo hace no negociable.

---

## Instalación

Elige según lo que leíste en la tabla Skill vs Comando más arriba.

### Opción A — Instalar como skill (este repo)

La skill está diseñada para ser instalada por **Claude Code** en nombre del usuario. El workflow más simple:

#### 1. Pídele a Claude que la instale

En cualquier sesión de Claude Code, pega este prompt:

```
Instala la skill qa-engineer desde https://github.com/valfrio/qa-engineer-skill
en ~/.agents/skills/qa-engineer/. Sigue los pasos de instalación del README de ese repo.
Después de instalar, verifica que SKILL.md carga correctamente y reporta.
```

Claude:
1. Clonará el repo en `~/.agents/skills/qa-engineer/` (creando el directorio si hace falta).
2. Verificará que el directorio contiene `SKILL.md` y el subdirectorio `references/`.
3. Leerá `SKILL.md` para confirmar que el frontmatter parsea (`name: qa-engineer`).
4. Reportará éxito o cualquier error.

#### 2. Instalación manual (alternativa)

Si prefieres instalarla tú mismo:

```bash
# Linux / macOS
mkdir -p ~/.agents/skills
cd ~/.agents/skills
git clone https://github.com/valfrio/qa-engineer-skill qa-engineer

# Windows (Git Bash)
mkdir -p ~/.agents/skills
cd ~/.agents/skills
git clone https://github.com/valfrio/qa-engineer-skill qa-engineer
```

#### 3. Verificación

En Claude Code:

```
Lee ~/.agents/skills/qa-engineer/SKILL.md y confirma que la skill está instalada correctamente.
```

Claude debería responder con el nombre de la skill (`qa-engineer`), la versión, y un resumen breve de los 8 ángulos adversariales. Si lo hace, la skill está lista.

#### 4. (Solo el primer uso) Configura Playwright en el proyecto destino

La primera vez que la skill se invoque en un proyecto, Claude detectará que Playwright no está configurado y se ofrecerá a configurarlo. Solo di que sí. El setup está completamente documentado en [`references/setup.md`](references/setup.md) — Claude lo seguirá paso a paso.

Necesitarás:
- Node.js >= 18
- Una app web corriendo (URL local o URL de staging)
- (Si está protegida) credenciales de usuario de test

---

### Opción B — Instalar como comando `/qa`

El comando `/qa` vive en [AppsurDesarrollo/claude-commands](https://github.com/AppsurDesarrollo/claude-commands) junto con el resto de comandos del equipo. Si ya tienes instalados los comandos de AppsurDesarrollo, basta con ejecutar:

```
/actualizar-comandos
```

Esto descarga la última versión del repo y sincroniza `qa.md` + `qa/references/*` en `~/.claude/commands/`.

Si no tienes los comandos instalados todavía, el one-liner:

```bash
gh repo clone AppsurDesarrollo/claude-commands /tmp/appsur-claude-commands \
  && mkdir -p ~/.claude/commands \
  && cp /tmp/appsur-claude-commands/*.md ~/.claude/commands/ \
  && for dir in /tmp/appsur-claude-commands/*/; do \
       name=$(basename "$dir"); \
       [ "$name" = ".git" ] && continue; \
       rm -rf "$HOME/.claude/commands/$name"; \
       cp -R "$dir" "$HOME/.claude/commands/$name"; \
     done \
  && rm -rf /tmp/appsur-claude-commands
```

Para usarlo, escribe `/qa` en Claude Code. El comando ejecuta el mismo pase adversarial (6 fases: verificación, scope, 8 ángulos, Planner→Generator→Healer, metodología, report, commit) — la diferencia con la skill está solo en cómo se dispara, no en lo que hace.

---

## Cómo Claude usa esta skill — dos modos

La skill tiene **dos modos de activación**. Entender la diferencia es la clave para usarla de forma cost-effective.

### Modo SUGGEST (automático, ~50 tokens)

Después de que Claude haga cualquier **cambio behavioral** en tu proyecto (nueva feature, bug fix, refactor, cambio de schema, cambio de formulario/flujo), añade una sola línea al final de su mensaje:

> 💡 *Este cambio toca el formulario de checkout. ¿Lanzo QA adversarial? → di* `haz QA del checkout`

Eso es todo. No se ejecutan tests. No se disparan agentes. No se cargan archivos de referencia. La línea es un recordatorio de que QA está disponible, nada más. Coste: ~50 tokens.

Si ignoras la línea, no pasa nada. Si dices "no gracias" o "luego", Claude deja de sugerir QA en el resto de la conversación.

### Modo EXECUTE (manual, ~100K-300K tokens por feature)

Cuando pides QA explícitamente (o respondes sí a una línea SUGGEST), Claude ejecuta el workflow completo:

1. **Identifica qué testear Y su radio de explosión.** Claude lista 3–5 features cercanas que podrían estar afectadas (input del Ángulo 7).
2. **Ejecuta la suite de regresión existente** si la hay. Caza cualquier cosa que ya esté rota.
3. **Planea tests adversariales para la feature objetivo.** Claude llama al agente Planner de Playwright con los 8 ángulos inline.
4. **Audita el plan contra los 8 ángulos** antes de generar código. Si falta algún ángulo, Claude devuelve el plan para revisión.
5. **Genera tests de Playwright** vía el agente Generator.
6. **Los ejecuta.** Si hay fallos, el agente Healer auto-repara los selectores y los re-ejecuta.
7. **Reporta resultados** en un report estructurado de QA (ver `references/example-qa-report.md`).
8. **Commitea los tests que pasan** a la suite de regresión — solo después de tu aprobación.

### Cómo disparar el modo EXECUTE

Dile a Claude cualquiera de estas cosas (mezcla español e inglés libremente):

- "haz QA del checkout" / "testea esto" / "rompe el formulario de login"
- "valida la creación de pedidos adversarialmente"
- "run QA on the order flow" / "test this adversarially"
- "lanza la suite de regresión"

O simplemente responde "sí" / "yes" / "dale" / "go" a una línea SUGGEST que recibiste antes.

### Cuándo ejecutar el modo EXECUTE en la práctica

No hay regla. Patrones comunes:

- **Antes de mergear un PR** — pase adversarial sobre la feature modificada.
- **Antes de desplegar** — suite de regresión completa contra staging.
- **Después de un bug fix complicado** — verifica el fix y revisa el Ángulo 7 (rotura cercana).
- **Periódicamente** — pase completo sobre los flujos críticos semanalmente.
- **Sáltatelo** para prototipos, spikes, código throwaway. La línea SUGGEST seguirá apareciendo, pero ignórala tranquilamente.

### Los 8 ángulos adversariales (el corazón de la skill)

| # | Ángulo | Ejemplo de ataque |
|---|-------|----------------|
| 1 | Inputs vacíos | Submit de formulario vacío, body `{}` |
| 2 | Datos inválidos | Email `not-an-email`, fecha `2026-13-45` |
| 3 | Valores límite | `0`, `-1`, `MAX_INT`, string de 10K chars |
| 4 | Caracteres especiales / inyección | `<script>`, SQL injection, emoji, RTL |
| 5 | Doble click / submit rápido | Spam de submit 5 veces en 200ms |
| 6 | Edges de navegación | Botón atrás tras submit, refresh a mitad de flujo, multi-pestaña |
| 7 | **Regresión en features cercanas** | Tras editar el formulario X, re-testea la vista de lista, las páginas hermanas, los componentes compartidos, los exports |
| 8 | Edges de auth / permisos | Sin sesión, rol equivocado, sesión expirada, cross-tenant |

Descripciones completas, rationale y ejemplos de ataque por ángulo en [`SKILL.md`](SKILL.md).

### Lo que tú (el humano) revisas

Después de que Claude ejecute QA, revisa:

1. **El report de QA** que produce Claude (formato en `references/example-qa-report.md`). Mira: Top 5 bugs críticos, huecos de cobertura, tests nuevos añadidos.
2. **Los archivos de trace** en cualquier fallo: `npx playwright show-trace test-results/.../trace.zip`. Timeline interactivo con snapshots del DOM, network, console.
3. **Los archivos de test nuevos** que Claude commiteó a `tests/`. Pasan a ser permanentes — asegúrate de que están testeando lo correcto.
4. **La sección del Ángulo 7** del report. Si Claude no re-testeó las features cercanas, pregunta por qué.

---

## Ejecutar tests en local contra un servidor remoto (dev local → staging/prod)

No necesitas CI/CD para usar esta skill. El workflow más común es **ejecutar Playwright desde la máquina local del dev contra un servidor desplegado** (staging o similar a producción). Setup:

```bash
# En tu proyecto, después de instalar Playwright:
export BASE_URL=https://staging.example.com
export TEST_USER_EMAIL=qa@example.com
export TEST_USER_PASSWORD=tu_password_de_test

npx playwright test
```

O pásalas inline:

```bash
BASE_URL=https://staging.example.com TEST_USER_EMAIL=qa@example.com TEST_USER_PASSWORD=xxx npx playwright test
```

`playwright.config.ts` ya lee `BASE_URL` del entorno (ver [`references/setup.md`](references/setup.md)). Los tests pegarán al servidor desplegado, capturarán vídeos/traces localmente en `test-results/`, y producirán el report en tu máquina.

**Por qué esto basta para la mayoría de equipos:**

- Sin secrets de GitHub que gestionar.
- Sin minutos de CI consumidos.
- Los traces y vídeos se quedan en tu máquina — más fácil de inspeccionar.
- Tú decides cuándo ejecutar, no un pipeline.

La integración con GitHub Actions está documentada en [`references/ci-cd.md`](references/ci-cd.md) para los equipos que la quieran, pero es **opcional**.

---

## Estructura del repositorio

```
qa-engineer/
├── SKILL.md                          # Archivo principal de la skill (Claude lo lee al invocarse)
├── README.md                         # Este archivo (legible por humanos)
├── INSTALL.md                        # Guía de instalación paso a paso para Claude
├── CHANGELOG.md                      # Historial de versiones
├── LICENSE                           # MIT
├── playwright-qa-agents.skill        # Artefacto compilado (gzip), regenerado en cada release
└── references/
    ├── setup.md                      # Setup inicial de Playwright, auth, config de vídeo/trace
    ├── qa-methodology.md             # Categorías de test, patrones, referencia de la API de Playwright
    ├── ci-cd.md                      # GitHub Actions, Docker, script post-deploy
    ├── spec-template.md              # Ejemplo: plan de test bien escrito cubriendo los 8 ángulos
    └── example-qa-report.md          # Ejemplo: report de QA con bugs, severidad, recomendaciones
```

---

## Vídeo, traces y artefactos

Playwright captura tres tipos de evidencia en cada fallo (configurado en `playwright.config.ts`):

| Artefacto | Qué es | Cuándo usarlo |
|---------|-----------|-------------|
| **Vídeo** (`.webm`) | Grabación completa de la sesión del browser | Reproducción visual rápida de lo que el usuario "vio" |
| **Trace** (`.zip`) | Timeline interactivo: snapshots del DOM, network, console, log de acciones | **Herramienta principal de debugging.** Ábrelo con `npx playwright show-trace`. Click en cualquier paso → ve el DOM en ese momento |
| **Screenshot** (`.png`) | Imagen estática en el momento del fallo | Para incrustar en reports, compartir rápido |

Defaults configurados por esta skill:
- `video: 'on-first-retry'` — solo grabar en fallos
- `trace: 'on-first-retry'` — igual
- `screenshot: 'only-on-failure'`

En CI, los tres se suben como artefactos de GitHub Actions (configurado en [`references/ci-cd.md`](references/ci-cd.md)). Cualquiera (o cualquier sesión futura de Claude) puede descargarlos y revisar qué pasó.

---

## Contribuir

Las contribuciones son bienvenidas — fork, branch, PR. Antes de enviar:

1. El cambio debe alinearse con la filosofía adversarial-first. Los PRs que diluyan el Ángulo 7 o suavicen el mantra "intenta romperlo" serán rechazados.
2. Los ejemplos de ataque / patrones nuevos deberían añadirse a [`references/qa-methodology.md`](references/qa-methodology.md) y (si aplica) a la tabla de los 8 ángulos en [`SKILL.md`](SKILL.md).
3. Sube la versión en el frontmatter de `SKILL.md` y añade una entrada a `CHANGELOG.md`.

---

## Licencia

MIT — ver [`LICENSE`](LICENSE).

---

## Agradecimientos

Construida sobre los [Playwright Test Agents](https://playwright.dev/docs/test-agents) de Microsoft (Planner / Generator / Healer). Inspirada por la filosofía de testing adversarial de [`expect-cli`](https://www.npmjs.com/package/expect-cli), extendida con loops de agentes estructurados, disciplina de regresión e integración CI/CD.
