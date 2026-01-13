I completed the "Login Bypass" lab on PortSwigger today. By analyzing the SQL query structure, I successfully neutralized the password verification step using the comment sequence (--) and gained administrator access. It was excellent practice for understanding vulnerabilities in authentication mechanisms. 🔓

**Vulnerability:** SQL Injection (Login Bypass)

**Lab:** PortSwigger Apprentice

**Payload Used:** `administrator'--`

**Result:** Logged in as admin without password.

