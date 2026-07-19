# MySQL Table vs MongoDB Document

Both ultimately store data as field → value pairs, so the difference isn't in that basic idea. It's in structure, schema enforcement, and how relationships are modeled.

## 1. Structure: flat vs nested

- **MySQL row**: always flat. Every column holds a single scalar value (or blob) — no lists or sub-objects inside a cell.
- **MongoDB document**: values can be nested objects or arrays of objects, arbitrarily deep.

MySQL example:
```
users table
| id | name  | city   |
|----|-------|--------|
| 1  | Sanjay| Mumbai |
```

MongoDB example:
```json
{
  "_id": 1,
  "name": "Sanjay",
  "address": { "city": "Mumbai", "pincode": "400001" },
  "orders": [
    { "item": "Book", "price": 300 },
    { "item": "Pen", "price": 20 }
  ]
}
```

Representing the same nested data in MySQL requires separate `addresses` and `orders` tables joined via foreign keys, since a cell can't hold a list or nested object.

## 2. Schema: enforced vs flexible

- MySQL: columns and types are fixed for the whole table. Every row must conform. Adding a new field means altering the table.
- MongoDB: each document in a collection can have a different set of fields. No migration needed to add a field to just one document.

## 3. Relationships: joins vs embedding

- MySQL: related data lives in its own table linked by a foreign key (e.g., `orders.user_id`); reassembled via `JOIN` at query time.
- MongoDB: related data is typically embedded directly inside the parent document (e.g., the `orders` array above), so one read returns everything with no join. References to other collections (like a foreign key) are also possible, but embedding is the idiomatic pattern for tightly-coupled data.

## 4. When to use which

- MySQL: naturally tabular/relational data, need strong consistency across related tables (e.g., financial ledgers).
- MongoDB: naturally hierarchical or variable-shaped data per record, want to fetch a whole object in one read (e.g., product catalogs with differing attributes per product).

## Summary

The core difference isn't "field → value" (same in both) — it's whether a value can be nested/multi-valued (Mongo: yes, MySQL: no) and whether every record must share the same shape (MySQL: yes, Mongo: no).
