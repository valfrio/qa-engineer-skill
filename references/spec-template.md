# Spec Template — Adversarial Test Plan

This is a **fully-worked example** of what a good test plan looks like. Use it as a reference when prompting the Planner agent, when reviewing its output, or when writing a plan by hand.

The example covers a generic **e-commerce "Place Order"** flow — a domain everyone understands and which has the same adversarial richness (line items, totals, taxes, state transitions, inventory, customer snapshot, multi-role permissions) as most real business features. The same structure applies to any feature in any app — the angles are the same; only the concrete attacks change.

---

## How to use this template

1. **As a reference for the Planner.** Tell Claude: *"Generate a plan for [feature] in the same shape as `references/spec-template.md`."* The Planner will produce a plan with all 8 angles laid out the same way.
2. **As an audit checklist.** When the Planner returns a plan, compare it section-by-section against this template. Missing angle = revise.
3. **As a worked example for new team members.** Read this once and you'll understand what "adversarial QA" means in concrete terms.

---

# Test Plan: Place Order (`/checkout`)

**Feature:** Customer fills cart, enters shipping/payment, places order. System creates Order, decrements inventory, charges payment, sends confirmation email.
**URL:** `https://staging.example.com/checkout`
**Spec author:** Claude (Planner agent), reviewed by Claude (QA loop)
**Reference seed:** `tests/seed.spec.ts`

---

## Mindset

Try to **break** the checkout. Do not check that it renders — assume it renders. Look for: silent duplicate orders, broken inventory decrements, payment double-charges, tax/total miscalculations, and side effects (confirmation emails) firing twice or not at all.

## Affected areas (blast radius)

Identified before generating tests, used as input for Angle 7.

| # | Neighbor | Why it might break |
|---|---------|--------------------|
| 1 | `/orders` (order list) | Consumes the same Order model; new order must appear with correct totals |
| 2 | `/orders/{id}` (order detail) | Reads the customer snapshot; if snapshot drifts, detail shows wrong data |
| 3 | `/products` and inventory store | Items in the order must decrement stock atomically |
| 4 | Order confirmation email | Triggered on order creation; must fire exactly once with correct totals |
| 5 | `/cart` (cart page) | Shares the line-item / pricing component with checkout |
| 6 | Admin order management UI | Reads the same Order model from a different role |

---

## Test inventory

Plan covers all 8 adversarial angles plus the happy path. Total: 24 tests.

| Angle | # tests |
|-------|---------|
| 0 — Happy paths (minority) | 3 |
| 1 — Empty inputs | 3 |
| 2 — Invalid data | 3 |
| 3 — Boundary values | 3 |
| 4 — Special chars / injection | 2 |
| 5 — Double-click / rapid submit | 2 |
| 6 — Navigation edge cases | 3 |
| 7 — Regression in nearby features | 4 |
| 8 — Auth / permission edges | 1 |

---

## Angle 0 — Happy paths (the minority)

### TEST 0.1 — Place order with one item, valid card, no discount

- **Action:** Add 1 item ($50) to cart, navigate to checkout, fill shipping (valid address), fill card (valid test card), submit.
- **Expected:** Order created in `paid` state. Subtotal $50.00, tax $4.00, total $54.00. Appears at top of order list.
- **Assertions:**
  - URL changes to `/orders/{id}` after submit
  - `getByText(/order #/i)` visible in detail header
  - `getByText('$54.00')` visible in totals
  - `getByText(/paid/i)` (status badge) visible
  - Network: `POST /api/orders` returns 200, no console errors

### TEST 0.2 — Place order with multiple items and mixed tax rates

- **Action:** Add 3 items: ($50 taxable 8%), ($30 taxable 8%, qty 2), ($100 tax-exempt). Submit.
- **Expected:** Subtotal $210.00, tax $8.80, total $218.80.
- **Assertions:** Each line visible in detail with its individual subtotal/tax breakdown. Total matches calculated value.

### TEST 0.3 — Apply valid discount code

