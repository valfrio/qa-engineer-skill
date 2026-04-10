# Example QA Report — Reference Output

This is a **fully-worked example** of the QA report Claude must produce after every QA session. Use it as the structural template for real reports.

A QA report has three jobs:

1. **Tell the human what's broken** in priority order, with reproduction steps.
2. **Tell future-Claude what was tested** so the next QA session doesn't re-do work.
3. **Tell the team what got committed** to the regression suite as permanent coverage.

If a real report omits any of these, it's incomplete.

The example below is paired with [`spec-template.md`](spec-template.md) — same feature (e-commerce "Place Order"), so you can read both side by side.

---

# QA Report — Place Order Feature

**Tester:** Claude (qa-engineer skill v2.0.0)
**Feature:** Place Order (`/checkout`)
**Spec:** `specs/place-order.md`
**Test files:** `tests/place-order/*.spec.ts` (24 tests)
**Environment:** `https://staging.example.com`
**Playwright artifacts:** `test-results/` (videos, traces, screenshots)
**Run duration:** 4m 12s

---

## TL;DR

**Status: 🔴 BLOCKING — do not deploy to production.**

Of 24 adversarial tests, **18 passed, 4 failed, 2 skipped**. Two failures are **critical** (data integrity / money), one is **high** (regression in nearby feature), and one is **medium** (UX). The two skipped tests are blocked on a missing test fixture.

Recommendation: **fix the 2 critical bugs before merge**, then re-run the suite.

---

## Test results

