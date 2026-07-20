# Code Quality Patterns ? Performance, Legacy, Metadata, Compilation

Detailed code examples and patterns for performance, legacy/deprecated code,
metadata access, compilation, and OpenEdge compatibility review findings.

This reference is used by the `code-review` skill when reviewing Progress
OpenEdge ABL source code.

---

# G2 ? Performance and Database Access

## Database Queries Inside Loops (PERF-001) ? MAJOR

Repeated database access inside loops may cause significant performance
degradation, especially when processing large datasets.

### BAD

```progress
FOR EACH tt-item NO-LOCK:

    FIND FIRST item NO-LOCK
        WHERE item.it-codigo = tt-item.it-codigo
        NO-ERROR.

    IF AVAILABLE item THEN
        ASSIGN
            tt-item.descricao = item.descricao.

END.
````

The database is accessed once for every record in `tt-item`.

### GOOD ? Evaluate Alternative Access Strategies

When appropriate, preload the required data into a Temp-Table or redesign the
query to avoid repeated database access.

```progress
DEFINE TEMP-TABLE tt-item NO-UNDO
    FIELD it-codigo AS CHARACTER
    FIELD descricao AS CHARACTER.

FOR EACH item NO-LOCK
    WHERE item.it-codigo = pcCodigo:

    CREATE tt-item.

    ASSIGN
        tt-item.it-codigo = item.it-codigo
        tt-item.descricao = item.descricao.

END.
```

### Review Guidance

Do not automatically classify every query inside a loop as a defect.

Consider:

* Number of records
* Index availability
* Query selectivity
* Database size
* Expected execution frequency
* Whether the query uses an indexed field
* Whether caching or preloading is appropriate

Flag as `MAJOR` when repeated database access is likely to produce significant
performance degradation.

---

## Missing NO-LOCK on Read-Only Queries (DB-001) ? MAJOR

Read-only database access should normally use `NO-LOCK`.

### BAD

```progress
FIND FIRST customer
    WHERE customer.cust-num = piCustomer
    NO-ERROR.
```

### GOOD

```progress
FIND FIRST customer NO-LOCK
    WHERE customer.cust-num = piCustomer
    NO-ERROR.
```

### Review Guidance

Flag when:

* The record is clearly read-only
* No update is performed
* The lock mode is not explicitly required

Do not flag the absence of `NO-LOCK` when the record is intentionally being
prepared for modification or when the application requires a different lock
strategy.

---

## Excessive EXCLUSIVE-LOCK (DB-002) ? MAJOR

`EXCLUSIVE-LOCK` should only be used when the record will actually be modified
or deleted.

### BAD

```progress
FOR EACH customer EXCLUSIVE-LOCK
    WHERE customer.country = pcCountry:

    DISPLAY customer.name.

END.
```

The code only reads records but acquires exclusive locks.

### GOOD

```progress
FOR EACH customer NO-LOCK
    WHERE customer.country = pcCountry:

    DISPLAY customer.name.

END.
```

### Review Guidance

Flag unnecessary exclusive locks because they may:

* Increase contention
* Block concurrent users
* Increase deadlock risk
* Reduce application scalability

---

## Unfiltered Database Queries (PERF-002) ? MAJOR

Avoid iterating over entire database tables when only a subset of records is
required.

### BAD

```progress
FOR EACH customer NO-LOCK:

    IF customer.country = pcCountry THEN DO:
        /* Processing */
    END.

END.
```

### GOOD

```progress
FOR EACH customer NO-LOCK
    WHERE customer.country = pcCountry:

    /* Processing */

END.
```

Filtering should preferably be performed by the database.

---

## Repeated CAN-FIND or FIND Operations (PERF-003) ? MINOR / MAJOR

Repeated existence checks followed by another database access may result in
unnecessary database operations.

### BAD

```progress
IF CAN-FIND(FIRST customer
    WHERE customer.cust-num = piCustomer) THEN DO:

    FIND FIRST customer NO-LOCK
        WHERE customer.cust-num = piCustomer
        NO-ERROR.

    /* Processing */

