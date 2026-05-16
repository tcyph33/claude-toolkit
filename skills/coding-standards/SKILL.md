---
name: coding-standards
description: >-
  Enforces coding standards for software construction.
  Apply these standards to ALL code written or modified, regardless of language.
  These are mandatory principles governing design, variables, control flow,
  defensive programming, self-documenting code, refactoring, and testing.
metadata:
  author: Tyler Cypher
  keywords: "coding-standards, quality, clean-code, construction, best-practices"
  tags: "coding-standards, code-quality, construction"
---

# Coding Standards

These standards are **mandatory** and apply to all code you write or modify, in any language. Each standard includes the rule, the justification, and a concrete example.

---

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

---

## 2. Variables & Data

### 2.1 Initialize Variables Close to First Use

**Standard:** Declare and initialize variables as close as possible to where they are first used, and in the smallest scope that contains all their uses. If a variable is only needed inside a loop, conditional, or inner block, declare it there — not in the enclosing scope. Never declare all variables at the top of a routine unless the language requires it. This applies equally to constants — do not group all constants in a block at the top of a file or module. Define each constant in the innermost scope that uses it, whether that is inside a loop body, a function, at the top of a class, or at module level only if truly used across the module.

**Justification:** The span between initialization and use is "live" time where the variable can be misused or misunderstood. Short live time reduces the window for errors and makes it easier to see a variable's purpose in context. Declaring a variable in an outer scope when it is only used in an inner scope widens its live time unnecessarily — other code in the outer scope can accidentally reference or shadow it. A constants block at the top of a file forces the reader to scroll back and forth to understand what each value means; placing constants near their usage keeps context together.

**Example:**
```
// BAD: All constants grouped at the top, far from where they're used
MAX_RETRIES = 3
TIMEOUT_SECONDS = 30
PAGE_SIZE = 50
CACHE_TTL = 3600
// ... 100 lines later, MAX_RETRIES is used ...
// ... 200 lines later, CACHE_TTL is used ...

// BAD: Declared far from use
function processItems(items):
    total = 0
    taxRate = 0
    discount = 0
    formattedOutput = ""
    // ... 30 lines later ...
    taxRate = getTaxRate()
    // ... 20 more lines ...
    discount = getDiscount()

// BAD: Declared in outer scope when only used inside the loop
function fetchWithRetry(url):
    final int maxRetries = 3;
    final int timeoutSeconds = 30;
    for attempt in range(maxRetries):
        response = fetch(url, timeout=timeoutSeconds)
        ...

// GOOD: Constants declared in the smallest scope that uses them
function fetchWithRetry(url):
    final int maxRetries = 3;
    for attempt in range(maxRetries):
        final int timeoutSeconds = 30;
        response = fetch(url, timeout=timeoutSeconds)
        ...

function paginate(items: List) -> List:
    final int pageSize = 50;
    final List chunkItems = chunk(items, pageSize);
    return chunkItems
```

### 2.2 Minimize Variable Scope

**Standard:** Give each variable the smallest scope possible. Prefer local variables over instance variables, instance variables over class variables, and class variables over globals. If a variable is only needed inside a loop or conditional, declare it there.

**Justification:** Wide scope increases the number of places a variable can be read or modified, making it harder to reason about its state at any given point. Narrow scope means fewer interactions and fewer bugs.

**Example:**
```
// BAD: Variable scoped too broadly
class OrderProcessor:
    private tempTotal: float  // only used in one method

    function calculate(order):
        tempTotal = 0
        for item in order.items:
            tempTotal += item.price
        return tempTotal

// GOOD: Variable scoped to where it's needed
class OrderProcessor:
    function calculate(order: Order) -> double:
        double total = 0;
        for item in order.items:
            total += item.price
        return total
```

### 2.3 Use Meaningful, Precise Names

**Standard:** Names should fully and accurately describe what the variable represents. Use names that are specific enough to distinguish the variable from others. Never abbreviate words — spell out every word completely. The only exceptions are universally understood abbreviations that would look unnatural written out (e.g., `id`, `url`, `html`, `api`). If a word can be spelled out without sounding awkward, spell it out. Boolean names should read as true/false assertions.

**Justification:** A variable's name is its primary documentation. Ambiguous names force the reader to trace code to understand meaning, which is the single most common source of misunderstanding in code. Abbreviations save a few keystrokes at the cost of forcing every future reader to mentally expand them — and different readers may expand them differently.

**Example:**
```
// BAD: Vague or abbreviated
d = getDate()
temp = calculate(x)
flag = check(record)
lst = getAll()
param_count = args.length
cmd_result = execute(input)
repo_dir = getClonePath()

// GOOD: Precise and descriptive
Date currentDate = getDate();
double monthlyRevenue = calculateRevenue(salesData);
boolean isEligibleForDiscount = checkDiscountEligibility(customer);
List activeEmployees = getActiveEmployees();
int parameterCount = args.length;
String commandResult = execute(input);
String repositoryDirectory = getClonePath();
```

### 2.4 Name Booleans in the Positive Form

**Standard:** Always name boolean variables in the positive/affirmative form — expressing what something *is*, *has*, or *can* do (e.g., `isEmpty`, `isValid`, `hasPermission`, `canRetry`). Never embed negation in the name (e.g., `isNotEmpty`, `isInvalid`, `hasNoPermission`). When you need the negative sense, negate the positive-form variable with `!` (e.g., `!isEmpty`).

**Justification:** Negation in a name forces every reader to mentally invert the meaning. Worse, when code negates an already-negative name (`!isNotEmpty`), the double negative is confusing and error-prone. Positive-form names compose cleanly with the `!` operator and read naturally in both the true and false cases.

**Example:**
```
// BAD: Negation embedded in the name
final boolean isNotEmpty = (list.size() > 0);
final boolean isInvalid = (!schema.validate(input));
final boolean hasNoPermission = (!user.canAccess(resource));

// Leads to confusing double negatives:
if (!isNotEmpty):  // what does this mean?
if (!isInvalid):   // hard to parse

// GOOD: Positive-form names, negated with ! when needed
final boolean isEmpty = (list.size() == 0);
final boolean isValid = (schema.validate(input));
final boolean hasPermission = (user.canAccess(resource));

// Clear in both positive and negative use:
if (isEmpty):       // clearly: the list has no items
if (!isValid):      // clearly: validation failed
if (!hasPermission): // clearly: access denied
```

### 2.5 Use Named Constants Instead of Magic Numbers

**Standard:** Replace every literal number or string that has meaning with a named constant. The only acceptable literal numbers are 0 and 1 when used in their obvious mathematical sense.

**Justification:** Magic numbers hide intent. A reader cannot tell why the number 86400 appears in code, but `SECONDS_PER_DAY` is immediately clear. Named constants also create a single point of change when the value needs updating.

