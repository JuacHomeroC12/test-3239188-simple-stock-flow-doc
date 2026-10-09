# Requirements — Simple Stock Flow

> **Primary source:** [`spec/data-model.md`](../spec/data-model.md). References such as `§n`, `Q-n`, `FK-n`, `D-n`, `ADR-n`, and `DP-n` point to that model. Decisions that cannot be demonstrated from the model are identified as **assumptions**.

## 1. Purpose

Reconstruct the functional and quality requirements implied by the data model: product catalog, reference categories, internal users, immutable sales recording, and retrieval of an aggregated report. This documentation does not expand the scope beyond the available evidence (§1–§11).

## 2. Actors

- **Administrator (`admin`):** Internal operator authorized to create seller accounts. The initial administrator identity is provisioned at startup from environment configuration; it is not seeded through SQL (D-09, D-10, §9.2, DP-04).
- **Seller (`seller`):** Internal operator who authenticates and records sales (§1, §2.5).
- **Image storage service:** External dependency that stores image binaries and returns an opaque key; the model does not prescribe a specific provider (D-08).

**Assumption SR-1 — permissions are partially undefined:** DP-04 and the closed H-3 decision specify that the initial `admin` role is provisioned from the deployment environment and that an administrator creates seller accounts (§9.2, §11, §13). The model still does not fully define who may modify each catalog field, who may record sales under every circumstance, or who may view reports. Configure those permissions explicitly in the API; the `role` column alone does not enforce authorization.

## 3. User Stories

### US-01 — Authenticate

**As** an internal operator, **I want** to sign in with my username and password, **so that** I can access authorized functions.

**Acceptance criteria**
- User lookup uses an exact match on the normalized username (Q10, §6.1).
- The system verifies the password through the hash port; the domain does not receive the plaintext password (D-09, §2.5, §7).
- The hash does not appear in responses, logs, projections, or errors (§7).
- Usernames are normalized to lowercase and trimmed; they are unique (§2.5, §4).

### US-02 — Create Seller Accounts

**As** an administrator, **I want** to create seller users, **so that** internal operators can record sales.

**Acceptance criteria**
- The username is required, non-empty, normalized, and unique; the role belongs to the `admin` / `seller` set (§2.5, §4).
- The password is transformed into a hash by the appropriate port; it is not stored or logged in plaintext (D-09, §7, §9.2).
- The initial user with the `admin` role is provisioned at startup from environment variables; credentials and precomputed hashes are not included in the repository (§9.2).
- Only an administrator creates seller accounts; the `admin` role itself is provisioned by deployment and is not granted at runtime (DP-04, §9.2, H-3 in §11 and §13). The model's later register reports the anonymous-registration defect A-1 as closed (§13); this document does not claim an independent runtime retest.

### US-03 — Search Active Products

**As** an operator, **I want** to search for products by partial text and category, **so that** I can quickly locate available items.

**Acceptance criteria**
- Search excludes soft-deleted products and can filter by text and category (Q1, §6.1).
- Results are sorted by name and support pagination; the total needed for pagination is returned (Q1, §6.1).
- No product fields unsupported by the model are added; the catalog consists of name, price, stock, category, and optional image (DP-03, §1, §3).

### US-04 — Retrieve a Product

**As** an operator, **I want** to retrieve a product by its identifier, **so that** I can review its current information before an operation.

**Acceptance criteria**
- The query identifies the product by its primary key (Q2, §6.1).
- Soft-deleted products are not presented as active (D-03, ADR-003, §2.2).
- Stock cannot be represented as negative; the database constraint is the final safeguard (§2.2, §4, ADR-002).

### US-05 — Maintain the Catalog and Stock

**As** an authorized operator, **I want** to create and update permitted product data and soft-delete a product, **so that** the catalog and stock remain available for sales.

**Acceptance criteria**
- A product contains only name, price, stock, category, and optional image (DP-03).
- The name is required and trimmed; the price must be strictly positive; the category must exist (§2.2).
- Stock never falls below zero. More units than available cannot be withdrawn; stock additions use a valid domain operation (`Product.Restock`) (§2.2, ADR-002).
- Existing categories can be queried but cannot be created, renamed, or deleted in the system; they are reference data seeded by the initial migration (§2.1, D-10, §9.1).
- Product removal is a soft delete through `deleted_at`, never a physical delete (§2.2, D-03, ADR-003). The model's later debt register reports T-09 as completed (§13, D-1), while its older §10.1 output omits the column; refresh the database evidence before claiming independent verification.
- **Assumption SR-1:** The exact authorization for creating, editing, or soft-deleting products is not fully defined by the model and must be specified in the API.

### US-06 — Manage a Product Image

**As** an authorized operator, **I want** to attach or replace a product image, **so that** I can display a visual reference without storing the binary in the database.

**Acceptance criteria**
- The database stores only an opaque key, never the binary or a physical path (D-08, §1, §3).
- The absence of an image is represented by `NULL`, not an empty string (§2.2).
- When replacing an image or soft-deleting the product, the key is unlinked first and the binary is deleted afterward; if the second step fails, an orphaned binary may remain (D-08, §7.1, H-2).
- **Assumption:** Image formats, maximum sizes, and validation rules are not defined by the model.

### US-07 — Browse Categories

**As** an operator, **I want** to view available categories, **so that** I can classify and locate products.

**Acceptance criteria**
- The category catalog can be listed alphabetically (Q4, §6.1), and a category can be retrieved by identifier (Q5, §6.1).
- Five categories are seeded by the initial migration; the category repository is read-only (D-10, §2.1, §9.1).
- A category name is unique, required, and non-empty (§2.1, §4).

### US-08 — Record a Sale