END.
```

The record is searched twice.

### GOOD

```progress
FIND FIRST customer NO-LOCK
    WHERE customer.cust-num = piCustomer
    NO-ERROR.

IF AVAILABLE customer THEN DO:

    /* Processing */

END.
```

### Review Guidance

Flag duplicated database access when the same record can be retrieved once
and checked with `AVAILABLE`.

Use `CAN-FIND` when only existence needs to be determined and no record data
is required.

---

## Missing Index Consideration (PERF-004) ? MAJOR

Queries using large database tables should use selective fields supported by
appropriate indexes whenever possible.

### Potentially Problematic

```progress
FOR EACH customer NO-LOCK
    WHERE customer.name BEGINS pcName:

    /* Processing */

END.
```

Review whether the predicate can efficiently use an available index.

### Review Guidance

Do not assume an index is missing solely from the source code.

When database schema or index definitions are unavailable:

* Report the finding as a performance risk
* Recommend validating the query plan and index usage
* Do not claim a confirmed full table scan

---

## Large Result Sets (PERF-005) ? MINOR / MAJOR

Flag code that unnecessarily loads very large datasets into memory.

Potential patterns:

* Large Temp-Tables
* Large Datasets
* Unbounded queries
* `QUERY:GET-NEXT()` loops without filtering
* Full-table data copies
* Excessive `BUFFER-COPY`

Recommend:

* Filtering at the database level
* Processing records incrementally
* Limiting result sets
* Releasing temporary resources
* Avoiding unnecessary duplication

Severity depends on expected data volume.

---

## Repeated Dynamic Query Creation (PERF-006) ? MAJOR

Creating dynamic queries repeatedly inside a high-volume loop can be expensive.

### BAD

```progress
FOR EACH tt-item:

    CREATE QUERY hQuery.

    /* Prepare and execute query */

    DELETE OBJECT hQuery.

END.
```

### GOOD

Create the query once when the query structure is reusable.

```progress
CREATE QUERY hQuery.

/* Configure query */

FOR EACH tt-item:

    /* Reuse query when appropriate */

END.

IF VALID-HANDLE(hQuery) THEN
    DELETE OBJECT hQuery NO-ERROR.
```

### Review Guidance

Evaluate:

* Query creation frequency
* Query complexity
* Dataset size
* Whether query preparation can be reused

---

# G3 ? Legacy and Deprecated Code

## Legacy Global State (LEG-001) ? MINOR / MAJOR

Excessive use of global or shared state increases coupling and makes code
harder to test and maintain.

Review:

* `DEFINE NEW GLOBAL SHARED VARIABLE`
* `DEFINE SHARED VARIABLE`
* Excessive `SESSION` state
* Global handles
* Shared Temp-Tables

### BAD

```progress
DEFINE NEW GLOBAL SHARED VARIABLE gcCodigo AS CHARACTER NO-UNDO.
```

Multiple procedures depend directly on mutable global state.

### GOOD

Use explicit parameters:

```progress
DEFINE INPUT PARAMETER pcCodigo AS CHARACTER NO-UNDO.
```

### Review Guidance

Do not automatically flag all shared variables.

Consider:

* Existing framework architecture
* Legacy Datasul conventions
* Persistent procedure contracts
* Whether replacing shared state would require unsafe architectural changes

---

## Legacy Error Handling (LEG-002) ? MINOR / MAJOR

Identify error handling patterns that obscure failures.

### Potentially Problematic

```progress
FIND FIRST customer NO-LOCK
    WHERE customer.cust-num = piCustomer
    NO-ERROR.

/* Error ignored */
```

If the record is required, silently continuing may cause incorrect behavior.

### Better

```progress
FIND FIRST customer NO-LOCK
    WHERE customer.cust-num = piCustomer
    NO-ERROR.

