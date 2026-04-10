---
name: qa-engineer
description: "QA E2E adversarial. Dos modos de activación — modo SUGGEST (auto, bajo coste) y modo EXECUTE (a petición del usuario, caro). El mantra central: NO estás testeando para confirmar que las features funcionan — estás testeando para INTENTAR ROMPERLAS. Usa los Playwright Test Agents (Planner → Generator → Healer) orquestados por Claude, estructurados en torno a 8 ángulos adversariales obligatorios (inputs vacíos, datos inválidos, valores límite, caracteres especiales/inyección, doble click, edge cases de navegación, regresión en features cercanas, edges de auth/permisos). El Ángulo 7 (regresión en features cercanas) es no negociable: cada pase de QA re-testea los vecinos del código que cambió. Modo SUGGEST: carga esta skill en contexto siempre que Claude haga cualquier cambio behavioral (nueva feature, bug fix, refactor, cambio de schema/API, modificación de un formulario o flujo). En modo SUGGEST Claude NO ejecuta ningún test — añade una línea de sugerencia al final del mensaje de implementación ofreciendo un pase adversarial con la frase exacta para invocarlo. Modo EXECUTE: cuando el usuario pide QA explícitamente con frases como 'haz QA', 'testea esto', 'rompe esto', 'valida esto', 'prueba la app', 'lanza tests', 'run QA', 'test this adversarially', o responde sí/dale/yes/go a una línea SUGGEST, ejecuta el workflow completo Planner → Generator → Healer. NUNCA ejecutes el workflow completo sin consentimiento explícito del usuario. Cubre: setup inicial de Playwright, setup de auth para apps protegidas, generación adversarial de planes de test, ejecución de tests, self-healing, integración CI/CD opcional con artefactos de vídeo/trace, y una metodología completa de QA para validación exhaustiva. Funciona con cualquier app web independientemente del stack."
license: MIT
metadata:
  author: AppsurDesarrollo
  version: "2.2.0"
  repository: "https://github.com/AppsurDesarrollo/qa-engineer-skill"
---

# qa-engineer — Skill de Testing E2E Adversarial

## Mantra Central

> **No estás testeando para confirmar que la feature funciona. Estás testeando para intentar romperla.**

Esta única frase es el alma de la skill. Interiorízala antes de cada sesión de QA:

- Un test que "verifica que la página renderiza" es **inútil**. Solo caza los bugs que nadie habría enviado a producción de todos modos.
- Un test que envía un formulario vacío, hace doble click en el botón, navega hacia atrás a mitad del flujo, y verifica que una feature cercana sigue funcionando **se gana su sitio**.
- Si terminas una sesión de QA sin haber intentado romper algo, **no hiciste QA** — hiciste un smoke test.

Los agentes Planner, Generator y Healer son herramientas. La mentalidad es tuya. Cada prompt que mandas al Planner, cada test que escribe el Generator, cada arreglo que aplica el Healer — todo debe estar al servicio de *intentar romper el sistema* en nombre del usuario.

> **"Si no lo has validado intentando romperlo, no funciona."**

---

## REGLAS DE ACTIVACIÓN — DOS MODOS (CRÍTICO — léelo antes de hacer nada)

Esta skill tiene **dos modos de activación**. El modo determina si Claude *sugiere* QA o *ejecuta* QA. **Confundirlos cuesta tokens y confianza del usuario.**

```
                  ┌─────────────────────┐
                  │   Skill cargada en  │
                  │  contexto de Claude │
                  └──────────┬──────────┘
                             │
              ┌──────────────┴───────────────┐
              │                              │
              ▼                              ▼
    ┌─────────────────┐            ┌──────────────────┐
    │   Modo SUGGEST  │            │   Modo EXECUTE   │
    │ (auto, ~50 tok) │            │ (manual, ~200K)  │
    └─────────────────┘            └──────────────────┘
    Disparado porque Claude        Disparado porque el
    está haciendo un cambio        usuario pide QA
    behavioral.                    explícitamente.
    
    Claude añade UNA línea         Claude ejecuta el
    de sugerencia al final         workflow completo
    de su mensaje.                 Planner → Generator →
    NO se ejecutan tests.          Healer. Tests corren,
                                   se produce un report.
```