**Example:**
```
// BAD: Magic numbers
if retryCount > 3:
    sleep(86400)
    sendAlert("threshold exceeded")

// GOOD: Named constants
final int MAX_RETRIES = 3;

final boolean isOverRetryLimit = (retryCount > MAX_RETRIES);
if (isOverRetryLimit):
    final int SECONDS_PER_DAY = 86400;
    sleep(SECONDS_PER_DAY)
    final String THRESHOLD_ALERT_MESSAGE = "threshold exceeded";
    sendAlert(THRESHOLD_ALERT_MESSAGE)
```

### 2.6 One Purpose Per Variable

**Standard:** Each variable should have exactly one purpose. Never reuse a variable for a different meaning later in the same routine (e.g., using `temp` for a running total and then later for a formatted string).

**Justification:** Reusing variables for multiple purposes creates hidden dependencies between sections of code. It makes the code fragile — changing one section can break another — and defeats the reader's ability to trust what a name means.

**Example:**
```
// BAD: Variable reused for different purposes
value = input.length()
// ... processing ...
value = formatOutput(result)  // now means something completely different

// GOOD: Distinct variables for distinct purposes
int inputLength = input.length();
// ... processing ...
String formattedResult = formatOutput(result);
```

### 2.7 Always Use Explicit Type Declarations

**Standard:** Always declare variable types explicitly, even when the language supports type inference. This applies to all variable declarations, function parameters, and return types. In dynamically typed languages, use type annotations (e.g., Python type hints, TypeScript types) rather than relying on inference.

**Justification:** Explicit types communicate what kind of value a variable holds without requiring the reader to trace the assignment or hover for IDE tooltips. Type inference saves keystrokes at the cost of clarity — a reader must mentally evaluate the right-hand side to know the type. Explicit types make code self-documenting at every declaration site and catch type mismatches at compile time rather than runtime.

**Example:**
```
// BAD: Relying on type inference — reader must evaluate the right side to know types
val total = calculateTotal(items)
const user = await fetchUser(id)
let config = loadConfig()

// BAD: Python without type hints
def process_payment(amount, currency, account):
    result = charge(amount, currency)
    return result

// GOOD: Explicit types in every declaration
final double total = calculateTotal(items);
const User user = await fetchUser(id);
let Config config = loadConfig();

// GOOD: Python with type annotations
def process_payment(amount: Decimal, currency: str, account: Account) -> PaymentResult:
    result: PaymentResult = charge(amount, currency)
    return result

// GOOD: TypeScript with explicit types (not inferred)
const retryCount: number = 3;
const isEnabled: boolean = (config.featureFlag === "on");
const users: User[] = await fetchUsers(teamId);
```

### 2.8 Parenthesize All Math Expressions

**Standard:** Always use explicit parentheses in arithmetic and mathematical expressions to make the order of operations unambiguous. Do not rely on operator precedence rules — even when the default precedence would produce the correct result, parentheses make intent visible.

**Justification:** Operator precedence varies subtly across languages and is easy to misremember. Parentheses eliminate any ambiguity about evaluation order, making the expression's intent explicit to every reader regardless of their familiarity with precedence rules. They also prevent bugs when expressions are later modified — adding a term to an unparenthesized expression can silently change the result.

**Example:**
```
// BAD: Relies on precedence — reader must mentally evaluate order
final double total = subtotal * taxRate + subtotal
final double discount = price * quantity - rebate / 2

// GOOD: Parentheses make evaluation order explicit
final double total = ((subtotal * taxRate) + subtotal)
final double discount = ((price * quantity) - (rebate / 2))
```

### 2.9 Never Return Expressions Directly

**Standard:** Never return a computed expression directly from a `return` statement. Always assign the result to a named variable first, then return that variable. The variable name documents what the value represents.

**Justification:** A bare `return` with an expression forces the reader to mentally parse the expression and infer what it represents. A named variable before the return acts as documentation — it names the result and makes the function's output self-evident. It also provides a convenient inspection point during debugging.

**Example:**
```
// BAD: Returning expressions directly
function calculateTotal(subtotal, taxRate):
    return ((subtotal * taxRate) + subtotal)

function getFullName(firstName, lastName):
    return firstName + " " + lastName

// GOOD: Named variable before return
function calculateTotal(subtotal, taxRate):
    final double totalWithTax = ((subtotal * taxRate) + subtotal)
    return totalWithTax

function getFullName(firstName, lastName):
    final String fullName = firstName + " " + lastName
    return fullName
```

---

## 3. Control Flow

### 3.1 Put the Normal Case First

**Standard:** In conditional statements where both branches continue execution (no early return), put the most common or expected path in the `if` clause. Reserve `else` for the less common case. This standard does NOT apply to guard clauses (see 3.4) — when one branch exits early, prefer the guard clause pattern instead.

**Justification:** Readers expect the primary path first. Putting the normal case in the `if` clause allows a reader to quickly understand the expected behavior without reading through edge cases. However, when a branch can exit early, a guard clause produces flatter code and should be preferred.

**Example:**
```
// BAD: Uncommon case first in a two-path conditional
if order.isInternational():
    applyCustomsDuty(order)
    applyInternationalShipping(order)
else:
    applyDomesticShipping(order)

// GOOD: Common case first when both branches continue
final boolean isDomesticOrder = (order.isDomestic());
if (isDomesticOrder):
    applyDomesticShipping(order)
else:
    applyCustomsDuty(order)
    applyInternationalShipping(order)

// NOTE: When one path exits early, use a guard clause instead:
final boolean isInvalid = (!order.isValid());
if (isInvalid):
    Error invalidOrderError = error("Invalid order");
    return invalidOrderError
// ... normal processing continues at base indentation
```

### 3.2 Extract Conditionals into Named Boolean Variables

**Standard:** Every `if` statement and `while` loop condition must evaluate a named boolean variable — never an inline expression. Each boolean variable should represent exactly one logical condition, wrapped in parentheses to make the comparison explicit and the intended order of operations unambiguous. Each sub-expression within a compound comparison should also be parenthesized.

**Justification:** Inline conditions mix "what are we checking?" with "what do we do about it?" in a single line, violating the principle that each line should do one thing. A named boolean on its own line describes the condition; the `if` on the next line describes the decision. Parentheses around each comparison eliminate ambiguity about operator precedence and make the boundaries of each comparison explicit. One condition per variable makes each check independently readable, testable, and debuggable.

**Example:**
```
// BAD: Inline condition in the if statement
if (order.total > 100.00):
    applyDiscount(order)

// BAD: Missing parentheses around comparison
final boolean isLargeOrder = order.total > 100.00;

// GOOD: Even single conditions get extracted and parenthesized
final int ONE_MONTH = 30;
final boolean isOverdue = (invoice.daysPastDue > ONE_MONTH);
if (isOverdue):
    sendReminder(invoice)

// GOOD: Each sub-expression parenthesized for clarity
final int MINIMUM_LARGER_ORDER = 100;
final int LOYALTY_THRESHOLD = 2;
final boolean isLargeOrder = (order.total > MINIMUM_LARGER_ORDER);
final boolean isLoyalCustomer = (customer.loyaltyYears > LOYALTY_THRESHOLD);
final boolean isEligibleForDiscount = (isLargeOrder && isLoyalCustomer);
if (isEligibleForDiscount):
    applyDiscount(order)
```

