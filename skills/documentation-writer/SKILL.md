---
name: documentation-writer
description: 'Generate comment blocks for Openedge source code. Use when a user says "document this function", "write documentation", "document this class/method", "document this code" or needs structured source-code documentation.'
license: MIT
metadata:
  domain: Openedge
  author: Consultoria Incode
  version: '1.0.0'
  category: Documentation and Planning
---

# Documentation Writer

You are an expert in writing documentation comment blocks for Openedge source code.

## When to Use

- Adding documentation to new or existing functions, classes, methods, program
- Generating documentation blocks for undocumented source files
- Reviewing and correcting existing documentation blocks for completeness
- Batch-documenting all elements in a `.i`, `.cls` or `.p` source file

---

### Rules

- The block must open with /* followed by a space and the identifier line.
- The identifier line must match exactly based on the element type:
  - Programs: The program filename (e.g., integration_queue.p)
  - Classes: The class name (e.g., OrderTrackingService)
  - Methods: ClassName::MethodName (e.g., OrderTrackingService::SendTrackingUpdate)
  - Procedures: The internal procedure name (e.g., piUpdateQueueStatus)
  - Functions: The internal function name (e.g., fnValidatePayload)
- The brief description is a concise sentence mapped directly to the Purpose or Description fields inside the block.
- The block must close with */ as part of the dashed divider line.

---

## Templates by Element

### Program Header
```
/*------------------------------------------------------------------------
  File        : <filename>.p
  Purpose     : <Brief description of what the program does>

  Description : <description of what the function does>

  Author(s)   : <author of program>
  Created     : <date when created>
  Version     : <version of program>
  Notes       : <something important(behaviors or logic annotations)>
  ----------------------------------------------------------------------*/

/* ***************************  Definitions  ************************** */

/* ************************  Internal Functions  ********************** */

/* ***********************  Internal Procedures  ********************** */
```

### Procedure Header

```
/*------------------------------------------------------------------------
  Purpose : <Brief description of what the procedure does>
  Params  : <inputs or outputs parameter received by procedure enumerated> 
  Notes   : <something important(behaviors or logic annotations)>  
  ----------------------------------------------------------------------*/
```

### Function Header

```
/*------------------------------------------------------------------------
  Purpose : <Brief description of what the function does>
  Params  : <inputs or outputs parameter received by function enumerated> 
  Notes   : <something important(behaviors or logic annotations)>
  ----------------------------------------------------------------------*/
```

### Class

```
/*------------------------------------------------------------------------
  File        : <filename>.cls
  Purpose     : <Brief description of what the class does>
  Syntax      : NEW <classname>(<params>)
  Implements  : <Interfaces or Inherit>
  Notes       : <something important(behaviors or logic annotations)>
  ----------------------------------------------------------------------*/
```

### Method

```
/*--------------------------------------------------------------------
  Purpose : <Brief description of what the method does>
  Params  : <inputs or outputs parameter received by method enumerated>
  Notes   : <something important(behaviors or logic annotations)>
  --------------------------------------------------------------------*/
```

---

## Complete Example: Program
```
/*------------------------------------------------------------------------
  File        : integration_queue.p
  Purpose     : Gerencia a fila de sincronização de pedidos com sistemas satélites.

  Description : Lê os registros pendentes da tabela de integração, processa os 
                dados de envio e atualiza o status da fila para evitar duplicidade.

  Author(s)   : Thiago Fofano - INSTI
  Created     : 17/07/2026
  Version     : 1.0.0
  Notes       : Deve ser executado via RPW ou persistido na sessão para escuta ativa.
  ----------------------------------------------------------------------*/

/* ***************************  Definitions  ************************** */
DEFINE TEMP-TABLE ttIntegrationQueue NO-UNDO
    FIELD idQueue     AS INT64
    FIELD cDocument   AS CHARACTER
    FIELD cPayload    AS CHARACTER
    FIELD cStatus     AS CHARACTER
    INDEX idxQueue IS UNIQUE PRIMARY KEY idQueue.

DEFINE VARIABLE iTotalProcessed AS INTEGER NO-UNDO.

/* ************************  Internal Functions  ********************** */

/*------------------------------------------------------------------------
  Purpose : Valida se o formato do payload do documento está legível.
  Params  : 1. INPUT pcPayload AS CHARACTER - String JSON/XML enviada
  Notes   : Retorna TRUE se o payload conter os nós obrigatórios de cabeçalho.
  ----------------------------------------------------------------------*/
FUNCTION validatePayload RETURNS LOGICAL (INPUT pcPayload AS CHARACTER):
    IF pcPayload = "" OR pcPayload = ? THEN 
        RETURN FALSE.
        
    RETURN (INDEX(pcPayload, "header") > 0).
END FUNCTION.

/* ***********************  Internal Procedures  ********************** */

/*------------------------------------------------------------------------
  Purpose : Atualiza o status do registro da fila após a tentativa de envio.
  Params  : 1. INPUT piQueueId AS INT64 - Identificador único da fila
            2. INPUT pcNewStatus AS CHARACTER - Novo status (Ex: "SUCCESS", "ERROR")
  Notes   : Realiza a alteração dentro de uma transação exclusiva por registro.
  ----------------------------------------------------------------------*/
PROCEDURE updateQueueStatus:
    DEFINE INPUT PARAMETER piQueueId   AS INT64     NO-UNDO.
    DEFINE INPUT PARAMETER pcNewStatus AS CHARACTER NO-UNDO.

    FIND FIRST ttIntegrationQueue WHERE ttIntegrationQueue.idQueue = piQueueId EXCLUSIVE-LOCK NO-ERROR.
    IF AVAILABLE ttIntegrationQueue THEN DO:
        ASSIGN ttIntegrationQueue.cStatus = pcNewStatus.
        VALIDATE ttIntegrationQueue.
    END.
END PROCEDURE.
```