### Modo SUGGEST — cuándo usarlo

**Disparador:** Claude acaba de hacer o está a punto de hacer un **cambio behavioral** en el proyecto del usuario. Ejemplos:

- Nueva feature o componente
- Bug fix que cambia el comportamiento en runtime
- Refactor que toca un flujo
- Endpoint de API añadido o modificado
- Migración de schema de base de datos
- Formulario, flujo o interacción de UI modificados

**Acción:** Termina la implementación normalmente. Reporta lo que hiciste. Después, **al final de tu mensaje**, añade exactamente una línea de sugerencia en el idioma del usuario:

> 💡 *Este cambio toca [feature]. ¿Lanzo QA adversarial? → di* `haz QA del [feature]`

o en inglés:

> 💡 *This change touches [feature]. Want me to run an adversarial QA pass? → say* `run QA on [feature]`

**Reglas duras del modo SUGGEST:**

- **NO** ejecutes ningún test, agente o comando de Playwright. El sentido del modo SUGGEST es no quemar tokens.
- **NO** leas `qa-methodology.md`, `setup.md` ni ningún otro archivo de referencia. Solo se cargan en modo EXECUTE.
- **NO** generes un plan de test, llames al Planner ni invoques ninguna herramienta de browser.
- **NO** añadas la línea de sugerencia a mensajes que no impliquen un cambio behavioral (docs puras, comentarios, estilos, ajustes de config, conversación, planificación).
- **NO** añadas la línea de sugerencia si el usuario ya rechazó QA antes en la misma conversación. Pregunta una vez, respeta la respuesta.
- **NO** añadas más de una línea de sugerencia por mensaje. Una línea, al final. Punto.

### Modo EXECUTE — cuándo usarlo

**Disparador:** el usuario pide QA explícitamente con cualquiera de estas frases (en cualquier idioma):

- "haz QA", "testea esto", "rompe esto", "valida esto", "prueba la app", "lanza los tests", "haz un pase adversarial"
- "run QA", "test this", "test this adversarially", "break this", "regression test", "run the playwright suite"
- El usuario responde sí / claro / dale / yes / go a una línea SUGGEST que añadiste antes
- El usuario invoca la skill por nombre (`/qa`, "usa la skill qa-engineer")
- Cualquier mención explícita a QA, testing E2E, Playwright, regresión, break testing o testing adversarial en forma de petición

**Acción:** Ejecuta el **Workflow: Pase de QA bajo demanda** completo (Pasos 1–6) más abajo. Lee los archivos de referencia que necesites. Genera el plan, ejecuta los agentes, produce el report de QA.

### Tabla de resolución de modos

| Situación | Modo | Qué hace Claude |
|-----------|------|------------------|
| Usuario dice "añade un nuevo campo al formulario de pedidos" | SUGGEST | Implementa → al final del mensaje: *"💡 Este cambio toca el formulario de pedidos. ¿Lanzo QA adversarial?"* |
| Usuario dice "arregla el typo del README" | NINGUNO | No hay cambio behavioral. No hay línea de sugerencia. |
| Usuario dice "haz QA del checkout" | EXECUTE | Ejecuta el workflow completo sobre la feature checkout |
| Usuario dice "sí" justo después de una línea SUGGEST | EXECUTE | Ejecuta sobre la feature mencionada en la línea SUGGEST |
| Usuario dice "no gracias" o "luego" a una línea SUGGEST | NINGUNO | Deja de sugerir QA en el resto de la conversación |
| Usuario hace una pregunta sobre QA sin pedir ejecutarlo | NINGUNO | Responde la pregunta, sin línea de sugerencia |
| Usuario dice "revisa este código" | NINGUNO | Es code review, no QA. Sin línea de sugerencia. |

