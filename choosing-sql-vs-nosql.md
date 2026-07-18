# Choosing SQL (MySQL) vs NoSQL (MongoDB) for a Problem

See also: [[mysql-table-vs-mongo-document]] for the structural differences.

This isn't a single rule — it's a checklist you weigh together against your actual access pattern.

## Decision checklist

| Question | Lean MySQL | Lean MongoDB |
|---|---|---|
| Does every record have basically the same fields? | Yes | No — fields vary a lot per record |
| Do you need strict, multi-table transactions (e.g., debit here, credit there, both or neither)? | Yes | No / rare |
| Will you frequently query across entities (joins, aggregates, reports)? | Yes | No — mostly fetch one entity at a time |
| Is the data relationship many-to-many and complex? | Yes | No — mostly parent→children, self-contained |
| Do you expect the schema to change often as the product evolves? | No | Yes |
| Do you need to scale writes horizontally across many servers? | Harder | Easier |

## Worked example: e-commerce app

**Orders & Payments → MySQL**
- Every order has the same shape: `order_id`, `user_id`, `amount`, `status`.
- Needs an all-or-nothing transaction: reduce inventory, create the order, charge payment — together or not at all. ACID transactions across tables handle this.
- Reports like "total revenue per user per month" need `JOIN` + `GROUP BY` across `orders`, `users`, `payments` — SQL's strength.

**Product Catalog → MongoDB**
- A "Laptop" product has `ram`, `cpu`, `screenSize`. A "T-Shirt" product has `size`, `color`, `fabric`. Forcing this into one MySQL table means many nullable columns or extra join tables per category.
- Product pages fetch one product at a time as a whole (specs, images, reviews) — a single document read instead of reassembling from several joined tables.
- New product categories can be added over time with new attribute sets, without a schema migration.

## The core question

"When I read this data, do I read it as one self-contained blob, or do I need to slice/join across many related records?"

- Self-contained, variable-shaped → document (MongoDB).
- Needs strict cross-record consistency and relational querying → table (MySQL).

Many real systems use both — e.g., MySQL for orders/payments/users, MongoDB for product catalog/content — rather than forcing one database to handle everything.
