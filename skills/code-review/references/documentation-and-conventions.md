# Documentation and Conventions ? OpenEdge ABL, Clean Code, OOABL

Detailed patterns for OpenEdge ABL documentation, clean code conventions,
naming standards, procedural ABL, and Object-Oriented ABL review checks.

This reference is used by the `code-review` skill when reviewing Progress
OpenEdge ABL source code.

---

# Documentation Standards

OpenEdge ABL does not have a single universal documentation standard
equivalent to ProtheusDOC.

Documentation should therefore follow the conventions established by the
project, team, or organization.

When no project-specific standard exists, use structured comment blocks
immediately before the element being documented.

The objective is to document:

- Purpose
- Business responsibility
- Parameters
- Return values
- Exceptions
- Side effects
- Database access
- External dependencies
- Important business rules

Documentation should explain **why** the code exists when the intent is not
obvious from the implementation.

---

## Program Documentation

Every significant `.p` program should contain a file header.

### Recommended Structure

```progress
/*------------------------------------------------------------------------
    File        : process-order.p
    Purpose     : Process customer orders.

    Description :
        Validates and processes customer orders according to the
        application's business rules.

    Parameters  :
        INPUT  pcOrderId - Order identifier.

    Output      :
        None.

    Notes       :
        Updates order and item records.

    Author(s)   : Consultoria Incode
    Created     : 20/07/2026
    Version     : 1.0.0
  ----------------------------------------------------------------------*/
````

### Review Checklist

* [ ] File purpose is documented
* [ ] Main responsibility is clear
* [ ] Important business rules are documented
* [ ] Inputs and outputs are documented
* [ ] Important side effects are documented
* [ ] Database updates are documented when relevant
* [ ] External integrations are documented
* [ ] Author and creation information exist when required by project standards

---

# Internal Procedure Documentation

Important internal procedures should have a documentation block.

### Recommended

```progress
/*------------------------------------------------------------------------
    Procedure : ProcessOrder
    Purpose   : Validate and process an order.
    Parameters:
        INPUT  pcOrderId - Order identifier.
    Notes:
        Updates order status and related items.
  ----------------------------------------------------------------------*/
PROCEDURE ProcessOrder:

    DEFINE INPUT PARAMETER pcOrderId AS CHARACTER NO-UNDO.

    /* Processing */

END PROCEDURE.
```

Documentation is especially recommended when the procedure:

* Performs database updates
* Starts transactions
* Calls external services
* Has complex business rules
* Is part of a public procedural API
* Is called by multiple programs

---

# Function Documentation

Functions with non-trivial logic should be documented.

### Recommended

```progress
/*------------------------------------------------------------------------
    Function  : CalculateTotal
    Purpose   : Calculates the total order value after applying a discount.
    Parameters:
        INPUT pcCustomer - Customer identifier.
        INPUT deDiscount - Discount percentage.
    Returns   :
        DECIMAL - Calculated order total.
  ----------------------------------------------------------------------*/
FUNCTION CalculateTotal RETURNS DECIMAL
    (INPUT pcCustomer AS CHARACTER,
     INPUT deDiscount AS DECIMAL):

    RETURN 0.

END FUNCTION.
```

---

# Class Documentation

Every public class should have a class-level documentation block.

### Recommended

```progress
/*------------------------------------------------------------------------
    Class     : CustomerService
    Purpose   : Provides customer-related business operations.
    Responsibilities:
        - Customer lookup
        - Customer validation
        - Customer status management
    Dependencies:
        - Customer database table
  ----------------------------------------------------------------------*/
CLASS CustomerService:

END CLASS.
```

---

# Method Documentation

Public methods should be documented.

Private methods should be documented when their purpose or behavior is not
obvious.

### Recommended

```progress
/*------------------------------------------------------------------------
    Method    : FindCustomer
    Purpose   : Retrieves a customer by identifier.
    Parameters:
        INPUT piCustomer - Customer identifier.
    Returns   :
        Customer - Customer object when found.
    Errors    :
        Throws an error when the customer cannot be retrieved.
  ----------------------------------------------------------------------*/
METHOD PUBLIC FindCustomer
    (INPUT piCustomer AS INTEGER):

    /* Implementation */

