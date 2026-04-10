# Referencia de Metodología de QA — Testing E2E Exhaustivo (Genérico)

## Filosofía de Testing

> **"No estás testeando para confirmar que las features funcionan — estás testeando para romperlas."**
> **"Si no lo has validado intentando romperlo, no funciona."**

Actúa como un QA Engineer senior con mentalidad de "romper el sistema". No asumas que nada funciona hasta haberlo verificado. Prioriza bugs críticos: pérdida de datos, estados inconsistentes, acciones duplicadas, y fallos de seguridad.

---

## Cómo este documento se relaciona con los 8 Ángulos Adversariales

`SKILL.md` define **8 ángulos adversariales** (los *vectores de ataque*). Este documento de metodología define **7 categorías funcionales de test** (los *dominios a atacar*). Son ortogonales — los dos hay que aplicarlos:

| | **Categorías Funcionales (este doc)** | **Ángulos Adversariales (SKILL.md)** |
|---|---|---|
| **Pregunta que responden** | *¿Qué dominio de comportamiento necesito testear?* | *¿Cómo ataco cada dominio?* |
| **Ejemplos** | CRUD, Máquinas de Estado, Automation, Real-Time, Edge Cases, Consistencia, Responsive | Inputs vacíos, datos inválidos, valores límite, inyección, doble click, navegación, regresión en features cercanas, auth |
| **Granularidad** | Una por área funcional de la app | Una por vector de ataque aplicado a *cada* área funcional |
| **Por dónde empezar** | Mapea la app: ¿qué categorías aplican? | Para cada categoría aplicable, ejecuta los 8 ángulos |

### Cómo usar ambos juntos

1. **Mapea la feature a categorías funcionales.** ¿Es CRUD? ¿Tiene transiciones de estado? ¿Dispara automatizaciones? Cada Sí abre una categoría de este documento.
2. **Para cada categoría abierta, ejecuta los 8 ángulos de `SKILL.md`.** Un formulario CRUD debe testearse con inputs vacíos, datos inválidos, valores límite, inyección, doble click, edges de navegación, regresión en features cercanas y edges de auth. Lo mismo para una transición de estado. Lo mismo para una automatización.
3. **Usa este documento para los patrones y ejemplos de código.** Cuando necesites escribir un test de "acceso concurrente" o un test de "tarea programada", los patrones canónicos están abajo. El *framing adversarial* de esos patrones viene de los ángulos.

### Mapeo rápido: qué ángulos aplican a qué categorías

| Categoría Funcional ↓  /  Ángulo → | 1 Vacío | 2 Inválido | 3 Límite | 4 Inyección | 5 Doble-click | 6 Navegación | 7 Cercano | 8 Auth |
|---|---|---|---|---|---|---|---|---|
| 1. CRUD                    | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 2. Máquina de Estados      | ⚠️ | ✅ | — | — | ✅ | ✅ | ✅ | ✅ |
| 3. Automation/Integración  | — | ✅ | ✅ | ✅ | ✅ | — | ✅ | ✅ |
| 4. Real-Time/Multi-Usuario | — | ✅ | ✅ | — | ✅ | ✅ | ✅ | ✅ |
| 5. Edge Cases              | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 6. Consistencia            | — | — | — | — | ✅ | ✅ | ✅ | — |
| 7. Responsive/A11y         | — | — | ✅ | — | — | ✅ | — | — |

✅ = obligatorio  ⚠️ = situacional  — = normalmente no aplicable

> **TL;DR:** los ángulos te dicen *qué romper*. Las categorías te dicen *qué tipos de cosas existen que puedan romperse*. Aplica los dos, siempre.

---

## Cómo Aplicar Esta Metodología

Esto es un **framework genérico**. Cuando testee una app específica, Claude debe:

1. **Identificar el dominio de la app** — ¿Qué hace? (e-commerce, SaaS, booking, CMS, dashboard...)
2. **Mapear las entidades** — ¿Cuáles son los objetos núcleo? (users, orders, posts, tickets, messages...)
3. **Mapear las operaciones** — ¿Qué puede hacerse con cada entidad? (CRUD, transiciones de estado, automatizaciones...)
4. **Mapear las integraciones** — ¿Qué sistemas externos están involucrados? (email, SMS, pagos, APIs, webhooks...)
5. **Aplicar cada categoría de test de abajo** a las entidades y operaciones mapeadas

---

## Categorías de Test