**As** an authenticated seller, **I want** to record a sale with one or more products and their quantities, **so that** the transaction is recorded and stock is updated.

**Acceptance criteria**
- Before recording a sale, the products in the batch are retrieved and only active products are considered (Q3, §6.1).
- Each quantity is strictly positive, and a product cannot be repeated within the same sale (§2.3–§2.4).
- Each sale item freezes the product name and unit price in effect at the time of sale; the category label is also intended to be frozen for reporting. The physical status of `sale_item.category_name` remains disputed because §2.4 calls it engine-enforced while §3/§10.1 call it pending/absent. Subtotal and sale total are calculated, not persisted (§1, §2.3–§2.4, D-05, D-06).
- A sale must contain at least one item to be confirmed. Adding the item and withdrawing stock are part of one domain operation (§2.3).
- The sale date is stored as `timestamptz`; the system uses one currency and adds no currency columns (D-05, §3, §8).
- **Assumption SA-2:** The sale and stock changes are persisted within a single transaction. **Assumption SA-3:** The responsible operator comes from the authenticated identity, not from an arbitrary value supplied by the HTTP client.

### US-09 — Query Sales by Date Range

**As** an authorized operator, **I want** to query sales within a date range, **so that** I can review recorded transactions.

**Acceptance criteria**
- Results are filtered by date range, sorted by descending date, and support pagination and a total result count (Q7, §6.1).
- The end of the date range cannot precede its start (§1).
- Sales are retained and cannot be edited or deleted (§1, §7.1).
- **Assumption SR-1:** The role allowed to query sales and the exact visibility rules must be configured in the API because the model does not fully define them.

### US-10 — View Sale Details

**As** an authorized operator, **I want** to retrieve a sale with its items, **so that** I can review the products, quantities, and values that were recorded.

**Acceptance criteria**
- The sale is retrieved by identifier together with its items (Q6, §6.1).
- Historical name, price, and category values come from the stored sale items, not from the current catalog (§2.4, D-06, ADR-004).
- A recorded sale is immutable, and sale items cannot exist outside their sale (§2.3–§2.4, FK-2, §7.1).

### US-11 — View the Aggregated Sales Report

**As** an authorized operator, **I want** to query a sales report for a date range, **so that** I can analyze amounts by sold product.

**Acceptance criteria**
- The report is computed by a read query over sales and sale items; it is not stored in a table or calculated by traversing the entire collection in memory (D-06, Q9, §6.1).
- It groups by product identifier, frozen product name, and frozen category label; displays the aggregated amount; and sorts by descending amount (D-06, DP-02, §11.1).
- The report is not broken down by seller (DP-02).
- If the category label changes during the date range, separate groups are preserved for each frozen value; the report does not choose a single winning label. The owner decision H-1 is closed in favor of this behavior (§11.1).
- **Documentation follow-up:** the signed criterion CA-06.1 still says “one row per product” and must be revised to “one row per product and frozen label.” The decision is not pending; alignment of the criterion's wording is pending (§11.1).
- **Assumption SR-1:** The model does not fully determine which role may view the report; configure the permission in the API rather than inferring it from `user.role`.

## 4. Derived Non-Functional Requirements

The NFR identifiers below organize this documentation; the model does not provide a normative list with these same codes. These requirements are therefore reconstructed from existing rules and queries.

| ID | Requirement | Evidence and verification |
|---|---|---|
| NFR-01 | Maintain monetary amounts at two-decimal precision and in a single currency. | `numeric(18,2)`, `Money`, D-05, §1 and §3. Verify that no currency columns are added. |
| NFR-02 | Prevent negative stock even under concurrency or writes outside the normal flow. | `CHECK stock >= 0`, optimistic concurrency using `xmin`, ADR-002, §2.2 and §4. Simulate two concurrent sales competing for limited stock. |
| NFR-03 | Support search, pagination, date-range sales queries, and the aggregated report using appropriate indexes. | Q1–Q10 and §6. The report must be aggregated by the database engine; Q8 has no consumer and should be removed. |
| NFR-04 | Preserve temporal consistency using timezone-aware timestamps and valid date ranges. | `timestamptz`, UTC, §1, §3, and §8. |
| NFR-05 | Protect authentication secrets. | D-09, `password_hash`, §7 and §9.2. The hash must never appear in responses or logs and must not be indexed. |
| NFR-06 | Restrict exposure of usernames, roles, and sale authorship. | Attribute-level privacy policy in §7; access to personal data is restricted. Endpoint authorization must be confirmed in the API. |
| NFR-07 | Keep image binaries separate from relational data and allow them to be deleted when an image is replaced or a product is soft-deleted. | D-08, §1, §7.1. The database persists only an opaque key. |
| NFR-08 | Maintain a single ownership and evolution mechanism for the schema. | ADR-001, EF migrations, §3.2 and §9. All schema DDL must be managed through migrations. |

## 5. Decisions and Open Boundaries

- Rules that the database engine can express should ultimately be backed by database constraints; process-only rules remain in the domain until a viable database solution exists (§4, ADR-002, T-20).
- The data model contains internal inconsistencies concerning the column count, the placement/status of `category_name`, the unique sale-item index, `deleted_at`, and FK-3. §13 reports D-1 and D-2 as paid off on 2026-09-20, while older §10 snapshots dated 2026-09-19 and the §12 sign-off retain prior statuses. Preserve that chronology and refresh engine evidence instead of silently reconciling it (§0, §3–§6, §10, §12–§13).
- No customers, payments, multiple currencies, sale editing/deletion, seller-level report breakdown, or `created_at` / `updated_at` audit fields are introduced (DP-02, DP-03, D-05, §7–§8).
