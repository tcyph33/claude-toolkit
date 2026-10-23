## Defensive Programming

### DFP-1 Validate Data at System Boundaries

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

### DFP-2 Handle Errors Deliberately — Don't Swallow or Ignore

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

### DFP-3 Fail Fast

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

### DFP-4 Define Boundaries for Acceptable Input (Barricades)

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