### Por qué dos modos

La versión anterior de esta skill ejecutaba el workflow completo automáticamente tras cada cambio. Eso costaba 100K-300K tokens por cambio. Con este diseño de dos modos:

- **El modo SUGGEST cuesta ~50 tokens por mensaje de implementación** (una línea añadida). Funcionalmente gratis.
- **El modo EXECUTE cuesta el pase de QA completo** pero solo cuando el usuario opta por ejecutarlo.

El usuario siempre sabe que QA está disponible porque Claude se lo recuerda (SUGGEST). El usuario siempre controla el coste porque Claude espera la luz verde (EXECUTE). Lo mejor de ambos.

---

## Visión general

Esta skill configura y ejecuta QA exhaustivo automatizado usando los **Playwright Test Agents** (Planner → Generator → Healer) con Claude como loop de IA orquestador. Valida cualquier app web a través de happy paths, edge cases, condiciones de error, break testing y suites de regresión.

**Tres agentes trabajando juntos:**
- **🎭 Planner** — Explora la app, produce planes de test en Markdown
- **🎭 Generator** — Transforma los planes en archivos de Playwright Test ejecutables
- **🎭 Healer** — Ejecuta los tests fallidos, auto-repara selectores/waits, los re-ejecuta

---

## Referencia rápida

Antes de empezar, lee el archivo de referencia que corresponda:

| Tarea | Archivo de referencia |
|------|----------------|
| Setup inicial de Playwright + Agentes | `references/setup.md` |
| Metodología de QA, categorías de test y patrones | `references/qa-methodology.md` |
| Integración CI/CD y workflow post-deploy | `references/ci-cd.md` |

**Lee siempre `references/setup.md` primero si el proyecto no tiene Playwright configurado.**

---

## Los 8 Ángulos Adversariales (memoriza este checklist)

Cada plan de test, cada prompt al Planner, cada archivo de spec que escribas **debe** considerar estos ocho ángulos. Son la superficie mínima de un pase de QA real. Si un plan de test cubre menos de cinco para una feature no trivial, el plan está incompleto — devuélvelo al Planner.

| # | Ángulo | Qué significa | Ejemplo de ataque |
|---|-------|--------------|----------------|
| **1** | **Inputs vacíos** | Enviar formularios / llamar endpoints sin nada relleno | Submit del checkout con carrito vacío, PATCH con body `{}` |
| **2** | **Datos inválidos** | Tipos equivocados, formatos equivocados, shapes equivocados | Email `not-an-email`, teléfono `abc`, fecha `2026-13-45`, JSON sin las claves obligatorias |
| **3** | **Valores límite** | Los bordes donde se esconden los bugs: `0`, `-1`, `null`, `""`, `MAX_INT`, `10K caracteres` | Cantidad `0`, precio negativo, nombre de 10.000 caracteres, año `1900` y `2099` |
| **4** | **Caracteres especiales e inyección** | Unicode, emoji, RTL, scripts, SQL, path traversal | `<script>alert(1)</script>`, `'; DROP TABLE--`, `🎉日本語`, `../../etc/passwd` |
| **5** | **Doble click / submit rápido** | Pone al usuario contra sí mismo y contra la red | Doble click en "Place order", spam en "Send", click en submit 5 veces en 100ms |
| **6** | **Edge cases de navegación** | Mecánicas del browser que se saltan las suposiciones de la app | Botón atrás tras submit, refresh a mitad del pago, URL directa al paso 3, abrir el mismo flujo en 2 pestañas, deep-link a un recurso borrado |
| **7** | **Regresión en features cercanas** | El cambio que acabas de hacer probablemente rompió algo *al lado* — ve a mirar | Tras editar el formulario de checkout, verifica que la paginación del carrito, los filtros, la lista de pedidos y el formulario *quote/draft* siguen funcionando (¡componentes compartidos!) |
| **8** | **Edges de auth y permisos** | Sin sesión, rol equivocado, sesión expirada, cross-tenant | Pegar a la URL como guest, como rol equivocado, después de `clearCookies()`, con el ID de otro tenant en la URL |

