# Business Rules — Simple Stock Flow

> **Primary source:** [`spec/data-model.md`](../spec/data-model.md). References such as `§n`, `FK-n`, `Q-n`, `D-n`, `ADR-n`, `DP-n`, and `T-xx` identify evidence in the supplied data model. This document consolidates business rules inferred from that model; it does not introduce new product capabilities.

## 1. Purpose and interpretation

This document states the business rules that the catalog, inventory, sales, users, and sales report must preserve. It distinguishes three statuses:

- **Database-enforced:** the model reports a PostgreSQL constraint or index that currently enforces the rule.
- **Domain-enforced:** the application domain enforces the rule; a direct database write may bypass it.
- **Pending or to verify:** the model identifies the implementation as pending, or its own sections disagree. Such a rule must not be described as a guaranteed database constraint until verified.

The actual database engine takes precedence over contradictory descriptions in the model (§0, §10). The later debt register (§13, dated 2026-09-20) marks D-1 and D-2 as paid off, but older status descriptions, the §10 query snapshots dated 2026-09-19, and the §12 sign-off were not fully synchronized. This document records the latest declared status and also preserves the verification caveat; it does not claim that the database was re-queried for this review.

## 2. Product and inventory rules

| ID | Business rule | Current enforcement / evidence |
|---|---|---|
| BR-01 | A product contains only a name, current price, stock quantity, category, and optional image. No SKU, description, reference code, or additional product attributes are part of the stated scope. | Domain/scope decision DP-03; §1, §3. |
| BR-02 | A product name is required, non-empty, and stored trimmed. | `NOT NULL` is database-enforced; non-empty validation and trimming are domain rules (§2.2, §4). |
| BR-03 | A product price must be strictly greater than zero. | Domain-enforced today; adding a database check is listed under T-20 (§2.2, §4). Do not assume the database currently rejects zero. |
| BR-04 | Stock must never be negative. | PostgreSQL `CHECK (stock >= 0)` is database-enforced. Stock withdrawals also validate available quantity in `Product.Withdraw` (§2.2, §4, ADR-002). |
| BR-05 | A stock withdrawal greater than the available stock must fail. | Domain process rule enforced by `Product.Withdraw`; the database `CHECK` is a final safeguard against negative results (§2.2, ADR-002). |
| BR-06 | Every product must reference an existing category. A category in use cannot be deleted. | Database-enforced by FK-1 with `ON DELETE RESTRICT` (§2.2, §4, §5). |
| BR-07 | Removing a product from the active catalog is a **soft delete**, not a physical deletion. Historical sales must continue to refer to the product that was sold. | D-03 and ADR-003. The later debt register says D-1 was paid off and `product.deleted_at` exists with a global filter (§13); the older §10.1 output dated 2026-09-19 omits it. The latest model status is therefore “engine, as declared in §13,” with a stale-evidence caveat until the query is refreshed. |
| BR-08 | A product image is stored outside the relational database. The database keeps only an opaque image key, not image bytes or a filesystem/HTTP path. An absent image is represented as `NULL`, not an empty string. | D-08, §2.2, §3, §7.1. |
| BR-09 | When an image is replaced or removed with a product, clear and persist the image key first, then delete the external binary. | D-08 and §7.1. The database and external storage do not share one atomic transaction; an orphaned binary is safer than a key pointing to a missing binary. |
| BR-10 | The category catalog consists of five seeded reference categories and is read-only in the application. Users cannot create, rename, or delete categories through the application. | D-10, §2.1, §9.1. Seed names in the supplied model are `General`, `Herramientas`, `Electricidad`, `Fontanería`, and `Pinturas`. |
| BR-11 | Category names must be required, non-empty, trimmed, and unique. | Name uniqueness is database-enforced by `IX_category_name`; non-empty/trimmed validation is domain-enforced today (§2.1, §4). |

## 3. User and security rules

