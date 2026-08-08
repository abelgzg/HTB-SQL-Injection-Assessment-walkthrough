# 🔐 HTB SQL Injection Fundamentals — Skills Assessment

> **Hands-on SQL Injection assessment completed in an authorized Hack The Box Academy lab environment.**

[![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Academy-9FEF00?logo=hackthebox\&logoColor=black)](https://academy.hackthebox.com/)
[![Kali Linux](https://img.shields.io/badge/OS-Kali%20Linux-557C94?logo=kalilinux\&logoColor=white)](https://www.kali.org/)
[![Burp Suite](https://img.shields.io/badge/Tool-Burp%20Suite-orange?logo=burpsuite\&logoColor=white)](https://portswigger.net/burp)
[![SQL](https://img.shields.io/badge/Language-SQL-336791?logo=mysql\&logoColor=white)](https://www.mysql.com/)
[![Status](https://img.shields.io/badge/Status-Completed-success)]()

---

## 📌 Overview

This repository contains my personal notes and technical walkthrough from the **SQL Injection Fundamentals Skills Assessment** on **Hack The Box Academy**.

The assessment combines multiple techniques covered throughout the module, including:

* 🔎 SQL injection identification
* 🔐 Authentication and registration bypass
* 📨 HTTP request manipulation
* 🧰 Burp Suite interception and modification
* 💉 UNION-based SQL injection
* 🗄️ Database enumeration
* 📋 Table and column enumeration
* 🔑 Credential/hash extraction
* ⚙️ Server configuration discovery
* 📁 File operations through SQL injection
* 🌐 Nginx web-root discovery
* 💻 Remote command execution

> [!WARNING]
> **Educational use only.**
>
> All testing documented in this repository was performed against an authorized Hack The Box lab environment. The techniques described here should only be used against systems where you have explicit permission to test.

---

## 🧭 Navigation

| Section                                                           | Description                          |
| ----------------------------------------------------------------- | ------------------------------------ |
| [🎯 Objectives](#-objectives)                                     | Assessment goals                     |
| [🛠️ Tools Used](#️-tools-used)                                   | Tools and technologies               |
| [1️⃣ Target Connection](#1️⃣-connecting-to-the-target)            | Initial HTTPS discovery              |
| [2️⃣ Login Investigation](#2️⃣-investigating-the-login)           | Authentication testing               |
| [3️⃣ Registration Bypass](#3️⃣-bypassing-the-invitation-code)     | Invitation-code injection            |
| [4️⃣ Injection Point](#4️⃣-finding-an-sql-injection-point)        | Identifying the vulnerable parameter |
| [5️⃣ Query Structure](#5️⃣-understanding-the-query-structure)     | Understanding the SQL context        |
| [6️⃣ Database Enumeration](#6️⃣-identifying-the-current-database) | Identifying the current database     |
| [7️⃣ Table Enumeration](#7️⃣-enumerating-tables)                  | Discovering database tables          |
| [8️⃣ Column Enumeration](#8️⃣-enumerating-columns)                | Discovering useful columns           |
| [9️⃣ File Privileges](#9️⃣-checking-secure_file_priv)             | Checking `secure_file_priv`          |
| [🔟 Nginx Root](#-finding-the-nginx-web-root)                     | Discovering the document root        |
| [1️⃣1️⃣ File Writing](#1️⃣1️⃣-file-writing-and-command-execution) | Writing and accessing a PHP file     |
| [1️⃣2️⃣ Flag](#1️⃣2️⃣-retrieving-the-flag)                        | Final assessment step                |
| [🧠 Key Takeaways](#-key-takeaways)                               | Lessons learned                      |
| [🛡️ Defensive Recommendations](#️-defensive-recommendations)     | Security mitigations                 |

---

## 🎯 Objectives

The main objectives of the assessment were to:

1. Identify SQL injection vulnerabilities.
2. Understand the SQL query context.
3. Determine the number of columns returned by the query.
4. Perform UNION-based SQL injection.
5. Enumerate database metadata.
6. Identify application tables and columns.
7. Retrieve relevant database information.
8. Investigate server-side file permissions.
9. Identify the application's Nginx document root.
10. Demonstrate the impact of SQL injection through controlled command execution.

---

## 🛠️ Tools Used

| Tool / Technology           | Purpose                                    |
| --------------------------- | ------------------------------------------ |
| 🐉 **Kali Linux**           | Security testing environment               |
| 🦋 **Burp Suite**           | HTTP interception and request manipulation |
| 🌐 **Web Browser**          | Application interaction                    |
| 🗄️ **MySQL**               | Database technology                        |
| 📊 **`information_schema`** | Database metadata enumeration              |
| 🌐 **Nginx**                | Web server configuration analysis          |
| 💉 **SQL Injection**        | Vulnerability exploitation technique       |

---

# 1️⃣ Connecting to the Target

After spawning the target, I initially attempted to access the target directly over HTTP.

```text
http://TARGET_IP
```

The server returned:

```text
400 Bad Request
```

The target appeared to require an HTTPS connection.

I therefore switched to:

```text
https://TARGET_IP
```

After switching to HTTPS, the application loaded correctly.

---

# 2️⃣ Investigating the Login

The application presented a login page, but I did not have valid credentials.

Since the module focused on SQL injection, I initially tested whether the login functionality was vulnerable.

I tested the username and password parameters with SQL injection techniques.

The initial attempts did not bypass authentication, and the application continued returning an invalid-credentials response.

Rather than spending excessive time on the login functionality, I moved to the registration functionality.

---

# 3️⃣ Bypassing the Invitation Code

The registration page required an invitation code.

The browser-side validation made it difficult to directly enter an arbitrary value, so I used **Burp Suite** to intercept the registration request.

The request could then be modified before being forwarded to the server.

I tested SQL injection against the invitation-code parameter.

One successful approach was:

```sql
' OR '1'='1
```

The important point was that the actual HTTP request was modified rather than relying on restrictions implemented by the browser interface.

After sending the modified request, the server accepted the registration and returned an account-created response.

I could then continue into the application.

---

# 4️⃣ Finding an SQL Injection Point

After logging in, I discovered a conversation/search functionality.

Search functionality is interesting from a security-testing perspective because user-controlled input may be incorporated into a backend database query.

I began testing the input to determine whether it was injectable.

One of the first things I wanted to determine was the number of columns returned by the underlying query.

A UNION-style test was used:

```sql
SELECT 1,2,3,4
```

The response reflected values from the query, indicating that the underlying query was working with **four columns**.

I also identified specific positions where returned values were visible in the application's response.

---

# 5️⃣ Understanding the Query Structure

Based on the application's behavior, I developed a rough model of what the backend query could look like:

```sql
SELECT message
FROM msgdb
WHERE (user='admin' AND data LIKE '%search%');
```

> [!NOTE]
> The exact backend implementation is not visible. This query represents an inferred structure based on the application's responses.

The important observation was that the supplied input appeared to be placed inside a quoted value and surrounded by parentheses.

Therefore, the injection needed to escape the existing SQL syntax before introducing the UNION query.

A successful test followed this general structure:

```sql
') UNION SELECT 1,2,3,4 -- -
```

The `')` portion allowed the original expression to be closed before introducing the UNION statement.

Once the UNION query worked, the reflected columns could be used to retrieve database information.

---

# 6️⃣ Identifying the Current Database

After confirming the UNION injection, I began enumerating database information.

The main information I wanted to identify was:

| Information          | Purpose                               |
| -------------------- | ------------------------------------- |
| 🗄️ Current database | Identify the application's database   |
| 🧩 Database version  | Understand the database environment   |
| 📋 Tables            | Identify potentially interesting data |
| 🧱 Columns           | Determine useful fields               |
| 👤 User information  | Identify relevant accounts            |

I used:

```sql
') UNION SELECT 1,2,database(),4
```

The current database was identified as:

```text
chattr
```

This allowed the subsequent enumeration to focus on the application's database.

---

# 7️⃣ Enumerating Tables

I then queried the database metadata to identify available tables.

MySQL's `information_schema` database provides metadata about databases, tables, columns, and other database objects.

The conceptual query was:

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema='chattr';
```

The results contained several tables.

One table immediately stood out because the assessment involved identifying user credentials:

```text
users
```

---

# 8️⃣ Enumerating Columns

Next, I inspected the columns belonging to the `users` table.

The goal was to identify fields containing useful account information, particularly:

* Username
* Password/hash
* Other account information

After identifying the relevant columns, I constructed a UNION query to retrieve the username and password hash.

For example:

```sql
') UNION SELECT 1,2,username,password FROM users -- -
```

This allowed me to retrieve the hash associated with the administrative account.

> [!NOTE]
> Password hashes should be handled carefully and should never be reused against real accounts or systems without authorization.

---

# 9️⃣ Checking `secure_file_priv`

At this stage, I investigated whether the database account had permissions that could potentially allow file operations on the underlying server.

For MySQL, the `secure_file_priv` setting can restrict file operations.

I queried the relevant configuration information:

```sql
SELECT variable_name, variable_value
FROM information_schema.global_variables
WHERE variable_name='secure_file_priv';
```

The returned value was:

```text
NULL
```

This indicated that the usual `secure_file_priv` directory restriction was not configured.

That was an important finding because it suggested that file operations could potentially be possible depending on the privileges of the database account.

---

# 🔟 Finding the Nginx Web Root

My first attempt at writing a file to:

```text
/var/www/html
```

did not work as expected.

The target was running **Nginx**, so I could not simply assume that `/var/www/html` was the directory serving the application.

I therefore inspected the Nginx configuration.

The main configuration file was:

```text
/etc/nginx/nginx.conf
```

The configuration showed that additional configuration was loaded from:

```text
/etc/nginx/sites-enabled/default
```

Inspecting this configuration revealed the application's actual document root:

```text
/var/www/chattr-prod
```

This explained why the earlier attempt using `/var/www/html` was not being served by the application.

---

# 1️⃣1️⃣ File Writing and Command Execution

After identifying the correct web root, I used the file-writing capability available through the SQL injection to place a PHP file inside the application's document root.

The file provided controlled command execution for the purposes of the assessment.

Once the file was accessible through the web server, I tested it with a harmless command:

```text
/shell.php?cmd=id
```

The response showed the user under which the web application was executing.

This confirmed that the SQL injection had progressed from database-level access to controlled command execution on the underlying server.

> [!CAUTION]
> A web-accessible command-execution file is extremely dangerous on a real production system. This technique should only be demonstrated in an authorized lab environment.

---

# 1️⃣2️⃣ Retrieving the Flag

With command execution confirmed, the final step was to locate the assessment flag.

I first inspected the filesystem and checked the root directory.

The target flag file was located at:

```text
/flag__.txt
```

I then used the controlled command-execution mechanism to read the file.

The returned value was the flag required by the assessment.

> [!NOTE]
> The actual flag value is intentionally omitted from this README.

---

# 🧠 Key Takeaways

This assessment reinforced several important concepts in web application security.

### 🔎 SQL Injection

User-controlled input should never be directly concatenated into SQL queries.

Parameterized queries / prepared statements should be used instead.

### 🧩 UNION-Based Enumeration

Once the number of columns and reflected positions were identified, UNION-based SQL injection could be used to retrieve additional database information.

### 🗄️ `information_schema`

Database metadata can reveal valuable information about:

* Databases
* Tables
* Columns
* Application structure

### ⚙️ Server Configuration

Understanding the underlying server configuration was critical to identifying the correct application web root.

### 📁 File Operations

Database file-writing capabilities can significantly increase the impact of an SQL injection vulnerability when the database account has sufficient privileges.

### 💻 Impact

The assessment demonstrated how a vulnerability that initially appears to provide database-level access can potentially lead to much greater impact when combined with additional server-side weaknesses and excessive privileges.

---

# 🛡️ Defensive Recommendations

The techniques demonstrated in this assessment also highlight several defensive measures.

| Vulnerability / Risk             | Recommended Mitigation                          |
| -------------------------------- | ----------------------------------------------- |
| SQL Injection                    | Use parameterized queries / prepared statements |
| Weak input handling              | Validate input server-side                      |
| Excessive DB privileges          | Apply least privilege                           |
| Dangerous file operations        | Restrict database file privileges               |
| Sensitive configuration exposure | Secure server configuration                     |
| Web shell persistence            | Monitor and restrict unexpected files           |
| Command execution                | Apply strict application and OS-level controls  |
| Credential storage               | Use strong password hashing algorithms          |

---

# 🧰 Repository Structure

```text
HTB-SQL-Injection-Assessment-walkthrough/
│
├── README.md
│
└── payloads/
    └── ...
```

---

# 📚 References

* [Hack The Box Academy](https://academy.hackthebox.com/)
* [PortSwigger Web Security Academy](https://portswigger.net/web-security)
* [MySQL Documentation](https://dev.mysql.com/doc/)
* [Nginx Documentation](https://nginx.org/en/docs/)

---

## ⚠️ Disclaimer

This repository is intended for **educational and authorized security-testing purposes only**.

The techniques demonstrated here were performed against a controlled **Hack The Box Academy** environment.

Do not use these techniques against systems, applications, accounts, or networks without explicit authorization.

---

## 👨‍💻 Author

**Abel Gzg**

Cybersecurity learner focused on:

`Web Security` • `Penetration Testing` • `CTFs` • `SQL Injection` • `Burp Suite`

⭐ If you found this write-up useful, consider giving the repository a star.