### 3.3 Simplify Complex Boolean Expressions

**Standard:** Break complex boolean expressions into well-named intermediate variables or extract them into descriptive functions. A boolean expression with more than 2-3 terms should be simplified. Compound conditions must be composed from individually named booleans, not combined in a single expression. Each boolean variable should hold exactly one logical condition. When combining 3 or more boolean variables into a single compound expression, break the expression across multiple lines with one term per line.

**Justification:** Complex boolean expressions are the #1 source of off-by-one logic errors and are nearly impossible to verify by reading. Named booleans make the intent explicit and each sub-condition independently testable. Combining multiple conditions into a single variable hides the individual checks and makes it impossible to inspect intermediate state when debugging.

**Example:**
```
// BAD: Multiple conditions combined in one variable
final boolean isEligible = order.total > 100.00 && customer.loyaltyYears > 2;

// BAD: Complex inline boolean
if (user.age >= 18 and user.hasValidID and not user.isBanned and
    (user.country == "US" or user.country == "CA") and
    user.accountAge > 30):
    grantAccess(user)

// GOOD: Complex condition decomposed into named parts, each parenthesized
final int ADULT_AGE = 18;
final int ESTABLISHED_ACCOUNT_AGE = 30;
final boolean isAdult = (user.age >= ADULT_AGE);
final boolean isVerified = (user.hasValidID && !user.isBanned);
final boolean isInSupportedRegion = ((user.country == "US") || (user.country == "CA"));
final boolean isEstablishedAccount = (user.accountAge > ESTABLISHED_ACCOUNT_AGE);
final boolean isEligible = (
    isAdult &&
    isVerified &&
    isInSupportedRegion &&
    isEstablishedAccount);
if (isEligible):
    grantAccess(user)
```

### 3.4 Use Loop Variables Correctly

**Standard:** A loop must have a single, predictable mechanism for advancing. Do not combine a `for` header's automatic increment with manual index mutations in the body — this creates two competing sources of control. If a loop requires conditional advancement, use a `while` loop where the body is the sole owner of the index. Use meaningful names for loop variables when the loop body is more than a few lines.

**Justification:** When a `for` loop header increments an index and the body also mutates it, there are two competing mechanisms controlling iteration. This makes termination conditions unpredictable and is a common source of infinite loops and off-by-one errors. A `while` loop with explicit index management in the body makes the single advancement mechanism visible and auditable.

**Example:**
```
// BAD: For loop header and body both control the index
for i = 0 to items.length:
    if items[i].isSpecial:
        i = i + 2  // competes with the for header's increment
    process(items[i])

// GOOD: While loop with a single advancement mechanism
index = 0
final boolean hasItemsRemaining = (index < items.length);
while (hasItemsRemaining):
    item = items[index]

    // Process the item.
    final boolean isSpecialItem = (item.isSpecial);
    if (isSpecialItem):
        processSpecial(item)
        index += 3
    else:
        process(item)
        index += 1

    // Check if items remain.
    hasItemsRemaining = (index < items.length)
```

### 3.5 Avoid Deep Nesting with Early Exit

**Standard:** Use guard clauses (early returns, early continues, early breaks) to handle edge cases and errors at the top of a block, leaving the main logic at the base nesting level.

**Justification:** Every level of nesting adds cognitive load. Guard clauses eliminate entire branches from consideration, letting the reader focus on the happy path without tracking multiple nested contexts.

**Example:**
```
// BAD: Deep nesting
function processPayment(payment):
    if payment != null:
        if payment.amount > 0:
            if payment.isAuthorized():
                result = charge(payment)
                if result.success:
                    return receipt(result)
    return error("Payment failed")

// GOOD: Guard clauses
function processPayment(payment):
    // Ensure payment exists
    final boolean isPaymentNull = (payment == null);
    if (isPaymentNull):
        Error nullPaymentError = error("Null payment");
        return nullPaymentError

    // Reject non-positive amount
    final boolean isInvalidAmount = (payment.amount <= 0);
    if (isInvalidAmount):
        Error invalidAmountError = error("Invalid amount");
        return invalidAmountError

    // Reject unauthorized payment
    final boolean isUnauthorized = (!payment.isAuthorized());
    if (isUnauthorized):
        Error unauthorizedError = error("Not authorized");
        return unauthorizedError

    // Attempt the charge
    result = charge(payment)

    // Reject failed charge
    final boolean isChargeFailed = (!result.success);
    if (isChargeFailed):
        Error chargeError = error("Charge failed");
        return chargeError

    Receipt paymentReceipt = receipt(result);
    return paymentReceipt
```

### 3.6 Use Structured Control Flow

**Standard:** Use only standard control structures: sequence, selection (if/else, switch/case), and iteration (for, while). Avoid goto, deep recursion where iteration suffices, and complex exception-based control flow.

**Justification:** Structured programming exists because unstructured jumps make it impossible to reason about code state at any given point. Standard control structures have predictable entry and exit points that enable local reasoning.

**Example:**
```
// BAD: Exception used for control flow
function findItem(items, target):
    try:
        for item in items:
            if item.id == target:
                throw FoundException(item)
    catch FoundException as e:
        return e.item
    return null

// GOOD: Standard control flow
function findItem(items, target):
    // Search for the matching item
    for item in items:
        // Check if item matches
        final boolean isMatchingItem = (item.id == target);
        if (isMatchingItem):
            return item

    return null
```

---

## 4. Defensive Programming

### 4.1 Validate Data at System Boundaries

**Standard:** Validate all data that crosses a trust boundary: user input, API responses, file contents, database results, and inter-service messages. Once validated, internal code can trust the data without re-validating.

**Justification:** Garbage in, garbage out. Boundary validation means the rest of your code operates on known-good data, eliminating hundreds of internal defensive checks. It concentrates validation logic in one visible place rather than scattering it throughout the codebase.

**Example:**
```
// BAD: Validation scattered everywhere
function calculateAge(birthYear):
    if birthYear == null:  // checked here
        return null
    // ...
function formatAge(age):
    if age == null or age < 0:  // and here
        return "Unknown"
    // ...

// GOOD: Validate at the boundary, trust internally
function handleRequest(request):
    // Reject invalid birth year input
    birthYear = request.get("birth_year")
    final boolean isBirthYearMissing = (birthYear == null);
    final boolean isBirthYearMalformed = (!isValidYear(birthYear));
    final boolean isInvalidBirthYear = (isBirthYearMissing || isBirthYearMalformed);
    if (isInvalidBirthYear):
        Response birthYearError = errorResponse("Invalid birth year");
        return birthYearError

    // Calculate and return formatted age
    // Birth year is guaranteed valid here.
    int age = calculateAge(birthYear);
    String formattedAge = formatAge(age);
    Response ageResponse = successResponse(formattedAge);
    return ageResponse
```