## Complete Example: Class

```
/*------------------------------------------------------------------------
  File        : OrderTrackingService.cls
  Purpose     : Centraliza a comunicação de rastreamento com a API externa Tudo Entregue.
  Syntax      : NEW OrderTrackingService(INPUT poHttpClient)
  Implements  : IOrderTrackingService
  Notes       : Depende de uma instância válida de componente HTTP injetada no construtor.
  ----------------------------------------------------------------------*/

USING Progress.Lang.*.

CLASS OrderTrackingService IMPLEMENTS IOrderTrackingService:

    DEFINE PRIVATE VARIABLE oHttp AS HANDLE NO-UNDO.

    /* Construtor da Classe */
    CONSTRUCTOR PUBLIC OrderTrackingService (INPUT phHttpClient AS HANDLE):
        ASSIGN oHttp = phHttpClient.
    END CONSTRUCTOR.

    /*--------------------------------------------------------------------
      Purpose : Dispara o evento de atualização de rota para o parceiro logístico.
      Params  : 1. INPUT pcTrackingCode AS CHARACTER - Código de rastreio do parceiro
                2. INPUT pcStatusDescription AS CHARACTER - Texto descritivo do evento
                3. OUTPUT plSuccess AS LOGICAL - Indica se a API aceitou a requisição
      Notes   : Executa de forma síncrona. Caso falhe, gera log no appserver.
      --------------------------------------------------------------------*/
    METHOD PUBLIC LOGICAL SendTrackingUpdate(INPUT  pcTrackingCode      AS CHARACTER,
                                             INPUT  pcStatusDescription AS CHARACTER,
                                             OUTPUT plSuccess           AS LOGICAL):
        
        /* Lógica fictícia de consumo */
        IF pcTrackingCode = "" THEN DO:
            ASSIGN plSuccess = FALSE.
            RETURN FALSE.
        END.

        ASSIGN plSuccess = TRUE.
        RETURN TRUE.

    END METHOD.

END CLASS.
```

---

## Workflow

Follow this process for every documentation request:

### 1. Analyze the Source Code

- Identify all documentable elements: Programs (.p), Classes (.cls), Methods, Procedures, and Internal Functions.
- Identify parameters with their data types, modes (INPUT, OUTPUT, INPUT-OUTPUT), and whether they are optional/handles.
- Identify return types for functions and methods.
- Spot database tables, temp-tables, or buffers used within the scope.

### 2. Ask Clarifying Questions (if needed)

If the source code alone does not provide enough information, ask about:

- **Author:** Who wrote or maintains this code?
- **Created/Since:** When was this element introduced?
- **Version:** Which product/patch version is this for?
- **Tables/Buffers:** Which database tables or temp-tables does the element access or lock?
- **Notes:** Any non-obvious behaviors, transactions, or side effects?

### 3. Generate Documentation Blocks

- Place each block **immediately before** the element it documents
- Write descriptions in the same language as the existing code comments (Portuguese or English)

### 4. Validate

Verify each generated block against this checklist:

- [ ] Block uses the proper standard template format (/*--------------------...) matching the element type.
- [ ] Contains a clear and concise Purpose line.
- [ ] All parameters are explicitly listed under Params with their modes, names, data types, and descriptions.
- [ ] If it is a Function or Method with a return, the return type and meaning are documented under Params or Purpose.
- [ ] Special conditions (locks, transaction scopes, or exceptions) are documented under Notes.
- [ ] Block is placed immediately before the documented element without gaps.

