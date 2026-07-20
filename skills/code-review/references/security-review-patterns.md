# Security Review Patterns ? OpenEdge ABL and Datasul

Detailed code examples and patterns for security-related code review findings
in Progress OpenEdge ABL applications, REST services, integrations, database
access, and TOTVS Datasul customizations.

This reference is used by the `code-review` skill when reviewing security risks
in OpenEdge ABL source code.

---

# S1 ? Injection and Dynamic Query Security

## SQL Injection Through Dynamic SQL (SEC-001) ? CRITICAL

Dynamic SQL constructed through string concatenation may allow external input
to alter the intended SQL statement.

### BAD

```progress
DEFINE VARIABLE cQuery AS CHARACTER NO-UNDO.

ASSIGN
    cQuery = "SELECT * FROM customer " +
             "WHERE name = '" + pcCustomerName + "'".
````

If `pcCustomerName` originates from an external or untrusted source, the
query may be vulnerable to injection.

### GOOD

Prefer parameterized SQL or APIs that support parameter binding.

```progress
DEFINE VARIABLE cQuery AS CHARACTER NO-UNDO.
DEFINE VARIABLE hQuery AS HANDLE NO-UNDO.

CREATE QUERY hQuery.

/* Prepare query using controlled query structure and parameters. */
```

When using dynamic SQL, separate:

1. SQL structure
2. Parameters
3. User-controlled values

Never concatenate untrusted input directly into executable SQL.

### Review Guidance

Flag as `CRITICAL` when:

* User input is directly concatenated into executable SQL
* External HTTP parameters are concatenated into SQL
* Request body values are concatenated into SQL
* File or integration data is treated as trusted without validation

If the value is fully controlled internally and cannot be influenced by an
external source, classify according to actual risk.

---

## Dynamic Query Injection (SEC-002) ? MAJOR / CRITICAL

OpenEdge Dynamic Queries can also become unsafe when query strings are built
from uncontrolled external input.

### BAD

```progress
ASSIGN
    cWhere = "customer.name = '" + pcName + "'".

hQuery:QUERY-PREPARE(
    "FOR EACH customer NO-LOCK WHERE " + cWhere
).
```

### Better

Keep the query structure controlled by the application.

```progress
ASSIGN
    cQuery = "FOR EACH customer NO-LOCK " +
             "WHERE customer.name = ?".

/* Configure query parameters separately when supported. */
```

### Review Guidance

Do not assume that using `QUERY-PREPARE()` automatically makes a query safe.

The security risk depends on how the query string is constructed.

---

## Dynamic Table or Field Names From External Input (SEC-003) ? MAJOR

Dynamic table names, field names, and database object identifiers should not
be accepted directly from untrusted input.

### BAD

```progress
ASSIGN
    cTable = pcTableName.

CREATE BUFFER hBuffer FOR TABLE cTable.
```

If `pcTableName` is externally controlled, an attacker may access unintended
database structures.

### GOOD

Use an explicit allowlist.

```progress
IF LOOKUP(pcTableName, "customer,item,order") = 0 THEN
    RETURN ERROR "Tabela não permitida.".

CREATE BUFFER hBuffer FOR TABLE pcTableName.
```

### Review Guidance

Prefer allowlists over blocklists.

Do not allow arbitrary table or field access unless the architecture explicitly
requires a generic metadata-driven system.

---

# S2 ? Credentials and Secrets

## Hardcoded Credentials (SEC-004) ? CRITICAL

Credentials must never be stored directly in source code.

### BAD

```progress
DEFINE VARIABLE cUser AS CHARACTER NO-UNDO.
DEFINE VARIABLE cPassword AS CHARACTER NO-UNDO.

ASSIGN
    cUser     = "admin"
    cPassword = "admin123".
```

### GOOD

Use secure configuration mechanisms appropriate to the deployment environment.

```progress
ASSIGN
    cUser = GET-ENVIRONMENT("APP_DB_USER")
    cPassword = GET-ENVIRONMENT("APP_DB_PASSWORD").