### 4.2 Handle Errors Deliberately — Don't Swallow or Ignore

**Standard:** Every error path must be explicitly handled. Never use empty catch blocks. Choose a deliberate strategy for each error: recover, propagate with context, or fail fast with a meaningful message.

**Justification:** Swallowed errors create silent data corruption and impossible-to-debug failures. Every ignored error is a future incident. Explicit handling — even if that handling is "crash with a clear message" — makes failures visible and debuggable.

**Example:**
```
// BAD: Swallowed error
try:
    data = fetchFromAPI()
catch Exception:
    pass  // silently continues with no data

// BAD: Generic handling that hides the cause
try:
    data = fetchFromAPI()
catch Exception:
    return "Something went wrong"

// GOOD: Deliberate handling with context
try:
    data = fetchFromAPI()
catch NetworkError as e:
    log.warn("API unreachable, using cached data", error=e)
    data = loadFromCache()
catch ParseError as e:
    raise DataCorruptionError("API returned unparseable response", cause=e)
```

### 4.3 Fail Fast

**Standard:** When an error is detected that makes further processing meaningless or dangerous, stop immediately with a clear error message. Do not attempt to continue in a half-broken state.

**Justification:** Continuing after detecting an inconsistency spreads corruption further from its source, making root cause analysis exponentially harder. Fast failure preserves the state closest to the original error.

**Example:**
```
// BAD: Limping along in a broken state
function processConfig(configPath):
    config = loadFile(configPath)
    if config == null:
        config = {}  // "default" — now everything downstream gets empty values
    server = config.get("server", "localhost")  // silent wrong default
    port = config.get("port", 0)  // port 0? this will fail much later
    return connect(server, port)

// GOOD: Fail fast with clear message
function processConfig(configPath):
    // Reject missing config file
    config = loadFile(configPath)
    final boolean isConfigMissing = (config == null);
    if (isConfigMissing):
        fail("Cannot load required config file: " + configPath)

    // Reject config missing required fields
    final boolean isServerMissing = (!config.contains("server"));
    final boolean isPortMissing = (!config.contains("port"));
    final boolean isMissingRequiredFields = (isServerMissing || isPortMissing);
    if (isMissingRequiredFields):
        fail("Config missing required fields 'server' and 'port' in: " + configPath)

    // Connect using validated config
    Connection connection = connect(config["server"], config["port"]);
    return connection
```

### 4.4 Define Boundaries for Acceptable Input (Barricades)

**Standard:** Establish a clear boundary (a "barricade") in your architecture beyond which data is assumed clean. Input-handling code on the outside validates and sanitizes; code on the inside operates on trusted data.

**Justification:** This pattern eliminates redundant validation, makes the code cleaner on the inside, and makes it obvious where to look when bad data gets through — it must be a gap in the barricade.

**Example:**
```
// BAD: Validation scattered throughout — no clear boundary
function handleCreateUser(request):
    email = request.body.get("email")
    user = createUser(request.body.get("name"), email)
    if not isValidEmail(user.email):  // validated too late, after creation
        return error(400, "Invalid email format")
    sendWelcomeEmail(user.email)
    return success(user)

function createUser(name, email):
    if len(name) < 1 or len(name) > 200:  // validation buried inside
        return null
    return User(name, email)

// GOOD: Clear barricade at the API handler layer
// --- OUTSIDE THE BARRICADE (validates) ---
function handleCreateUser(request):
    // Validate email format
    email = request.body.get("email")
    final boolean isEmailInvalid = (!isValidEmail(email));
    if (isEmailInvalid):
        Error emailError = error(400, "Invalid email format");
        return emailError

    // Validate name length
    final int MIN_NAME_LENGTH = 1;
    final int MAX_NAME_LENGTH = 200;
    name = sanitize(request.body.get("name"))
    final boolean isNameTooShort = (name.length() < MIN_NAME_LENGTH);
    final boolean isNameTooLong = (name.length() > MAX_NAME_LENGTH);
    final boolean isNameInvalid = (isNameTooShort || isNameTooLong);
    if (isNameInvalid):
        String nameErrorMessage = "Name must be " + MIN_NAME_LENGTH + "-" + MAX_NAME_LENGTH + " characters";
        Error nameError = error(400, nameErrorMessage);
        return nameError

    // --- INSIDE THE BARRICADE (trusts) ---
    // Create user and send welcome email with validated data
    user = createUser(name, email)
    sendWelcomeEmail(user.email)
    Response successResponse = success(user);
    return successResponse
```

---

## 5. Self-Documenting Code

### 5.1 Make Code Read Without Comments

**Standard:** The primary documentation for code is the code itself. Use descriptive names, clear structure, and appropriate abstraction so the code communicates intent directly. If an individual line of code needs a comment to explain what it does, rewrite the line more clearly.

**Justification:** Comments rot — they become misleading when code changes but comments don't. Code that requires comments to be understood is code that should be rewritten more clearly. The gap between what code does and what a comment says it does is where bugs hide.

**Example:**
```
// BAD: Comment explains what the code does
// Loop through employees and add up salary for active full-time ones
total = 0
for i in range(len(emps)):
    if emps[i].st == 1 and emps[i].tp == "FT":
        total += emps[i].sal

// GOOD: Code explains itself
double totalFullTimeSalary = 0;
for Employee employee in employees:
    final boolean isEligible = (employee.isActive && employee.isFullTime);
    if (isEligible):
        totalFullTimeSalary += employee.salary
```

### 5.2 Comment the Why, Not the What

**Standard:** Write comments only when code cannot convey intent alone — typically to explain *why* a non-obvious decision was made, document external constraints, or note workarounds with ticket references.

**Justification:** "What" comments duplicate the code and become stale. "Why" comments preserve knowledge that is invisible in the code: business rules, regulatory requirements, performance constraints, or workarounds for external bugs.

**Example:**
```
// BAD: Comments that restate the code
x = x + 1  // increment x
user.setActive(false)  // set user to inactive
return null  // return null

// GOOD: Comments that explain why
// Tax calculation uses last year's rate until March (fiscal year transition, per finance team)
taxRate = getPreviousYearRate()

// Retry with exponential backoff — the vendor API rate-limits aggressively (JIRA-4521)
sleep(2 ** attemptNumber)
```

### 5.3 Comment Surprises and Non-Obvious Side Effects

**Standard:** When code has behavior that is not obvious from reading it — side effects, performance implications, external dependencies, unintuitive edge cases, or counterintuitive results — mark it with a comment. If a reader would be surprised by what the code does, that surprise must be documented.

