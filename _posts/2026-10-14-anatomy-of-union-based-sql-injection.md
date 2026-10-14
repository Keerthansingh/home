---
layout: post
title: "Anatomy of a Classic: Exploiting Union-Based SQL Injection in an Educational Web Portal"
date: 2026-10-14 12:00:00 +0000
categories: [pentesting, web-security]
---

# Anatomy of a Classic: Exploiting Union-Based SQL Injection in an Educational Web Portal

*Published by Keerthan Singh*

When conducting web application penetration testing, legacy systems and custom-built academic portals frequently harbor classic vulnerabilities. While modern frameworks often abstract database queries safely, older custom applications built with raw SQL interactions can leave massive security gaps. 

In this post, we’ll look at a real-world case study of a **Union-Based SQL Injection (SQLi)** vulnerability discovered in a legacy university web portal, walking through the manual exploitation process, database enumeration, and proper remediation.

---

## 1. Overview & Discovery

During a routine security assessment of an educational web application, attention was drawn to an event-viewing endpoint used to fetch news articles and updates:
`https://vulnerable-portal.internal/viewevent.php?type=news&id=45`

The `id` parameter immediately stood out as a prime candidate for input validation testing. Injecting a single quote (`'`) broke the application's page rendering, confirming that user input was being concatenated directly into the backend SQL query without proper sanitization.

---

## 2. Manual Exploitation: Step-by-Step

To verify the vulnerability and understand its scope, a manual exploitation approach was utilized.

### Step A: Confirming the Vulnerability
By injecting a basic boolean bypass (`' or '1'='1`), the application successfully returned valid server responses instead of throwing a standard error, confirming that the parameter was vulnerable to SQL injection[cite: 1].

### Step B: Determining Column Count (`ORDER BY`)
Before extracting data via a `UNION` query, it is necessary to determine how many columns the original SQL query returns. Using the `ORDER BY` clause[cite: 1]:
* `' order by 4-- -` resulted in a normal page load.
* `' order by 5-- -` caused the content to fail to display.

This confirmed that the original query returned exactly **4 columns**[cite: 1].

### Step C: Finding Reflected Columns (`UNION SELECT`)
To figure out which columns are rendered visibly on the webpage, a test payload was injected:
`-1' union select 1,2,3,4-- -`

Upon inspection, values corresponding to **column 2 and column 3** were dynamically reflected back into the HTML response[cite: 1]. This gave us our injection "window" to extract data.

---

## 3. Database & Schema Enumeration

With the reflection points identified, we could leverage `information_schema` to map out the database architecture[cite: 1].

1. **Enumerating Databases:** 
   Using a `GROUP_CONCAT` payload, the available databases on the backend server were successfully listed[cite: 1], revealing schemas such as `information_schema`, `csedb`, `jayalal`, `mrbs`, and internal trackers[cite: 1].
2. **Targeting Specific Tables:** 
   Drilling down into the custom database (`jayalal`), we queried `information_schema.tables`[cite: 1], exposing tables related to administration, user directories, feedback, and course listings (including an `admin` table and a large user registry)[cite: 1].

### Extracting Administrative Credentials
By querying the columns of the `admin` table (`information_schema.columns`) and concatenating the `username` and `password` fields[cite: 1], administrative credentials (`admin:hackmetheo`) were successfully retrieved directly through the vulnerable parameter[cite: 1].

*Note: Automated tools like `sqlmap` can also validate and scale this entire discovery workflow (`--dump-all`), rapidly extracting thousands of directory records[cite: 1].*

---

## 4. The "Read-Only" Misconception (A Note on Risk)

A common pushback from developers or administrators when discovering SQLi is: *"The data is read-only, and it's mostly public information anyway—why does it matter?"*

This case study highlights why that mindset is dangerous:
* **Privilege Escalation:** Extracting administrator credentials or session data allows an attacker to bypass application boundaries completely. 
* **Data Privacy:** Even if some data is public, internal configurations, staff phone numbers, system schemas, and user directories should never be exposed indiscriminately.
* **Integrity Risk:** Depending on the database server user privileges and configuration, SQLi can sometimes be escalated to interact with the underlying file system.

---

## 5. Remediation: How to Fix It

Vulnerabilities like this cannot be fixed effectively with basic input filtering or regex blocklists, which can often be bypassed. The permanent solution is to separate data from code entirely:

1. **Use Prepared Statements (Parameterized Queries):** Ensure user input is never concatenated directly into SQL statements. When using PHP (common in legacy portals), use PDO or MySQLi with prepared statements:
   ```php
   $stmt = $pdo->prepare('SELECT * FROM events WHERE id = ?');
   $stmt->execute([$id]);
   $event = $stmt->fetch();