| # | Test | Angle | Status | Severity | Notes |
|---|------|-------|--------|----------|-------|
| 0.1 | Place order with one item | 0 — Happy | ✅ | — | — |
| 0.2 | Place order with multiple items + mixed tax | 0 — Happy | ✅ | — | — |
| 0.3 | Apply valid discount code | 0 — Happy | ✅ | — | — |
| 1.1 | Submit checkout with empty cart | 1 — Empty | ✅ | — | Validation works |
| 1.2 | Submit with no shipping address | 1 — Empty | ✅ | — | — |
| 1.3 | Submit with no payment method | 1 — Empty | ✅ | — | — |
| 2.1 | Negative quantity | 2 — Invalid | ❌ | **Critical** | See BUG-001 |
| 2.2 | Invalid card number | 2 — Invalid | ✅ | — | Luhn check works |
| 2.3 | Malformed email | 2 — Invalid | ✅ | — | All 4 cases rejected |
| 3.1 | Quantity = 0 | 3 — Boundary | ⚠️ | Medium | See BUG-003 |
| 3.2 | Maximum quantity (999999) | 3 — Boundary | ✅ | — | Total displayed correctly, no overflow |
| 3.3 | Address with 10K chars | 3 — Boundary | ✅ | — | Truncated to 200 chars (silent — see improvement #2) |
| 4.1 | Customer name with HTML/script | 4 — Injection | ✅ | — | Properly escaped in detail and email |
| 4.2 | Address with unicode + emoji | 4 — Injection | ✅ | — | All 4 scripts render correctly |
| 5.1 | Double-click "Place Order" | 5 — Double-click | ❌ | **Critical** | See BUG-002 |
| 5.2 | Rapid retry on payment failure | 5 — Double-click | ✅ | — | Idempotent — only retries once |
| 6.1 | Back button after order | 6 — Nav | ✅ | — | No resubmission |
| 6.2 | Refresh during payment | 6 — Nav | ✅ | — | No double-charge |
| 6.3 | Two-tab simultaneous checkout | 6 — Nav | ✅ | — | Both succeed, inventory correct |
| 7.1 | Order list filters/pagination | 7 — Regression | ✅ | — | — |
| 7.2 | Order detail reads snapshot, not live | 7 — Regression | ❌ | **High** | See BUG-004 |
| 7.3 | Confirmation email fires once | 7 — Regression | ⏭️ | — | Skipped — depends on 7.2 fix |
| 7.4 | Inventory decremented exactly once | 7 — Regression | ✅ | — | Stock matches expected delta |
| 8.1 | Guest vs logged-in checkout | 8 — Auth | ⏭️ | — | Skipped — test fixture missing (no guest test path) |

**Totals:** 18 ✅ / 4 ❌ / 2 ⏭️

---

## 🔴 Top critical bugs

### BUG-001 — Negative quantity creates order with negative total

- **Severity:** 🔴 Critical (money flow corruption)
- **Test:** 2.1 — Negative quantity
- **Angle:** 2 — Invalid data
- **Trace:** `test-results/place-order-Negative-quantity-chromium/trace.zip`
- **Steps to reproduce:**
  1. Add an item to cart.
  2. In the cart, modify the line quantity to `-5` (via DOM or API).
  3. Proceed to checkout and submit.
- **Expected:** Server rejects with 422 OR field clamps to 1. Total cannot be negative.
- **Actual:** Order created with total `-$60.50`. Stored in DB. Visible in order list with negative total. **No validation on negative quantity in either client or server.**
- **Probable cause:** Missing `min: 1` validation rule on the line quantity field at both UI and API layers.
- **Suggested fix:** Add server-side validation rejecting `quantity <= 0` in the order creation endpoint. Add client-side input constraint `min="1"` on the quantity field. Both layers — server is the security boundary, client is the UX.
- **Risk if shipped:** Customer (or attacker) places orders with negative totals, effectively crediting their account. Money lost. Reconciliation nightmare.

### BUG-002 — Double-click on "Place Order" creates duplicate order and double-charges card

- **Severity:** 🔴 Critical (data integrity + duplicate charge)
- **Test:** 5.1 — Double-click submit
- **Angle:** 5 — Double-click / rapid submit
- **Trace:** `test-results/place-order-Double-click-chromium/trace.zip`
- **Steps to reproduce:**
  1. Fill the checkout form with valid data.
  2. Use double-click (or human double-click) on "Place Order".
- **Expected:** Exactly 1 order created. Card charged exactly 1 time.
- **Actual:** **2 orders created**, both for the same customer with the same items. Payment processor log shows **2 charges** of $54.00 each.
- **Probable cause:** Submit button is not disabled after the first click. No idempotency key sent with the request, so the server cannot deduplicate.
- **Suggested fix (minimum):** Disable the submit button immediately on click and re-enable only on response or error. Show a loading spinner.
- **Suggested fix (proper):** Generate an idempotency key on the client when checkout mounts. Send it with the order request. Server stores it in a 10-second window — duplicate requests with the same key return the original order, not a new one. Pattern compatible with Stripe-style idempotency keys.
- **Risk if shipped:** Customers see duplicate charges on their statements, dispute charges, file chargebacks. Reputation damage and processor fees.

### BUG-004 — Order detail reads from live customer data, not snapshot

- **Severity:** 🟠 High (snapshot guarantee broken — historical accuracy)
- **Test:** 7.2 — Order detail reads snapshot
- **Angle:** 7 — Regression in nearby features
- **Trace:** `test-results/place-order-Snapshot-chromium/trace.zip`
- **Steps to reproduce:**
  1. Place an order as customer "John Doe" with shipping address "123 Old St".
  2. Navigate to `/account` and update the customer's address to "456 New Ave".
  3. Navigate back to the order detail page.
- **Expected:** Detail view shows "123 Old St" (snapshot at moment of order).
- **Actual:** Detail view shows "456 New Ave" (live data). The snapshot guarantee is broken.
- **Probable cause:** The order detail template reads from the live `customer` relation instead of the `shipping_address_snapshot` field stored on the order at creation time.
- **Suggested fix:** Update the order detail view to read all customer/shipping fields from the snapshot fields on the order, not from the related customer record. Verify all snapshot fields are populated at order creation.
- **Note:** This bug **also breaks the confirmation email and any historical reporting**. Test 7.3 was skipped because it depends on this fix. Once 7.2 is fixed, re-run 7.3.
- **Risk if shipped:** Shipping labels print to the new address even though the order was placed with the old. Packages go to the wrong place. Legal/compliance issue for any regulated industry.

### BUG-003 — Quantity = 0 silently drops line from cart

- **Severity:** 🟡 Medium (UX confusion, no data corruption)
- **Test:** 3.1 — Quantity = 0 boundary
- **Angle:** 3 — Boundary values
- **Trace:** `test-results/place-order-Quantity-zero-chromium/trace.zip`
- **Steps to reproduce:**
  1. Add a line with quantity `0`, valid product.
  2. Submit checkout.
- **Expected:** Either rejected with explicit error, or accepted with line subtotal = $0.00.
- **Actual:** Line is **silently dropped** from the cart. No error message. User sees their line disappear with no explanation.
- **Probable cause:** Frontend filters out `qty <= 0` before submission without notifying the user.
- **Suggested fix:** Either show validation "Quantity must be at least 1", or accept and display the line with $0.00. Decide deterministically — silent drop is the worst option.

---

## 💡 Proposed Improvements (non-bug observations)

1. **Confirmation page should show order ID prominently.** Currently it's buried in the URL — users have to copy the URL to know their order number. Add a large "Order #1234" header.
2. **Address truncation should be visible.** Test 3.3 passed because the field truncates at 200 chars, but it does so silently. Add a character counter and visible truncation indicator.
3. **Idempotency keys for all money flows.** BUG-002 is a specific case of a general pattern. Recommend a global pattern: any endpoint that creates money-affecting records (orders, payments, refunds) accepts an idempotency key.
4. **Add a guest test path to the seeder.** Test 8.1 was skipped because the test environment requires login. Add a guest checkout fixture so guest tests can run in CI.

---

## 📋 Coverage assessment

- **Angle 1 (empty inputs):** 3/3 covered, all pass.
- **Angle 2 (invalid data):** 3/3 covered, 1 critical bug.
- **Angle 3 (boundary values):** 3/3 covered, 1 medium bug.
- **Angle 4 (special chars):** 2/2 covered, all pass. **Suggest expanding** to test RTL languages in product names (only customer names tested).
- **Angle 5 (double-click):** 2/2 covered, 1 critical bug.
- **Angle 6 (navigation):** 3/3 covered, all pass.
- **Angle 7 (nearby regression):** 4/4 covered, 1 high bug. **All 4 named neighbors tested.**
- **Angle 8 (auth):** 1/1 covered but skipped (missing fixture). **Insufficient coverage** — recommend a dedicated `specs/checkout-permissions.md` covering guest, customer, and admin matrices.

**Coverage gaps to address in next pass:**
- Guest checkout (after fixture is added).
- Concurrent checkout from two sessions on the last item in stock (race condition).
- International addresses (non-ASCII postal formats).
- Currency edge cases if multi-currency is supported.

---

## 🔄 New regression tests added

The following tests passed and were committed to the permanent regression suite:

```
tests/place-order/happy-paths.spec.ts          (3 tests)
tests/place-order/empty-inputs.spec.ts         (3 tests)
tests/place-order/invalid-data.spec.ts         (2 of 3 — BUG-001 test left as expected-failure pending fix)
tests/place-order/boundary-values.spec.ts      (2 of 3 — BUG-003 test left as expected-failure pending fix)
tests/place-order/injection.spec.ts            (2 tests)
tests/place-order/double-click.spec.ts         (1 of 2 — BUG-002 test left as expected-failure pending fix)
tests/place-order/navigation.spec.ts           (3 tests)
tests/place-order/regression-neighbors.spec.ts (3 of 4 — BUG-004 test left as expected-failure pending fix)
```

**Total committed:** 19 passing tests + 4 expected-failure tests (will turn green when bugs are fixed). The expected-failure tests act as a guarantee that the bug fix is verified before close.

---

## Next actions

1. **Developer:** Fix BUG-001, BUG-002, BUG-004 (in that order — critical → high). BUG-003 is medium, can ship in next iteration.
2. **After fixes:** Re-run `npx playwright test tests/place-order/`. The 4 expected-failure tests should now pass and be flipped to normal assertions.
3. **Request the next QA pass** when the fix commit lands — ask Claude with "haz QA del checkout otra vez" or "rerun QA on the order flow".
