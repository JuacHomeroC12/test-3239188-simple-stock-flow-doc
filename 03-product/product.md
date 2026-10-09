# Product Definition — Simple Stock Flow

> **Reconstruction basis:** [`spec/data-model.md`](../spec/data-model.md). This document presents a product vision inferred from the model; anything not confirmed by the model is marked as an assumption.

## 1. Vision

Simple Stock Flow is an internal tool for maintaining a product catalog with stock, recording sales attributed to operators, and querying an aggregated report of sold amounts over a date range. The system preserves the values that described a sale when it occurred, even if the catalog changes later (§1, §2, §6, §11).

**Vision statement:** Enable an authorized operator to find active products, record sales with inventory control, and consult sales history and reports while preserving stock integrity and the stability of historical records.

## 2. Problem to Be Solved

Without a consistent record, the catalog and sales can diverge: sold units may not be reflected in stock, a product may end up with negative stock, or a later change to a price/name may alter the interpretation of a past transaction. The model introduces domain rules and database constraints to reduce these risks (`Product.Withdraw`, `Sale.AddItem`, `CHECK stock >= 0`, `xmin`, §2.2–§2.4, ADR-002).

The model also defines specific queries for catalog search, lookup by identifier, sales within a date range, and the aggregated report (Q1–Q10, §6.1). The report is computed in the database and uses the frozen historical values stored for each sale item (D-06, ADR-004, §11.1).

## 3. Intended Users

- **Administrator (`admin`):** Internal identity allowed to create seller accounts. The initial administrator is provisioned from the environment at startup (DP-04, D-09, §9.2).
- **Seller (`seller`):** Internal operator who authenticates and records sales (§1, §2.5).
- **Permission assumption:** The API is expected to distinguish the operations available to `admin` and `seller`, but the full permission matrix for catalog, sales, and reports cannot be derived entirely from the model. It must be explicitly configured; the existence of a `role` field does not itself enforce authorization.

The existence of registered buyers or customer accounts cannot be inferred: `User` represents an internal operator, and a sale records who registered it, not who purchased it (§1, §7).

## 4. Included Capabilities

1. Authentication for internal operators and password handling through hashing (D-09, §2.5, §7, §9.2).
2. Product maintenance for name, price, stock, category, and optional image; search for active products by partial text and category (DP-03, Q1–Q3, §2.2, §6.1).
3. Read access to the categories seeded by the initial migration; they are read-only reference data, not a user-editable catalog (§2.1, D-10, §9.1).
4. Recording sales with one or more items, positive quantities, stock protection, and values frozen at the time of sale (§2.3–§2.4).
5. Querying sales history by date range and retrieving sale details with their items (Q6–Q7, §6.1).
6. An aggregated report by product, frozen name, and frozen category, ordered by amount descending and without a seller breakdown (D-06, DP-02, Q9, §11.1).
7. External image storage referenced by an opaque key (D-08, §1, §7.1).


## 5. Explicit Boundaries

According to the decisions reflected in the model, the product **does not** include:

- An end-customer entity, buyer data, or customer management (§1, §7).
- Payments, cards, cash register operations, or payment reconciliation (§7).
- Multiple currencies or currency columns in tables (D-05, §3).
- Additional product attributes such as description, SKU, or reference code (DP-03, §1).
- Editing or deleting recorded sales; sales and their items are immutable (§2.3–§2.4, §7.1).
- Reports broken down by seller (DP-02) or a persisted report table (D-06).
- Generic `created_at` / `updated_at` audit columns; the model recognizes `sold_at` as the business timestamp and `deleted_at` for soft deletion (D-03, §8).
- Category administration through write operations; the five categories are created as seed data (§2.1, D-10, §9.1).
## 6. Product Acceptance Indicators

- A valid sale reduces stock for its products and never causes stock to fall below zero (§2.2–§2.3, ADR-002).
- The history preserves the name, price, and category each product had at the time of sale (§2.4, D-06, ADR-004).
- Search and queries follow their declared access patterns and pagination rules (Q1, Q7, Q9, §6.1).
- A soft-deleted product no longer appears in active-product searches, but its historical information is not destroyed (D-03, ADR-003, §2.2, §7.1).
- Authentication secrets are never exposed in responses or logs (§7).

## 7. Assumptions Requiring Confirmation

- The exact authorization matrix for administrators and sellers.
- The visual form of the interface and supported image formats/sizes; the model does not specify them.
- The operational meaning of “report” in the interface; the data defines aggregation by product and frozen category, but not its visual presentation.
- The final wording of the report acceptance criterion in light of decision H-1 (§11.1): grouping by product and frozen label can produce more than one row for the same product.
