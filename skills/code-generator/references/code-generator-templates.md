---
name: code-generator-templates
description: 'Complete OpenEdge ABL code templates for data access, CRUD operations, and structured multi-layer service logic.'
license: MIT
metadata:
  domain: Openedge
  author: Consultoria Incode
  version: '1.0.0'
  category: Reference Templates
---

# OpenEdge Code Templates

Complete OpenEdge ABL structural templates for data persistence, single-record entity handling, and transaction blocks. Use these as starting points and adapt to specific database tables, temp-tables, and business logic layers.

---

## Table of Contents

- [Template: Single Entity CRUD Layer](#template-single-entity-crud-layer)
- [Template: Master-Detail Transaction Buffer](#template-master-detail-transaction-buffer)
- [Data Validation Handlers](#data-validation-handlers)
- [Commit and Transaction Scope Handlers](#commit-and-transaction-scope-handlers)
- [Template: Business Logic Program (Procedural .p)](#template-business-logic-program-procedural-p)
- [Template: Definition File (Include .i)](#template-definition-file-include-i)

---

## Template: Single Entity CRUD Layer

A clean Data Access / Service pattern class for executing standard CRUD operations safely on a single database table using data hiding and local buffers.

```progress
/*------------------------------------------------------------------------
  File        : CustomerService.cls
  Purpose     : Handles basic CRUD operations for the Customer table.
  Syntax      : NEW CustomerService()
  Implements  : Progress.Lang.Object
  Notes       : Uses strong encapsulation and localized scopes to prevent locks.
  ----------------------------------------------------------------------*/

USING Progress.Lang.*.

CLASS CustomerService:

    /* *************************** Definitions *************************** */
    DEFINE TEMP-TABLE ttCustomer NO-UNDO
        FIELD custNum     AS INTEGER
        FIELD custName    AS CHARACTER
        FIELD custStatus  AS CHARACTER
        INDEX idxCust IS UNIQUE PRIMARY KEY custNum.

    /* *************************** Constructors ************************** */
    CONSTRUCTOR PUBLIC CustomerService():
        /* Initialization logic if needed */
    END CONSTRUCTOR.

    /* ***************************** Methods ***************************** */

    /*--------------------------------------------------------------------
      Purpose : CustomerService::GetCustomer
                Fetches a single customer record into the local temp-table buffer.
      Params  : 1. INPUT piCustNum AS INTEGER - Unique customer number
                2. OUTPUT DATASET FOR ttCustomer - Output data container
      Notes   : Uses NO-LOCK on the database to prevent row-level blocking.
      --------------------------------------------------------------------*/
    METHOD PUBLIC VOID GetCustomer(INPUT  piCustNum AS INTEGER, 
                                   OUTPUT DATASET FOR ttCustomer):
        
        EMPTY TEMP-TABLE ttCustomer.
        
        FIND FIRST customer WHERE customer.custNum = piCustNum NO-LOCK NO-ERROR.
        if AVAILABLE customer THEN DO:
            CREATE ttCustomer.
            BUFFER-COPY customer TO ttCustomer.
        END.
    END METHOD.

    /*--------------------------------------------------------------------
      Purpose : CustomerService::SaveCustomer
                Persists temp-table modifications back into the core database.
      Params  : 1. INPUT-OUTPUT DATASET FOR ttCustomer - Current state payload
                2. OUTPUT plSuccess AS LOGICAL - Result flag indicating state
      Notes   : Wraps execution context inside a strict transactional scope.
      --------------------------------------------------------------------*/
    METHOD PUBLIC VOID SaveCustomer(INPUT-OUTPUT DATASET FOR ttCustomer,
                                    OUTPUT       plSuccess AS LOGICAL):
        
        DEFINE BUFFER bCustomer FOR customer.
        ASSIGN plSuccess = FALSE.

        FIND FIRST ttCustomer NO-LOCK NO-ERROR.
        IF NOT AVAILABLE ttCustomer THEN RETURN.

        /* Execute business data consistency checks */
        IF NOT THIS-OBJECT:ValidateCustomer(DATASET ttCustomer BY-REFERENCE) THEN RETURN.

        /* Explicit localized transaction block */
        DO TRANSACTION:
            FIND FIRST bCustomer WHERE bCustomer.custNum = ttCustomer.custNum EXCLUSIVE-LOCK NO-ERROR.
            IF NOT AVAILABLE bCustomer THEN DO:
                CREATE bCustomer.
            END.
            
            BUFFER-COPY ttCustomer TO bCustomer.
            ASSIGN plSuccess = TRUE.
        END. /* Transaction End */
        
    END METHOD.

END CLASS.

```

---

## Template: Master-Detail Transaction Buffer

A master-detail model simulation processing parent headers (e.g., Orders) bound structurally to dynamic collections of children data lines (e.g., Order Items).

```progress
/*------------------------------------------------------------------------
  File        : OrderManager.cls
  Purpose     : Manages transactional scopes for compound Master-Detail Order records.
  Syntax      : NEW OrderManager()
  Implements  : Progress.Lang.Object
  Notes       : Ensures atomicity; if an item fails validation, the entire order rolls back.
  ----------------------------------------------------------------------*/

USING Progress.Lang.*.

CLASS OrderManager:

    /* *************************** Definitions *************************** */
    DEFINE TEMP-TABLE ttOrderHeader NO-UNDO
        FIELD orderNum    AS INTEGER
        FIELD custNum     AS INTEGER
        FIELD totalAmount AS DECIMAL
        INDEX idxOrderMaster IS UNIQUE PRIMARY KEY orderNum.

    DEFINE TEMP-TABLE ttOrderItem NO-UNDO
        FIELD orderNum    AS INTEGER
        FIELD itemLine    AS INTEGER
        FIELD itemCode    AS CHARACTER
        FIELD quantity    AS INTEGER
        INDEX idxOrderLine IS UNIQUE PRIMARY KEY orderNum itemLine.

    DEFINE DATASET dsOrder FOR ttOrderHeader, ttOrderItem
        DATA-RELATION relOrder FOR ttOrderHeader, ttOrderItem
            RELATION-FIELDS (orderNum, orderNum).

    /* ***************************** Methods ***************************** */

    /*--------------------------------------------------------------------
      Purpose : OrderManager::ProcessOrderCommit
                Applies the full Master-Detail structural layout data payload to the DB.
      Params  : 1. INPUT-OUTPUT DATASET FOR dsOrder - Relational tracking payload
      Notes   : Executes under explicit transaction locks, parsing items inside an active loop.
      --------------------------------------------------------------------*/
    METHOD PUBLIC LOGICAL ProcessOrderCommit(INPUT-OUTPUT DATASET dsOrder):
        
        DEFINE BUFFER bOrderHeader FOR orderHeader.
        DEFINE BUFFER bOrderItem   FOR orderItem.
        
        DEFINE VARIABLE lValidTransaction AS LOGICAL NO-UNDO INITIAL TRUE.

        FIND FIRST ttOrderHeader NO-LOCK NO-ERROR.
        IF NOT AVAILABLE ttOrderHeader THEN RETURN FALSE.

        /* Component Global Parent Transaction Guard */
        DO TRANSACTION ON ERROR UNDO, LEAVE:
            
            /* 1. Persist Master Record */
            FIND FIRST bOrderHeader WHERE bOrderHeader.orderNum = ttOrderHeader.orderNum EXCLUSIVE-LOCK NO-ERROR.
            IF NOT AVAILABLE bOrderHeader THEN DO:
                CREATE bOrderHeader.
            END.
            BUFFER-COPY ttOrderHeader TO bOrderHeader.

            /* 2. Process Detail Records Grid */
            FOR EACH ttOrderItem WHERE ttOrderItem.orderNum = ttOrderHeader.orderNum NO-LOCK:
                
                /* In-line specific field validation */
                IF ttOrderItem.quantity <= 0 THEN DO:
                    lValidTransaction = FALSE.
                    UNDO, LEAVE. /* Force immediate roll-back block exit */
                END.

                FIND FIRST bOrderItem WHERE bOrderItem.orderNum = ttOrderItem.orderNum 
                                        AND bOrderItem.itemLine = ttOrderItem.itemLine 
                                        EXCLUSIVE-LOCK NO-ERROR.
                IF NOT AVAILABLE bOrderItem THEN DO:
                    CREATE bOrderItem.
                END.
                BUFFER-COPY ttOrderItem TO bOrderItem.
            END.

            /* Final Structural Check evaluation */
            IF NOT lValidTransaction THEN DO:
                UNDO, LEAVE.
            END.
        END. /* End of transaction block */

        RETURN lValidTransaction.

    END METHOD.

END CLASS.

```

---

## Data Validation Handlers

Isolate data validation rules using private or protected methods returning logic evaluation flags before opening transactions:

```progress
    /*--------------------------------------------------------------------
      Purpose : CustomerService::ValidateCustomer
                Executes structural business rule validation arrays on customer states.
      Params  : 1. INPUT DATASET FOR ttCustomer - Isolated verification payload
      Notes   : Does not manipulate database records directly; read-only operations.
      --------------------------------------------------------------------*/
    METHOD PRIVATE LOGICAL ValidateCustomer(INPUT DATASET ttCustomer):
        
        FIND FIRST ttCustomer NO-LOCK NO-ERROR.
        IF NOT AVAILABLE ttCustomer THEN RETURN FALSE.

        /* Rule 1: Empty Data Guards */
        IF ttCustomer.custName = "" OR ttCustomer.custName = ? THEN DO:
            /* Custom log or application message instantiation here */
            RETURN FALSE.
        END.

        /* Rule 2: Code Format Verification */
        IF ttCustomer.custStatus = "INACTIVE" AND ttCustomer.custNum < 100 THEN DO:
            RETURN FALSE.
        END.

        RETURN TRUE.
    END METHOD.

```

---

## Commit and Transaction Scope Handlers

Critical rules for transaction isolation in Progress OpenEdge ABL:

```progress
    /*--------------------------------------------------------------------
      Purpose : OrderManager::ExecuteSafeCommit
                Demonstrates how to structure custom post-commit events properly.
      Params  : 1. INPUT piOrderNum AS INTEGER - Final processing target reference
      Notes   : ALWAYS split post-save notifications away from exclusive lock windows.
      --------------------------------------------------------------------*/
    METHOD PUBLIC VOID ExecuteSafeCommit(INPUT piOrderNum AS INTEGER):
        
        DEFINE VARIABLE lCommitSuccess AS LOGICAL NO-UNDO.

        /* Scope A: Locking / Mutex State Execution block */
        DO TRANSACTION:
            /* Keep this scope as micro-sized as possible */
            FIND FIRST orderHeader WHERE orderHeader.orderNum = piOrderNum EXCLUSIVE-LOCK NO-ERROR.
            IF AVAILABLE orderHeader THEN DO:
                ASSIGN orderHeader.totalAmount = orderHeader.totalAmount * 1.
                lCommitSuccess = TRUE.
            END.
        END. /* Locks are fully released right here */

        /* Scope B: Triggering External events or Satellite Systems APIs */
        IF lCommitSuccess THEN DO:
            /* Call external messaging or async services like 'Tudo Entregue' queue hooks.
               Reason: Putting API or HTTP calls inside DO TRANSACTION causes database freeze. */
        END.

    END METHOD.

```

---

## Template: Business Logic Program (Procedural .p)

A structured program containing advanced internal procedures and functions, demonstrating dynamic query building and memory cleanup.

```progress
/*------------------------------------------------------------------------
  File        : order_processing.p
  Purpose     : Handles mass processing and status updates for orders.
  Syntax      : RUN order_processing.p (INPUT piBatchId, OUTPUT pcLogMessage).
  Author(s)   : Thiago Fofano - INSTI
  Created     : 17/07/2026
  Version     : 1.0.0
  Notes       : Designed to run as a backend job or batch routine.
  ----------------------------------------------------------------------*/

/* ***************************  Definitions  ************************** */
DEFINE INPUT  PARAMETER piBatchId    AS INTEGER   NO-UNDO.
DEFINE OUTPUT PARAMETER pcLogMessage AS CHARACTER NO-UNDO.

/* Include containing standard temp-tables definitions */
{ src/shared/tt_order_definitions.i }

DEFINE VARIABLE hQuery AS HANDLE NO-UNDO.

/* ***************************  Main Block  *************************** */
RUN loadBatchItems(INPUT piBatchId, INPUT-OUTPUT DATASET dsOrder).

/* Loop through memory items and execute calculation via function */
FOR EACH ttOrderItem NO-LOCK:
    IF NOT validateItemStock(ttOrderItem.itemCode, ttOrderItem.quantity) THEN DO:
        RUN logErrorProcedure(INPUT ttOrderItem.itemCode, INPUT "Insufficient Stock").
        NEXT.
    END.
    
    RUN updateItemStatus(INPUT-OUTPUT ttOrderItem).
END.

ASSIGN pcLogMessage = "Batch " + STRING(piBatchId) + " processed successfully.".

/* ************************  Internal Functions  ********************** */

/*------------------------------------------------------------------------
  Purpose : validateItemStock
            Checks if a given item has enough unallocated stock in the warehouse.
  Params  : 1. INPUT pcItemCode AS CHARACTER - The unique item identifier
            2. INPUT piRequestedQty AS INTEGER - Quantity needed for the order
  Notes   : Returns LOGICAL. Uses NO-LOCK to prevent transactional gridlocks.
  ----------------------------------------------------------------------*/
FUNCTION validateItemStock RETURNS LOGICAL (INPUT pcItemCode AS CHARACTER, 
                                            INPUT piRequestedQty AS INTEGER):
    
    DEFINE BUFFER bItemWarehouse FOR itemWarehouse.
    
    FIND FIRST bItemWarehouse WHERE bItemWarehouse.itemCode = pcItemCode NO-LOCK NO-ERROR.
    IF NOT AVAILABLE bItemWarehouse THEN 
        RETURN FALSE.
        
    RETURN (bItemWarehouse.quantityOnHand - bItemWarehouse.quantityAllocated >= piRequestedQty).

END FUNCTION.

/* ***********************  Internal Procedures  ********************** */

/*------------------------------------------------------------------------
  Purpose : loadBatchItems
            Populates the Dataset dynamically using a dynamic tracking query.
  Params  : 1. INPUT piSourceBatch AS INTEGER - The tracking batch ID
            2. INPUT-OUTPUT DATASET dsOrder - Target dataset to populate
  Notes   : Instantiates a dynamic query handle that MUST be deleted on exit.
  ----------------------------------------------------------------------*/
PROCEDURE loadBatchItems:
    DEFINE INPUT        PARAMETER piSourceBatch AS INTEGER NO-UNDO.
    DEFINE INPUT-OUTPUT PARAMETER DATASET FOR dsOrder.

    EMPTY TEMP-TABLE ttOrderHeader.
    EMPTY TEMP-TABLE ttOrderItem.

    CREATE QUERY hQuery.
    hQuery:SET-BUFFERS(BUFFER orderHeader:HANDLE).
    hQuery:QUERY-PREPARE("FOR EACH orderHeader WHERE orderHeader.batchId = " + QUOTES(STRING(piSourceBatch)) + " NO-LOCK").
    hQuery:QUERY-OPEN().

    hQuery:GET-FIRST().
    DO WHILE NOT hQuery:QUERY-OFF-END:
        CREATE ttOrderHeader.
        BUFFER-COPY orderHeader TO ttOrderHeader.
        
        /* Fill child records sequentially */
        FOR EACH orderItem WHERE orderItem.orderNum = orderHeader.orderNum NO-LOCK:
            CREATE ttOrderItem.
            BUFFER-COPY orderItem TO ttOrderItem.
        END.
        
        hQuery:GET-NEXT().
    END.

    FINALLY:
        /* Critical cleanup block to avoid Memory Leaks with Handles */
        IF VALID-HANDLE(hQuery) THEN DO:
            hQuery:CLOSE().
            DELETE OBJECT hQuery.
        END.
    END FINALLY.
END PROCEDURE.

/*------------------------------------------------------------------------
  Purpose : updateItemStatus
            Modifies the state of an item record using dynamic parameters.
  Params  : 1. INPUT-OUTPUT TABLE ttOrderItem - Uses TABLE-HANDLE syntax for performance
  Notes   : Encapsulates database writes strictly inside its local loop block.
  ----------------------------------------------------------------------*/
PROCEDURE updateItemStatus:
    DEFINE INPUT-OUTPUT PARAMETER BUFFER bttItem FOR ttOrderItem.
    DEFINE BUFFER bRealOrderItem FOR orderItem.

    DO TRANSACTION:
        FIND FIRST bRealOrderItem WHERE bRealOrderItem.orderNum = bttItem.orderNum 
                                    AND bRealOrderItem.itemLine = bttItem.itemLine 
                                    EXCLUSIVE-LOCK NO-ERROR.
        IF AVAILABLE bRealOrderItem THEN DO:
            ASSIGN 
                bRealOrderItem.status = "PROCESSED"
                bttItem.status        = "PROCESSED".
        END.
    END.
END PROCEDURE.

/*------------------------------------------------------------------------
  Purpose : logErrorProcedure
            Writes processing exceptions directly to a persistent log line.
  Params  : 1. INPUT pcEntityKey AS CHARACTER - Context reference identifier
            2. INPUT pcReason AS CHARACTER - Detailed failure text
  Notes   : Independent transaction block.
  ----------------------------------------------------------------------*/
PROCEDURE logErrorProcedure:
    DEFINE INPUT PARAMETER pcEntityKey AS CHARACTER NO-UNDO.
    DEFINE INPUT PARAMETER pcReason    AS CHARACTER NO-UNDO.

    DO TRANSACTION:
        CREATE integrationLog.
        ASSIGN 
            integrationLog.logDate   = TODAY
            integrationLog.logTime   = TIME
            integrationLog.entityKey = pcEntityKey
            integrationLog.errorText = pcReason.
    END.
END PROCEDURE.

```

---

## Template: Definition File (Include `.i`)

A standard data contract layout used to synchronize data definitions across multiple `.p` and `.cls` components cleanly.

```progress
/*------------------------------------------------------------------------
  File        : tt_order_definitions.i
  Purpose     : Shared definition for the Order relational business contract.
  Author(s)   : Thiago Fofano - INSTI
  Created     : 17/07/2026
  Notes       : Included across service layers to guarantee structural consistency.
  ----------------------------------------------------------------------*/

/* Prevent duplicate definitions if included multiple times across namespaces */
&IF DEFINED(tt_order_definitions_i) = 0 &THEN
&GLOBAL-DEFINE tt_order_definitions_i

DEFINE TEMP-TABLE ttOrderHeader NO-UNDO
    FIELD orderNum    AS INTEGER
    FIELD custNum     AS INTEGER
    FIELD totalAmount AS DECIMAL
    FIELD batchId     AS INTEGER
    INDEX idxOrderMaster IS UNIQUE PRIMARY KEY orderNum.

DEFINE TEMP-TABLE ttOrderItem NO-UNDO
    FIELD orderNum    AS INTEGER
    FIELD itemLine    AS INTEGER
    FIELD itemCode    AS CHARACTER
    FIELD quantity    AS INTEGER
    FIELD status      AS CHARACTER
    INDEX idxOrderLine IS UNIQUE PRIMARY KEY orderNum itemLine.

DEFINE DATASET dsOrder FOR ttOrderHeader, ttOrderItem
    DATA-RELATION relOrder FOR ttOrderHeader, ttOrderItem
        RELATION-FIELDS (orderNum, orderNum).

&ENDIF

```