### 1. TESTING FUNCIONAL — CRUD y Flujos Núcleo

Para **cada entidad** en la aplicación, testea todas las operaciones:

#### Create — Happy Path
```typescript
test('create [entity] with valid data', async ({ page }) => {
  // Navega al formulario de creación
  // Rellena todos los campos obligatorios con datos válidos
  // Submit
  // Assert: mensaje de éxito, la entidad aparece en la lista, datos correctos mostrados
});
```

#### Create — Inválido y Edge Cases
Testea cada uno de estos escenarios por formulario de creación:

| Escenario | Qué testear | Esperado |
|----------|-------------|----------|
| Campos obligatorios vacíos | Submit sin nada relleno | Errores de validación en cada campo |
| Formato inválido | Email, teléfono, URL, fechas mal formados | Mensajes de error específicos por campo |
| Valores límite | Min/max lengths, 0, -1, 999999, string vacío | Rechazo o aceptación de forma controlada |
| Strings extremos | 10K+ chars, solo espacios, solo chars especiales | Truncado o error de validación |
| Inyección | `<script>alert(1)</script>`, `'; DROP TABLE--` | Sanitizado, sin ejecutar |
| Unicode y emoji | Nombres con ñ, ü, 日本語, 🎉 | Almacenados y mostrados correctamente |
| Duplicado | Crear la misma entidad dos veces | Error o deduplicar, nunca duplicado en silencio |
| Fechas pasadas | Si hay campo de fecha, prueba fechas en el pasado | Rechazar si no está permitido |
| Extremos futuros | Año 2099, año 1900 | Rechazar o manejar con elegancia |

#### Read / List
```typescript
test('list [entities] with pagination', async ({ page }) => {
  // Navega a la vista de lista
  // Assert: items visibles, count coincide
  // Testea paginación: next, previous, jump to page
  // Testea sorting: por cada columna ordenable
  // Testea filtering: por cada filtro disponible
  // Testea search: match exacto, match parcial, sin resultados
});
```

#### Update
| Escenario | Esperado |
|----------|----------|
| Cambio válido de un solo campo | Actualizado, confirmación mostrada |
| Cambio válido multi-campo | Todos los campos actualizados atómicamente |
| Cambio a un valor en conflicto | Error (ej: email duplicado, slot ocupado) |
| Update de entidad borrada/archivada | Bloqueado o error |
| Update de entidad en estado terminal | Bloqueado (ej: no se puede editar un order "completed") |
| Update concurrente por 2 usuarios | Last-write-wins o resolución de conflictos |

#### Delete / Archive
| Escenario | Esperado |
|----------|----------|
| Delete de entidad activa | Quitada de la lista, prompt de confirmación |
| Delete de entidad ya borrada | Idempotente o 404 |
| Delete de entidad con dependencias | Cascade o bloqueo con explicación |
| Doble click en botón delete | Solo una eliminación, no error |
| Soft delete vs hard delete | Verifica cuál aplica, comprueba opción de recuperación |

---

### 2. TESTING DE MÁQUINA DE ESTADOS

La mayoría de entidades tienen estados (pending → confirmed → completed → cancelled, etc). Para cada transición de estado:

```
Para cada [estado_actual] → [estado_objetivo]:
  ✅ Test: la transición válida funciona
  ❌ Test: la transición inválida está bloqueada
  🔄 Test: los efectos secundarios se disparan correctamente (emails, webhooks, updates de UI)
  ⏱️ Test: los timestamps se actualizan correctamente
```

Patrón de ejemplo:
```typescript
test('transition [entity] from [state_A] to [state_B]', async ({ page }) => {
  // Setup: crea entidad en state_A
  // Action: dispara la transición (button click, API call, etc)
  // Assert: el estado ahora es state_B
  // Assert: la UI refleja el nuevo estado
  // Assert: efectos secundarios disparados (notificaciones, logs, etc)
  // Assert: no se puede volver a state_A (si es irreversible)
});
```

**Comprobaciones críticas:**
- Dibuja el diagrama completo de estados de cada entidad
- Cada flecha es un test case (transición válida)
- Cada flecha que falta es un test case (la transición inválida debe estar bloqueada)
- Cada estado debe tener al menos un camino de salida (sin estados muertos)

---

### 3. TESTING DE AUTOMATION E INTEGRACIÓN

Para cada efecto secundario automatizado en el sistema (emails, SMS, webhooks, notificaciones, tareas programadas):