### Cómo usar el checklist

- **Por feature:** para cualquier cambio no trivial, el Planner debe producir al menos un test por ángulo aplicable. Cinco de ocho es el suelo; ocho de ocho es el estándar.
- **Por prompt al Planner:** pega los ángulos relevantes directamente en el prompt — no confíes en que el agente los recuerde. Ver *Cómo escribir instrucciones para el Planner* más abajo.
- **Por revisión:** al leer el output del Planner antes de generar tests, cuenta los ángulos. Ángulo faltante = revisa el plan, no continúes al Generator.
- **El Ángulo 7 es el más olvidado.** Hazlo regla dura: cada cambio de feature se entrega con al menos un test de regresión contra una feature cercana. Ver la sección dedicada más abajo.

---

## Ángulo 7 en profundidad — Regresión en Features Cercanas

La mayoría de los "bugs pequeños" no son bugs en la cosa que cambiaste. Son bugs en la cosa *al lado* de lo que cambiaste — un componente hermano, un servicio compartido, una vista de lista que consumía una API cuyo shape acaba de cambiar.

Cuando el Planner explora una feature modificada, también debe recibir la orden de **dar un paso hacia afuera** y re-testear los vecinos. Concretamente:

### Qué cuenta como "cercano"

| Tipo de cambio | Features cercanas a re-testear |
|---------------|----------------------------|
| Formulario / página | Otras páginas que usan el mismo componente, páginas hermanas en el mismo módulo, la vista de lista a la que alimenta este formulario |
| Modelo / relación de ORM | Cada página que lista, filtra o agrega este modelo; cada PDF / export / endpoint de API que lo serializa |
| Endpoint de API | Cada consumidor de ese endpoint (UI, móvil, webhooks, terceros) |
| Componente compartido (botones, cards, formularios reutilizables) | Al menos 3 páginas aleatorias que ya usan el componente |
| Transición de máquina de estados | Todas las demás transiciones de la misma entidad (una flecha nueva puede romper las otras) |
| Migración / schema | Cada seeder, cada factory, cada report que consulta la tabla modificada |
| Auth / middleware | Cada ruta protegida en el rol afectado |

### Cómo aplicarlo

1. **Antes de que el Planner ejecute**, lista en voz alta los 3–5 vecinos más cercanos al cambio. Escríbelos en el prompt del Planner como URLs / features explícitas.
2. **El plan de test del Planner debe incluir una sección "Regresión — features cercanas"** con un test por vecino.
3. **El Generator debe producir esos tests**, aunque parezcan "obviamente fine" en el diff. El punto *no es* si parecen fine — el punto es *probar* que siguen funcionando tras el cambio.
4. **Si saltas el Ángulo 7, escribe por qué** en el report de QA. "Salté la regresión en features cercanas porque X" fuerza que la decisión sea consciente en vez de accidental.

> Regla práctica: si tu report de QA tiene cero tests de regresión contra features cercanas, no hiciste el Ángulo 7. Repite el pase de QA.

---

## Cómo escribir instrucciones para el Planner

El Planner es tan bueno como el prompt que le das. Prompts vagos producen planes de test vagos. La palanca más grande que tienes sobre la calidad del QA es **cómo instruyes al Planner**.

### El test Bad / Good

Lee estos dos prompts. El primero es el que escribe un QA junior. El segundo es el que escribe un QA senior. Escribe siempre el segundo.

**❌ Mal — pasivo, comprueba renderizado, un solo ángulo:**

```
Use the Playwright Planner to test the login page at /login.
Verify that the login form works.
```

