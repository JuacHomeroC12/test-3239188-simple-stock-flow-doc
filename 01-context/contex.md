# System Context — Simple Stock Flow

> **Basis:** [`spec/data-model.md`](../spec/data-model.md). This document reconstructs the system context from its entities, rules, and access patterns. Any element not established by the model is explicitly identified as an assumption.

## 1. Overview

Simple Stock Flow is an internal product-catalog, inventory-control, and sales-recording system. It allows authenticated operators to find active products, record which items were sold, reduce stock consistently, and consult sales history and an aggregated report by product. Each sale preserves a copy of the commercial values that applied to the product when the transaction took place (§1, §2, §6, §11).

The system relies on PostgreSQL 16, using the `sales` schema and five main tables: `category`, `product`, `sale`, `sale_item`, and `user` (§0, §2–§3). The model specifies an API, application logic, and adapters for persistence, password hashing, and external image storage (§0, §2.5; architectural details are documented in `05-architecture/architecture.md`).

## 2. Actors and External Dependencies

| Element | Interaction with the system | Evidence |
|---|---|---|
| Administrator (`admin`) | Manages seller accounts according to decision DP-04. The initial account is provisioned from the environment at startup. | §2.5, DP-04, §9.2 |
| Seller (`seller`) | Internal operator who authenticates and records sales. | §1, §2.5 |
| PostgreSQL 16 | Persists the catalog, categories, sales, sale items, and users; enforces the stated integrity constraints. | §0, §3–§6, §10 |
| External image service | Stores and deletes image binaries; the database retains only an opaque key. | D-08, §1, §7.1 |
| Hash adapter | Generates/verifies password hashes without exposing plaintext passwords to the domain. | D-09, §1, §7, §9.2 |

**Context assumption:** The interface that consumes the API (web, desktop, or another type) and the specific image-storage provider are not defined by the model. No technology product is selected for either one.

## 3. Inside the System Boundary

- Authenticate internal operators and look up users by normalized username (Q10, §6.1).
- Maintain products and stock; search active products by partial text/category, retrieve a product by identifier, and retrieve products in a batch (Q1–Q3, §6.1).
- Read the five seeded categories, without user-facing category-maintenance operations (D-10, §2.1, §9.1).
- Record sales containing one or more items, freeze commercial values, attribute each sale to an operator, and control stock (§2.3–§2.4).
- Query sales by date range, retrieve sale details, and obtain an aggregated report for a date range (Q6, Q7, Q9, §6.1).
- Manage images through opaque references to external storage (D-08).
- Protect authentication information and apply the privacy and retention rules in §7.

## 4. Outside the System Boundary

- End-customer registration, buyer records, or external customer profiles: the model's user is an internal operator (§1, §7).
- Payments, card processing, cash register operations, or payment reconciliation: no supporting entities or columns are present (§7).
- Multi-currency operation: the model uses a single currency and has no currency columns (D-05, §3).
- Additional product attributes such as description, SKU, or reference code (DP-03).
- Editing or deleting recorded sales; sales and their items remain immutable (§2.3–§2.4, §7.1).
- Reports broken down by seller or persistence of the report as a table (DP-02, D-06, Q9).
- Editable maintenance of the category catalog; the set is created by the initial migration (D-10, §2.1, §9.1).
- Generic auditing based on `created_at` / `updated_at` or a change log, since the model excludes it (§8).

## 5. Main Context Flows

1. **Authentication:** An operator submits credentials; the application looks up the user and uses the hash port to verify the password. The hash is neither returned nor logged (D-09, Q10, §7).
2. **Catalog:** The application searches for or retrieves active products and reference categories. The category foreign key protects the relationship; soft deletion excludes a product from active queries without destroying its history (Q1–Q5, FK-1, D-03).
3. **Sale:** The application reads active products in a batch, validates quantities and stock, adds items with frozen values, and saves the operation while preventing negative stock (§2.2–§2.4, Q3, D-04, ADR-002).
4. **Queries:** An authorized user requests sales for a date range, a sale detail, or a report. The report aggregates data in the database engine by product and frozen historical values; it does not calculate sales by seller (Q6, Q7, Q9, D-06, DP-02, §11.1).
5. **Image:** The application stores the binary in external storage and retains its key. When replacing or deleting an image, it first clears the reference and then attempts to delete the binary (D-08, §7.1).

## 6. Explicit Boundaries and Uncertainties

- The exact permission matrix for each operation is not fully defined by the model. The role is limited to `admin` and `seller`; the API must enforce authorization. **Assumption SR-1:** the administrator is expected to manage the catalog and accounts, but who can view reports and who can modify each field requires confirmation.
- The retention policy for an orphaned image binary, if deletion fails after the reference is removed, is unresolved. The model identifies this as H-2, an operational concern outside the schema (§7.1, §11).
- Some rule statuses and structural details are internally contradictory. The architecture must disclose these contradictions rather than replace the actual database state with a documentary inference (§3, §4, §10, §13).
- The exact report grouping criterion must reconcile any earlier requirement of “one row per product” with the decision to group by product and frozen category (§11.1).