| Comprobación | Cómo |
|-------|-----|
| Se dispara en el evento correcto | Crea el evento → verifica que la automatización se disparó |
| NO se dispara en el evento equivocado | Crea un evento similar pero distinto → verifica que NO hay automatización |
| El contenido es correcto | Verifica que el payload/body contiene datos actuales, no stale |
| Sin duplicados | Dispara el evento una vez → verifica exactamente 1 automatización, no 2+ |
| Retry en fallo | Simula fallo de integración → verifica el mecanismo de retry |
| El timing es correcto | Si es delayed (ej: "send 24h before"), verifica el timing exacto |
| Cancelación detiene automatizaciones pendientes | Cancela la entidad → verifica que las automatizaciones programadas están canceladas |
| Update se refleja en automatizaciones pendientes | Modifica la entidad → verifica que las automatizaciones pendientes usan los datos NUEVOS |

#### Patrón: Testing de Comunicaciones Automatizadas
```typescript
test('automation fires on [event] with correct content', async ({ page, request }) => {
  // 1. Realiza la acción que dispara la automatización (vía UI o API)
  // 2. Espera a que la automatización se procese
  // 3. Consulta el endpoint de test / mailbox / log para verificar:
  //    - Exactamente 1 mensaje enviado (no 0, no 2+)
  //    - El destinatario es correcto
  //    - El contenido refleja los datos actuales de la entidad
  //    - El template se renderizó (sin {{variables}} en crudo)
});
```

#### Patrón: Testing de Tareas Programadas / Con Tiempo
```typescript
test('scheduled task fires at correct time', async ({ page }) => {
  // Usa la Playwright Clock API para controlar el tiempo
  await page.clock.install({ time: new Date('2026-04-14T10:00:00') });
  
  // Crea entidad que disparará la automatización con tiempo
  // Avanza rápido hasta justo antes del trigger esperado
  await page.clock.fastForward('23:59:00');
  // Assert: la automatización NO se ha disparado todavía
  
  // Avanza rápido pasando el punto del trigger
  await page.clock.fastForward('00:02:00');
  // Assert: la automatización SÍ se ha disparado
});
```

---

### 4. TESTING REAL-TIME Y MULTI-USUARIO

Para apps con features en tiempo real (chat, live updates, edición colaborativa, dashboards):

```typescript
test('real-time sync between two users', async ({ browser }) => {
  const ctx1 = await browser.newContext();
  const ctx2 = await browser.newContext();
  const page1 = await ctx1.newPage();
  const page2 = await ctx2.newPage();
  
  // El Usuario 1 realiza la acción
  // Assert: el Usuario 2 ve el cambio sin refrescar
  
  await ctx1.close();
  await ctx2.close();
});
```

| Comprobación | Qué verificar |
|-------|---------------|
| Sincronización | Una acción del usuario A es visible al usuario B en tiempo real |
| Ordenamiento | Los mensajes/eventos aparecen en orden cronológico correcto |
| Sin pérdida de datos | Envía N items rápidamente → llegan los N |
| Reconexión | Simula desconexión → reconecta → no se perdió ningún dato |
| Conflicto | Ambos usuarios editan lo mismo → resuelto, no corrupto |

---

### 5. EDGE CASES Y BREAK TESTING

Aplica estos a **cada flujo crítico** de la app:

#### Concurrencia
```typescript
test('concurrent access does not corrupt data', async ({ browser }) => {
  const ctx1 = await browser.newContext();
  const ctx2 = await browser.newContext();
  const page1 = await ctx1.newPage();
  const page2 = await ctx2.newPage();
  
  // Ambos usuarios intentan la misma acción simultáneamente
  await Promise.all([
    performCriticalAction(page1),
    performCriticalAction(page2),
  ]);
  
  // Assert: uno tiene éxito, el otro recibe error de conflicto
  // O: ambos tienen éxito si la acción soporta concurrencia
  // NUNCA: corrupción de datos, registros duplicados o fallo silencioso
  
  await ctx1.close();
  await ctx2.close();
});
```

#### Fallos de Red
```typescript
test('app handles network failure gracefully', async ({ page }) => {
  // Navega y rellena el formulario con datos
  
  // Bloquea las llamadas a la API
  await page.route('**/api/**', route => route.abort());
  
  // Intenta la acción
  // Assert: error user-friendly mostrado (no pantalla en blanco ni error en crudo)
  // Assert: los datos del formulario se conservan (el usuario no pierde su input)
  // Assert: la app sigue siendo usable después del error
});
```

