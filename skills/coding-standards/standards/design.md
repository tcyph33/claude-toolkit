## 1. Design & Complexity

### 1.1 Keep Routines Small and Focused

**Standard:** Every routine (function, method) should do one thing and do it well. A routine should have a single, clearly identifiable purpose. If you cannot describe what a routine does with a concise name, it is doing too much.

**Justification:** Complexity is the primary enemy of software quality. Small routines are easier to name, understand, test, debug, and reuse. When a routine does one thing, errors are isolated and changes have minimal blast radius.

**Example:**
```
// BAD: Does multiple things
function processOrder(order):
    validate(order)
    calculateTax(order)
    applyDiscount(order)
    chargePayment(order)
    sendConfirmationEmail(order)
    updateInventory(order)

// GOOD: Orchestrator delegates to focused routines
function processOrder(order: Order):
    Order validatedOrder = validateOrder(order);
    Order pricedOrder = applyPricing(validatedOrder);
    chargePayment(pricedOrder)
    fulfillOrder(pricedOrder)

function applyPricing(order: Order) -> Order:
    Tax tax = calculateTax(order);
    Discount discount = calculateDiscount(order);
    order.tax = tax
    order.discount = discount
    return order
```

### 1.2 Minimize Coupling Between Modules

**Standard:** Modules and classes should depend on as few other modules as possible. Connections between modules should be narrow (few parameters), visible (explicit, not through globals), and flexible (using abstractions rather than concrete implementations).

**Justification:** Tight coupling means changes propagate unpredictably. Loose coupling allows you to understand, modify, and test one module without understanding the entire system.

**Example:**
```
// BAD: Function reaches into internals of another module
function formatReport(database):
    records = database.connection.cursor.execute("SELECT ...")
    return records.format()

// GOOD: Depend on an abstraction, not the implementation
function formatReport(recordProvider: RecordProvider) -> String:
    List records = recordProvider.getRecords();
    String formattedReport = formatAsTable(records);
    return formattedReport
```

### 1.3 Practice Information Hiding

**Standard:** Each module should hide a design decision — a piece of complexity — behind a simple interface. Internal data structures, algorithms, and implementation choices should not be visible to callers.

**Justification:** Information hiding is the most powerful technique for managing complexity. When implementation details are hidden, they can change without affecting the rest of the system. It isolates the "what" from the "how."

**Example:**
```
// BAD: Exposes internal representation
class Employee:
    public salaryInCents: int
    public taxBracketCode: string

// GOOD: Hides representation behind behavior
class Employee:
    private salaryInCents: int
    private taxBracketCode: string

    function getNetPay() -> Money:
        const netPay: double = calculateNetFromGross(salaryInCents, taxBracketCode)
        return netPay
```

### 1.4 Limit Nesting and Complexity per Routine

**Standard:** Keep cyclomatic complexity low (aim for under 7 decision points per routine). Nesting deeper than 3 levels is a signal to extract a subroutine or restructure.

**Justification:** Deeply nested or highly branching code exceeds human working memory. Studies show defect rates rise sharply when complexity exceeds 7-10 decision points. Flat, linear code is dramatically easier to trace mentally.

**Example:**
```
// BAD: Deep nesting
function getDiscount(customer, order):
    if customer != null:
        if customer.isActive:
            if order.total > 100:
                if customer.loyaltyYears > 5:
                    return 0.20
                else:
                    return 0.10
            else:
                return 0.05
    return 0

// GOOD: Early returns flatten the logic
function getDiscount(customer: Customer, order: Order) -> double:
    final double NO_DISCOUNT = 0;
    final double SMALL_ORDER_DISCOUNT = 0.05;
    final double LOYAL_CUSTOMER_DISCOUNT = 0.20;
    final double STANDARD_DISCOUNT = 0.10;
    final double MINIMUM_ORDER_FOR_TIERED_DISCOUNT = 100;
    final int LOYALTY_YEARS_THRESHOLD = 5;

    final boolean isIneligible = ((customer == null) || (!customer.isActive));
    if (isIneligible):
        return NO_DISCOUNT
    final boolean isSmallOrder = (order.total <= MINIMUM_ORDER_FOR_TIERED_DISCOUNT);
    if (isSmallOrder):
        return SMALL_ORDER_DISCOUNT
    final boolean isLoyalCustomer = (customer.loyaltyYears > LOYALTY_YEARS_THRESHOLD);
    if (isLoyalCustomer):
        return LOYAL_CUSTOMER_DISCOUNT
    return STANDARD_DISCOUNT
```

### 1.5 Design at the Right Level of Abstraction

**Standard:** Each routine should operate at a single, consistent level of abstraction. Do not mix high-level orchestration with low-level detail in the same routine.

**Justification:** Mixing abstraction levels forces the reader to constantly context-switch between "what is being done" and "how it is being done." Consistent abstraction makes code read like well-organized prose.

**Example:**
```
// BAD: Mixes levels of abstraction
function generateReport(data):
    connection = openSocket("report-server", 443)
    connection.setTLSVersion(1.3)
    reportData = aggregate(data)
    html = "<html><body>" + reportData.toHTML() + "</body></html>"
    connection.send(html.encode("utf-8"))

// GOOD: Single level of abstraction
function generateReport(data: DataSet):
    ReportData reportData = aggregate(data);
    String html = renderAsHTML(reportData);
    sendToReportServer(html)
```
