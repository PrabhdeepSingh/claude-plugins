# Comments — the budget applied to one function

The four checks from §3, applied to a function the way a coding agent typically emits it. Read the generated version first and count: nine lines of code, fifteen lines of comment (six of doc block, nine inline), and not one of them tells the reader something the code does not.

## As generated

```ts
/**
 * Calculates the total for an invoice.
 * @param invoice - The invoice to calculate the total for.
 * @param taxRate - The tax rate to apply.
 * @returns The total amount in cents.
 */
function calculateInvoiceTotal(invoice: Invoice, taxRate: number): number {
  // Start with a subtotal of zero
  let subtotalCents = 0;
  // Loop through each line item and add its amount
  for (const lineItem of invoice.lineItems) {
    // Multiply quantity by unit price to get the line amount
    subtotalCents += lineItem.quantity * lineItem.unitPriceCents;
  }
  // Apply the discount if there is one
  // Updated to use the new discount field as requested in the ticket
  const discountedCents = subtotalCents - (invoice.discountCents ?? 0);
  // Calculate tax by multiplying by the tax rate and rounding.
  // We round here because floating point math can produce fractional
  // cents and we need to make sure we always return a whole number.
  const taxCents = Math.round(discountedCents * taxRate);
  // Return the discounted amount plus tax
  return discountedCents + taxCents;
}
```

## Under the budget

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

One comment survived. It names an invariant the code cannot state (whole cents, rounded exactly once), it sits above the block it governs, and it is one line.

## What each removed comment failed

| Comment | Check it failed |
|---|---|
| `Start with a subtotal of zero` | 1 — restates `let subtotalCents = 0` |
| `Loop through each line item and add its amount` | 1 — restates the `for` line |
| `Multiply quantity by unit price…` | 1 — restates the expression; 3 — interleaved inside the loop body |
| `Apply the discount if there is one` | 1 — restates `?? 0` |
| `Updated to use the new discount field as requested in the ticket` | 1 — narrates the edit and the task; commit-message material |
| `Calculate tax by multiplying…` + two more lines | 1 — first line restates the code; 4 — three lines for one why |
| `Return the discounted amount plus tax` | 1 — restates `return` |
| The six-line doc block | Docstring rule — a module-private helper gets none; and its `@param`/`@returns` lines only restate names and types the signature already declares |

Had a second why been genuinely needed inside this body, check 2 says the answer is a split (`applyDiscount`, `roundedTax`) with the why moving to the smaller function's single line — not a second comment.

## Docstrings: the public line versus the private silence

The same function, when it *is* the exported API of a billing module, earns a docstring in the language's convention — one summary line that says what the caller gets, with no signature echo.

```python
def calculate_invoice_total(invoice: Invoice, tax_rate: Decimal) -> int:
    """Return the invoice total in minor units, tax rounded once."""
    ...

def _line_subtotal(line_items: list[LineItem]) -> int:
    return sum(item.quantity * item.unit_price_cents for item in line_items)
```

```go
// CalculateInvoiceTotal returns the invoice total in minor units with tax rounded once.
func CalculateInvoiceTotal(invoice Invoice, taxRate float64) int64 { … }

func lineSubtotal(lineItems []LineItem) int64 { … }
```

The public function's docstring exists because callers who never open this file will read it in their editor. The unexported helper has a name that already says everything a one-line docstring would, so it gets nothing.

Avoid — the summary line that repeats the signature:

```python
def calculate_invoice_total(invoice: Invoice, tax_rate: Decimal) -> int:
    """calculate_invoice_total(invoice, tax_rate) -> int. Takes an invoice and a tax rate."""
```

Prefer — the summary line that tells the caller what they get and the one fact they could not guess:

```python
def calculate_invoice_total(invoice: Invoice, tax_rate: Decimal) -> int:
    """Return the invoice total in minor units, tax rounded once."""
```