IF ERROR-STATUS:ERROR THEN
    RETURN ERROR "Erro ao consultar cliente.".

IF NOT AVAILABLE customer THEN
    RETURN ERROR "Cliente não encontrado.".
```

Or, in modern error-handling flows:

```progress
DO ON ERROR UNDO, THROW:

    FIND FIRST customer NO-LOCK
        WHERE customer.cust-num = piCustomer
        NO-ERROR.

    IF NOT AVAILABLE customer THEN
        THROW NEW Progress.Lang.AppError(
            "Cliente não encontrado.",
            0
        ).

END.
```

### Review Guidance

The use of `NO-ERROR` itself is not a defect.

The finding is the failure to handle an error when the operation requires
successful execution.

---

## Unnecessary NO-ERROR (LEG-003) ? MINOR

Avoid suppressing errors when the application should allow them to propagate.

### BAD

```progress
RUN processar-pedido NO-ERROR.
```

No subsequent validation occurs.

### GOOD

```progress
RUN processar-pedido NO-ERROR.

IF ERROR-STATUS:ERROR THEN
    RETURN ERROR ERROR-STATUS:GET-MESSAGE(1).
```

Or allow the runtime error to propagate when appropriate.

---

## Console Debug Output (LEG-004) ? INFO / MINOR

Flag temporary debugging output left in production code.

Examples:

```progress
MESSAGE "DEBUG" VIEW-AS ALERT-BOX.
```

```progress
DISPLAY cCodigo.
```

```progress
PUT UNFORMATTED "DEBUG: " cCodigo SKIP.
```

### Recommendation

Use the application's standard logging mechanism.

Do not flag legitimate user-facing messages or intentional batch output.

---

## Hardcoded Configuration (LEG-005) ? MINOR / MAJOR

Flag hardcoded:

* File paths
* URLs
* Ports
* Environment names
* Credentials
* Connection strings

### BAD

```progress
ASSIGN
    cUrl = "http://192.168.0.100:8080/api".
```

### Better

Load configuration from:

* Environment configuration
* Application configuration
* Datasul configuration mechanisms
* Secure secret management

Severity:

* `MAJOR` for credentials or secrets
* `MINOR` for environment-specific configuration

---

## Hardcoded Magic Values (LEG-006) ? MINOR

### BAD

```progress
IF iStatus = 3 THEN DO:
```

### GOOD

```progress
DEFINE VARIABLE iStatusProcessed AS INTEGER NO-UNDO INITIAL 3.

IF iStatus = iStatusProcessed THEN DO:
```

Prefer named constants or clearly named variables.

Do not create constants for trivial values where this would reduce readability.

---

## Deprecated or Obsolete APIs (LEG-007) ? INFO / MINOR

Flag APIs known to be deprecated for the target OpenEdge version.

When a deprecated API is identified:

1. Confirm the target OpenEdge version.
2. Identify the recommended replacement.
3. Explain migration impact.
4. Do not invent replacement APIs.

If the version is unknown, classify the finding as `INFO` and recommend
verification against the OpenEdge documentation.

---

# G4 ? Metadata and Schema Access

OpenEdge applications may access metadata through several mechanisms.

Unlike Protheus, there is no universal rule that direct database metadata
access is prohibited.

The review must distinguish between:

* Application data
* Database schema metadata
* Runtime metadata
* Dynamic ABL metadata
* Datasul framework metadata

---

## Direct Schema Metadata Access (META-001) ? INFO / MINOR

Review direct access to database schema metadata.

Potential mechanisms include:

* `BUFFER-FIELD`
* `BUFFER-FIELD()`
* `BUFFER-VALUE`
* `BUFFER-FIELD:DATA-TYPE`
* `BUFFER-FIELD:FORMAT`
* `BUFFER-FIELD:LABEL`
* `BUFFER-FIELD:EXTENT`
* `BUFFER-FIELD:DECIMALS`
* `BUFFER-FIELD:MANDATORY`

These are valid OpenEdge mechanisms and should not be flagged as inherently
unsafe.

### Example

```progress
ASSIGN
    hField = BUFFER customer:BUFFER-FIELD("cust-num").