```

When using environment variables is not appropriate, use the application's
secure configuration mechanism.

### Review Guidance

Flag as `CRITICAL`:

* Database passwords
* API keys
* OAuth client secrets
* Private keys
* Access tokens
* Encryption keys

Also review credentials stored in:

* `.p`
* `.cls`
* `.i`
* `.json`
* `.xml`
* `.ini`
* Configuration files committed to source control

---

## Hardcoded API Tokens (SEC-005) ? CRITICAL

### BAD

```progress
ASSIGN
    cToken = "eyJhbGciOiJIUzI1NiIs...".
```

### GOOD

Load credentials from secure configuration.

```progress
ASSIGN
    cToken = GET-ENVIRONMENT("API_TOKEN").
```

Never log authentication tokens.

---

## Sensitive Data in Logs (SEC-006) ? MAJOR

Do not log:

* Passwords
* API tokens
* Authentication headers
* Credit card information
* Personal identification data
* Sensitive customer information

### BAD

```progress
LOG-MANAGER:WRITE-MESSAGE(
    "Authorization: " + cAuthorization
).
```

### GOOD

```progress
LOG-MANAGER:WRITE-MESSAGE(
    "External service authentication completed."
).
```

If diagnostic information is required, mask sensitive values.

---

# S3 ? External Input Validation

## Unvalidated REST Input (SEC-007) ? MAJOR

External HTTP input must be validated before being used in business logic,
database queries, file operations, or external integrations.

Review:

* Query parameters
* Path parameters
* HTTP headers
* JSON request bodies
* XML request bodies
* Form data

### BAD

```progress
ASSIGN
    iCustomerId = INTEGER(pcCustomerId).

FIND FIRST customer NO-LOCK
    WHERE customer.cust-num = iCustomerId
    NO-ERROR.
```

The conversion and input validation strategy should be explicitly considered.

### GOOD

Validate the input before processing.

```progress
ASSIGN
    iCustomerId = INTEGER(pcCustomerId) NO-ERROR.

IF ERROR-STATUS:ERROR THEN
    RETURN ERROR "Identificador de cliente inválido.".

IF iCustomerId <= 0 THEN
    RETURN ERROR "Identificador de cliente inválido.".
```

### Review Guidance

Validation should consider:

* Data type
* Range
* Length
* Allowed characters
* Business constraints
* Required fields
* Null or empty values

---

## Trusting External JSON Without Validation (SEC-008) ? MAJOR

JSON received from external systems should not be assumed to be valid or
complete.

Review:

* Required properties
* Data types
* Null values
* Unexpected values
* Maximum payload size
* Business validation

Do not trust client-side validation alone.

---

# S4 ? File and Path Security

## Path Traversal (SEC-009) ? CRITICAL

Do not construct file paths directly from untrusted input.

### BAD

```progress
ASSIGN
    cFilePath = "/app/files/" + pcFileName.

INPUT FROM VALUE(cFilePath).
```

An attacker may attempt to access files outside the intended directory.

### GOOD

Validate the filename and restrict access to an approved directory.

```progress
IF INDEX(pcFileName, "..") > 0 THEN
    RETURN ERROR "Nome de arquivo inválido.".