**Justification:** Surprises in code cause bugs when future developers make reasonable assumptions that turn out to be wrong. A comment on non-obvious behavior acts as a warning sign: "this does more (or less) than you think." Without it, developers will misuse the code, skip necessary preconditions, or break it during refactoring.

**Example:**
```
// BAD: Non-obvious side effects with no warning
function getUser(userId):
    user = cache.get(userId)
    if user == null:
        user = database.fetch(userId)
        cache.put(userId, user)
        analytics.trackLookup(userId)  // side effect: sends data to external service
    return user

// GOOD: Side effects documented
function getUser(userId):
    user = cache.get(userId)
    if user == null:
        user = database.fetch(userId)
        cache.put(userId, user)
        // Sends a tracking event to the analytics service on cache miss
        analytics.trackLookup(userId)
    return user

// GOOD: Unintuitive behavior documented
// This closes the underlying stream — caller must not read from source after this call
function readAll(source):
    data = source.read()
    source.close()
    return data
```

### 5.4 Comment Each Code Block with High-Level Intent

**Standard:** Every logical block of code within a routine must be preceded by a short comment describing the high-level intent — what it accomplishes, not how. A "logical block" is identified by blank-line boundaries: any contiguous group of lines bounded above and below by blank lines (or by the start/end of a routine) constitutes a block requiring an intent comment. If a chunk of code has no comment above it, it violates this standard — no exceptions. The comment should be short (one line) and name the purpose of the block — do not restate what the code does, do not explain the obvious consequence, and do not add reasoning that the reader can infer from context. A good intent comment could replace the block with a single sentence summary if the code were removed. If a routine has no block without an intent comment above it, you are doing it right.

When in doubt about whether two groups of statements are one block or two, ask: can you describe both groups with a single short intent? If you need the word "and" or "then" to connect two different purposes, they are separate blocks requiring separate comments.

**Justification:** Even well-named variables and clear code require reading to understand. Intent comments at the block level serve as a table of contents within a routine, enabling a reader to jump directly to the section they care about. They also create a contract: if the code does not match the stated intent, either the code or the comment is wrong — both cases reveal bugs.

**Example:**
```
// BAD: No block-level comments — must read every line to find what you need
function processMonthlyBilling(accounts):
    activeAccounts = filter(accounts, account -> account.isActive)
    for account in activeAccounts:
        charges = calculateCharges(account)
        invoice = createInvoice(account, charges)
        sendInvoice(account.email, invoice)
        if account.balance < 0:
            applyLateFee(account)
            flagForReview(account)

// BAD: Over-explains the obvious consequence
// Return focus to Terminal so the script can continue executing subsequent commands
osascript -e 'tell application "Terminal" to activate'

// GOOD: Names the purpose without over-explaining
// Return focus to Terminal
osascript -e 'tell application "Terminal" to activate'

// GOOD: Intent comments enable scanning and verify code matches purpose
function processMonthlyBilling(accounts):
    // Filter to only billable accounts
    activeAccounts = filter(accounts, account -> account.isActive)

    // Process billing for each active account
    for account in activeAccounts:
        // Generate and deliver the monthly invoice
        charges = calculateCharges(account)
        invoice = createInvoice(account, charges)
        sendInvoice(account.email, invoice)

        // Flag delinquent accounts for collections review
        if account.balance < 0:
            applyLateFee(account)
            flagForReview(account)
```

### 5.5 Use Routine Names as Documentation

**Standard:** A routine's name should describe everything it does. If you cannot name a routine concisely, it is doing too much. Function names should be verb phrases for procedures and noun/adjective phrases for functions that return values.

**Justification:** The routine name is the contract. A name like `processData` tells the caller nothing; `calculateMonthlyRevenue` tells them exactly what they get. Good names eliminate the need to read the implementation to understand usage.

**Example:**
```
// BAD: Vague names
function handle(data)
function doWork(input)
function process(x)
function manage(items)

// GOOD: Names describe the contract
function calculateShippingCost(order, destination)
function isEligibleForRefund(transaction)
function formatAsCSV(records)
function sendPasswordResetEmail(user)
```

### 5.6 Document Function Intent, Inputs, Output, and Exceptions

**Standard:** Every function must have a documentation comment that describes: (1) a high-level summary of what the function does, (2) each input parameter with its name, type, and purpose, (3) what the function returns — only if it actually returns a value (omit the `@returns` tag entirely for void/side-effect-only functions that return nothing), and (4) any exceptions or errors the function can throw. Use the idiomatic documentation format for the language (e.g., JSDoc for TypeScript/JavaScript, docstrings with Args/Returns/Raises for Python, Javadoc with @param/@return/@throws for Java, `///` for Rust, doc comments for Go).

**Justification:** A function name conveys the "what" at a glance, but callers need to know the full contract: what arguments to provide, what constraints exist on those arguments, what they get back, and what can go wrong. Without this, callers must read the implementation to use the function correctly — which defeats encapsulation. Documenting inputs, outputs, and exceptions at the function boundary ensures the function can be understood, used, and tested without reading its body.

**Example:**
```
// BAD: No documentation — caller must read the body to understand usage
function calculateShippingCost(order, destination, expedited):
    ...

// GOOD: Full contract documented using the language's idiomatic format
/**
 * Calculates the total shipping cost for an order based on destination
 * and delivery speed.
 *
 * @param order - The order containing items and their weights
 * @param destination - The delivery address (must include country and postal code)
 * @param expedited - Whether to use expedited (2-day) shipping rates
 * @returns The total shipping cost, never negative
 * @throws {InvalidAddressError} If destination is missing country or postal code
 * @throws {UnsupportedRegionError} If destination country is not in the shipping table
 */
function calculateShippingCost(order: Order, destination: Address, expedited: boolean): Money {
    ...
}
```

### 5.7 Use Naming Conventions Consistently

**Standard:** Choose a naming convention for each element type (variables, functions, classes, constants) and apply it consistently throughout the project. Follow the dominant convention of the language ecosystem. Constants whose value is known at write time (compile-time constants, fixed configuration values, magic number replacements) must use UPPER_SNAKE_CASE. Runtime constants whose value is computed or derived from input at execution time use the same casing as regular variables (e.g., camelCase or snake_case depending on language).

**Justification:** Inconsistent naming creates unnecessary cognitive overhead. When conventions are consistent, the name's form tells you what kind of thing it is (constant, class, function) without requiring you to look it up. The UPPER_SNAKE_CASE distinction for fixed constants signals to the reader "this value never changes and was decided by the author," while regular casing signals "this value is fixed for this execution but depends on runtime input."