#### Doble Submit
```typescript
test('double submit does not create duplicate', async ({ page }) => {
  // Rellena formulario
  // Doble click en el botón submit
  await page.getByRole('button', { name: /submit|save|create|confirm/i }).dblclick();
  
  // Assert: solo 1 entidad creada, no 2
});
```

#### Expiración de Sesión
```typescript
test('expired session redirects to login', async ({ page }) => {
  // Limpia las cookies/storage de auth
  await page.context().clearCookies();
  
  // Intenta una acción protegida
  // Assert: redirecciona al login, no error 500
  // Assert: tras re-login, el usuario vuelve a la página deseada
});
```

#### Input Malformado de API
```typescript
test('API rejects malformed payload', async ({ request }) => {
  const cases = [
    { data: null },
    { data: {} },
    { data: { id: 'not-a-number' } },
    { data: { required_field: null } },
    { data: { number_field: -1 } },
    { data: { string_field: 'x'.repeat(100000) } },
  ];
  
  for (const { data } of cases) {
    const resp = await request.post('/api/endpoint', { data });
    expect(resp.status()).toBeGreaterThanOrEqual(400);
    expect(resp.status()).toBeLessThan(500); // Error de cliente, no crash de servidor
  }
});
```

#### Edge Cases del Browser
- **Botón atrás** después de submit de formulario → no debería resubmitir
- **Refresh** durante una operación → no debería corromper el estado
- **Múltiples pestañas** con la misma sesión → deberían mantenerse en sync
- **Conexión lenta** (throttle de red) → estados de loading, sin timeouts
- **Zoom / escalado de fuente** → la UI sigue siendo usable

---

### 6. TESTING DE CONSISTENCIA

Verifica que los datos se mantienen sincronizados en todas las capas:

| Capa A | Capa B | Comprobación |
|---------|---------|-------|
| Estado de DB | Display de UI | El estado de la entidad coincide con lo mostrado |
| Estado de entidad | Contenido de comunicación | Si status = "confirmed", el email dice "confirmed" |
| Timestamp de acción | Timestamp de log | Los eventos están loggeados en el momento correcto |
| Count de lista | Items reales | "Showing 15 results" → exactamente 15 items renderizados |
| Sumas/totales | Valores individuales | Total del dashboard = suma de las entradas individuales |
| Datos cacheados | Datos fuente | Después de un update, las vistas cacheadas reflejan los nuevos datos |

---

### 7. RESPONSIVE Y ACCESIBILIDAD

```typescript
// Testea en viewport mobile
test.use({ viewport: { width: 375, height: 667 } });

test('critical flow works on mobile', async ({ page }) => {
  // Navega por el flujo principal en mobile
  // Assert: todos los botones tap-eables, formularios rellenables, sin scroll horizontal
});
```

Comprobaciones rápidas de a11y:
- Cada elemento interactivo alcanzable por teclado (navegación con Tab)
- Cada campo de formulario tiene un label
- Los mensajes de error están asociados con su campo
- Contraste de color suficiente
- El screen reader puede navegar por los flujos críticos

---

## Estrategia de Ejecución de Tests

### Orden de Prioridad (qué testear primero)

1. **Camino crítico** — La cosa principal que hacen los usuarios (signup, compra, crear entidad núcleo)
2. **Flujos de dinero** — Cualquier cosa que involucre pagos, billing, créditos
3. **Integridad de datos** — Operaciones que crean/modifican/borran datos permanentes
4. **Automatizaciones** — Efectos secundarios que no se pueden deshacer fácilmente (emails, notificaciones)
5. **Auth y seguridad** — Login, permisos, gestión de sesiones
6. **Edge cases** — Concurrencia, fallos, límites
7. **UX y polish** — Responsive, estados de loading, mensajes de error

### Cuándo usar Playwright Agents vs Tests Manuales

| Usa Agentes (Planner → Generator) | Escribe Tests Manuales |
|----------------------------------|--------------------|
| Exploración de feature nueva | Validación de lógica de negocio compleja |
| Cobertura amplia | Regresión específica para bugs conocidos |
| Testing de flujos UI | Testing a nivel de API |
| Después de refactors mayores | Aserciones sensibles a performance |

---

## Formato de Output de Test

Para cada test ejecutado, produce:

```markdown
### TEST: [nombre descriptivo]
- **Acción:** [qué se hizo]
- **Esperado:** [qué debería pasar]
- **Real:** [qué pasó realmente]
- **Estado:** ✅ OK / ❌ ERROR
- **Severidad:** Low / Medium / High / Critical
- **Detalles:** [info técnica si hay error — stack trace, screenshot, network log]
- **Causa probable:** [hipótesis]
- **Fix sugerido:** [cómo podría resolverse]
```

## Clasificación de Severidad

| Severidad | Criterio | Ejemplos |
|----------|----------|---------|
| **Critical** | Pérdida de datos, brecha de seguridad, acciones duplicadas, dinero afectado | Doble cargo, fuga de datos, registros corruptos |
| **High** | Flujo núcleo roto, usuario bloqueado | No se puede crear/editar/borrar entidad principal, auth roto |
| **Medium** | Funcionalidad degradada, UX confusa | Datos equivocados en notificación, error no descriptivo |
| **Low** | Cosmético, impacto menor | Typo, problema de alineación, hover state que falta |

---

## Resumen de Sesión

Al final de cada sesión de QA, produce:

```markdown
## 📊 Resumen de QA

### Tests ejecutados: X
- ✅ Pasados: X
- ❌ Fallidos: X  
- ⏭️ Saltados: X

### 🔴 Top 5 Bugs Críticos
1. [Bug más crítico con descripción y pasos de reproducción]
2. ...

### 💡 Mejoras Propuestas
- [Mejoras de arquitectura, lógica o UX detectadas]

### 📋 Cobertura
- [Features testeadas vs pendientes]
- [Áreas sin cobertura identificadas]

### 🔄 Tests de Regresión Nuevos Añadidos
- [Lista de archivos de test nuevos commiteados a la suite]
```

---

## Referencia rápida de la API de Playwright

### Locators (SIEMPRE semánticos)
```typescript
page.getByRole('button', { name: 'Submit' })
page.getByLabel('Email')
page.getByPlaceholder('Search...')
page.getByText('Welcome')
page.getByTestId('submit-btn')     // fallback cuando no hay opción semántica
```

### Aserciones (auto-retry incorporado)
```typescript
await expect(page).toHaveTitle(/Dashboard/);
await expect(page).toHaveURL('/dashboard');
await expect(locator).toBeVisible();
await expect(locator).toBeHidden();
await expect(locator).toHaveText('Expected');
await expect(locator).toHaveValue('input-value');
await expect(locator).toBeEnabled();
await expect(locator).toBeDisabled();
await expect(locator).toHaveCount(5);
await expect(locator).toHaveAttribute('href', '/path');
await expect(locator).toContainText('partial');
```

### Interceptación de Red
```typescript
// Mock de respuesta de API
await page.route('**/api/endpoint', route =>
  route.fulfill({ status: 200, body: JSON.stringify({ id: 1 }) })
);

// Bloquea requests (simula fallo)
await page.route('**/api/endpoint', route => route.abort());

// Espera una llamada de API específica
const response = await page.waitForResponse('**/api/endpoint');
expect(response.status()).toBe(200);

// Intercepta y modifica la respuesta
await page.route('**/api/endpoint', async route => {
  const response = await route.fetch();
  const json = await response.json();
  json.modified = true;
  await route.fulfill({ response, json });
});
```

### Clock API (para tests dependientes de timing)
```typescript
await page.clock.install({ time: new Date('2026-04-14T20:00:00') });
await page.clock.fastForward('01:00:00');
await page.clock.setFixedTime(new Date('2026-04-15T20:00:00'));
await page.clock.resume(); // vuelve al tiempo real
```

### Testing de API (sin browser)
```typescript
test('API: create entity', async ({ request }) => {
  const response = await request.post('/api/entities', {
    data: { name: 'Test', type: 'example' },
    headers: { Authorization: 'Bearer token' },
  });
  expect(response.ok()).toBeTruthy();
  const data = await response.json();
  expect(data.id).toBeDefined();
});
```

### Multi-context (simulación multi-usuario)
```typescript
test('two users interact', async ({ browser }) => {
  const ctx1 = await browser.newContext();
  const ctx2 = await browser.newContext();
  const page1 = await ctx1.newPage();
  const page2 = await ctx2.newPage();
  
  // ... interacción del test ...
  
  await ctx1.close();
  await ctx2.close();
});
```
