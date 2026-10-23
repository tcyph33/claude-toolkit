## Sequential Dependencies & Temporal Coupling

### SEQ-1 Make Order Dependencies Visible in Code Structure

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

### SEQ-2 Organize Straight-Line Code to Show Dependencies

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

### SEQ-3 Encapsulate Multi-Step Sequences

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
