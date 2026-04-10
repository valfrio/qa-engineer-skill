# Ejemplo de Report de QA — Output de Referencia

Este es un **ejemplo completamente desarrollado** del report de QA que Claude debe producir al final de cada sesión de QA. Úsalo como plantilla estructural para reports reales.

Un report de QA tiene tres trabajos:

1. **Decirle al humano qué está roto** en orden de prioridad, con pasos de reproducción.
2. **Decirle al Claude futuro qué se testeó** para que la siguiente sesión de QA no rehaga el trabajo.
3. **Decirle al equipo qué se commiteó** a la suite de regresión como cobertura permanente.

Si un report real omite cualquiera de estos, está incompleto.

El ejemplo de abajo va emparejado con [`spec-template.md`](spec-template.md) — misma feature (e-commerce "Place Order"), así que puedes leer ambos lado a lado.

---

# Report de QA — Feature Place Order

**Tester:** Claude (skill qa-engineer v2.2.1)
**Feature:** Place Order (`/checkout`)
**Spec:** `specs/place-order.md`
**Archivos de test:** `tests/place-order/*.spec.ts` (24 tests)
**Entorno:** `https://staging.example.com`
**Artefactos de Playwright:** `test-results/` (vídeos, traces, screenshots)
**Duración del run:** 4m 12s

---

## TL;DR

**Estado: 🔴 BLOCKING — no desplegar a producción.**

De 24 tests adversariales, **18 pasaron, 4 fallaron, 2 saltados**. Dos fallos son **críticos** (integridad de datos / dinero), uno es **alto** (regresión en feature cercana), y uno es **medio** (UX). Los dos saltados están bloqueados por una fixture de test que falta.

Recomendación: **arreglar los 2 bugs críticos antes de mergear**, después re-ejecutar la suite.

---

## Resultados de tests

