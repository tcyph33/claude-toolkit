## Self-Documenting Code

### DOC-1 Make Code Read Without Comments

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

### DOC-2 Comment the Why, Not the What

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

### DOC-3 Comment Surprises and Non-Obvious Side Effects

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

### DOC-4 Comment Each Code Block with High-Level Intent

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

### DOC-5 Use Routine Names as Documentation

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

### DOC-6 Document Function Intent, Inputs, Output, and Exceptions

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

### DOC-7 Use Naming Conventions Consistently

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