**Example:**
```
// BAD: Inconsistent conventions
class order_processor:      // Python class should be PascalCase
    MaxRetries = 5          // constant mixed with variable style
    def CalculateTotal():   // method should be snake_case in Python
        pass

// BAD: Fixed constant using variable casing
local expected_argument_count=3
local max_retries=5

// BAD: Runtime constant using UPPER_SNAKE_CASE
TOTAL_WITH_TAX = subtotal * taxRate

// GOOD: Consistent with ecosystem
class OrderProcessor:
    MAX_RETRIES = 5
    def calculate_total(self):
        pass

// GOOD: Fixed constants in UPPER_SNAKE_CASE
local EXPECTED_ARGUMENT_COUNT=3
local MAX_RETRIES=5
final int SECONDS_PER_DAY = 86400;

// GOOD: Runtime constants in regular variable casing
local total_with_tax=$((subtotal * tax_rate))
final double discountedPrice = basePrice * (1 - discountRate);
```

---

## 6. Refactoring & Quality

### 6.1 Refactor When You See Duplication

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

### 6.2 Refactor When a Routine is Too Long

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

### 6.3 Refactor When Naming is Difficult

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

### 6.4 Fix Broken Windows Immediately

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

### 6.5 Leave Code Better Than You Found It (Boy Scout Rule)

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

---

## 7. Testing

### 7.1 Write Tests That Verify Intent, Not Implementation

**Standard:** Tests should assert *what* the code is supposed to accomplish, not *how* it accomplishes it. Tests should not break when internal implementation details change but external behavior stays the same.

**Justification:** Implementation-coupled tests create a maintenance burden that actively discourages refactoring. They test the "how" twice (once in the code, once in the test) without adding confidence that the system works correctly.

**Example:**
```
// BAD: Tests implementation details
function testCalculateDiscount():
    calculator = DiscountCalculator()
    // Testing that specific internal methods are called in order
    assert calculator.internalLookupTable.size == 5
    assert calculator.appliedRuleNames == ["RULE_A", "RULE_B"]

// GOOD: Tests observable behavior
function testCalculateDiscount():
    calculator = DiscountCalculator()
    result = calculator.calculate(orderOf(150.00), goldCustomer())
    assert result.discountAmount == 22.50
    assert result.appliedDiscount == "15% Gold Member"
```

### 7.2 Test Boundary Conditions

**Standard:** For every routine, identify and explicitly test the boundary conditions: empty inputs, single-element inputs, maximum values, off-by-one boundaries, null/nil, duplicates, and type edges.

**Justification:** The majority of bugs live at boundaries — empty lists, zero values, max-length strings, first/last elements. Code that works for "normal" inputs often fails at edges. Boundary tests catch the bugs that matter most.

**Example:**
```
// GOOD: Testing boundaries of a pagination function
function testPaginate():
    // Empty input
    assert paginate([], pageSize=10) == []

    // Exactly one page
    assert paginate([1,2,3], pageSize=3) == [[1,2,3]]

    // One more than a page
    assert paginate([1,2,3,4], pageSize=3) == [[1,2,3], [4]]

    // Page size of 1
    assert paginate([1,2], pageSize=1) == [[1], [2]]

    // Single item
    assert paginate([1], pageSize=10) == [[1]]
```

### 7.3 Make Each Test Independent

**Standard:** Each test must be able to run in isolation, in any order, and produce the same result. Tests must not depend on shared mutable state, execution order, or side effects from other tests.

**Justification:** Interdependent tests are fragile, produce non-deterministic results, and make failures misleading. When test B fails only after test A runs, you waste time debugging test interactions instead of product code.

**Example:**
```
// BAD: Tests depend on shared state
sharedUser = null

function testCreateUser():
    sharedUser = createUser("Alice")
    assert sharedUser.name == "Alice"

function testDeleteUser():
    deleteUser(sharedUser)  // fails if testCreateUser didn't run first
    assert sharedUser.isDeleted

// GOOD: Each test is self-contained
function testCreateUser():
    user = createUser("Alice")
    assert user.name == "Alice"

function testDeleteUser():
    user = createUser("Bob")  // creates its own test data
    deleteUser(user)
    assert user.isDeleted
```

### 7.4 Test One Thing Per Test

**Standard:** Each test should verify a single behavior or scenario. If a test fails, it should be immediately obvious which behavior is broken without reading the test body.

**Justification:** Multi-assertion tests that verify several behaviors at once provide poor failure diagnostics. When they fail, you must read the test to understand which of the several things went wrong. Single-behavior tests pinpoint failures instantly.

**Example:**
```
// BAD: Multiple behaviors in one test
function testUserRegistration():
    user = register("alice@test.com", "password123")
    assert user.email == "alice@test.com"
    assert user.isActive == true
    assert user.welcomeEmailSent == true
    assert user.defaultRole == "member"
    assert user.createdAt != null

// GOOD: One behavior per test
function testRegistrationSetsEmail():
    user = register("alice@test.com", "password123")
    assert user.email == "alice@test.com"

function testRegistrationActivatesUser():
    user = register("alice@test.com", "password123")
    assert user.isActive == true

function testRegistrationSendsWelcomeEmail():
    user = register("alice@test.com", "password123")
    assert user.welcomeEmailSent == true

function testRegistrationAssignsDefaultRole():
    user = register("alice@test.com", "password123")
    assert user.defaultRole == "member"
```

### 7.5 Name Tests as Specifications

**Standard:** Test names should read as a specification of behavior: what scenario is being tested and what the expected outcome is. A reader should understand what the system does just by reading the test names.

**Justification:** Test names are the living documentation of your system's behavior. When tests read as specifications, you can understand the system's contract by reading the test list alone — without reading any implementation code.

**Example:**
```
// BAD: Vague test names
function test1()
function testProcess()
function testEdgeCase()
function testBug123()

// GOOD: Specification-style names
function testEmptyCartReturnsZeroTotal()
function testExpiredCouponIsRejectedWithMessage()
function testOversizedOrderTriggersManualReview()
function testConcurrentWithdrawalsDoNotOverdraft()
```

---

## 8. Data Structures & Table-Driven Methods

### 8.1 Replace Complex Logic with Table Lookups

**Standard:** When you have complex conditional logic that maps inputs to outputs (especially chains of if/else or switch/case with many branches), replace it with a data-driven table lookup. The table encodes the decision; the code simply indexes into it.

**Justification:** Table-driven methods separate data from logic. Adding a new case means adding a row to a table, not editing a control structure. This eliminates the risk of introducing bugs in branching logic and makes the mapping explicitly visible and auditable.

**Example:**
```
// BAD: Long conditional chain
function getShippingRate(region, weight):
    if region == "US":
        if weight < 5:
            return 5.99
        elif weight < 20:
            return 12.99
        else:
            return 24.99
    elif region == "EU":
        if weight < 5:
            return 8.99
        elif weight < 20:
            return 18.99
        else:
            return 34.99
    // ... more regions ...

// GOOD: Table-driven
SHIPPING_RATES = {
    "US": {5: 5.99, 20: 12.99, MAX: 24.99},
    "EU": {5: 8.99, 20: 18.99, MAX: 34.99},
    "APAC": {5: 12.99, 20: 24.99, MAX: 44.99},
}

function getShippingRate(region, weight):
    regionRates = SHIPPING_RATES[region]
    for threshold, rate in sorted(regionRates.items()):
        const boolean isUnderThreshold = (weight < threshold);
        if isUnderThreshold:
            return rate
    const double maxRate = regionRates[MAX];
    return maxRate
```

