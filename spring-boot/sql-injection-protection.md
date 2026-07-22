# Does Spring Boot Protect Against SQL Injection?

## The short answer

Not directly — there's no Spring Boot filter, annotation, or "SQL sanitizer" scanning your queries for attacks. What actually protects you is **parameterized queries** (prepared statements), and Spring Data JPA/`JdbcTemplate` generate those automatically *when used as intended*. The protection comes from how the query-building abstractions work, not from a bolted-on security feature — and it's entirely possible to write SQL-injection-vulnerable code inside a Spring Boot app if you bypass those abstractions with string concatenation.

Spring Security (authentication/authorization) is a separate concern entirely — it has nothing to do with SQL injection, which is purely about how a query string and its data get sent to the database.

## What a prepared statement actually is

A prepared statement is a SQL statement with placeholders (`?` or named parameters) instead of literal values, sent to the database and parsed/compiled *once* — the database returns a handle to that compiled statement rather than executing it immediately. You then **execute** that same handle one or more times, supplying different parameter values (bound separately, in their own message) each time, without the SQL text itself being re-sent or re-parsed.

```java
// Plain JDBC — the raw API everything else (JdbcTemplate, Hibernate) sits on top of
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement(
         "SELECT * FROM users WHERE username = ? AND status = ?")) { // 1. prepare once: SQL text with placeholders

    stmt.setString(1, username); // 2. bind values separately — never concatenated into the SQL string
    stmt.setString(2, "ACTIVE");

    ResultSet rs = stmt.executeQuery(); // 3. execute — driver sends only the bound values, not new SQL text
    while (rs.next()) {
        System.out.println(rs.getString("email"));
    }
}
```

### What actually crosses the wire

`stmt.setString(1, username)` doesn't talk to the database at all — it's purely local bookkeeping, storing the value inside the `PreparedStatement` Java object until you call `execute`. The database only sees two distinct messages, sent at two different times:

```
-- 1. At prepare time (conn.prepareStatement(...)) — sent once
Client → DB:  PARSE "SELECT * FROM users WHERE username = ? AND status = ?"
DB → Client:  OK, statement handle #42, expects 2 parameters (VARCHAR, VARCHAR)

-- 2. At execute time (stmt.executeQuery()) — sent on every call, can repeat with different values
Client → DB:  EXECUTE handle #42 WITH ["alice", "ACTIVE"]
DB → Client:  <result rows>

-- a second execute, different values, reusing the same handle — no SQL text sent again at all
Client → DB:  EXECUTE handle #42 WITH ["bob", "ACTIVE"]
DB → Client:  <result rows>
```

The `EXECUTE` message contains **no SQL syntax whatsoever** — no `SELECT`, no `WHERE`, no quotes — just a statement handle and two raw values in a parameter slot the driver's wire protocol tags as data, not text-to-be-parsed. There is nothing for a malicious value to "escape into" at that point, because the database's SQL parser already finished its one and only pass — back at `PARSE` time, against the placeholder template, before any user input existed anywhere in the exchange.

Compare that to a plain `Statement`, where the value has to be baked into the SQL string by hand before it's sent:

```java
// Plain Statement — no placeholders exist; the value is part of the SQL text itself
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT * FROM users WHERE username = '" + username + "'"); // vulnerable
```

On the wire, that's a *single* message, with no separate parse step at all:

```
Client → DB:  QUERY "SELECT * FROM users WHERE username = 'bob' OR '1'='1'"
DB → Client:  <result rows — including every user, not just bob>
```

The *entire* thing sent to the database is raw SQL text that its parser reads character by character — and by the time it's parsing, `'bob' OR '1'='1'` and the query's own `WHERE` clause are indistinguishable from each other; both are just SQL syntax in the same string. That's the concrete meaning of "data and code get merged": with a prepared statement, user input travels in a channel the parser never looks at as syntax; with concatenation, it travels in the *same* channel as the SQL keywords themselves.

### Worked example: the classic `' AND 0=0 --` login bypass

A concrete walkthrough of a real payload against a typical login query, `WHERE name = '<name>' AND password = '<password>'`, with `name` set to `John' AND 0=0 --`:

**Concatenated (vulnerable)** — the actual text sent to the database becomes:
```sql
SELECT * FROM users WHERE name = 'John' AND 0=0 --' AND password = 'whatever'
```
The parser reads this structurally: `'John'` closes the string literal, `AND 0=0` is genuine SQL (always true), and `--` starts a SQL line comment — **everything after it is discarded**, including `' AND password = 'whatever'`. What actually runs is just:
```sql
SELECT * FROM users WHERE name = 'John' AND 0=0   -- equivalent to: WHERE name = 'John'
```
The password check is gone entirely — if a user named John exists, the attacker is now logged in as John with no password at all. The `0=0` isn't doing the real work here (the name match alone would have found John anyway); the `--` is what matters, since it erases whatever check was meant to come after the injection point.

