## Refactoring & Quality

### REF-1: Refactor When You See Duplication

**Standard:** When the same logic appears in three or more places, extract it into a shared routine. Two occurrences may be coincidental; three is a pattern. When extracting, ensure the abstraction is genuine — the duplicated code must represent the same concept, not just happen to look similar.

**Justification:** Duplicated code means duplicated bugs and duplicated maintenance. When a fix is needed, it must be applied in every copy — and inevitably one copy gets missed. A single authoritative implementation is the only reliable approach.

**Example:**
```
// BAD: Same validation in three handlers
function handleCreate(request):
    if not request.email or "@" not in request.email:
        return error("Invalid email")
    ...

function handleUpdate(request):
    if not request.email or "@" not in request.email:
        return error("Invalid email")
    ...

function handleInvite(request):
    if not request.email or "@" not in request.email:
        return error("Invalid email")
    ...

// GOOD: Extracted once
function validateEmail(email):
    // Reject missing or malformed email
    final boolean isEmailMissing = (email == null);
    final boolean isMissingAtSign = (!email.contains("@"));
    final boolean isEmailInvalid = (isEmailMissing || isMissingAtSign);
    if (isEmailInvalid):
        raise ValidationError("Invalid email")

function handleCreate(request):
    validateEmail(request.email)
    ...
```

### REF-2: Refactor When a Routine is Too Long

**Standard:** If a routine exceeds roughly 50-100 lines or you need to scroll to see it all, look for opportunities to extract logical subsections into well-named helper routines. The right length is whatever it takes to do one thing — no more.

**Justification:** Long routines accumulate multiple responsibilities, making them difficult to understand, test, and modify. Extracting sections into named routines effectively adds documentation (via the routine name) and makes each piece independently testable.

**Example:**
```
// BAD: One long function doing everything
function generateInvoice(order):
    // 20 lines calculating line items
    // 15 lines calculating tax
    // 25 lines formatting PDF
    // 10 lines sending email
    // total: 70+ lines

// GOOD: Orchestrator with focused subroutines
function generateInvoice(order):
    final List<LineItem> lineItems = calculateLineItems(order)
    final Money tax = calculateTax(lineItems, order.region)
    final PDF pdf = renderInvoicePDF(lineItems, tax, order.customer)
    sendInvoiceEmail(order.customer.email, pdf)
```

### REF-3: Refactor When Naming is Difficult

**Standard:** If you struggle to name a variable, routine, or class, treat it as a design smell. Difficulty naming usually means the entity lacks a single coherent purpose. Restructure until a clear name emerges naturally.

**Justification:** Naming difficulty is a reliable signal that the abstraction is wrong. A well-designed entity with a single purpose practically names itself. Forcing a name onto a muddled entity just hides the design problem.

**Example:**
```
// BAD: Hard to name because it does too much
function handleStuff(data):  // "stuff" reveals confusion about purpose
    validate(data)
    transform(data)
    save(data)
    notify(data.owner)

// GOOD: Clear names emerge from clear responsibilities
function saveValidatedRecord(data):
    final Record record = transformToRecord(data)
    repository.save(record)

function notifyRecordOwner(record):
    sendNotification(record.owner, "Record saved")
```

### REF-4: Fix Broken Windows Immediately

**Standard:** When you encounter code that is clearly wrong, misleading, or poorly structured while working in that area, fix it then and there — as long as the fix is small and safe. Do not leave known problems for later when the cost of fixing now is minimal.

**Justification:** The Broken Windows theory: one piece of bad code signals that quality doesn't matter here, which invites more bad code. Small fixes prevent quality erosion. However, large refactors should be planned separately — this standard applies to incidental improvements within your current scope.

**Example:**
```
// While implementing a new feature, you notice:
function calc(a, b):  // terrible name, unclear params
    return a * b + a

// Since you're already working in this file, fix it:
function calculateTotal(subtotal, taxRate):
    final int total = ((subtotal * taxRate) + subtotal);
    return total
```

### REF-5: Leave Code Better Than You Found It (Boy Scout Rule)

**Standard:** When you modify a file, improve the immediate surroundings if the improvement is small, safe, and unambiguous. Rename an unclear variable, remove a dead code path, or fix a misleading comment. Do not pile on large changes unrelated to your task.

**Justification:** Incremental improvement prevents code from degrading over time without requiring dedicated refactoring sprints. The constraint to keep improvements small and local prevents scope creep while steadily raising quality.

**Example:**
```
// Before your change, you see nearby:
int x = getTimeout();  // timeout in ms
// You rename it while you're here:
int timeoutMs = getTimeout();

// But do NOT restructure the entire file — that's a separate task.
```