| # | Test | Ángulo | Estado | Severidad | Notas |
|---|------|-------|--------|----------|-------|
| 0.1 | Pedido con un item | 0 — Happy | ✅ | — | — |
| 0.2 | Pedido con varios items + tax mixto | 0 — Happy | ✅ | — | — |
| 0.3 | Aplicar código de descuento válido | 0 — Happy | ✅ | — | — |
| 1.1 | Submit checkout con carrito vacío | 1 — Vacío | ✅ | — | La validación funciona |
| 1.2 | Submit sin dirección de envío | 1 — Vacío | ✅ | — | — |
| 1.3 | Submit sin método de pago | 1 — Vacío | ✅ | — | — |
| 2.1 | Cantidad negativa | 2 — Inválido | ❌ | **Crítico** | Ver BUG-001 |
| 2.2 | Número de tarjeta inválido | 2 — Inválido | ✅ | — | El check de Luhn funciona |
| 2.3 | Email malformado | 2 — Inválido | ✅ | — | Los 4 casos rechazados |
| 3.1 | Cantidad = 0 | 3 — Límite | ⚠️ | Medio | Ver BUG-003 |
| 3.2 | Cantidad máxima (999999) | 3 — Límite | ✅ | — | Total mostrado correctamente, sin overflow |
| 3.3 | Dirección con 10K chars | 3 — Límite | ✅ | — | Truncado a 200 chars (silencioso — ver mejora #2) |
| 4.1 | Nombre de cliente con HTML/script | 4 — Inyección | ✅ | — | Escapado correctamente en detalle y email |
| 4.2 | Dirección con unicode + emoji | 4 — Inyección | ✅ | — | Los 4 scripts renderizan correctamente |
| 5.1 | Doble click en "Place Order" | 5 — Doble-click | ❌ | **Crítico** | Ver BUG-002 |
| 5.2 | Retry rápido en fallo de pago | 5 — Doble-click | ✅ | — | Idempotente — solo reintenta una vez |
| 6.1 | Botón atrás después del pedido | 6 — Nav | ✅ | — | Sin resubmit |
| 6.2 | Refresh durante el pago | 6 — Nav | ✅ | — | Sin doble cargo |
| 6.3 | Checkout simultáneo en dos pestañas | 6 — Nav | ✅ | — | Ambas tienen éxito, inventario correcto |
| 7.1 | Filtros/paginación de la lista de pedidos | 7 — Regresión | ✅ | — | — |
| 7.2 | Detalle del pedido lee snapshot, no live | 7 — Regresión | ❌ | **Alto** | Ver BUG-004 |
| 7.3 | Email de confirmación se dispara una vez | 7 — Regresión | ⏭️ | — | Saltado — depende del fix de 7.2 |
| 7.4 | Inventario decrementado exactamente una vez | 7 — Regresión | ✅ | — | Stock coincide con el delta esperado |
| 8.1 | Checkout guest vs logueado | 8 — Auth | ⏭️ | — | Saltado — falta fixture de test (sin ruta de test guest) |

**Totales:** 18 ✅ / 4 ❌ / 2 ⏭️

---

## 🔴 Top bugs críticos

### BUG-001 — La cantidad negativa crea un pedido con total negativo

- **Severidad:** 🔴 Crítica (corrupción de flujo de dinero)
- **Test:** 2.1 — Cantidad negativa
- **Ángulo:** 2 — Datos inválidos
- **Trace:** `test-results/place-order-Negative-quantity-chromium/trace.zip`
- **Pasos para reproducir:**
  1. Añade un item al carrito.
  2. En el carrito, modifica la cantidad de la línea a `-5` (vía DOM o API).
  3. Procede al checkout y submit.
- **Esperado:** El servidor rechaza con 422 O el campo se clamp a 1. El total no puede ser negativo.
- **Real:** Pedido creado con total `-$60.50`. Almacenado en DB. Visible en la lista de pedidos con total negativo. **Sin validación de cantidad negativa ni en cliente ni en servidor.**
- **Causa probable:** Falta la regla de validación `min: 1` en el campo de cantidad de la línea, tanto en la capa de UI como en la de API.
- **Fix sugerido:** Añadir validación server-side rechazando `quantity <= 0` en el endpoint de creación de pedidos. Añadir constraint `min="1"` en el input client-side del campo de cantidad. Las dos capas — el servidor es la frontera de seguridad, el cliente es el UX.
- **Riesgo si se envía:** El cliente (o atacante) hace pedidos con totales negativos, efectivamente acreditando su cuenta. Dinero perdido. Pesadilla de reconciliación.

### BUG-002 — El doble click en "Place Order" crea pedido duplicado y dobla el cargo de tarjeta

- **Severidad:** 🔴 Crítica (integridad de datos + cargo duplicado)
- **Test:** 5.1 — Doble click submit
- **Ángulo:** 5 — Doble click / submit rápido
- **Trace:** `test-results/place-order-Double-click-chromium/trace.zip`
- **Pasos para reproducir:**
  1. Rellena el formulario de checkout con datos válidos.
  2. Usa doble click (o doble click humano) en "Place Order".
- **Esperado:** Exactamente 1 pedido creado. Tarjeta cargada exactamente 1 vez.
- **Real:** **2 pedidos creados**, ambos para el mismo cliente con los mismos items. El log del procesador de pago muestra **2 cargos** de $54.00 cada uno.
- **Causa probable:** El botón submit no se desactiva tras el primer click. No se manda idempotency key con la request, así que el servidor no puede deduplicar.
- **Fix sugerido (mínimo):** Desactivar el botón submit inmediatamente al click y reactivarlo solo en respuesta o error. Mostrar un loading spinner.
- **Fix sugerido (correcto):** Generar una idempotency key en el cliente cuando el checkout monta. Mandarla con la request del pedido. El servidor la guarda en una ventana de 10 segundos — requests duplicadas con la misma key devuelven el pedido original, no uno nuevo. Patrón compatible con el estilo de idempotency keys de Stripe.
- **Riesgo si se envía:** Los clientes ven cargos duplicados en sus extractos, disputan cargos, abren chargebacks. Daño a la reputación y comisiones del procesador.

### BUG-004 — El detalle del pedido lee de los datos vivos del cliente, no del snapshot

- **Severidad:** 🟠 Alta (la garantía del snapshot está rota — precisión histórica)
- **Test:** 7.2 — El detalle del pedido lee snapshot
- **Ángulo:** 7 — Regresión en features cercanas
- **Trace:** `test-results/place-order-Snapshot-chromium/trace.zip`
- **Pasos para reproducir:**
  1. Haz un pedido como cliente "John Doe" con dirección de envío "123 Old St".
  2. Navega a `/account` y actualiza la dirección del cliente a "456 New Ave".
  3. Vuelve a la página de detalle del pedido.
- **Esperado:** El detalle muestra "123 Old St" (snapshot en el momento del pedido).
- **Real:** El detalle muestra "456 New Ave" (datos vivos). La garantía del snapshot está rota.
- **Causa probable:** El template de detalle del pedido lee de la relación viva `customer` en vez del campo `shipping_address_snapshot` guardado en el pedido al momento de la creación.
- **Fix sugerido:** Actualizar la vista de detalle del pedido para que lea todos los campos de cliente/envío de los campos de snapshot del pedido, no del registro del cliente relacionado. Verificar que todos los campos de snapshot se rellenan al crear el pedido.
- **Nota:** Este bug **también rompe el email de confirmación y cualquier reporte histórico**. El test 7.3 se saltó porque depende de este fix. Una vez 7.2 esté arreglado, re-ejecutar 7.3.
- **Riesgo si se envía:** Las etiquetas de envío imprimen la dirección nueva aunque el pedido se hizo con la antigua. Los paquetes llegan al sitio equivocado. Problema legal/compliance para cualquier industria regulada.

### BUG-003 — Cantidad = 0 elimina silenciosamente la línea del carrito

- **Severidad:** 🟡 Media (confusión de UX, sin corrupción de datos)
- **Test:** 3.1 — Boundary cantidad = 0
- **Ángulo:** 3 — Valores límite
- **Trace:** `test-results/place-order-Quantity-zero-chromium/trace.zip`
- **Pasos para reproducir:**
  1. Añade una línea con cantidad `0`, producto válido.
  2. Submit del checkout.
- **Esperado:** O rechazada con error explícito, o aceptada con subtotal de línea = $0.00.
- **Real:** La línea es **eliminada silenciosamente** del carrito. Sin mensaje de error. El usuario ve su línea desaparecer sin explicación.
- **Causa probable:** El frontend filtra `qty <= 0` antes del submit sin notificar al usuario.
- **Fix sugerido:** O mostrar validación "La cantidad debe ser al menos 1", o aceptar y mostrar la línea con $0.00. Decidir de forma determinista — el silent drop es la peor opción.

---

## 💡 Mejoras Propuestas (observaciones no-bug)

1. **La página de confirmación debería mostrar el ID del pedido prominentemente.** Actualmente está enterrado en la URL — los usuarios tienen que copiar la URL para saber su número de pedido. Añade un header grande "Pedido #1234".
2. **El truncado de dirección debería ser visible.** El test 3.3 pasó porque el campo trunca a 200 chars, pero lo hace silenciosamente. Añade un contador de caracteres y un indicador visible de truncado.
3. **Idempotency keys para todos los flujos de dinero.** BUG-002 es un caso específico de un patrón general. Recomendamos un patrón global: cualquier endpoint que cree registros que afectan dinero (pedidos, pagos, refunds) acepta una idempotency key.
4. **Añadir una ruta de test guest al seeder.** El test 8.1 se saltó porque el entorno de test requiere login. Añade una fixture de checkout guest para que los tests guest puedan correr en CI.

---

## 📋 Evaluación de cobertura

- **Ángulo 1 (inputs vacíos):** 3/3 cubiertos, todos pasan.
- **Ángulo 2 (datos inválidos):** 3/3 cubiertos, 1 bug crítico.
- **Ángulo 3 (valores límite):** 3/3 cubiertos, 1 bug medio.
- **Ángulo 4 (caracteres especiales):** 2/2 cubiertos, todos pasan. **Sugerimos expandir** a testear lenguajes RTL en nombres de producto (solo se testearon nombres de cliente).
- **Ángulo 5 (doble click):** 2/2 cubiertos, 1 bug crítico.
- **Ángulo 6 (navegación):** 3/3 cubiertos, todos pasan.
- **Ángulo 7 (regresión cercana):** 4/4 cubiertos, 1 bug alto. **Los 4 vecinos nombrados testeados.**
- **Ángulo 8 (auth):** 1/1 cubierto pero saltado (falta fixture). **Cobertura insuficiente** — recomendamos un `specs/checkout-permissions.md` dedicado cubriendo las matrices guest, customer y admin.

**Huecos de cobertura a abordar en el siguiente pase:**
- Checkout guest (después de añadir la fixture).
- Checkout concurrente desde dos sesiones sobre el último item en stock (race condition).
- Direcciones internacionales (formatos postales no-ASCII).
- Edge cases de moneda si se soporta multi-moneda.

---

## 🔄 Tests de regresión nuevos añadidos

Los siguientes tests pasaron y fueron commiteados a la suite de regresión permanente:

```
tests/place-order/happy-paths.spec.ts          (3 tests)
tests/place-order/empty-inputs.spec.ts         (3 tests)
tests/place-order/invalid-data.spec.ts         (2 de 3 — el test del BUG-001 dejado como expected-failure pendiente del fix)
tests/place-order/boundary-values.spec.ts      (2 de 3 — el test del BUG-003 dejado como expected-failure pendiente del fix)
tests/place-order/injection.spec.ts            (2 tests)
tests/place-order/double-click.spec.ts         (1 de 2 — el test del BUG-002 dejado como expected-failure pendiente del fix)
tests/place-order/navigation.spec.ts           (3 tests)
tests/place-order/regression-neighbors.spec.ts (3 de 4 — el test del BUG-004 dejado como expected-failure pendiente del fix)
```

**Total commiteado:** 19 tests pasando + 4 tests expected-failure (se pondrán en verde cuando los bugs se arreglen). Los tests expected-failure actúan como garantía de que el fix del bug está verificado antes de cerrarlo.

---

## Próximas acciones

1. **Developer:** Arreglar BUG-001, BUG-002, BUG-004 (en ese orden — crítico → alto). BUG-003 es medio, puede ir en la siguiente iteración.
2. **Después de los fixes:** Re-ejecutar `npx playwright test tests/place-order/`. Los 4 tests expected-failure deberían ahora pasar y se flipean a aserciones normales.
3. **Pedir el siguiente pase de QA** cuando el commit del fix esté en main — pídele a Claude "haz QA del checkout otra vez" o "rerun QA on the order flow".