MESSAGE
    hField:DATA-TYPE
    hField:LABEL
    VIEW-AS ALERT-BOX.
```

### Review Guidance

Flag only when:

* Metadata access is unnecessarily repeated
* Dynamic metadata is used where static typing would be simpler
* Reflection introduces unnecessary complexity
* Metadata is accessed inside high-volume loops without caching

---

## Dynamic Field Access Inside Loops (META-002) ? MINOR / MAJOR

### Potentially Problematic

```progress
FOR EACH tt-item:

    hField = BUFFER tt-item:BUFFER-FIELD(pcField).

    cValue = STRING(hField:BUFFER-VALUE).

END.
```

If the field definition never changes, resolve the field handle once.

### Better

```progress
hField = BUFFER tt-item:BUFFER-FIELD(pcField).

FOR EACH tt-item:

    cValue = STRING(hField:BUFFER-VALUE).

END.
```

Severity depends on execution frequency and complexity.

---

## Dynamic Table Access (META-003) ? INFO / MINOR

Review usage of:

```progress
BUFFER
CREATE BUFFER
CREATE-LIKE
CREATE-LIKE-SEQUENTIAL
TEMP-TABLE-PREPARE
```

Dynamic table access is appropriate when:

* Generic frameworks are required
* Table names are configuration-driven
* Metadata-driven processing is intentional

Flag when dynamic access is used unnecessarily instead of statically typed
code.

---

## Datasul Metadata Access

When reviewing TOTVS Datasul code, distinguish between:

1. Standard Datasul framework APIs
2. Application database tables
3. Temporary structures
4. Metadata and configuration tables

Do not classify direct access to a Datasul table as a defect solely because
the table is accessed directly.

Evaluate:

* Whether the table is part of the public application contract
* Whether a standard Datasul API exists
* Whether direct access bypasses required business rules
* Whether the access is read-only or modifies data
* Whether the operation is supported by the target Datasul version

---

# G5 ? Compilation and Compatibility

## Syntax and Compilation Errors (COMP-001) ? MAJOR

Identify obvious compilation problems.

Check for:

* Missing periods
* Incorrect block termination
* Invalid keywords
* Incorrect parameter definitions
* Invalid data types
* Invalid method signatures
* Invalid class syntax
* Incorrect `USING`
* Incorrect `CATCH`
* Incorrect `FINALLY`
* Invalid `DO` blocks
* Invalid `FOR EACH`
* Invalid `FIND`
* Incorrect `END PROCEDURE`
* Incorrect `END FUNCTION`
* Incorrect `END METHOD`
* Incorrect `END CLASS`

Example:

```progress
PROCEDURE processar:

    MESSAGE "Processando".

/* Missing END PROCEDURE */
```

Flag only issues that can reasonably be identified from the provided source.

---

## Undefined or Unresolved References (COMP-002) ? MAJOR

Check for:

* Undefined variables
* Undefined parameters
* Undefined Temp-Tables
* Undefined buffers
* Undefined procedures
* Undefined functions
* Undefined classes
* Missing includes
* Missing namespaces
* Invalid method calls

When an external dependency is not provided, do not automatically classify
the reference as invalid.

Clearly identify it as:

> Unable to validate because the referenced dependency was not provided.

---

## Include Dependencies (COMP-003) ? MINOR

Review:

```progress
{include-file.i}
```

Check for:

* Missing include files
* Circular dependencies
* Excessive include usage
* Hidden global state introduced by includes
* Conflicting definitions

Do not flag a valid include merely because it is used.

---

## OpenEdge Version Compatibility (COMP-004) ? MAJOR / MINOR

Review usage of APIs or language features that may depend on the OpenEdge
version.

If the target version is known:

* Validate compatibility against that version.

If the target version is unknown:

* Do not assume the newest syntax is supported.
* Mark version-sensitive recommendations as `INFO` or `MINOR`.

Example:

```text
The implementation uses a feature introduced in a newer OpenEdge release.
Validate compatibility with the target OpenEdge version.
```

---

## Encoding and Character Handling (COMP-005) ? MINOR / MAJOR

Review encoding when source code or external files contain:

* Accented characters
* Unicode
* JSON
* XML
* HTTP payloads
* File imports
* File exports

Potential risks include:

* UTF-8 vs. ISO-8859-1
* Code page mismatches
* Character conversion errors
* Corrupted accented characters
* Incorrect JSON encoding
* Incorrect HTTP charset handling

Do not assume a single encoding is correct for every OpenEdge environment.

Recommend validating against:

* OpenEdge session code page
* Database code page
* Operating system
* External system encoding

---

## Internationalization (COMP-006) ? MINOR

Flag hardcoded user-facing messages when the application architecture requires
internationalization.

### Example

```progress
MESSAGE "Cliente não encontrado."
    VIEW-AS ALERT-BOX.
