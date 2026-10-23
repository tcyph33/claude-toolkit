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