END METHOD.
```

---

# Documentation Checklist

Review the following:

* [ ] Significant `.p` programs have a file header
* [ ] Public classes have class documentation
* [ ] Public methods are documented
* [ ] Important internal procedures are documented
* [ ] Non-trivial functions are documented
* [ ] Parameters are documented when necessary
* [ ] Return values are documented
* [ ] Exceptions are documented
* [ ] Database side effects are documented
* [ ] External dependencies are documented
* [ ] Important business rules are documented
* [ ] Documentation matches the actual implementation
* [ ] No obsolete or misleading documentation exists

---

# Documentation Anti-Patterns

## Obvious Comments

### BAD

```progress
/* Increment counter */
iCounter = iCounter + 1.
```

The comment adds no useful information.

### GOOD

```progress
/* Retry counter is limited to three attempts to avoid repeated
   calls to the external service. */
iRetryCount = iRetryCount + 1.
```

Comments should explain intent when the intent is not obvious.

---

## Outdated Comments

### BAD

```progress
/* Updates customer status to ACTIVE */
customer.status = "PENDING".
```

Documentation must reflect the actual behavior.

---

## Commented-Out Code

### BAD

```progress
/*
FIND FIRST customer NO-LOCK
    WHERE customer.cust-num = piCustomer
    NO-ERROR.
*/

/* New implementation */
```

Remove obsolete code instead of leaving it commented out.

Version control should preserve historical implementations.

---

# Clean Code and Naming Conventions

Naming conventions should prioritize clarity and consistency.

If an existing project has an established naming standard, follow the project
standard instead of imposing a new convention.

---

## Variable Naming

Recommended prefixes:

| Prefix | Type        | Example       |
| ------ | ----------- | ------------- |
| `c`    | CHARACTER   | `cName`       |
| `i`    | INTEGER     | `iCount`      |
| `int`  | INTEGER     | `intCustomer` |
| `de`   | DECIMAL     | `deTotal`     |
| `d`    | DATE        | `dDueDate`    |
| `dt`   | DATETIME    | `dtCreated`   |
| `dtz`  | DATETIME-TZ | `dtzUpdated`  |
| `l`    | LOGICAL     | `lActive`     |
| `h`    | HANDLE      | `hQuery`      |
| `o`    | OBJECT      | `oService`    |
| `tt`   | TEMP-TABLE  | `ttCustomer`  |
| `ds`   | DATASET     | `dsCustomer`  |
| `b`    | BUFFER      | `bCustomer`   |
| `q`    | QUERY       | `qCustomer`   |

### Example

```progress
DEFINE VARIABLE cCustomerName AS CHARACTER NO-UNDO.
DEFINE VARIABLE iCustomerId AS INTEGER NO-UNDO.
DEFINE VARIABLE deTotal AS DECIMAL NO-UNDO.
DEFINE VARIABLE lProcessed AS LOGICAL NO-UNDO.
DEFINE VARIABLE hQuery AS HANDLE NO-UNDO.
```

Follow existing project conventions when they differ.

Do not flag naming prefixes as defects when the project consistently uses a
different standard.

---

# Strong Typing

Prefer explicit data types.

### BAD

Use ambiguous or weakly defined structures when strong typing is possible.

### GOOD

```progress
DEFINE VARIABLE cName AS CHARACTER NO-UNDO.
DEFINE VARIABLE iCount AS INTEGER NO-UNDO.
DEFINE VARIABLE deAmount AS DECIMAL NO-UNDO.
DEFINE VARIABLE lActive AS LOGICAL NO-UNDO.
```

For parameters:

```progress
DEFINE INPUT PARAMETER pcCustomer AS CHARACTER NO-UNDO.
DEFINE INPUT PARAMETER piCustomer AS INTEGER NO-UNDO.
```

For methods:

```progress
METHOD PUBLIC CalculateTotal RETURNS DECIMAL
    (INPUT pcCustomer AS CHARACTER).
```

Flag missing or ambiguous typing when it materially reduces readability,
compile-time validation, or maintainability.

---

# NO-UNDO Convention

Variables and Temp-Tables should normally use `NO-UNDO`.

### Recommended

```progress
DEFINE VARIABLE cName AS CHARACTER NO-UNDO.

DEFINE TEMP-TABLE tt-item NO-UNDO
    FIELD it-codigo AS CHARACTER.
```

Review exceptions when transaction rollback semantics are intentionally
required.

Do not flag missing `NO-UNDO` automatically if the variable explicitly needs
to participate in transaction undo processing.

---

# Variable Scope

Prefer the narrowest possible scope.

### Preferred

```progress
PROCEDURE ProcessOrder:

    DEFINE VARIABLE cOrderId AS CHARACTER NO-UNDO.

    /* cOrderId only exists within this procedure */