**Por qué está mal:** "Funciona" no es un objetivo de test. El agente producirá un plan solo de happy path ("rellena email, rellena password, click login, ve dashboard") y enviarás a producción una feature que se rompe con input vacío.

**✅ Bien — activo, adversarial, multi-ángulo, con lista de ataques explícita:**

```
Use the Playwright Planner agent to explore the login flow at https://app.example.com/login.

Mindset: try to BREAK the login. Do not check that it renders — assume it renders.
Produce a test plan covering at least these adversarial angles:

1. Empty inputs — submit with both fields empty, with only email, with only password.
2. Invalid data — malformed emails (missing @, no domain, with spaces), wrong password format.
3. Boundary values — 1-char password, 256-char email, password with only spaces.
4. Special chars & injection — emoji in email, <script> in password, SQL injection patterns.
5. Double-click / rapid submit — double-click "Sign in", spam the button 5 times in 200ms.
6. Navigation edge cases — back button after successful login, refresh mid-submission, direct URL to /dashboard while logged out, open /login in two tabs and submit in both.
7. Regression in nearby features — after a successful login, verify the password reset link still navigates correctly, the registration link works, and logout from the dashboard returns to /login cleanly.
8. Auth & permission edges — login with a deactivated account, expired session, wrong role redirect.

For each test, specify:
- The exact action sequence
- The expected outcome (success or specific error)
- What to assert on (URL, visible text, console errors, network status)

Reference seed test: tests/seed.spec.ts
Save the plan to: specs/login-flow.md
```

> **Nota:** los prompts al Planner se escriben en inglés porque el agente Playwright procesa mejor instrucciones en inglés. Tú piensas en español, pero traduces el prompt antes de enviarlo. La calidad del plan depende de esto.

**Por qué está bien:** le dice al agente la *mentalidad*, nombra cada ángulo del checklist explícitamente, da ejemplos de ataque concretos, exige especificidad por test, y fija el path del output.

### Plantilla — copia esto para cada prompt al Planner

```
Use the Playwright Planner agent to explore [FEATURE NAME] at [URL].

Mindset: try to BREAK [FEATURE]. Do not check that it renders — assume it renders.
Produce a test plan covering these adversarial angles:

1. Empty inputs — [concrete attack examples for this feature]
2. Invalid data — [concrete attack examples]
3. Boundary values — [concrete attack examples]
4. Special chars & injection — [concrete attack examples]
5. Double-click / rapid submit — [which buttons / actions]
6. Navigation edge cases — [which navigation flows are risky here]
7. Regression in nearby features — [list 3–5 named neighbors and what to verify on each]
8. Auth & permission edges — [which roles, which auth states]

For each test, specify the action sequence, expected outcome, and what to assert on
(URL, visible text, console errors, network status).

Reference seed: tests/seed.spec.ts
Save plan to: specs/[feature-slug].md
```

### Reglas para escribir prompts al Planner

1. **Nombra siempre la mentalidad.** "Try to break X" debe aparecer en cada prompt. El agente no inventará el frame adversarial por su cuenta.
2. **Lista siempre los ángulos inline.** No digas "cubre edge cases" — pega los ángulos. Concreto > abstracto siempre.
3. **Da siempre ejemplos de ataque.** "Datos inválidos" es demasiado abstracto. "Email sin @, teléfono con letras, fecha `2026-13-45`" es accionable.
4. **Exige siempre especificidad por test.** Acción + resultado esperado + objetivo de la aserción. Sin manos sueltas.
5. **Nombra siempre los vecinos para el Ángulo 7.** No confíes en que el agente los encuentre. Tú conoces el codebase; el agente no.
6. **Fija siempre el path del output.** `specs/[feature-slug].md` — predecible, revisable, commiteable.
7. **Nunca le pidas al Planner "test the feature".** Pídele que *break* la feature.

### Reglas para revisar el output del Planner antes de mandarlo al Generator

