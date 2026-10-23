## Control Flow

### CTL-1 Put the Normal Case First

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

### CTL-2 Extract Conditionals into Named Boolean Variables

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

### CTL-3 Simplify Complex Boolean Expressions

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

### CTL-4 Use Loop Variables Correctly

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

### CTL-5 Avoid Deep Nesting with Early Exit

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

### CTL-6 Use Structured Control Flow

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