END PROCEDURE.
```

Avoid unnecessary:

* Global variables
* Shared variables
* Session-level state
* Persistent mutable state

Prefer explicit parameter passing.

### Recommended

```progress
RUN ProcessOrder (
    INPUT cOrderId
).
```

instead of relying on hidden shared state.

---

# Global and Shared Variables

Global and shared variables are not automatically defects.

Review their use for:

* Hidden dependencies
* Mutable shared state
* Difficult testing
* Unclear lifecycle
* Unexpected side effects
* Tight coupling

Flag as `MINOR` or `MAJOR` when shared state materially harms maintainability
or correctness.

Do not flag legitimate framework or legacy architecture without explaining
the impact.

---

# Function and Procedure Size

There should be no absolute universal line limit.

Instead, evaluate:

* Number of responsibilities
* Cyclomatic complexity
* Nesting depth
* Number of parameters
* Database operations
* Error-handling branches
* External dependencies

A large procedure may be acceptable if it represents one cohesive operation.

A small procedure may still be problematic if it performs multiple unrelated
responsibilities.

---

# Single Responsibility

A procedure, function, or method should have a clear primary responsibility.

### Potentially Problematic

```progress
PROCEDURE ProcessOrder:

    /* Validate input */

    /* Query database */

    /* Calculate prices */

    /* Call REST API */

    /* Generate file */

    /* Send email */

END PROCEDURE.
```

Consider separating responsibilities when the implementation becomes
difficult to understand, test, or maintain.

---

# Magic Numbers and Strings

Avoid unexplained business values.

### BAD

```progress
IF iStatus = 3 THEN DO:

    ASSIGN
        deDiscount = deTotal * 0.15.

END.
```

### GOOD

```progress
DEFINE VARIABLE iStatusApproved AS INTEGER NO-UNDO INITIAL 3.
DEFINE VARIABLE deDiscountRate AS DECIMAL NO-UNDO INITIAL 0.15.

IF iStatus = iStatusApproved THEN DO:

    ASSIGN
        deDiscount = deTotal * deDiscountRate.

END.
```

When appropriate, use constants or centralized configuration.

Do not extract trivial values merely for the sake of creating constants.

---

# Dead Code

Flag:

* Unused variables
* Unreachable code
* Unused parameters
* Unused procedures
* Unused functions
* Commented-out code
* Obsolete branches

### BAD

```progress
DEFINE VARIABLE cUnused AS CHARACTER NO-UNDO.

/* Never used */
```

Remove dead code unless it exists for an intentional framework contract.

---

# Deep Nesting

Avoid excessive nested conditions.

### Potentially Problematic

```progress
IF lValid THEN DO:

    IF AVAILABLE customer THEN DO:

        IF customer.active THEN DO:

            IF customer.balance > 0 THEN DO:

                /* Business logic */

            END.

        END.

    END.

END.
```

Consider guard clauses or extracting business logic.

### Alternative

```progress
IF NOT lValid THEN
    RETURN.

IF NOT AVAILABLE customer THEN
    RETURN.

IF NOT customer.active THEN
    RETURN.

IF customer.balance <= 0 THEN
    RETURN.

/* Business logic */
```

Use this approach when it improves readability and is compatible with the
procedure's control flow.

---

# Duplicate Logic

Identify repeated business logic that can lead to inconsistent behavior.

Potential solutions:

* Internal procedures
* Functions
* Classes
* Services
* Shared business components

Do not abstract code merely because two blocks look superficially similar.

The duplicated logic must represent the same business responsibility.

---

# Error Handling Conventions

Error handling should follow the architecture of the application.

For procedural ABL:

```progress
DO ON ERROR UNDO, THROW:

    /* Processing */

END.
```

For Object-Oriented ABL:

```progress
DO ON ERROR UNDO, THROW:

    /* Processing */

END.

CATCH oError AS Progress.Lang.Error:

    /* Error handling */

END CATCH.
```

Review whether:

* Errors are propagated correctly
* Errors retain useful context
* `NO-ERROR` is checked
* Exceptions are not silently ignored
* Cleanup occurs after failures

---

# OOABL Conventions

When reviewing `.cls` files, verify:

* Explicit access modifiers
* Encapsulation
* Strong typing
* Constructor design
* Interface usage
* Inheritance
* Method visibility
* Property visibility
* Static members
* Dependency management

---

## Access Modifiers

Public APIs should be intentionally exposed.

### Recommended

```progress
CLASS CustomerService:

    DEFINE PRIVATE VARIABLE cConnection AS CHARACTER NO-UNDO.

    METHOD PUBLIC ProcessCustomer():
        /* Implementation */
    END METHOD.

END CLASS.
```

Avoid exposing internal implementation details as public members.

---

# Class Responsibilities

A class should have a clear responsibility.

### Potentially Problematic

```progress
CLASS OrderService:

    /* Database access */
    /* REST communication */
    /* File generation */
    /* Email sending */
    /* UI interaction */
    /* Business rules */

