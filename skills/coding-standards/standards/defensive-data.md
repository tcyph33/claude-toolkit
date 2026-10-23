## Defensive Data Handling

### IMM-1: Prefer Immutable Data by Default

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

### IMM-2: Make Copies at Trust Boundaries

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

### IMM-3: Avoid Aliasing of Mutable Data

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
