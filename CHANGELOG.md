# Changelog

Todos los cambios notables de la skill `qa-engineer` están documentados aquí.

El formato está basado en [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), y esta skill sigue [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [2.2.1] — 2026-04-10

Traducción completa al español de toda la documentación de la skill (SKILL.md, README, INSTALL, CHANGELOG y los 5 archivos de `references/`). El comportamiento es idéntico a v2.2.0 — solo cambia el idioma.

### Cambiado

- **Toda la documentación traducida al español.** SKILL.md, README.md, INSTALL.md, CHANGELOG.md y los 5 archivos de `references/`. Los términos técnicos (Planner, Generator, Healer, trace, snapshot, browser context, etc.) se mantienen en inglés. Los bloques de código, comandos CLI y nombres de variables también se mantienen en inglés. Las frases trigger de activación del modo EXECUTE están en ambos idiomas para que la skill matchee tanto si el dev escribe en español como en inglés.
- **Los prompts al Planner se documentan en inglés** porque el agente Playwright procesa mejor instrucciones en inglés. SKILL.md ahora explica explícitamente este patrón: tú piensas en español, traduces el prompt antes de mandarlo al agente.
- **Bumped frontmatter `version` a `2.2.1`** (patch — solo cambio de idioma, comportamiento idéntico).

### Por qué

El equipo de AppsurDesarrollo trabaja en español. Tener la skill en inglés añadía fricción de lectura al revisar reports de QA, debuggear comportamiento de la skill, o entender el flujo de los dos modos. La traducción reduce esa fricción a coste cero (es solo documentación, no cambia el código de Playwright ni la forma en que la skill se activa).

---

## [2.2.0] — 2026-04-10

Reintroducida la activación automática de forma token-safe. La skill ahora tiene **dos modos**: SUGGEST (auto, ~50 tokens) y EXECUTE (manual, ~100K-300K tokens). Reemplaza el modelo on-demand puro de v2.1.0, que perdió el beneficio de descubribilidad de la auto-activación.

### Añadido

- **Modo SUGGEST.** Cuando Claude hace un cambio behavioral, la skill se carga automáticamente y Claude añade una línea de sugerencia de QA al final de su mensaje — por ejemplo: *"💡 Este cambio toca el formulario de checkout. ¿Lanzo QA adversarial? → di `haz QA del checkout`"*. No se ejecutan tests, no se disparan agentes, no se cargan archivos de referencia. Coste: ~50 tokens por sugerencia.
- **Modo EXECUTE.** Disparado cuando el usuario pide QA explícitamente O responde "sí" / "yes" / "dale" / "go" a una línea SUGGEST. Ejecuta el workflow completo Planner → Generator → Healer.
- **Tabla de resolución de modos** en `SKILL.md` cubriendo las situaciones más comunes (cambio behavioral, fix de docs, petición de code review, petición explícita de QA, rechazo) y qué modo aplica a cada una.
- **Reglas duras del modo SUGGEST** que impiden a Claude ejecutar tests, leer archivos de referencia, llamar a agentes, o repetir sugerencias después de que el usuario rechace.
- **Sección de README.md "Ejecutar tests en local contra un servidor remoto".** Documenta el workflow dev-local-contra-staging que la mayoría de equipos usan en realidad, con ejemplos de la variable de entorno `BASE_URL`. Hace que CI/CD sea explícitamente opcional.

### Cambiado

- **`description` del frontmatter** reescrito para definir ambos modos con precisión y que Claude pueda distinguirlos en el momento de activación. Incluye las frases explícitas de invocación del modo EXECUTE y las condiciones del trigger del modo SUGGEST.
- **Sección "ACTIVATION RULE" de `SKILL.md`** reescrita como "REGLAS DE ACTIVACIÓN — DOS MODOS" con un diagrama de flujo ASCII, subsecciones separadas de "Cuándo usar" para cada modo, la tabla de resolución y el rationale del diseño.
- **Principio Clave #2** cambiado de *"Solo activación on-demand"* a *"Dos modos: SUGGEST (auto, gratis) y EXECUTE (manual, caro)"*.
- **Anti-patrones** actualizados: quitado *"Auto-disparar QA sin que se pida"*, añadidos cuatro anti-patrones más específicos: ejecutar EXECUTE sin consentimiento, leer archivos de referencia en modo SUGGEST, añadir la línea de sugerencia a cambios no behaviorales, repetir la sugerencia después de que el usuario rechace.
- **Mensaje de confirmación del Paso 7 de `INSTALL.md`** reescrito para explicar los dos modos al usuario desde el principio, con la distinción SUGGEST/EXECUTE y la garantía dura de que EXECUTE nunca corre sin consentimiento.
- **Sección "How Claude uses this skill" de `README.md`** reescrita en torno al modelo de dos modos con los números de coste por modo visibles.
- **Bumped `version` del frontmatter a `2.2.0`** (minor — cambio de comportamiento aditivo, totalmente backwards compatible con prompts de v2.1.0).

### Por qué

v2.1.0 arregló el problema de coste de tokens de v2.0.0 haciendo la skill totalmente on-demand, pero introdujo un problema nuevo: **descubribilidad**. Los usuarios tenían que acordarse de pedir QA. Muchos se olvidaban. La disciplina que el Ángulo 7 se suponía que tenía que aplicar se saltaba porque nadie invocaba la skill.

v2.2.0 resuelve ambos problemas con el diseño de dos modos:

- **SUGGEST es funcionalmente gratis** (~50 tokens por mensaje de cambio behavioral). Claude le recuerda al usuario que QA está disponible, el usuario siempre lo sabe.
- **EXECUTE solo corre con consentimiento.** Sin gasto sorpresa de tokens. Misma disciplina de coste que v2.1.0.

La descubribilidad se recupera sin pagar el impuesto de tokens de v2.0.0. Lo mejor de las dos versiones anteriores.

---

## [2.1.0] — 2026-04-10

Modelo de activación cambiado de auto-trigger a on-demand. Motivado por el coste de tokens: el loop del agente es caro (5–10× el coste de la implementación que testea), y ejecutarlo tras cada cambio quemaba presupuesto sin darle al usuario control sobre el timing del QA.

### Cambiado

- **La activación es ahora SOLO ON DEMAND.** La skill ya no se auto-dispara después de implementaciones, bug fixes o refactors. Claude solo la invoca cuando el usuario pide QA explícitamente ("haz QA", "test this", "rompe esto", "run QA on X", etc.).
- **Sección "AUTO-TRIGGER RULE" de `SKILL.md`** renombrada a "ACTIVATION RULE". Reescrita para listar las frases explícitas que activan la skill y los casos que **no** la activan. Incluye el rationale (coste de tokens + control del usuario).
- **Sección de Workflow de `SKILL.md`** renombrada de "Post-Implementation QA (Auto-Triggered)" a "On-Demand QA Pass".
- **Principio Clave #2** cambiado de *"Cada implementación dispara QA"* a *"Solo activación on-demand"*.
- **Anti-patrón añadido:** *"Auto-disparar QA sin que se pida"*.
- **`description` del frontmatter** reescrito para hacer las reglas de activación inequívocas para Claude al matchear la skill: frases explícitas que activan, casos explícitos que no.
- **Paso 6 de `INSTALL.md`** cambiado de "auto-detectar y ejecutar smoke check" a "detección de contexto solo lectura". La instalación ya no ejecuta tests, ya no crea archivos, ya no instala Playwright. Todo eso pasa más tarde, solo cuando el usuario pide QA.
- **Mensaje de confirmación del Paso 7 de `INSTALL.md`** reescrito para decirle al usuario que la skill es on-demand y listar las frases de invocación de ejemplo.
- **Sección "How Claude uses this skill" de `README.md`** reescrita en torno a la invocación explícita. Incluye guidance de "Cuándo invocarla" (antes de mergear, antes de desplegar, después de bug fixes complicados, periódicamente).
- **Bumped `version` del frontmatter a `2.1.0`** (minor — cambio de comportamiento, sin break de API).

### Por qué

Las versiones anteriores de esta skill eran agresivamente automáticas: cada implementación disparaba un pase de QA. Esto tenía dos problemas reales:

1. **Coste de tokens.** El Planner de Playwright lee los accessibility trees de cada página que explora. Un pase de QA completo sobre una feature no trivial cuesta 5–10× los tokens de la implementación que lo disparó. Ejecutar QA tras cada cambio trivial (renombrar una variable, ajustar una regla CSS, arreglar un typo en el nombre de una ruta) quemaba presupuesto sin valor proporcional.
2. **Control del usuario.** El timing del QA es una decisión del developer. Pueden querer agruparlo al final de la sesión, ejecutarlo solo antes de mergear, saltárselo para spikes, o ejecutarlo en una máquina diferente. Auto-disparar le quitaba esa decisión.

La skill sigue siendo agresiva sobre la *calidad* cuando se invoca — los 8 ángulos, el Ángulo 7, el checklist de auditoría, el mantra — pero la invocación es ahora decisión del usuario.

---

## [2.0.0] — 2026-04-10

Reescritura mayor. La skill es ahora adversarial-first por diseño y empaquetada para distribución vía GitHub.

### Añadido

- **Mantra central** al principio de `SKILL.md`: *"No estás testeando para confirmar que las features funcionan — estás testeando para romperlas."*
- **El checklist de los 8 Ángulos Adversariales** como concepto de primer nivel en `SKILL.md`. Cada plan de test debe cubrir los ángulos aplicables. Cinco de ocho es el mínimo para features no triviales.
- **Sección profunda del Ángulo 7** — *Regresión en features cercanas*. Promovido de un bullet olvidable a una regla no negociable con una tabla de mapeo "tipo de cambio → features cercanas" y un workflow de aplicación de 4 pasos.
- **Sección "Cómo escribir instrucciones para el Planner"** con una comparación de prompt bad/good, una plantilla copiable, 7 reglas para escribir prompts, y un checklist de auditoría de 5 preguntas para revisar planes antes de mandarlos al Generator.
- **`README.md`** — introducción legible por humanos con instrucciones de instalación optimizadas para Claude como ejecutor.
- **`INSTALL.md`** — guía de instalación one-shot orientada a Claude. Le guía paso a paso por clone, verify, frontmatter check y report.
- **`references/spec-template.md`** — ejemplo completamente desarrollado de un plan de test adversarial cubriendo los 8 ángulos.
- **`references/example-qa-report.md`** — ejemplo completamente desarrollado de un report de QA con bugs clasificados por severidad, top 5 críticos, huecos de cobertura y tests nuevos añadidos.
- **`LICENSE`** — MIT.
- **Metadata del frontmatter**: `license`, `metadata.author`, `metadata.version`, `metadata.repository`.

### Cambiado

- **`name` del frontmatter** cambiado de `playwright-qa-agents` a `qa-engineer` para alinear con el nombre del directorio y reflejar mejor el framing basado en rol.
- **`description` del frontmatter** reescrito para mencionar el framing adversarial, los 8 ángulos y el Ángulo 7 explícitamente. Esto mejora el matching del auto-trigger y le da a Claude el frame correcto en la primera carga.
- **Paso 1 del workflow post-implementación** renombrado a *"Identifica qué cambió Y qué hay al lado"* y ahora requiere que Claude nombre 3–5 vecinos específicos antes de que el Planner ejecute.
- **Paso 3a (Plan)** reescrito para usar la plantilla adversarial con los 8 ángulos inline. Los prompts vagos son ahora un anti-patrón explícito.
- **Principios Clave** expandidos de 10 a 12. El Principio #1 es ahora el mantra central. Los Principios #3 y #4 codifican el suelo de los 8 ángulos y el Ángulo 7 como no negociable.
- **Anti-patrones** expandidos con 5 entradas nuevas: prompts vagos al Planner, render-checking, saltarse el Ángulo 7, planes mayoritariamente happy path, confiar ciegamente en el Planner.
- **Estructura de archivos** — `setup.md`, `qa-methodology.md` y `ci-cd.md` movidos a `references/` para que coincida con el path que `SKILL.md` ya documentaba.

### Filosofía

La versión 1.x de esta skill era un documento de metodología con un auto-trigger. La versión 2.x es una **disciplina codificada como workflow**: Claude ya no puede "hacer QA" escribiendo un plan de test happy path. Los 8 ángulos y la auditoría del Ángulo 7 hacen la disciplina mecánica.

---

## [1.0.0] — 2026-03-15

Versión inicial (interna, sin empaquetar).

### Añadido

- Auto-trigger después de cada implementación.
- Integración con los Playwright Test Agents (Planner, Generator, Healer).
- Metodología de QA genérica con 7 categorías de test.
- Guía de setup, guía de CI/CD, referencia de troubleshooting.
- Patrones de testing multi-contexto multi-usuario.
- Patrones de la API Clock para tests dependientes de timing.
