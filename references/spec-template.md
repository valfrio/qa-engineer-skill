# Plantilla de Spec — Plan de Test Adversarial

Este es un **ejemplo completamente desarrollado** de cómo se ve un buen plan de test. Úsalo como referencia cuando hagas prompt al agente Planner, cuando revises su output, o cuando escribas un plan a mano.

El ejemplo cubre un flujo genérico de **e-commerce "Place Order"** — un dominio que todos entienden y que tiene la misma riqueza adversarial (líneas de pedido, totales, impuestos, transiciones de estado, inventario, snapshot de cliente, permisos multi-rol) que la mayoría de features reales de negocio. La misma estructura aplica a cualquier feature en cualquier app — los ángulos son los mismos; solo cambian los ataques concretos.

---

## Cómo usar esta plantilla

1. **Como referencia para el Planner.** Dile a Claude: *"Genera un plan para [feature] con la misma forma que `references/spec-template.md`."* El Planner producirá un plan con los 8 ángulos dispuestos de la misma forma.
2. **Como checklist de auditoría.** Cuando el Planner devuelva un plan, compáralo sección por sección contra esta plantilla. Ángulo faltante = revisar.
3. **Como ejemplo desarrollado para nuevos miembros del equipo.** Léelo una vez y entenderás qué significa "QA adversarial" en términos concretos.

---

# Plan de Test: Place Order (`/checkout`)

**Feature:** El cliente rellena el carrito, introduce envío/pago, y hace el pedido. El sistema crea Order, decrementa inventario, cobra el pago, envía email de confirmación.
**URL:** `https://staging.example.com/checkout`
**Autor del spec:** Claude (agente Planner), revisado por Claude (loop de QA)
**Seed de referencia:** `tests/seed.spec.ts`

---

## Mentalidad

Intenta **romper** el checkout. No verifiques que renderiza — asume que renderiza. Busca: pedidos duplicados en silencio, decrementos de inventario rotos, doble cargo en el pago, miscálculos de impuestos/totales, y efectos secundarios (emails de confirmación) disparándose dos veces o ninguna.

## Áreas afectadas (radio de explosión)

Identificadas antes de generar tests, usadas como input para el Ángulo 7.

| # | Vecino | Por qué podría romperse |
|---|---------|--------------------|
| 1 | `/orders` (lista de pedidos) | Consume el mismo modelo Order; el pedido nuevo debe aparecer con totales correctos |
| 2 | `/orders/{id}` (detalle de pedido) | Lee el snapshot del cliente; si el snapshot deriva, el detalle muestra datos equivocados |
| 3 | `/products` y store de inventario | Los items del pedido deben decrementar stock atómicamente |
| 4 | Email de confirmación de pedido | Se dispara al crear el pedido; debe dispararse exactamente una vez con totales correctos |
| 5 | `/cart` (página de carrito) | Comparte el componente de líneas/precios con checkout |
| 6 | UI de gestión de pedidos del admin | Lee el mismo modelo Order desde un rol distinto |

---

## Inventario de tests

El plan cubre los 8 ángulos adversariales más el happy path. Total: 24 tests.

| Ángulo | # tests |
|-------|---------|
| 0 — Happy paths (minoría) | 3 |
| 1 — Inputs vacíos | 3 |
| 2 — Datos inválidos | 3 |
| 3 — Valores límite | 3 |
| 4 — Caracteres especiales / inyección | 2 |
| 5 — Doble click / submit rápido | 2 |
| 6 — Edge cases de navegación | 3 |
| 7 — Regresión en features cercanas | 4 |
| 8 — Edges de auth / permisos | 1 |

---

## Ángulo 0 — Happy paths (la minoría)

### TEST 0.1 — Hacer pedido con un item, tarjeta válida, sin descuento

- **Acción:** Añade 1 item ($50) al carrito, navega al checkout, rellena envío (dirección válida), rellena tarjeta (tarjeta de test válida), submit.
- **Esperado:** Pedido creado en estado `paid`. Subtotal $50.00, tax $4.00, total $54.00. Aparece arriba en la lista de pedidos.
- **Aserciones:**
  - La URL cambia a `/orders/{id}` después del submit
  - `getByText(/order #/i)` visible en la cabecera del detalle
  - `getByText('$54.00')` visible en los totales
  - `getByText(/paid/i)` (badge de estado) visible
  - Network: `POST /api/orders` devuelve 200, sin errores de console

### TEST 0.2 — Hacer pedido con varios items y tasas de impuesto mixtas

- **Acción:** Añade 3 items: ($50 con tax 8%), ($30 con tax 8%, qty 2), ($100 exento de tax). Submit.
- **Esperado:** Subtotal $210.00, tax $8.80, total $218.80.
- **Aserciones:** Cada línea visible en el detalle con su breakdown individual de subtotal/tax. El total coincide con el valor calculado.

### TEST 0.3 — Aplicar código de descuento válido

