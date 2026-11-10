# Comments — one example

The checks are in `SKILL.md` section 3. This is what passing them looks like.

```ts
function calculateInvoiceTotal(invoice: Invoice, taxRate: number): number {
  let subtotalCents = 0;
  for (const lineItem of invoice.lineItems) {
    subtotalCents += lineItem.quantity * lineItem.unitPriceCents;
  }
  const discountedCents = subtotalCents - (invoice.discountCents ?? 0);
  // Round once, here — tax on minor units must never carry fractional cents downstream.
  const taxCents = Math.round(discountedCents * taxRate);
  return discountedCents + taxCents;
}
```

The one comment names an invariant the code cannot state, sits above the line it governs and is one line. Comments that restate a line ("loop through each line item") or narrate the edit ("updated as requested") are not written.

When the function is exported, it gets a one-line docstring in the language's convention that says what the caller gets and does not repeat the signature. An unexported helper whose name says it all gets none.

```python
def calculate_invoice_total(invoice: Invoice, tax_rate: Decimal) -> int:
    """Return the invoice total in minor units, tax rounded once."""
    ...

def _line_subtotal(line_items: list[LineItem]) -> int:
    return sum(item.quantity * item.unit_price_cents for item in line_items)
```