Antes de dejar que el Generator convierta un plan en código, audita el plan contra los 8 ángulos:

- [ ] ¿El plan tiene al menos un test por ángulo aplicable?
- [ ] ¿Los happy paths son una *minoría* de los tests, no la mayoría?
- [ ] ¿El Ángulo 7 (regresión en features cercanas) tiene vecinos explícitos y nombrados?
- [ ] ¿Cada test especifica *qué assertear*, no solo "verificar que funciona"?
- [ ] ¿Hay al menos 3 tests que habrían cazado un bug real si la feature estuviera rota?

Si alguna respuesta es no, devuelve el plan con: *"Revisa el plan — el ángulo [N] falta o está flojo. Añade tests que [hueco específico]."*

---

## Workflow: Pase de QA bajo demanda

Este es el flujo que Claude ejecuta **cuando el usuario pide QA explícitamente** sobre una feature o cambio concreto. No lo ejecutes especulativamente.

### Paso 1 — Identifica qué cambió Y qué hay al lado

Antes de ejecutar tests, Claude debe entender tanto el alcance del cambio **como su radio de explosión hacia features cercanas** (Ángulo 7).

```
Preguntas que debes auto-contestarte:
- ¿Qué entidad/feature se creó o modificó?
- ¿Qué operaciones cambiaron? (create, read, update, delete, transición de estado)
- ¿Hay efectos secundarios? (emails, notificaciones, webhooks, tareas programadas)
- ¿Qué componentes / servicios / endpoints compartidos toca esto?
- VECINOS — nombra 3 a 5 features específicas adyacentes al cambio:
    * Otras páginas que usan el mismo componente
    * Páginas hermanas en el mismo módulo
    * La vista de lista que se alimenta del formulario modificado
    * Cualquier PDF, export o consumidor de API del modelo modificado
    * Cualquier otra transición de estado de la misma entidad
```

La lista de vecinos **no es opcional** — se convierte en el input del Ángulo 7 en el Paso 3a. Si no puedes nombrar al menos 3 vecinos, no entiendes el cambio lo suficiente como para hacerle QA. Re-lee el diff.

### Paso 2 — Ejecuta la suite de regresión existente

```bash
npx playwright test
```

Si algún test existente falla → la implementación rompió algo. Arregla antes de continuar.

### Paso 3 — Genera tests nuevos para la feature modificada

Usa el loop Planner → Generator → Healer:

**3a. Plan** — Pídele al Planner que explore la feature modificada usando la **plantilla adversarial** (ver "Cómo escribir instrucciones para el Planner" arriba). **No** uses lenguaje vago como "comprehensive test plan" — pega los 8 ángulos inline con ejemplos de ataque concretos y los vecinos nombrados del Paso 1.

Estructura mínima del prompt:

```
Use the Playwright Planner agent to explore [feature] at [URL].

Mindset: try to BREAK [feature]. Do not check that it renders — assume it renders.
Produce a test plan covering these adversarial angles:

1. Empty inputs — [examples]
2. Invalid data — [examples]
3. Boundary values — [examples]
4. Special chars & injection — [examples]
5. Double-click / rapid submit — [which buttons]
6. Navigation edge cases — [back, refresh, multi-tab, deep-link]
7. Regression in nearby features — [the 3–5 neighbors named in Step 1]
8. Auth & permission edges — [which roles, which states]

For each test, specify the action sequence, expected outcome, and assertion target.

Reference seed: tests/seed.spec.ts
Save to: specs/[feature-name].md
```

Antes de mandar el plan al Generator, audítalo contra el checklist de "Reglas para revisar el output del Planner" arriba. Rechaza y revisa si algún ángulo falta o está flojo.

**3b. Generate** — Convierte el plan en tests ejecutables:
```
Use the Playwright Generator agent to create tests from specs/[feature-name].md
Use tests/seed.spec.ts as the seed test.
Save to: tests/[feature-name]/
```