END CLASS.
```

Consider separating responsibilities when coupling becomes excessive.

---

# Constructor Design

Constructors should establish valid object state.

Review for:

* Excessive parameters
* External network calls
* Database transactions
* Long-running operations
* Complex business logic

Avoid performing expensive operations in constructors unless the architecture
explicitly requires it.

---

# Interfaces

Interfaces should represent contracts rather than implementation details.

### Recommended

```progress
INTERFACE ICustomerService:

    METHOD ProcessCustomer
        (INPUT piCustomer AS INTEGER).

END INTERFACE.
```

Review for:

* Excessively large interfaces
* Interfaces with unrelated responsibilities
* Contracts that expose internal implementation details

---

# Inheritance

Review inheritance carefully.

Flag:

* Deep inheritance chains
* Inappropriate inheritance
* Inheritance used only for code reuse
* Base classes with excessive responsibilities

Prefer composition when it better represents the relationship.

Do not automatically flag inheritance as a problem.

---

# Method Declaration and Implementation

Ensure that class methods are declared and implemented according to valid
OOABL syntax.

Review:

* Method visibility
* Return type
* Parameter types
* Parameter direction
* Method name consistency
* Declaration/implementation consistency

Example:

```progress
CLASS CustomerService:

    METHOD PUBLIC ProcessCustomer
        (INPUT piCustomer AS INTEGER)
        RETURNS LOGICAL.

END CLASS.
```

The implementation must match the declared contract.

---

# Properties

Review properties for appropriate visibility.

### Potentially Problematic

```progress
DEFINE PUBLIC PROPERTY CustomerName AS CHARACTER
    GET.
    SET.
```

If unrestricted mutation is not required, consider controlled mutation.

Prefer encapsulation where business invariants must be maintained.

---

# Static State

Review static variables and methods for:

* Global mutable state
* Thread/concurrency concerns
* Hidden dependencies
* Difficult testing
* Unexpected shared state

Static members are not inherently incorrect.

Flag only when the shared state introduces a meaningful design or runtime
risk.

---

# Procedural ABL Conventions

For `.p` programs, review:

* Clear program structure
* Definitions separated from logic
* Internal procedures
* Internal functions
* Parameter passing
* Transaction boundaries
* Database access
* Error handling

A recommended high-level structure is:

```progress
/* Header */

/* USING */

/* Definitions */

/* Temp-Tables */

/* Variables */

/* Buffers */

/* Main execution */

/* Internal Functions */

/* Internal Procedures */
```

The exact order may follow the project's existing conventions.

---

# Include Files

Review `.i` include files for:

* Hidden variable definitions
* Hidden business logic
* Global state
* Duplicate definitions
* Excessive nesting
* Unclear dependencies

Include files should have a clear purpose.

### Potentially Problematic

```progress
{common.i}
```

where `common.i` contains:

* Variables
* Temp-Tables
* Database buffers
* Business logic
* Procedures
* Global state

Large "everything" includes increase coupling.

Prefer smaller, focused includes when practical.

---

# Naming Consistency

Names should clearly represent their purpose.

### BAD

```progress
DEFINE VARIABLE c1 AS CHARACTER NO-UNDO.
DEFINE VARIABLE x AS CHARACTER NO-UNDO.
DEFINE VARIABLE temp AS CHARACTER NO-UNDO.
```

### GOOD

```progress
DEFINE VARIABLE cCustomerCode AS CHARACTER NO-UNDO.
DEFINE VARIABLE cCustomerName AS CHARACTER NO-UNDO.
DEFINE VARIABLE cOrderStatus AS CHARACTER NO-UNDO.
```

Avoid meaningless names unless they represent intentionally generic
framework concepts.

---

# Boolean and Logical Naming

Logical variables should communicate a boolean state.

### Recommended

```progress
DEFINE VARIABLE lActive AS LOGICAL NO-UNDO.
DEFINE VARIABLE lProcessed AS LOGICAL NO-UNDO.
DEFINE VARIABLE lFound AS LOGICAL NO-UNDO.
DEFINE VARIABLE lHasError AS LOGICAL NO-UNDO.
```

Avoid ambiguous names:

```progress
DEFINE VARIABLE lStatus AS LOGICAL NO-UNDO.
DEFINE VARIABLE lValue AS LOGICAL NO-UNDO.
```

Prefer names that communicate the condition represented.

---

# Parameter Naming

Parameters should use descriptive names and appropriate prefixes.

### Recommended

```progress
DEFINE INPUT PARAMETER pcCustomerCode AS CHARACTER NO-UNDO.
DEFINE INPUT PARAMETER piCustomerId AS INTEGER NO-UNDO.
DEFINE INPUT PARAMETER pdeAmount AS DECIMAL NO-UNDO.
DEFINE OUTPUT PARAMETER plSuccess AS LOGICAL NO-UNDO.
```

Follow project conventions if they differ.

---

# Code Formatting

Review for consistent:

* Indentation
* Keyword casing
* Alignment
* Block structure
* Spacing
* Line breaks

A common convention is:

* ABL keywords in uppercase
* Identifiers using project naming conventions
* Four spaces for indentation

Example:

```progress
FOR EACH customer NO-LOCK
    WHERE customer.country = pcCountry:

    IF customer.active THEN DO:

        DISPLAY
            customer.cust-num
            customer.name.

    END.