- **Action:** From checkout, enter discount code `SAVE10`. Submit.
- **Expected:** Total reduced by 10%. Order created with discount line item.
- **Assertions:** Discount visible in totals breakdown. Final total matches calculated value.

---

## Angle 1 — Empty inputs

### TEST 1.1 — Submit checkout with empty cart

- **Action:** Navigate directly to `/checkout` with an empty cart, submit anyway.
- **Expected:** Validation error or redirect back to cart. No order created.
- **Assertions:** Error message "Your cart is empty" visible OR redirected to `/cart`. No new entry in order list.

### TEST 1.2 — Submit with no shipping address

- **Action:** Add item, leave all shipping fields empty, fill payment, submit.
- **Expected:** Validation errors on each shipping field. Submit blocked.
- **Assertions:** Error messages visible. Network: no successful POST.

### TEST 1.3 — Submit with no payment method

- **Action:** Add item, fill shipping, leave card fields empty, submit.
- **Expected:** Validation error on payment. Submit blocked.
- **Assertions:** Error visible on the payment section specifically.

---

## Angle 2 — Invalid data

### TEST 2.1 — Negative quantity in cart

- **Action:** Modify cart line quantity to `-5` (via UI control or DOM).
- **Expected:** Either rejected with validation, or clamped to 0/1. Total never goes negative.
- **Assertions:** Error message OR field clamped. Total is non-negative.

### TEST 2.2 — Invalid card number

- **Action:** Enter card `1234 5678 9012 3456` (fails Luhn check), valid CVV/expiry.
- **Expected:** Card validation rejects. Submit blocked OR payment processor returns error.
- **Assertions:** Error message visible. No order created.

### TEST 2.3 — Malformed email in customer field

- **Action:** Enter `not-an-email`, `user@`, `@domain.com`, `user@@domain.com` in email field.
- **Expected:** Validation error on each.
- **Assertions:** Field-specific error visible. No order created.

---

## Angle 3 — Boundary values

### TEST 3.1 — Quantity = 0

- **Action:** Set cart line quantity to `0`.
- **Expected:** Either rejected with validation, or line removed from cart. Whatever the rule is, it must be **deterministic** — not silently dropped.
- **Assertions:** Line behavior matches rule. Total reflects the line correctly (or excludes it explicitly).

### TEST 3.2 — Maximum quantity (999999)

- **Action:** Set quantity to `999999`, unit price `99999.99`.
- **Expected:** Calculation does not overflow. Total displayed correctly. Rounding to 2 decimals.
- **Assertions:** Total displayed without scientific notation, no `Infinity`, no `NaN`. Stock check rejects if not available.

### TEST 3.3 — Address line with 10,000 characters

- **Action:** Paste a 10K-char string into shipping address line.
- **Expected:** Either truncated to a documented max (e.g., 200 chars) or rejected. Database does not throw 500.
- **Assertions:** No 500 error. Order detail does not break layout.

---

## Angle 4 — Special chars / injection

### TEST 4.1 — Customer name with HTML/script

- **Action:** Customer name = `<script>alert(1)</script><img src=x onerror=alert(2)>`.
- **Expected:** Rendered as escaped text in order detail and confirmation email. No alert fires. No image loads.
- **Assertions:** `getByText('<script>')` is visible (literal text, not interpreted). Console has no errors.

### TEST 4.2 — Address with unicode + emoji + RTL

- **Action:** Shipping name `José María 山田 🎉 مرحبا`. Submit order.
- **Expected:** Order stores name correctly. Detail view and email render all characters.
- **Assertions:** Detail view shows the full name. DB field matches input byte-for-byte.

---

## Angle 5 — Double-click / rapid submit

### TEST 5.1 — Double-click "Place Order" button

- **Action:** Fill checkout form completely. Use `dblclick()` on submit button.
- **Expected:** Exactly **one** order created, not two. Card charged once.
- **Assertions:** Order list contains exactly one new order. Payment processor log shows one charge.

### TEST 5.2 — Rapid resubmit on payment failure retry