**3c. Run & Heal**:
```bash
# Ejecuta los tests nuevos
npx playwright test tests/[feature-name]/

# Si hay fallos, invoca al Healer
# "Use the Playwright Healer agent to fix failing tests in tests/[feature-name]/"
```

### Paso 4 — Aplica la metodología de QA

Lee `references/qa-methodology.md` y aplica las categorías de test relevantes según lo que cambió:

| Qué cambió | Aplica estas categorías |
|-------------|----------------------|
| Feature CRUD nueva | Funcional (Create/Read/Update/Delete), Edge Cases, Consistencia |
| Transiciones de estado | Testing de máquina de estados, Testing de automatizaciones |
| Endpoint de API | Input malformado, Fallos de red, Testing de API |
| Feature en tiempo real | Testing multi-usuario, Sincronización, Pérdida de datos |
| Cualquier cambio user-facing | Responsive, Doble Submit, Botón atrás, Expiración de sesión |
| Integraciones (email, SMS) | Testing de automatizaciones, Timing, Sin duplicados |

### Paso 5 — Reporta los resultados

Produce el report estandarizado (ver `references/qa-methodology.md` para el formato):
- Resultados individuales por test con estado y severidad
- Top 5 bugs críticos
- Mejoras propuestas
- Evaluación de cobertura
- Lista de tests de regresión nuevos añadidos

### Paso 6 — Commitea los tests a la suite de regresión

Los tests nuevos que pasen pasan a formar parte de la suite permanente, asegurando que la feature siga testeada en cada cambio futuro.

---

## Workflow: Setup inicial

Si el proyecto todavía no tiene Playwright:

1. Lee `references/setup.md`
2. Instala Playwright + browsers
3. Inicializa los agentes: `npx playwright init-agents --loop=claude`
4. Crea `playwright.config.ts` adaptado al proyecto
5. Crea el setup de auth si la app requiere login
6. Crea `tests/seed.spec.ts`
7. Crea el directorio `specs/`
8. Ejecuta la primera exploración del Planner sobre la app

---

## Principios Clave

1. **No estás testeando para confirmar — estás testeando para romper.** Este es el mantra central. Si un test no cazaría un bug real, bórralo.
2. **Dos modos: SUGGEST (auto, gratis) y EXECUTE (manual, caro)** — Los cambios behaviorales auto-cargan la skill, que añade una línea de sugerencia de QA. El workflow completo solo corre cuando el usuario dice sí explícitamente. Ver las Reglas de Activación arriba.
3. **Los 8 ángulos son el suelo, no el techo** — Un plan de test que omite un ángulo aplicable está incompleto por definición.
4. **El Ángulo 7 es no negociable** — Cada pase de QA incluye tests de regresión contra features cercanas nombradas. Saltarlo es una decisión documentada, no un default.
5. **Sin asunciones** — Valida todo explícitamente. "Probablemente sigue funcionando" es la frase que va justo antes de un incidente en producción.
6. **Prioriza bugs críticos** — Acciones duplicadas, pérdida de datos, estados inconsistentes, flujos de dinero.
7. **Tests deterministas** — Cada test reproducible, sin dependencia de estado externo.
8. **Aislamiento** — Contexto de browser fresco por test (default de Playwright).
9. **Locators semánticos** — `getByRole`, `getByLabel`, `getByTestId`. Nunca CSS frágil.
10. **Sin waits artificiales** — Confía en el auto-wait + retry assertions de Playwright.
11. **Los tests son permanentes** — Una vez escritos, se quedan en la suite de regresión para siempre.
12. **Patrones genéricos** — Aplica la misma metodología independientemente del dominio de la app.

---

## Estructura del proyecto