END.
```

Formatting issues should generally be `INFO` unless they materially reduce
readability.

---

# Business Rule Documentation

Complex business rules should be documented.

### Example

```progress
/* Orders above the configured credit limit require manual approval.
   The validation must occur before the order status is changed. */
IF deOrderTotal > deCreditLimit THEN
    ASSIGN
        lRequiresApproval = TRUE.
```

Avoid documenting obvious implementation details.

Document the business reason when it is not evident from the code.

---

# Datasul-Specific Documentation

When reviewing TOTVS Datasul code, document relevant dependencies such as:

* Datasul programs
* Persistent procedures
* Super Procedures
* Business Entities
* Standard APIs
* Database tables
* Business rules
* Framework dependencies

Example:

```progress
/*------------------------------------------------------------------------
    Dependency : Standard Datasul customer validation procedure.
    Side Effect: May update customer status.
    Notes      : Must execute within the same transaction as order creation.
  ----------------------------------------------------------------------*/
```

Do not invent Datasul dependencies that cannot be identified from the source.

---

# Documentation Review Severity

| Rule     | Pattern                                   | Severity      |
| -------- | ----------------------------------------- | ------------- |
| DOC-001  | Missing significant program documentation | INFO / MINOR  |
| DOC-002  | Missing public class documentation        | MINOR         |
| DOC-003  | Missing public method documentation       | MINOR         |
| DOC-004  | Missing parameter documentation           | INFO / MINOR  |
| DOC-005  | Missing return documentation              | INFO / MINOR  |
| DOC-006  | Outdated or incorrect documentation       | MINOR         |
| DOC-007  | Missing business rule documentation       | INFO / MINOR  |
| DOC-008  | Missing side-effect documentation         | MINOR         |
| DOC-009  | Commented-out code                        | INFO / MINOR  |
| DOC-010  | Obvious or redundant comments             | INFO          |
| QUAL-001 | Poor naming                               | MINOR         |
| QUAL-002 | Excessive procedure/method complexity     | MINOR / MAJOR |
| QUAL-003 | Excessive global/shared state             | MINOR / MAJOR |
| QUAL-004 | Magic numbers or strings                  | MINOR         |
| QUAL-005 | Dead code                                 | MINOR         |
| QUAL-006 | Deep nesting                              | MINOR         |
| QUAL-007 | Duplicate business logic                  | MINOR / MAJOR |
| QUAL-008 | Excessive class responsibility            | MINOR / MAJOR |
| OO-001   | Missing access control                    | MINOR / MAJOR |
| OO-002   | Excessive inheritance                     | MINOR         |
| OO-003   | Inappropriate inheritance                 | MINOR / MAJOR |
| OO-004   | Excessive static state                    | MINOR / MAJOR |
| OO-005   | Poor interface design                     | MINOR         |
| ABL-001  | Poor parameter design                     | MINOR         |
| ABL-002  | Excessive include coupling                | MINOR / MAJOR |
| ABL-003  | Hidden include dependencies               | MINOR         |
| DS-001   | Missing Datasul dependency documentation  | INFO / MINOR  |

---

# Review Guidance

When applying these rules:

1. Follow project-specific conventions when they exist.
2. Do not impose ProtheusDOC, TLPP, or AdvPL conventions on ABL.
3. Do not require documentation for every trivial line or variable.
4. Prioritize documentation of public APIs and business rules.
5. Consider procedural ABL and OOABL separately.
6. Consider the target OpenEdge version.
7. Consider Datasul architecture when applicable.
8. Do not classify formatting preferences as functional defects.
9. Do not flag legacy code solely because it is old.
10. Explain the maintenance or runtime impact of significant findings.
11. Prefer incremental improvements over unnecessary rewrites.
12. Never invent project conventions or Datasul framework requirements.