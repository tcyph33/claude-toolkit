## Data Structures & Table-Driven Methods

### STR-1 Replace Complex Logic with Table Lookups

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

### STR-2 Choose Data Structures Deliberately

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

### STR-3 Encapsulate Complex Data Structures

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