| ID | Business rule | Current enforcement / evidence |
|---|---|---|
| BR-12 | A user represents an internal operator, not a customer or buyer. The data model defines no customer entity. | §1, §7, DP-02/DP-03. |
| BR-13 | The supported user roles are limited to `admin` and `seller`. | Domain validation through `Roles.IsValid`; currently domain-only, with database enforcement listed under T-20 (§2.5, §4). |
| BR-14 | A username is required and unique, and is normalized to lowercase with surrounding whitespace removed. | Uniqueness is database-enforced by `IX_user_username`; normalization is domain-enforced (§2.5, §4). |
| BR-15 | A password must never be handled by domain objects in plaintext. The domain receives/retains a password hash only. The hash is required and must never be included in logs, responses, projections, or error messages, and it must not be indexed. | D-09, §2.5, §7. The `NOT NULL` column is database-enforced; non-empty and privacy behavior depend on application design. |
| BR-16 | User creation, catalog permissions, and permission to view reports must be authorized by role; the exact policy must not be invented from the data model. | **Partly settled:** DP-04 and H-3 (§11, §13) state that the administrator creates seller accounts and the `admin` role is provisioned by deployment, not granted at runtime. §13 also reports defect A-1 as closed. The exact permissions for catalog operations and report access remain unspecified and must be configured explicitly. |

## 4. Sale and sale-item rules

| ID | Business rule | Current enforcement / evidence |
|---|---|---|
| BR-17 | Each sale records when it occurred and identifies the internal operator responsible for it. The operator value is mandatory and non-empty. | `sold_at` and `sold_by` exist in the documented schema; `sold_by_user_id` and FK-4 are pending T-12 (§2.3, §3, §5). Assumption SA-3 says the operator should come from the authenticated identity. |
| BR-18 | A sale can be confirmed only when it contains at least one sale item. | Domain-enforced by `Sale.EnsureConfirmable`; this cross-row rule is not expressible as a simple `CHECK` constraint (§2.3). |
| BR-19 | A product must not be repeated in the same sale. | Enforced by `Sale.AddItem` (§2.3). The later debt register reports D-2 paid off and the composite unique index present (§13); the older §6.2/§10.3 snapshot dated 2026-09-19 does not list it, and §12 retains an older status. Latest model-declared status: **engine**; the source evidence has not been fully refreshed. |
| BR-20 | Each sale item must reference a product and its quantity must be strictly greater than zero. | The later debt register reports D-2 paid off and FK-3 (`sale_item.product_id` → `product.id`, `RESTRICT`) present (§13); older §10 output and §12 still show the earlier status. Latest model-declared status for FK-3: **engine, pending a refreshed query**. `quantity > 0` remains described as domain-only in §2.4/§4. |
| BR-21 | Only active products are eligible for a new sale. The application must retrieve and validate the batch of requested product identifiers before recording the sale. | Q3 and §5. The implementation of the active-product filter depends on the soft-delete status described in BR-07. |
| BR-22 | Recording a sale item and withdrawing its stock are one domain operation. If stock is insufficient, the operation fails rather than recording an invalid sale item. | `Sale.AddItem` calls `Product.Withdraw` before adding the item (§2.2–§2.3). Assumption SA-2 requires the sale and stock changes to be persisted in a single database transaction. |
| BR-23 | A sale item preserves a snapshot of the product name and unit price at the time of sale. Later catalog edits must not rewrite historical sale values. | §2.3–§2.4, D-06, ADR-004. |
| BR-24 | The category name used by a sale report should also be frozen at the time of sale and should not track later category/catalog changes. | The intended behavior is stated in D-06 and ADR-004. Physical status remains **disputed**: §2.4 describes `sale_item.category_name` as engine-enforced, while §3/§10.1 mark it pending/absent. T-11 is not listed among the debt items paid off in §13. Do not claim this field is deployed until the schema and migration are verified. |
| BR-25 | A sale and its sale items are immutable once recorded. They cannot be edited or deleted through application operations. A sale item has no independent lifecycle outside its sale. | §2.3–§2.4, FK-2, §7.1. The sale-item-to-sale relationship is intended to cascade, although the application does not provide sale deletion. |
| BR-26 | A line subtotal is calculated as quantity multiplied by unit price, and the sale total is calculated as the sum of line subtotals. These values are calculated, not persisted as separate columns. | §1, §2.3–§2.4, D-05. |
| BR-27 | The system uses one currency and monetary amounts have two-decimal precision. No currency column or multi-currency workflow is in scope. | D-05, D-07, §1, §3. `Money` rounds to two decimals using `MidpointRounding.AwayFromZero` (§2.2). |