IF INDEX(pcFileName, "/") > 0
OR INDEX(pcFileName, "~\") > 0 THEN
    RETURN ERROR "Nome de arquivo inválido.".
```

Prefer stronger path validation and canonicalization when available.

### Review Guidance

Pay special attention to:

* File download endpoints
* File upload endpoints
* Import routines
* Export routines
* Report generation
* Temporary files

---

## Unrestricted File Access (SEC-010) ? MAJOR

Applications should not allow external users to access arbitrary filesystem
locations.

### BAD

```progress
INPUT FROM VALUE(pcFilePath).
```

when `pcFilePath` is externally controlled.

### GOOD

Use an allowlisted directory and validate the requested resource.

---

## Unsafe File Upload (SEC-011) ? MAJOR

File upload functionality should validate:

* Filename
* Extension
* File size
* File location
* Content type
* Storage location

Do not rely exclusively on the filename extension.

---

# S5 ? REST and HTTP Security

## Unencrypted HTTP Communication (SEC-012) ? MAJOR

Sensitive information should be transmitted over HTTPS.

### BAD

```progress
ASSIGN
    cUrl = "http://api.example.com/customers".
```

### GOOD

```progress
ASSIGN
    cUrl = "https://api.example.com/customers".
```

### Review Guidance

Flag HTTP when:

* Credentials are transmitted
* Tokens are transmitted
* Personal data is transmitted
* Business-sensitive data is transmitted

Plain HTTP may be acceptable for non-sensitive internal traffic only when the
environment explicitly guarantees secure transport.

---

## Disabled TLS Certificate Validation (SEC-013) ? CRITICAL

Never disable TLS certificate validation in production to bypass certificate
errors.

Flag patterns that:

* Accept invalid certificates
* Disable hostname validation
* Ignore certificate errors
* Trust all certificates

Such configurations may allow man-in-the-middle attacks.

---

## Missing HTTP Timeout (SEC-014) ? MAJOR

External HTTP calls should have reasonable timeout handling.

Without appropriate timeouts, application resources may remain occupied
indefinitely.

Review:

* Connection timeout
* Read timeout
* Overall request timeout

---

## Uncontrolled Retry Logic (SEC-015) ? MAJOR

Retry logic must not create infinite or excessive retries.

### BAD

```progress
DO WHILE lError:

    RUN CallExternalService.

END.
```

### GOOD

Use bounded retries.

```progress
DO iRetry = 1 TO 3:

    RUN CallExternalService NO-ERROR.

    IF NOT ERROR-STATUS:ERROR THEN
        LEAVE.

END.
```

Consider exponential backoff for external services.

---

## Sensitive Data in URLs (SEC-016) ? MAJOR

Avoid transmitting sensitive data through URL query parameters.

### BAD

```text
https://api.example.com/login?user=admin&password=secret
```

Prefer secure request bodies and appropriate authentication mechanisms.

---

# S6 ? Authentication and Authorization

## Missing Authentication (SEC-017) ? CRITICAL

Sensitive REST or integration endpoints must enforce authentication.

Review whether endpoints expose:

* Customer data
* Financial data
* Employee data
* Administrative operations
* Database modification
* Business-critical operations

Do not assume authentication is handled unless the application architecture
clearly establishes it.

---

## Missing Authorization (SEC-018) ? CRITICAL

Authentication verifies who the caller is.

Authorization verifies what the caller is allowed to do.

Review whether authenticated users can:

* Access other users' data
* Modify records outside their scope
* Execute administrative operations
* Access restricted business functions

### Example Risk

```progress
/* User authenticated successfully. */

FIND FIRST customer NO-LOCK
    WHERE customer.cust-num = piCustomer
    NO-ERROR.

/* No authorization check */
```

Authentication alone is not sufficient.

---

## IDOR / Insecure Direct Object Reference (SEC-019) ? CRITICAL

Endpoints that accept object identifiers must verify that the current user is
authorized to access the requested object.

### Risk

```text
GET /customers/100
GET /customers/101
GET /customers/102
```

If changing the identifier allows access to another customer's information,
the endpoint may be vulnerable.

Review authorization based on:

* User
* Company
* Branch
* Role
* Permission
* Business ownership

---

# S7 ? Database Security

## Excessive Database Access (SEC-020) ? MAJOR

Applications should access only the data required to perform the operation.

Avoid unrestricted queries that expose unnecessary fields.

### BAD

```progress
FOR EACH customer NO-LOCK:

    /* Return entire customer record */

END.
```

### Better

Select or expose only the required data through the application layer.

---

## Sensitive Data Exposure (SEC-021) ? MAJOR

Review whether APIs, logs, or output expose:

* Password hashes
* Personal identification data
* Financial information
* Internal database identifiers
* Internal system information
* Security configuration

Return only data required by the caller.

---

## Direct Database Access Bypassing Business Rules (SEC-022) ? MAJOR

In Datasul or other ERP environments, direct database updates may bypass
business rules enforced by standard application programs.

### Potentially Dangerous

```progress
FOR FIRST customer EXCLUSIVE-LOCK:

    ASSIGN
        customer.credit-limit = 999999999.

END.
```

If the standard business process requires validation, authorization, logging,
or related updates, direct modification may create data integrity and security
risks.

### Review Guidance

Consider:

* Standard Datasul APIs
* Business Entities
* Standard application procedures
* Validation routines
* Authorization rules

Do not assume direct table access is always prohibited.

---

# S8 ? Dynamic Execution and Runtime Code

## Unsafe Dynamic Procedure Execution (SEC-023) ? CRITICAL

Avoid executing procedure names derived directly from untrusted input.

### BAD

```progress
RUN VALUE(pcProcedureName).
```

If `pcProcedureName` is externally controlled, arbitrary procedures may be
executed.

### GOOD

Use an explicit allowlist.

```progress
IF LOOKUP(
    pcProcedureName,
    "process-order.p,process-customer.p"
) = 0 THEN
    RETURN ERROR "Procedimento não permitido.".

RUN VALUE(pcProcedureName).
```

---

## Unsafe Dynamic Class Instantiation (SEC-024) ? CRITICAL

Do not instantiate arbitrary classes based on untrusted input.

### BAD

```progress
CREATE OBJECT oObject
    TYPE pcClassName.
```

### GOOD

Use a controlled factory or allowlist.

```progress
CASE pcClassName:

    WHEN "CustomerService" THEN
        oObject = NEW CustomerService().

    WHEN "OrderService" THEN
        oObject = NEW OrderService().

    OTHERWISE
        RETURN ERROR "Classe não permitida.".

END CASE.
```

---

# S9 ? Error Handling and Information Disclosure

## Sensitive Information in Error Messages (SEC-025) ? MAJOR

Do not expose internal implementation details to external users.

### BAD

```progress
RETURN ERROR
    "Database connection failed: " +
    cConnectionString.
```

### GOOD

Return a generic external message while logging technical details securely.

```progress
LOG-MANAGER:WRITE-MESSAGE(
    "Database connection failed."
).

RETURN ERROR
    "Não foi possível concluir a operação.".
```

---

## Stack Trace Exposure (SEC-026) ? MAJOR

Do not expose internal stack traces, filesystem paths, database details, or
internal class names in public API responses.

Internal diagnostics should be logged securely.

---

# S10 ? Secrets in Configuration

## Plaintext Secrets in Configuration Files (SEC-027) ? CRITICAL

Review:

* `.ini`
* `.json`
* `.xml`
* `.yaml`
* `.properties`

Do not commit plaintext credentials or secrets to source control.

### BAD

```ini
[Database]
Password=SuperSecret123
```

### Better

Use:

* Environment variables
* Secret management
* Secure deployment configuration
* Protected credential stores

The exact mechanism depends on the deployment architecture.

---

# S11 ? Datasul Security

## Unauthorized Direct Business Data Modification (DS-SEC-001) ? CRITICAL / MAJOR

Review direct database updates in Datasul customizations.

### Potential Risk

```progress
FOR FIRST customer EXCLUSIVE-LOCK:

    ASSIGN
        customer.credit-limit = deNewLimit.

END.
```

The operation may bypass:

* Authorization
* Validation
* Audit
* Related table updates
* Standard business rules

### Review Guidance

Do not automatically reject direct database updates.

Evaluate whether:

1. The update is part of an approved customization.
2. Standard Datasul APIs are available.
3. Business rules are bypassed.
4. Authorization is validated.
5. Audit requirements are satisfied.
6. Related data is maintained consistently.

---

## Datasul Credential Exposure (DS-SEC-002) ? CRITICAL

Never hardcode:

* Database credentials
* AppServer credentials
* REST authentication tokens
* External API credentials

Review source code and configuration files.

---

## Datasul Environment Manipulation (DS-SEC-003) ? MAJOR

Review code that dynamically changes application environment context.

Pay special attention to:

* Company
* Branch
* User context
* Session context
* Database connection context

Ensure that context changes are authorized and correctly restored.

---

# Security Severity Summary

| Rule       | Pattern                                 | Severity         |
| ---------- | --------------------------------------- | ---------------- |
| SEC-001    | SQL injection                           | CRITICAL         |
| SEC-002    | Dynamic query injection                 | MAJOR / CRITICAL |
| SEC-003    | Dynamic table/field from input          | MAJOR            |
| SEC-004    | Hardcoded credentials                   | CRITICAL         |
| SEC-005    | Hardcoded API tokens                    | CRITICAL         |
| SEC-006    | Sensitive data in logs                  | MAJOR            |
| SEC-007    | Unvalidated REST input                  | MAJOR            |
| SEC-008    | Unvalidated JSON input                  | MAJOR            |
| SEC-009    | Path traversal                          | CRITICAL         |
| SEC-010    | Unrestricted file access                | MAJOR            |
| SEC-011    | Unsafe file upload                      | MAJOR            |
| SEC-012    | Unencrypted HTTP                        | MAJOR            |
| SEC-013    | Disabled TLS validation                 | CRITICAL         |
| SEC-014    | Missing HTTP timeout                    | MAJOR            |
| SEC-015    | Uncontrolled retries                    | MAJOR            |
| SEC-016    | Sensitive data in URLs                  | MAJOR            |
| SEC-017    | Missing authentication                  | CRITICAL         |
| SEC-018    | Missing authorization                   | CRITICAL         |
| SEC-019    | IDOR                                    | CRITICAL         |
| SEC-020    | Excessive database access               | MAJOR            |
| SEC-021    | Sensitive data exposure                 | MAJOR            |
| SEC-022    | Business rule bypass                    | MAJOR            |
| SEC-023    | Unsafe dynamic procedure execution      | CRITICAL         |
| SEC-024    | Unsafe dynamic class instantiation      | CRITICAL         |
| SEC-025    | Sensitive error information             | MAJOR            |
| SEC-026    | Stack trace exposure                    | MAJOR            |
| SEC-027    | Plaintext secrets in configuration      | CRITICAL         |
| DS-SEC-001 | Unauthorized direct data modification   | CRITICAL / MAJOR |
| DS-SEC-002 | Datasul credential exposure             | CRITICAL         |
| DS-SEC-003 | Unsafe environment context manipulation | MAJOR            |

---

# Security Review Guidance

When applying these rules:

1. Never assume all dynamic ABL code is insecure.
2. Identify whether input originates from an untrusted source.
3. Distinguish internal configuration from external user input.
4. Consider the application's deployment architecture.
5. Consider whether authentication is implemented outside the reviewed code.
6. Do not report missing authentication when the endpoint is explicitly
   protected by infrastructure that is visible in the provided context.
7. Do not invent security controls that are not present.
8. Do not expose secrets in the review output.
9. Mask sensitive values if they appear in reviewed source code.
10. Never reproduce credentials, tokens, or private keys in the review.
11. Consider the target OpenEdge version.
12. Consider the Datasul architecture when applicable.
13. Prefer allowlists for dynamic execution and metadata access.
14. Validate external input before database, filesystem, or business operations.
15. Separate technical error details from public error responses.
16. Prioritize data protection, authorization, authentication, and integrity.