```

If the application supports multiple languages, recommend using the
application's internationalization mechanism.

Do not flag hardcoded messages when the application is intentionally
single-language.

---

# Resource Management

## Dynamic Handle Leak (RES-001) ? MAJOR

Flag dynamic resources that are created but not released.

Potential resources:

* Query handles
* Buffer handles
* Procedure handles
* Dynamic Temp-Tables
* Dynamic Datasets

### BAD

```progress
CREATE QUERY hQuery.

/* Query processing */

/* hQuery never released */
```

### GOOD

```progress
CREATE QUERY hQuery.

/* Query processing */

IF VALID-HANDLE(hQuery) THEN
    DELETE OBJECT hQuery NO-ERROR.
```

Review error paths as well.

Resources must be released even when an error occurs.

---

## Persistent Procedure Leak (RES-002) ? MAJOR

### BAD

```progress
RUN process.p PERSISTENT SET hProcess.
```

The procedure is never released.

### GOOD

```progress
RUN process.p PERSISTENT SET hProcess.

/* Processing */

IF VALID-HANDLE(hProcess) THEN
    DELETE PROCEDURE hProcess NO-ERROR.
```

Consider cleanup in all execution paths.

---

# Code Quality Patterns

## Excessive Procedure or Method Size (QUAL-001) ? MINOR / MAJOR

Large procedures may indicate excessive responsibilities.

Review for:

* Multiple unrelated responsibilities
* Deep nesting
* Repeated logic
* Long database operations
* Mixed UI and business logic
* Mixed integration and persistence logic

Do not enforce an arbitrary line limit.

Evaluate complexity and responsibility instead.

---

## Deep Nesting (QUAL-002) ? MINOR

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

---

## Duplicate Logic (QUAL-003) ? MINOR / MAJOR

Identify repeated business logic that can lead to inconsistent maintenance.

Consider extracting:

* Internal procedures
* Functions
* Classes
* Shared services

Do not abstract code merely because two blocks look superficially similar.

---

# Review Guidance

When applying these patterns:

1. Consider the target OpenEdge version.
2. Consider whether the code is procedural or OOABL.
3. Consider the Datasul architecture.
4. Consider database size and expected record volume.
5. Consider indexes when evaluating performance.
6. Distinguish confirmed defects from potential risks.
7. Do not invent missing schema information.
8. Do not invent Datasul APIs.
9. Do not treat every legacy pattern as a defect.
10. Prefer evidence-based findings.
11. Provide a practical recommendation.
12. Provide corrected ABL code when appropriate.

---

# Severity Summary

| Rule     | Pattern                                  | Severity      |
| -------- | ---------------------------------------- | ------------- |
| PERF-001 | Database access inside high-volume loops | MAJOR         |
| DB-001   | Missing NO-LOCK on read-only access      | MAJOR         |
| DB-002   | Unnecessary EXCLUSIVE-LOCK               | MAJOR         |
| PERF-002 | Unfiltered database query                | MAJOR         |
| PERF-003 | Repeated FIND/CAN-FIND                   | MINOR / MAJOR |
| PERF-004 | Potential missing index                  | MAJOR         |
| PERF-005 | Excessive result set                     | MINOR / MAJOR |
| PERF-006 | Repeated dynamic query creation          | MAJOR         |
| LEG-001  | Excessive global/shared state            | MINOR / MAJOR |
| LEG-002  | Improper error handling                  | MINOR / MAJOR |
| LEG-003  | Unnecessary NO-ERROR                     | MINOR         |
| LEG-004  | Debug/console output                     | INFO / MINOR  |
| LEG-005  | Hardcoded configuration                  | MINOR / MAJOR |
| LEG-006  | Magic values                             | MINOR         |
| LEG-007  | Deprecated API                           | INFO / MINOR  |
| META-001 | Unnecessary metadata access              | INFO / MINOR  |
| META-002 | Dynamic metadata access in loops         | MINOR / MAJOR |
| META-003 | Unnecessary dynamic table access         | INFO / MINOR  |
| COMP-001 | Compilation/syntax errors                | MAJOR         |
| COMP-002 | Undefined references                     | MAJOR         |
| COMP-003 | Include dependency problems              | MINOR         |
| COMP-004 | Version compatibility issue              | MINOR / MAJOR |
| COMP-005 | Encoding problems                        | MINOR / MAJOR |
| COMP-006 | Internationalization issue               | MINOR         |
| RES-001  | Dynamic handle leak                      | MAJOR         |
| RES-002  | Persistent procedure leak                | MAJOR         |
| QUAL-001 | Excessive procedure/method size          | MINOR / MAJOR |
| QUAL-002 | Deep nesting                             | MINOR         |
| QUAL-003 | Duplicate logic                          | MINOR / MAJOR |

```

