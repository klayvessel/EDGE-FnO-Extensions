# EDGEExtension — Approved Customer List (ACL) Enhancements

**Publisher:** Alithya Fullscope  
**Version:** 1.0.0  
**Layer:** USR (14)  
**Depends on:** AdvancedQualityManagement, ApplicationSuite, ApplicationCommon, ApplicationFoundation, ApplicationPlatform

---

## Overview

This Dynamics 365 Finance & Operations X++ extension enhances the standard **Advanced Quality Management (AQM) Approved Customer List** feature. It adds a configurable item lookup mode, a custom approved-items lookup form, and ACL validation helpers used across sales orders, sales quotations, and sales agreements.

Microsoft recently removed the approved customer list item lookup functionality from the Advanced Quality Management module. This extension can be used to restore that behavior, giving users the ability to look up only items approved for a customer when entering lines on sales orders, sales quotations, and sales agreements.

This extension is provided **as-is**, without warranty or support. Use it as a reference or starting point and validate it thoroughly in a non-production environment before deploying.

**Deployment compatibility:**

- This extension **can** be deployed alongside the **EDGE to AQMS Migration** module.
- This extension **cannot** be deployed if the original **EDGE for Operations** modules are present, as conflicts will occur.
- This extension should only be deployed **after** Microsoft releases the update that removes the ACL item lookup functionality from the Advanced Quality Management (AQMS) module. Deploying it before that update is applied may result in duplicate or conflicting behavior.

---

## Features

### 1. Approved Item Lookup Method setting

A new field, `PIP2ACLItemLookupMethod`, is added to the **Sales & Marketing parameters** table (`smmParametersTable`) and surfaced in the **smmParameters** form under the **Legacy Approved Customer** group.

| Enum value | Label | Behavior |
|---|---|---|
| `Standard` | Show approved items first | Opens the lookup with the Approved Items tab active (default) |
| `AllItemsFirst` | Show all items first | Opens the lookup with the All Items tab active |
| `Hide` | Do not show approved items | Skips the ACL item lookup entirely; standard item lookup runs instead |

### 2. Custom Approved Items Lookup form (`PIPApprovedItemsLookup`)

A tabbed lookup form with two tabs:

- **Approved** — lists only items currently approved for the customer (populated from `TmpPIPApprovedItems`)
- **All Items** — standard item lookup with no ACL filter

The form is triggered when the user selects an item ID on a sales order, sales quotation, or sales agreement line, provided the customer has at least one active ACL entry and the lookup method is not set to `Hide`.

A tab/combo toggle (`switchView`) lets users flip between the two tabs inline.

### 3. ACL validation and query helpers

Methods added to `QMSApprovedCustomerList` via a table extension:

| Method | Description |
|---|---|
| `pipExistValidForCustomer` | Returns `true` if the customer (by account, group, or All) has an active ACL entry for the given date range |
| `pipExistValidItemForCustomer` | Returns `true` if a specific item is approved for the customer on the given dates |
| `pipGetApprovedItems` | Populates a `TmpPIPApprovedItems` buffer with all items approved for the customer; supports an optional item filter string |
| `pipGetPriorityRelation` | Finds the most specific ACL record (Table beats Group beats All) for a given customer/item combination |
| `pipGetEffectiveAtSamePriorityLevel` | Returns the ACL record that is effective within a date range at the same priority level, optionally filtered by inventory dimension |
| `pipItemLookup` | Opens `PIPApprovedItemsLookup` as a form-run lookup bound to a string control |
| `pipShouldSkipACLItemLookup` | Cancels an `QMSItemIdLookupControlEventArgs` when the lookup method is `Hide` |

> **Performance note:** All multi-OR queries explicitly pair both `ItemCode`/`ItemRelation` and `CustomerCode`/`CustomerRelation` in each OR clause to ensure SQL Server uses an index seek rather than a clustered index scan.

### 4. ACL date resolution

A `pipACLDate()` display method is added to `SalesTable`, `SalesLine`, `SalesQuotationTable`, and `SalesQuotationLine`. It reads `QMSACLDateType` from parameters and returns the appropriate date for ACL validation:

| `QMSACLDateType` | Date returned |
|---|---|
| `OrderDate` | Document created date |
| `RequestedReceiptDate` | Receipt date requested on the line/header |
| `RequestedShipDate` | Shipping date requested on the line/header |
| `Today` | Current system date in the user's time zone |

For sales agreements, `pipACLFromDate()` and `pipACLToDate()` on `AgreementLine` return the line's effective/expiration dates, falling back to the agreement header's defaults when line dates are not yet set.

### 5. Form item ID lookup interception

Event handlers on the `ItemId` form control intercept the standard lookup on three forms:

| Form | Table in context | Trigger |
|---|---|---|
| `SalesTable` | `SalesLine` | `SalesLine_ItemId` OnLookup |
| `SalesQuotationTable` | `SalesQuotationLine` | `SalesQuotationLine_ItemId`, `SalesQuotationLine_ItemId1`, `Line_ItemId` OnLookup |
| `SalesAgreement` | `AgreementLine` | `AgreementLine_ItemId` OnLookup |

Each handler checks `pipShouldSkipACLItemLookup` first, then calls `pipExistValidForCustomer`. If the customer is on the ACL, the standard lookup is cancelled and `pipItemLookup` opens the custom form instead.

---

## Data objects

| Object | Type | Description |
|---|---|---|
| `TmpPIPApprovedItems` | TempDB table | Holds the approved item results for a single lookup session (ItemId, ItemName, NameAlias, ItemType, ItemCode, ApprovedItemGroup, ValidFrom, ValidTo) |
| `PIP2ACLItemLookupMethod` | Enum | Lookup mode: `Standard`, `AllItemsFirst`, `Hide` |
| `smmParametersTable.EDGEExtension` | Table extension | Adds `PIP2ACLItemLookupMethod` field to Sales & Marketing parameters |
| `smmParameters.EDGEExtension` | Form extension | Adds the lookup method combo box to the parameters UI |

---

## Class / extension inventory

| Artifact | Extends | Purpose |
|---|---|---|
| `PIPQMSApprovedCustomerList_Extension` | `QMSApprovedCustomerList` (table) | Core ACL query and lookup helpers |
| `PIPAgreementLine_Extension` | `AgreementLine` (table) | ACL from/to date helpers for agreement lines |
| `PIPSalesTable_Extension` | `SalesTable` (table) | `pipACLDate()` for sales orders |
| `PIPSalesLine_Extension` | `SalesLine` (table) | `pipACLDate()` for sales order lines |
| `PIPSalesQuotationTable_Extension` | `SalesQuotationTable` (table) | `pipACLDate()` for quotation headers |
| `PIPSalesQuotationLine_Extension` | `SalesQuotationLine` (table) | `pipACLDate()` for quotation lines |
| `PIPSalesTableForm_Extension` | `SalesTable` (form) | Intercepts ItemId lookup on sales orders |
| `PIPSalesQuotationTableForm_Extension` | `SalesQuotationTable` (form) | Intercepts ItemId lookup on sales quotations |
| `PIPSalesAgreementDataSourceEventHandler` | — (static class) | Intercepts ItemId lookup on sales agreements |