**Prepared statement (safe)** — `WHERE name = ? AND password = ?` was already parsed and locked in at prepare time, as exactly two value-blanks. At execute time the database is only ever asked: *does any row's `name` column equal the exact string `John' AND 0=0 --`?* Since no real user has that literal 22-character string as their name, it matches zero rows. The quote, `AND`, and `--` inside it are just characters in one value — never re-read as syntax — so there's nothing to comment out and no bypass.

**Two separate benefits fall out of this, and it's worth keeping them distinct:**

1. **Security**: because parameter values are sent in their own message and bound directly into the placeholder, a malicious value like `' OR '1'='1` can never "break out" into being interpreted as SQL — no matter what it contains, the database only ever treats it as one literal value for that one placeholder. String concatenation (the plain-`Statement` example above) skips this separation entirely: "data" and "code" are merged into a single string before the database ever sees it, so anything shaped like SQL syntax inside user input becomes part of the actual query.
2. **Performance**: parsing and query-planning a SQL statement isn't free. A prepared statement lets the database do that work once and then reuse the same compiled statement/plan across many executions with different parameter values — valuable for a query shape that runs frequently (e.g. inside a loop, or thousands of times a day for the same endpoint). Batch inserts are the clearest example: prepare one `INSERT ... VALUES (?, ?)` statement, then call `addBatch()`/bind new values in a loop and `executeBatch()` once, instead of re-parsing an `INSERT` for every single row.

```java
// Reusing one prepared statement across many executions — prepare once, bind+execute repeatedly
try (PreparedStatement stmt = conn.prepareStatement("INSERT INTO logs (message) VALUES (?)")) {
    for (String message : messages) {
        stmt.setString(1, message);
        stmt.addBatch(); // queue this set of bound values
    }
    stmt.executeBatch(); // send all of them in one round trip, against the one compiled statement
}
```

**In a Spring Boot app**, you rarely call `connection.prepareStatement()` directly — `JdbcTemplate.query(sql, params, ...)` and Spring Data JPA/Hibernate's generated JPQL/SQL both use `PreparedStatement` under the hood automatically. The code examples in this file's "Safe patterns" table above are all, at the JDBC level, doing exactly what the raw example does here.

**A nuance on "compiled once"**: not every database fully plans a query the instant you prepare it. Some (e.g. PostgreSQL) may defer full plan generation until the first execution, and can switch between a generic plan (reused as-is) and per-value "custom" plans after enough executions reveal that a value-specific plan would be faster. The security guarantee (placeholders are never re-parsed as SQL) holds regardless; it's specifically the performance benefit that varies by database and driver.

## Safe patterns (parameterized automatically)

| Pattern | Why it's safe |
|---|---|
| Derived query methods — `findByUsername(String username)` | Spring Data generates a parameterized JPQL query under the hood; the value is always bound, never concatenated |
| `@Query("SELECT u FROM User u WHERE u.username = :username")` + `@Param` | Named/positional JPQL parameters — parameterized |
| `@Query(value = "...", nativeQuery = true)` **with `:name`/`?1` placeholders** | Native SQL is just as safe as JPQL *when parameterized* — `nativeQuery = true` doesn't mean less safe by itself |
| `JdbcTemplate`/`NamedParameterJdbcTemplate` with `?`/named placeholders, values passed as separate arguments | The driver sends SQL and parameters separately — a true prepared statement |
| Criteria API / `Specification` / QueryDSL | Queries built programmatically, with no string concatenation possible in the first place |

```java
// Safe — value is bound as a parameter, never merged into the SQL text
@Query("SELECT u FROM User u WHERE u.username = :username")
User findByUsername(@Param("username") String username);

jdbcTemplate.queryForObject(
    "SELECT * FROM users WHERE username = ?", // placeholder
    new Object[] { username },                // bound separately
    User.class
);
```

## Unsafe patterns (Spring Boot won't stop these)

| Pattern | Why it's still vulnerable |
|---|---|
| `@Query(value = "SELECT * FROM users WHERE name = '" + name + "'", nativeQuery = true)` | The value is concatenated into the SQL text before the database ever sees it |
| `entityManager.createNativeQuery("... WHERE name = '" + name + "'")` | Same problem, one level lower |
| `jdbcTemplate.queryForList("SELECT * FROM users WHERE name = '" + name + "'")` | No separate parameter was ever passed — the whole string *is* the query |
| Sorting/filtering by a user-supplied **column or table name** | Placeholders only bind *values*, not identifiers — `ORDER BY ?` isn't valid JDBC. A user-controlled column name must be checked against an allow-list manually; parameter binding can't help here at all |
| Dynamic `LIKE` queries built by concatenating unescaped wildcards | A related string-building mistake (wildcard injection), not fixed by binding alone unless the wildcard characters are also escaped |

```java
// Vulnerable — nativeQuery = true doesn't make this any safer, the string is still hand-built
@Query(value = "SELECT * FROM users WHERE name = '" + "#{#name}" + "'", nativeQuery = true) // don't do this
```

## Common gotchas

- **`nativeQuery = true` is not inherently less safe, and JPQL is not inherently safer** — what matters is whether the query uses placeholders/binding, not which query language it's written in.
- **Bean Validation (`@Pattern`, `@Size`) is not a substitute for parameterized queries** — it can shrink the attack surface (e.g. rejecting obviously malformed input) but should never be relied on as *the* SQL injection defense; the fix is always parameterization at the query layer.
- **Identifiers (column/table names, `ORDER BY` direction) can't be parameterized the normal way** — JDBC placeholders are for values only. These need explicit allow-list validation against a fixed set of known-safe names.
- **A least-privilege database user is defense in depth, not prevention** — it limits blast radius if an injection does slip through, but doesn't stop the injection itself.

## Real-life analogy

A prepared statement is like a bank pre-printing a fill-in-the-blank withdrawal slip once — a box for "account number," a box for "amount" — and then reusing that same printed form for every customer all day, rather than hand-drafting a brand-new form from scratch for each person (that reuse is the performance benefit). No matter what a customer writes inside the "amount" box — even the literal words "and also transfer $10,000 to account X" — the teller only ever reads it as the *value* to put in that one box; it can never become a new instruction on the form (that's the security benefit). Building a query by string concatenation is like a teller who takes a customer's freeform sentence and retypes it directly into the bank's own official instruction sheet before it's imprinted — if that sentence happens to contain something shaped like an instruction, the bank's back office can no longer tell the original form apart from what the customer tacked on, because they were merged into one document before anyone could draw that line.