### 8.2 Choose Data Structures Deliberately

**Standard:** Select data structures based on the actual access patterns of your code. Consider insertion frequency, lookup frequency, ordering requirements, and memory constraints. Do not default to the most familiar structure — choose the one that matches the usage profile.

**Justification:** The wrong data structure forces workarounds that add complexity and harm performance. A list used for frequent lookups by key, a map used where ordering matters, or a flat array used for frequent insertions in the middle all result in code that fights its own foundation.

**Example:**
```
// BAD: Using a list for frequent lookups by ID
function findUser(users, userId):
    for user in users:  // O(n) every time
        if user.id == userId:
            return user
    return null

// Called in a loop:
for orderId in orderIds:
    user = findUser(allUsers, orders[orderId].userId)  // O(n*m)

// GOOD: Choose structure matching access pattern
usersByID = buildIndex(allUsers, key=user.id)  // O(n) once

for orderId in orderIds:
    user = usersByID[orders[orderId].userId]  // O(1) per lookup
```

### 8.3 Encapsulate Complex Data Structures

**Standard:** When a data structure has non-trivial access rules or invariants, wrap it in a class or module that enforces those rules. Do not expose raw structures (nested maps, arrays of arrays) to consuming code.

**Justification:** Raw complex structures spread knowledge of their shape throughout the codebase. When the structure changes, every access point breaks. Encapsulation creates a single place to enforce invariants and a single place to update when the structure evolves.

**Example:**
```
// BAD: Raw nested structure used directly everywhere
schedule = {monday: [{start: 9, end: 12}, {start: 13, end: 17}], ...}

// Consumers must know the shape:
mondaySlots = schedule["monday"]
for slot in mondaySlots:
    if slot["start"] <= currentHour < slot["end"]:
        return true

// GOOD: Encapsulated behind meaningful operations
class Schedule:
    private slots: Map<Day, List<TimeSlot>>

    function isAvailableAt(day: Day, hour: int) -> bool:
        final boolean hasMatchingSlot = (any(slot.contains(hour) for slot in slots[day]));
        return hasMatchingSlot

    function addSlot(day: Day, start: int, end: int):
        final boolean isValidRange = (start < end);
        if (!isValidRange):
            throw IllegalArgumentException("Invalid time range")
        slots[day].append(TimeSlot(start, end))
```

---

## 9. Routine Parameters

### 9.1 Limit Parameter Count

**Standard:** Routines should accept no more than 7 parameters. If a routine needs more, group related parameters into a cohesive object or structure. This limit applies to the conceptual interface — splitting parameters across multiple setter calls to dodge the limit violates the spirit of this standard.

**Justification:** Human working memory holds approximately 7 items. Routines with many parameters are difficult to call correctly (easy to swap arguments), difficult to read at the call site, and signal that the routine may have too many responsibilities.

**Example:**
```
// BAD: Too many parameters — easy to get wrong at call site
function createUser(firstName, lastName, email, phone, street, city,
                    state, zip, country, role, department, manager):
    ...

// Call site is unreadable:
createUser("Alice", "Smith", "a@b.com", "555-1234", "123 Main",
           "Springfield", "IL", "62701", "US", "admin", "eng", "Bob")

// GOOD: Group related parameters
class Address:
    street, city, state, zip, country

class UserRequest:
    firstName, lastName, email, phone
    address: Address
    role, department, manager

function createUser(request: UserRequest):
    ...
```

### 9.2 Use Consistent Parameter Ordering

**Standard:** Establish a consistent parameter ordering convention and follow it across all routines in a module. Common conventions: input parameters first, output parameters last; or primary subject first, modifiers after. Within a group of related functions, the same conceptual parameter should appear in the same position.

**Justification:** Inconsistent ordering forces callers to check documentation for every call. Consistent ordering builds muscle memory and makes incorrect usage visually obvious. It also reduces the chance of accidentally swapping arguments of the same type.

**Example:**
```
// BAD: Inconsistent ordering across related functions
function copyFile(destination, source)
function moveFile(source, destination)
function linkFile(linkName, target, overwrite)
function deleteFile(force, path)

// GOOD: Consistent convention (source/subject first, destination/target second, options last)
function copyFile(source, destination, options)
function moveFile(source, destination, options)
function linkFile(source, linkName, options)
function deleteFile(path, options)
```

### 9.3 Document Parameter Assumptions

**Standard:** When a parameter has constraints that are not obvious from its type (e.g., must be positive, must be non-empty, must be sorted, must be a valid enum value), document these assumptions at the routine boundary — either through type system enforcement, assertions, or a brief comment on the parameter.

**Justification:** Undocumented parameter assumptions are latent bugs. Callers will inevitably pass values that violate unstated constraints, producing subtle failures far from the root cause. Making assumptions explicit turns silent corruption into immediate, debuggable failures.

**Example:**
```
// BAD: Hidden assumptions
function paginate(items, page, pageSize):
    start = page * pageSize
    return items[start:start + pageSize]
// What happens with page=-1? pageSize=0? Empty items?

// GOOD: Assumptions made explicit
function paginate(items: List, page: int, pageSize: int) -> List:
    final boolean isValidPage = (page >= 0);
    if (!isValidPage):
        throw IllegalArgumentException("page must be non-negative")
    final boolean isValidPageSize = (pageSize > 0);
    if (!isValidPageSize):
        throw IllegalArgumentException("pageSize must be positive")
    // items may be empty — returns empty list (valid behavior)
    final int start = (page * pageSize);
    final List pageResults = items[start:(start + pageSize)];
    return pageResults
```

### 9.4 Avoid Flag Parameters

**Standard:** Do not use boolean parameters that switch a routine's behavior between two modes. Instead, create two routines with clear names, or use an enum/named constant if more than two modes exist.

**Justification:** A boolean parameter at the call site (`process(order, true)`) is unreadable — "true" tells the reader nothing. It also indicates the routine has two responsibilities. Separate routines make intent explicit at every call site without requiring the reader to look up what the flag means.

**Example:**
```
// BAD: Flag parameter — what does "true" mean?
render(template, true)
render(template, false)
sendEmail(user, message, true, false)

// GOOD: Separate routines with clear intent
renderWithHeaders(template)
renderBodyOnly(template)

sendEmail(user, message)
sendEmailWithAttachment(user, message, attachment)

// Or use a named option when a parameter object is warranted:
render(template, {includeHeaders: true})
```

---

## 10. Code Layout & Formatting

### 10.1 Use Whitespace to Reveal Logical Structure

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