- **Acción:** Desde checkout, introduce el código de descuento `SAVE10`. Submit.
- **Esperado:** Total reducido en 10%. Pedido creado con la línea de descuento.
- **Aserciones:** Descuento visible en el breakdown de totales. Total final coincide con el valor calculado.

---

## Ángulo 1 — Inputs vacíos

### TEST 1.1 — Submit checkout con carrito vacío

- **Acción:** Navega directamente a `/checkout` con un carrito vacío, submit igualmente.
- **Esperado:** Error de validación o redirect de vuelta al carrito. No se crea pedido.
- **Aserciones:** Mensaje "Your cart is empty" visible O redirigido a `/cart`. Sin entrada nueva en la lista de pedidos.

### TEST 1.2 — Submit sin dirección de envío

- **Acción:** Añade item, deja todos los campos de envío vacíos, rellena pago, submit.
- **Esperado:** Errores de validación en cada campo de envío. Submit bloqueado.
- **Aserciones:** Mensajes de error visibles. Network: sin POST exitoso.

### TEST 1.3 — Submit sin método de pago

- **Acción:** Añade item, rellena envío, deja los campos de tarjeta vacíos, submit.
- **Esperado:** Error de validación en pago. Submit bloqueado.
- **Aserciones:** Error visible específicamente en la sección de pago.

---

## Ángulo 2 — Datos inválidos

### TEST 2.1 — Cantidad negativa en carrito

- **Acción:** Modifica la cantidad de una línea del carrito a `-5` (vía control UI o DOM).
- **Esperado:** O rechazado con validación, o clamped a 0/1. El total nunca se vuelve negativo.
- **Aserciones:** Mensaje de error O campo clamped. El total no es negativo.

### TEST 2.2 — Número de tarjeta inválido

- **Acción:** Introduce tarjeta `1234 5678 9012 3456` (falla check de Luhn), CVV/expiry válidos.
- **Esperado:** La validación de tarjeta rechaza. Submit bloqueado O el procesador de pago devuelve error.
- **Aserciones:** Mensaje de error visible. No se crea pedido.

### TEST 2.3 — Email malformado en campo de cliente

- **Acción:** Introduce `not-an-email`, `user@`, `@domain.com`, `user@@domain.com` en el campo de email.
- **Esperado:** Error de validación en cada uno.
- **Aserciones:** Error específico de campo visible. No se crea pedido.

---

## Ángulo 3 — Valores límite

### TEST 3.1 — Cantidad = 0

- **Acción:** Pon la cantidad de una línea del carrito a `0`.
- **Esperado:** O rechazado con validación, o la línea se quita del carrito. Sea cual sea la regla, debe ser **determinista** — no se elimina en silencio.
- **Aserciones:** El comportamiento de la línea coincide con la regla. El total refleja la línea correctamente (o la excluye explícitamente).

### TEST 3.2 — Cantidad máxima (999999)

- **Acción:** Pon cantidad a `999999`, precio unitario `99999.99`.
- **Esperado:** El cálculo no hace overflow. Total mostrado correctamente. Redondeo a 2 decimales.
- **Aserciones:** Total mostrado sin notación científica, sin `Infinity`, sin `NaN`. Si no hay stock, el check de stock rechaza.

### TEST 3.3 — Línea de dirección con 10.000 caracteres

- **Acción:** Pega un string de 10K chars en la línea de dirección de envío.
- **Esperado:** O truncado a un máximo documentado (ej: 200 chars) o rechazado. La base de datos no lanza un 500.
- **Aserciones:** Sin error 500. El detalle del pedido no rompe el layout.

---

## Ángulo 4 — Caracteres especiales / inyección

### TEST 4.1 — Nombre de cliente con HTML/script

- **Acción:** Nombre del cliente = `<script>alert(1)</script><img src=x onerror=alert(2)>`.
- **Esperado:** Renderizado como texto escapado en el detalle del pedido y el email de confirmación. Sin alert. Sin carga de imagen.
- **Aserciones:** `getByText('<script>')` visible (texto literal, no interpretado). Console sin errores.

### TEST 4.2 — Dirección con unicode + emoji + RTL

- **Acción:** Nombre de envío `José María 山田 🎉 مرحبا`. Submit del pedido.
- **Esperado:** El pedido almacena el nombre correctamente. Detalle y email renderizan todos los caracteres.
- **Aserciones:** El detalle muestra el nombre completo. El campo de DB coincide con el input byte por byte.

---

## Ángulo 5 — Doble click / submit rápido

### TEST 5.1 — Doble click en el botón "Place Order"

- **Acción:** Rellena el formulario de checkout completamente. Usa `dblclick()` en el botón submit.
- **Esperado:** Exactamente **un** pedido creado, no dos. Tarjeta cargada una sola vez.
- **Aserciones:** La lista de pedidos contiene exactamente un pedido nuevo. El log del procesador de pago muestra un cargo.

### TEST 5.2 — Resubmit rápido en retry de fallo de pago

