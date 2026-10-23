## Code Layout & Formatting

### FMT-1: Use Whitespace to Reveal Logical Structure

**Standard:** Group related statements together and separate logical sections with blank lines. Align related assignments when it aids readability. Use indentation consistently to show containment relationships. The visual shape of the code should reflect its logical shape.

**Justification:** Code is read far more than it is written. Visual grouping allows the reader to perceive logical blocks at a glance without reading every line. Poor layout forces line-by-line reading and obscures the high-level structure.

**Example:**
```
// BAD: No visual structure
function processOrder(order):
    subtotal = calculateSubtotal(order.items)
    taxRate = getTaxRate(order.region)
    tax = subtotal * taxRate
    discount = getDiscount(order.customer)
    total = subtotal + tax - discount
    receipt = createReceipt(order, total)
    sendConfirmation(order.customer.email, receipt)
    updateInventory(order.items)
    logTransaction(order.id, total)

// GOOD: Blank lines separate logical phases
function processOrder(order):
    // Calculate final price from subtotal, tax, and discount
    subtotal = calculateSubtotal(order.items)
    taxRate = getTaxRate(order.region)
    tax = subtotal * taxRate
    discount = getDiscount(order.customer)
    total = subtotal + tax - discount

    // Generate and send order confirmation
    receipt = createReceipt(order, total)
    sendConfirmation(order.customer.email, receipt)

    // Record the completed transaction
    updateInventory(order.items)
    logTransaction(order.id, total)
```

### FMT-2: Keep Lines Readable

**Standard:** Limit line length to what can be read without horizontal scrolling (typically 80-120 characters depending on the project convention). When a line must be broken, break at a logical seam — after a comma, before an operator, or at a natural grouping boundary.

**Justification:** Long lines force horizontal scrolling or mental line-wrapping, both of which interrupt reading flow. Logical break points make continued lines read naturally and keep related elements visually grouped.

**Example:**
```
// BAD: One excessively long line
result = someService.transformAndValidate(inputData, configOptions, retryPolicy, timeoutMs, fallbackHandler, metricsCollector)

// GOOD: Broken at logical seams
result = someService.transformAndValidate(
    inputData,
    configOptions,
    retryPolicy,
    timeoutMs,
    fallbackHandler,
    metricsCollector
)
```

### FMT-3: Put Related Code Together

**Standard:** Code that operates on the same data or contributes to the same logical step should be physically adjacent. Do not interleave unrelated operations. If two statements must be understood together, they should appear together.

**Justification:** Proximity implies relationship. When related code is scattered, the reader must search and mentally reassemble fragments. Keeping related operations adjacent reduces the "span" a reader must hold in memory.

**Example:**
```
// BAD: Related operations interleaved with unrelated ones
startDate = parseDate(input.start)
logMetric("parse_started")
endDate = parseDate(input.end)
config = loadConfig()
duration = endDate - startDate
reportTitle = config.get("title")

// GOOD: Related operations grouped
// Parse date range and compute duration
startDate = parseDate(input.start)
endDate = parseDate(input.end)
duration = endDate - startDate

// Load report configuration
config = loadConfig()
reportTitle = config.get("title")

// Record processing telemetry
logMetric("parse_started")
```

### FMT-4: Use Formatting to Expose Errors

**Standard:** Format parallel structures in parallel. When multiple lines follow the same pattern (e.g., a series of assignments, a series of conditionals, a data table), use consistent formatting so that deviations from the pattern are visually obvious. Use exactly one space before and after operators — do not column-align assignments, as alignment is fragile and creates unnecessary churn when names change.

**Justification:** When parallel code is formatted consistently, a typo or logic error breaks the visual pattern and becomes visible at a glance. Irregular formatting (missing spaces, inconsistent operators) hides errors by making every line look different even when they should be uniform.

**Example:**
```
// BAD: Irregular formatting hides errors
config.timeout = 30
config.retries=3
config.backoffMs = 1000
config.maxConnections = 10
config.enableSSL= true
config.port =8080

// GOOD: Consistent single-space formatting exposes deviations
config.timeout = 30
config.retries = 3
config.backoffMs = 1000
config.maxConnections = 10
config.enableSSL = true
config.port = 8080
```
