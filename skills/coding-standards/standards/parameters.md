## Routine Parameters

### PRM-1 Limit Parameter Count

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

### PRM-2 Use Consistent Parameter Ordering

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

### PRM-3 Document Parameter Assumptions

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

### PRM-4 Avoid Flag Parameters

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