- **Acción:** Dispara un decline de pago, después click en "Try again" 5 veces en 200ms.
- **Esperado:** Pago reintentado **una** vez por intención de click, no 5. Sin estado parcial.
- **Aserciones:** Network muestra un count de retry controlado. Sin pedido duplicado.

---

## Ángulo 6 — Edge cases de navegación

### TEST 6.1 — Botón atrás después de pedido exitoso

- **Acción:** Haz un pedido. Después del redirect a la página de confirmación, click en el botón atrás del browser.
- **Esperado:** El usuario vuelve al checkout (estado limpio) O al carrito. **Sin** resubmit. **Sin** pedido duplicado.
- **Aserciones:** Sin segundo pedido en la lista.

### TEST 6.2 — Refresh durante el procesamiento del pago

- **Acción:** Submit del checkout, refresh de la página mientras el spinner se está mostrando.
- **Esperado:** Sin doble cargo. O el pedido se completa una vez, o el refresh muestra estado limpio.
- **Aserciones:** Exactamente un pedido en la lista. El procesador de pago muestra un cargo.

### TEST 6.3 — Abrir checkout en dos pestañas simultáneamente

- **Acción:** Abre `/checkout` en pestaña A y pestaña B con el mismo carrito. Submit en ambas.
- **Esperado:** Ambas tienen éxito independientemente O la segunda detecta que el carrito ya fue enviado. Sin decremento doble de inventario.
- **Aserciones:** El delta de inventario coincide con la cantidad real pedida, no el doble.

---

## Ángulo 7 — Regresión en features cercanas (NO NEGOCIABLE)

### TEST 7.1 — La lista de pedidos (`/orders`) sigue paginando y filtrando correctamente

- **Acción:** Después de crear 3 pedidos nuevos, navega a la lista. Aplica filtro por estado = "Paid". Aplica filtro por rango de fechas.
- **Esperado:** Los filtros funcionan. La paginación funciona. Los counts coinciden.
- **Aserciones:** La lista filtrada contiene solo pedidos coincidentes. El count total coincide.

### TEST 7.2 — El detalle del pedido lee del snapshot del cliente, no de los datos vivos

- **Acción:** Haz un pedido con cliente "John Doe" en "123 Old St". Después de crearlo, **actualiza la dirección del cliente en `/account`**. Recarga el detalle del pedido.
- **Esperado:** El detalle sigue mostrando la dirección **original** del snapshot, NO la nueva.
- **Aserciones:** La dirección en el detalle coincide con la dirección en el momento del pedido, no con la dirección actual del cliente.

### TEST 7.3 — El email de confirmación se dispara exactamente una vez

- **Acción:** Haz un pedido. Revisa el mailbox de test.
- **Esperado:** Exactamente un email de confirmación, con totales, items y nombre de cliente correctos.
- **Aserciones:** Count del mailbox = 1 (no 0, no 2+). El cuerpo del email contiene los datos del pedido actuales, sin `{{variables}}` en crudo.

### TEST 7.4 — El inventario fue decrementado exactamente una vez

- **Acción:** Anota el nivel de inventario del SKU pedido antes del test. Después de hacer el pedido, comprueba otra vez.
- **Esperado:** El stock decreció exactamente en la cantidad pedida, no más.
- **Aserciones:** Stock nuevo = stock viejo − cantidad de la línea.

---

## Ángulo 8 — Edges de auth / permisos

### TEST 8.1 — Checkout como guest vs cliente logueado

- **Acción:** Como guest (sin sesión), completa el checkout. Después como un usuario logueado con una cuenta distinta, completa otro checkout.
- **Esperado:** Ambos tienen éxito. El pedido del guest está asociado con el email, no con una cuenta de usuario. El pedido logueado está asociado con la cuenta de usuario. Sin cross-contaminación.
- **Aserciones:** El pedido del guest tiene `user_id = null`, email del cliente set. El pedido logueado tiene `user_id` set. Cada usuario solo ve sus propios pedidos en `/account/orders`.

---

## Fuera de scope (otros specs los cubren)

- Cancelación de pedidos y refunds → cubierto por `specs/order-cancel.md`
- Suscripciones / billing recurrente → cubierto por `specs/subscription.md`
- Gestión de pedidos del admin → cubierto por `specs/admin-orders.md`

---

## Evaluación de cobertura

| Ángulo | Cubierto | Notas |
|-------|---------|-------|
| 0 — Happy paths | ✅ | 3 tests |
| 1 — Inputs vacíos | ✅ | 3 tests |
| 2 — Datos inválidos | ✅ | 3 tests |
| 3 — Valores límite | ✅ | 3 tests |
| 4 — Caracteres especiales | ✅ | 2 tests |
| 5 — Doble click | ✅ | 2 tests |
| 6 — Edges de navegación | ✅ | 3 tests |
| 7 — Regresión cercana | ✅ | 4 tests, los 4 vecinos críticos cubiertos |
| 8 — Auth / permisos | ⚠️ | Solo 1 test — se recomienda matriz completa de roles como spec separado |

**Plan aceptado para generación.** Enviando al agente Generator.
