## Variables & Data

### VAR-1: Initialize Variables Close to First Use

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

### VAR-2: Minimize Variable Scope

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

### VAR-3: Use Meaningful, Precise Names

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

### VAR-4: Name Booleans in the Positive Form

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

### VAR-5: Use Named Constants Instead of Magic Numbers

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

### VAR-6: One Purpose Per Variable

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

### VAR-7: Always Use Explicit Type Declarations

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

### VAR-8: Parenthesize All Math Expressions

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

### VAR-9: Never Return Expressions Directly

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