- **Action:** Trigger a payment decline, then click "Try again" 5 times in 200ms.
- **Expected:** Payment retried **once** per click intent, not 5 times. No partial state.
- **Assertions:** Network shows controlled retry count. No duplicate order.

---

## Angle 6 — Navigation edge cases

### TEST 6.1 — Back button after successful order

- **Action:** Place an order. After redirect to confirmation page, click browser back.
- **Expected:** User returns to checkout (clean state) OR cart. **No** resubmission. **No** duplicate order.
- **Assertions:** No second order in list.

### TEST 6.2 — Refresh during payment processing

- **Action:** Submit checkout, refresh the page while the spinner is showing.
- **Expected:** No double-charge. Either the order completes once, or the refresh shows a clean state.
- **Assertions:** Exactly one order in list. Payment processor shows one charge.

### TEST 6.3 — Open checkout in two tabs simultaneously

- **Action:** Open `/checkout` in tab A and tab B with the same cart. Submit in both.
- **Expected:** Both succeed independently OR second one detects the cart was already submitted. No duplicate inventory decrement.
- **Assertions:** Inventory delta matches the actual quantity ordered, not double.

---

## Angle 7 — Regression in nearby features (NON-NEGOTIABLE)

### TEST 7.1 — Order list (`/orders`) still paginates and filters correctly

- **Action:** After creating 3 new orders, navigate to list. Apply filter by status = "Paid". Apply filter by date range.
- **Expected:** Filters work. Pagination works. Counts match.
- **Assertions:** Filtered list contains only matching orders. Total count matches.

### TEST 7.2 — Order detail reads customer snapshot, not live data

- **Action:** Place an order with customer "John Doe" at "123 Old St". After creation, **update the customer's address in `/account`**. Reload the order detail.
- **Expected:** Detail view still shows the **original** address from snapshot, NOT the new address.
- **Assertions:** Address in detail view matches the address at moment of order, not the current customer address.

### TEST 7.3 — Confirmation email fires exactly once

- **Action:** Place an order. Check the test mailbox.
- **Expected:** Exactly one confirmation email, with correct totals, items, and customer name.
- **Assertions:** Mailbox count = 1 (not 0, not 2+). Email body contains current order data, no raw `{{variables}}`.

### TEST 7.4 — Inventory was decremented exactly once

- **Action:** Note inventory level for the ordered SKU before the test. After order is placed, check again.
- **Expected:** Stock decreased by exactly the quantity ordered, not more.
- **Assertions:** New stock = old stock − line quantity.

---

## Angle 8 — Auth / permission edges

### TEST 8.1 — Guest checkout vs. logged-in customer

- **Action:** As a guest (no session), complete checkout. Then as a logged-in user with a different account, complete a separate checkout.
- **Expected:** Both succeed. Guest order is associated with the email, not a user account. Logged-in order is associated with the user account. No cross-contamination.
- **Assertions:** Guest order has `user_id = null`, customer email set. Logged-in order has `user_id` set. Each user only sees their own orders in `/account/orders`.

---

## Out of scope (other specs handle these)

- Order cancellation and refunds → covered by `specs/order-cancel.md`
- Subscription / recurring billing → covered by `specs/subscription.md`
- Admin order management → covered by `specs/admin-orders.md`

---

## Coverage assessment

| Angle | Covered | Notes |
|-------|---------|-------|
| 0 — Happy paths | ✅ | 3 tests |
| 1 — Empty inputs | ✅ | 3 tests |
| 2 — Invalid data | ✅ | 3 tests |
| 3 — Boundary values | ✅ | 3 tests |
| 4 — Special chars | ✅ | 2 tests |
| 5 — Double-click | ✅ | 2 tests |
| 6 — Navigation edges | ✅ | 3 tests |
| 7 — Nearby regression | ✅ | 4 tests, all 4 critical neighbors covered |
| 8 — Auth / permissions | ⚠️ | 1 test only — full role matrix recommended as separate spec |

**Plan accepted for generation.** Sending to Generator agent.