## 5. Report and date-range rules

| ID | Business rule | Current enforcement / evidence |
|---|---|---|
| BR-28 | A date range is invalid when its end precedes its start. | Date-range value object and §1. |
| BR-29 | The sales report is calculated by a read query over sales and sale items. It is not stored in a report table. | D-06, Q9, §6.1. |
| BR-30 | The report groups data by product identity and the frozen product/category labels, shows aggregated amounts, and sorts by amount descending. It does not break results down by seller. | D-06, DP-02, §11.1. **The business decision H-1 is closed:** group by the frozen `category_name`, so the same product may produce multiple rows when its frozen label differs. However, CA-06.1 still says “one row per product” and its signed wording remains to be updated (§11.1). |
| BR-31 | Sales are retained indefinitely and historical sales values are not rewritten when catalog information changes. | §2.4, §7.1, ADR-004. The report uses the snapshots stored with sale items. |

## 6. Cross-cutting integrity and retention rules

| ID | Business rule | Current enforcement / evidence |
|---|---|---|
| BR-32 | Concurrent sales must not cause stock to become negative. Conflicting updates must be detected or rejected, and the stock check remains the database's final guard. | Optimistic concurrency through PostgreSQL `xmin`, D-04, ADR-002, §2.2 and §4. The precise reject/retry behavior is an architecture assumption (SA-4). |
| BR-33 | The system does not introduce generic `created_at` / `updated_at` audit columns or an audit-history table without a separate requirement. | §8. `sale.sold_at` is the business timestamp. `product.deleted_at` is intended for soft deletion and is reported as present in the later §13 debt register; see the stale §10.1 evidence caveat in BR-07. |
| BR-34 | No customer management, payment processing, multiple currencies, sale editing/deletion, seller-level report breakdown, product SKU/description, or generic change auditing is included in the reconstructed scope. | DP-02, DP-03, D-05, §7–§8. |

## 7. Technical conventions are not business rules

The excerpt in §0 of the data model about table and attribute naming is a **technical naming convention**, not a business rule. It states that entity/class names use singular `PascalCase` in English, column/attribute names use singular `snake_case`, and database tables are named `category`, `product`, `sale`, `sale_item`, and `user`, while the schema remains `sales` (§0). These conventions can be referenced by the architecture/persistence documentation; they should not be mistaken for domain invariants such as positive prices, non-negative stock, or immutable sales.

## 8. Source inconsistencies that must remain visible

The supplied data model contains inconsistencies which this document does not silently resolve:

1. **Schema column count and placement:** §3 calls the physical model 22 columns, while the historical §10.1 output dated 2026-09-19 shows 21 rows. The `category_name` description is misplaced in the `product` table description and is also described as a `sale_item` field (§3, §10.1).
2. **Later debt status versus older evidence:** §13 says D-1 (`deleted_at`) and D-2 (`sale_item.sale_id NOT NULL`, the composite unique index, and FK-3) were paid off on 2026-09-20. Older status markers, the §10 outputs, and §12 were not consistently updated. If the later status is current, the derived verification targets are 22 columns, 9 constraints, and 13 indexes; these are expected counts, not measurements from this review. Refresh the evidence against the engine.
3. **Still unresolved physical field:** `sale_item.category_name` is marked as engine-enforced in §2.4 but pending/absent in §3/§10.1. It is not listed as paid off in §13 and remains disputed.
4. **User foreign key:** `sale.sold_by_user_id` and FK-4 remain pending T-12 (§3, §5, §13).
5. **Report wording:** the H-1 grouping decision is closed in §11.1, but the signed CA-06.1 wording “one row per product” still needs to be aligned with grouping by frozen labels.

The documented principle is that the actual database engine takes precedence over a contradictory description (§0, §10). Any rule above labelled **pending**, **open**, or **to verify** must be carried forward as such rather than presented as implemented fact.