### 10.2 Keep Lines Readable

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

### 10.3 Put Related Code Together

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

### 10.4 Use Formatting to Expose Errors

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

---

## 11. Sequential Dependencies & Temporal Coupling

### 11.1 Make Order Dependencies Visible in Code Structure

**Standard:** When operations must execute in a specific sequence, make the dependency enforced by the code structure — not by comments, documentation, or convention. Use return values that feed into subsequent calls, builder patterns, or type state to make it impossible to call things out of order.

**Justification:** If correctness depends on calling A before B, someone will eventually call B first. Comments like "must call init() before process()" are invisible at the call site. Structural enforcement makes the compiler or runtime catch ordering violations, not production incidents.

**Example:**
```
// BAD: Order dependency enforced only by convention
connection = createConnection()
connection.setHost("db.example.com")
connection.setPort(5432)
connection.authenticate("user", "pass")
connection.connect()  // must call after authenticate — but nothing prevents reordering

// GOOD: Return values enforce the sequence
const ConnectionConfig config = ConnectionConfig("db.example.com", 5432)
const AuthenticatedConfig authenticated = authenticate(config, "user", "pass")
const Connection connection = connect(authenticated)  // can't connect without an AuthenticatedConfig
```

### 11.2 Organize Straight-Line Code to Show Dependencies

**Standard:** When statements must execute in a specific order, structure the code so the dependency is obvious: use the result of one statement as the input to the next. When statements have no dependency, group them by logical affinity rather than by an arbitrary sequence.

**Justification:** A reader seeing a sequence of independent statements assumes they can be reordered safely. If reordering would break things, the code must make that visible — either through data flow or by extracting the sequence into a named routine that encapsulates the required order.

**Example:**
```
// BAD: Hidden dependency — revenue must be computed before tax
computeRevenue(report)
computeTax(report)      // silently reads report.revenue — breaks if reordered
computeTotal(report)    // silently reads report.tax

// GOOD: Data flow makes the dependency explicit
revenue = computeRevenue(report)
tax = computeTax(revenue)
total = computeTotal(revenue, tax)
```

### 11.3 Encapsulate Multi-Step Sequences

**Standard:** When a sequence of operations must always happen together in a fixed order, wrap them in a single routine. Do not require callers to know or reproduce the sequence.

**Justification:** If three steps must always happen together, exposing them individually means every caller must get the sequence right. A single routine encapsulates the ordering knowledge in one place, and callers simply invoke the operation without knowing the internal steps.

**Example:**
```
// BAD: Every caller must know the required sequence
resource = allocateResource()
resource.initialize()
resource.validate()
resource.register(registry)
// If any caller forgets validate(), subtle bugs ensue

// GOOD: Sequence encapsulated
function provisionResource(registry):
    resource = allocateResource()
    resource.initialize()
    resource.validate()
    resource.register(registry)
    return resource
```

---

## 12. Defensive Data Handling

### 12.1 Prefer Immutable Data by Default

**Standard:** Treat data as immutable unless mutation is explicitly required by the design. Use copies or new instances rather than modifying data that was passed in. When mutation is necessary, make it obvious and contained.

**Justification:** Shared mutable state is the root cause of an entire class of bugs: unexpected side effects, stale references, race conditions, and action-at-a-distance. Immutable data eliminates these by making data flow explicit — you can always trust that a value hasn't been changed behind your back.

**Example:**
```
// BAD: Mutates the input — caller's data is modified as a side effect
function applyDiscount(order, discountPercent):
    order.total = order.total * (1 - discountPercent)  // modifies caller's object
    return order

// GOOD: Returns a new value, input is untouched
function applyDiscount(order, discountPercent):
    discountedTotal = order.total * (1 - discountPercent)
    Order discountedOrder = order.withTotal(discountedTotal)  // creates new instance
    return discountedOrder
```

### 12.2 Make Copies at Trust Boundaries

**Standard:** When receiving mutable data from an external caller or returning internal mutable data to an external caller, create a defensive copy. This prevents external code from mutating your internal state or vice versa.

**Justification:** Without defensive copies, you lose control of your own state. A caller can retain a reference to your internal data and modify it after passing it to you, violating invariants you thought were protected. Copies at boundaries guarantee that each module owns its data.

**Example:**
```
// BAD: Stores a reference to mutable input — caller can change it later
class Schedule:
    function setSlots(slots):
        this.slots = slots  // external code still holds a reference

// Later, caller modifies the list:
mySlots.append(badSlot)  // Schedule's internal state is corrupted

// GOOD: Defensive copy on input
class Schedule:
    function setSlots(slots):
        this.slots = copy(slots)  // our own copy — caller can't affect it

// GOOD: Defensive copy on output
class Schedule:
    function getSlots():
        Slots slots = copy(this.slots)  // return a copy — caller can't affect our internal state
        return slots
```

### 12.3 Avoid Aliasing of Mutable Data

**Standard:** Do not create multiple references to the same mutable data structure unless the aliasing is intentional and documented. When two variables point to the same mutable object, modification through one alias silently affects the other.

**Justification:** Aliasing bugs are notoriously difficult to track down because the code that causes the problem and the code that observes the symptom are in entirely different locations. Avoiding aliases eliminates this category of bugs at the source.

**Example:**
```
// BAD: Unintentional aliasing
defaultConfig = {timeout: 30, retries: 3}
userConfig = defaultConfig  // alias, not a copy!
userConfig.timeout = 60     // oops — defaultConfig is now {timeout: 60, retries: 3}

// GOOD: Explicit copy prevents aliasing
defaultConfig = {timeout: 30, retries: 3}
userConfig = copy(defaultConfig)
userConfig.timeout = 60  // defaultConfig is unaffected
```

---

## Summary of Application

When writing or modifying code, apply these standards in this priority order:

1. **Correctness** — Code must do what it claims (testing, boundary validation)
2. **Clarity** — Code must be understandable without author context (naming, self-documentation)
3. **Simplicity** — Prefer the simplest solution that works (complexity management, single responsibility)
4. **Robustness** — Code must handle failure gracefully (defensive programming, fail fast)
5. **Maintainability** — Code must be easy to change (loose coupling, information hiding, refactoring)

These standards are not suggestions — they are enforced. Deviations require explicit justification from the user.

---

## Self-Review Requirement

After writing or modifying code, you MUST perform a complete self-review pass before considering the work done. Re-read every line of the code you produced and verify it against every standard in this document. Do not rely on having applied the standards correctly during the initial writing pass — treat the self-review as an independent audit. If the self-review reveals any violation, fix it immediately before presenting the work as complete.

After the general self-review, perform a second line-by-line audit. Check each line individually rather than reading for flow. For every line, ask: does this line comply with every applicable standard on its own? Reading for flow causes you to mentally group lines and skip over individual violations. The line-by-line pass forces you to evaluate each line in isolation.