```
project/
├── .github/                    # Definiciones de agentes (auto-generadas)
├── specs/                      # Planes de test (Markdown, por feature)
│   ├── auth-flow.md
│   ├── entity-crud.md
│   ├── dashboard.md
│   └── integrations.md
├── tests/
│   ├── fixtures.ts             # Fixtures custom (auth, setup de datos)
│   ├── seed.spec.ts            # Seed test para bootstrap de los agentes
│   ├── auth.setup.ts           # Setup de autenticación
│   ├── auth/                   # Tests por área de feature
│   ├── entity-crud/
│   ├── dashboard/
│   ├── integrations/
│   ├── edge-cases/
│   └── helpers/
│       └── test-data.ts        # Generadores de datos de test
├── playwright.config.ts
└── test-results/               # Traces, screenshots, vídeos
```

---

## Troubleshooting

| Problema | Solución |
|---------|----------|
| Los agentes se atascan en el login | Añade auth a `seed.spec.ts` o usa `storageState` en fixtures |
| Tests flaky | Revisa el timing → usa `expect().toBeVisible()` no `waitForTimeout` |
| Accessibility trees enormes | Guarda los snapshots localmente, no los inyectes en cada prompt |
| Context window se desborda | Usa "Code Mode" — el agente escribe código que llama a las herramientas |
| Crashes del lado servidor | Añade aserciones de health check antes de los pasos de interacción |
| Tests pasan en local, fallan en CI | Revisa env vars, base URL y dependencias del browser |

---

## Anti-patrones

- **Prompts vagos al Planner** — "Test the feature" / "verify it works" / "comprehensive test plan". Pega siempre los 8 ángulos inline con ejemplos de ataque concretos. Ver "Cómo escribir instrucciones para el Planner".
- **Comprobar renderizado en vez de break-testing** — Un test que solo verifica que la página carga es un smoke test, no QA. Si tus tests no intentan romper nada, no hiciste QA.
- **Saltarse el Ángulo 7** — Entregar QA sin tests de regresión contra features cercanas nombradas. El bug que se te escapó casi siempre está al lado del cambio, no en él.
- **Planes de test mayoritariamente happy path** — Si la mayoría de tus tests rellenan datos válidos y hacen click en submit, estás testeando la demo, no el producto. Los tests adversariales deberían ser más numerosos que los happy paths.
- **Confiar ciegamente en el Planner** — Audita siempre el plan contra los 8 ángulos antes de mandarlo al Generator. El agente producirá un plan flojo si le dejas.
- **`page.waitForTimeout()`** — Nunca. Usa auto-wait + aserciones.
- **Selectores CSS** — Usa locators semánticos (`getByRole`, `getByTestId`).
- **Estado compartido entre tests** — Cada test recibe un contexto de browser fresco.
- **Testear todo con agentes** — Mantén los tests estables como specs estáticos; usa los agentes para features nuevas y áreas flaky.
- **Ignorar los traces** — Revisa siempre el trace viewer en los fallos.
- **Ejecutar el modo EXECUTE sin consentimiento explícito** — Sugerir es gratis, ejecutar es caro. Nunca ejecutes el workflow completo Planner → Generator → Healer salvo que el usuario lo diga explícitamente. Luces verdes aceptables: "haz QA", "sí", "dale", "yes", "go", o cualquier frase de la lista de disparadores de EXECUTE.
- **Leer archivos de referencia en modo SUGGEST** — `qa-methodology.md`, `setup.md`, `spec-template.md`, `example-qa-report.md` son recursos del modo EXECUTE. Cargarlos en modo SUGGEST gasta contexto para nada.
- **Añadir la línea de sugerencia a cambios no behaviorales** — Docs puras, comentarios, estilos, ajustes de config, conversación. Sin línea de sugerencia. Solo añádela cuando hay un cambio behavioral real que merezca ser testeado.
- **Repetir la sugerencia después de que el usuario ya rechazó** — Pregunta una vez por conversación. Respeta la respuesta.
- **No commitear los tests** — Cada test que pasa va a la suite de regresión.