### O que eu alterei em relação ao `code-quality-patterns.md` do Protheus

A principal mudança é que não fiz uma conversão literal. O arquivo original tinha três grupos muito específicos do Protheus:

- **G2:** APIs AdvPL dentro de loops e transações;
- **G3:** APIs legadas do framework Protheus;
- **G4:** acesso direto às tabelas `SX*`;
- **G5:** encoding fixo em Windows-1252.

No OpenEdge, isso não se aplica. Em vez disso, a referência agora tem uma abordagem mais adequada ao ABL:

- `NO-LOCK` e `EXCLUSIVE-LOCK`;
- consultas dentro de loops;
- `FIND` vs. `CAN-FIND`;
- uso de índices;
- `NO-ERROR` mal utilizado;
- transações;
- vazamento de `HANDLE`;
- `DELETE PROCEDURE`;
- `CREATE QUERY`;
- `BUFFER-FIELD`;
- acesso dinâmico a metadados;
- `TEMP-TABLE`;
- dependências de `.i`;
- compatibilidade entre versões do OpenEdge;
- code page e encoding;
- estado global e `SHARED`;
- código legado;
- padrões de OOABL.

**Um ponto que considero especialmente importante para o seu agente Datasul:** eu evitaria colocar uma regra rígida dizendo que determinado acesso a uma tabela Datasul é proibido. O agente deve primeiro entender se existe uma API/framework padrão para aquela operação e, principalmente, se o acesso direto está **bypassando alguma regra de negócio**. Isso evita que o agente gere falsos positivos em código Datasul legítimo.

A próxima referência que eu faria seria **`database-review-patterns.md`**, mais profunda e específica para ABL, pois ela pode concentrar as regras de `FIND`, `FOR EACH`, `CAN-FIND`, `NO-LOCK`, `EXCLUSIVE-LOCK`, `AVAILABLE`, `LOCKED`, índices e transações. Isso deixaria o `SKILL.md` principal mais enxuto e daria ao agente uma base técnica bem mais forte para revisar código OpenEdge.
