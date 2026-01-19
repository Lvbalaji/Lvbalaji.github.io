---
layout: mission
title: "NoSQL Injection: The Theory & Mechanics (Part 1)"
date: 2026-01-01
categories: [Web Security, Theory, NoSQL]
toc: true
---

# NoSQL Injection: Breaking the Non-Relational Database

NoSQL databases have become increasingly popular for modern web applications due to their scalability and flexibility. However, just like SQL databases, they are vulnerable to injection attacks if user input is not properly sanitized.

Unlike SQL Injection, which relies on manipulating a structured query language string, NoSQL Injection often involves **injecting objects or arrays** to manipulate the query logic directly.

---

## 📦 1. MongoDB Architecture & Fundamentals

To understand how to break it, you must understand how it stores data. Unlike relational databases (MySQL, PostgreSQL) that use tables and rows, MongoDB stores data in **documents**.

### The Hierarchy
1.  **Databases:** The highest level container.
2.  **Collections:** Groups of related documents. This is equivalent to a "Table" in SQL (e.g., a "people" collection).
3.  **Documents:** JSON-like structures (associative arrays) holding key-value pairs.
    Example: `{"username": "lphillips", "age": "65"}`.


### Querying Logic
NoSQL does not use standard SQL sentences (like `SELECT * FROM`). Instead, it uses structured associative arrays and **Operators**.

**Filters:** Criteria used to select data.
    * *Example:* `['last_name' => 'Sandler']` retrieves documents where the last name is Sandler.
**Operators:** Special keys that allow complex logic.
    `$lt`: Less than
    `$ne`: Not equal
    `$or`: Logical OR.

---

## 💉 2. NoSQL Injection Principles

**The Root Cause:** Improper concatenation of untrusted user input into database commands allows attackers to alter the command's logic.

**The Difference:**
**SQL Injection:** Relies on string concatenation (e.g., `' OR '1'='1`).
**NoSQL Injection:** Frequently relies on injecting **arrays** or **objects** into the application to manipulate filters.

---

## ⚠️ 3. Types of NoSQL Injection

There are two primary categories of NoSQL injection: Syntax Injection and Operator Injection.

### 1️⃣ Syntax Injection
This is similar to traditional SQLi. It involves breaking the query syntax to inject a payload. This is rarer in NoSQL but usually occurs in custom JavaScript queries (like using the `$where` operator).

**Mechanism:**
If an application allows direct **string interpolation**, an attacker may modify the query structure.

**Example Insecure Query:**
```javascript
db.users.find({ username: userInput })
```

If you send a single quote `'` as input, it might break the syntax.

**Exploitable Input Example:**

**Payload:** `{"$gt": ""}`
  
**Effect:** When applied to a string field like `username`, MongoDB returns all usernames that are greater than an **empty string ("")**. This effectively dumps the database.
    

### 2️⃣ Operator Injection

This is the most common form of NoSQL injection. It involves injecting NoSQL operators (like `$ne`) to change the query behavior _without_ breaking the syntax.

Mechanism:

You pass a JSON object or an array where the application expects a simple string.

**Example Insecure Query:**
```
db.users.find({ username: userInput, password: passInput })
```

**Exploitable Input:**
```
{ "username": { "$ne": null }, "password": { "$ne": null } }
```

**Effect:** This query asks the database: "Find a user where the username is NOT null AND the password is NOT null." Since this is true for valid users, it **bypasses authentication**.

---

## 🛠️ 4. Attack Techniques

We can categorize the exploitation into four specific techniques ranging from simple bypasses to complex data extraction.

### 🔓 Technique 1: Authentication Bypass (Operator Injection)

Attackers can bypass login screens by injecting a "Not Equal" (`$ne`) operator.

The Logic:

The application expects a string for the username. PHP (and other languages) allow passing arrays via HTTP parameters (e.g., user[$ne]=val). If the code inserts $_POST['user'] directly into the query, it becomes ['username'=>['$ne'=>'xxxx']].

The Exploit:

By asserting that the username is not "randomjunk" and the password is not "randomjunk", the database returns the first document it finds (usually the Admin).

**Example Request:**

```
POST /login.php HTTP/1.1
Content-Type: application/x-www-form-urlencoded

user[$ne]=xxxx&pass[$ne]=yyyy
```

### 👤 Technique 2: Targeting Specific Users (`$nin`)

If the bypass above logs you in as Admin, but you want to access _other_ accounts, you use the `$nin` **(Not In)** operator.

The Logic:

"Give me a user where the username is NOT IN the list ['admin', 'jude']". This forces the database to return the next available user in the collection.

**Example Request:**

```
user[$nin][]=admin&user[$nin][]=jude&pass[$ne]=randomjunk
```

This iterates through the user list, allowing you to hijack arbitrary accounts.

### 🕵️ Technique 3: Data Extraction (Blind Injection)

Attackers can steal data (like passwords) using the `$regex` operator to play a game of "hangman" with the database.

Step 1: Guess Length

Use regex wildcards to determine the exact length of the field.

Payload: `pass[$regex]=^.{5}$` (Checks if length is exactly 5).
  

Step 2: Guess Characters

Iterate through characters to find matches.

Payload: `pass[$regex]=^a....$` (Checks if the password starts with 'a').
  

**Example Request:**

```
POST /login.php HTTP/1.1
user=admin&pass[$regex]=^.{5}$
```

If the server returns "Success" or a 302 Redirect, your guess was correct. If it returns "Login Failed," the regex did not match.

### 💥 Technique 4: Advanced Syntax Injection (`$where`)

This occurs when developers use the `$where` operator, which allows executing arbitrary JavaScript within the query.

Detection:

Injecting a single quote ' often causes a verbose traceback error or a behavior change.

Exploitation:

You can inject a JavaScript payload that always evaluates to True to dump data.

**Payload:** `'||1||'`
    
**Resulting Query:** `username == '' || 1 || ''`
    
**Effect:** Since `1` is Truthy in JavaScript, this condition is always true, causing the query to return all documents.
    

---

## 🛡️ 5. Summary of Operators (Cheat Sheet)

When testing for NoSQL injection, these are your primary tools:

|**Operator**|**Purpose**|**Description**|
|---|---|---|
|**`$ne`**|Auth Bypass|Matches anything "Not Equal" to the payload.|
|**`$nin`**|Enum Users|Matches anything "Not In" the provided array list.|
|**`$regex`**|Data Extraction|Matches data against a Regular Expression pattern.|
|**`$where`**|Syntax Injection|Allows execution of JavaScript (vulnerable to string concat).|
|**`$or`**|Logic Bypass|Can be used to force a successful login condition.|
|**`$gt`**|Logic Manipulation|"Greater Than" - often used with empty strings `""` to select all records.|


---

## ❓ 6. Interview Corner: 10 Common Questions

If you are interviewing for a security role, expect these questions regarding NoSQL databases (specifically MongoDB).

### Q1: What is the fundamental difference between SQL Injection and NoSQL Injection?

**Answer:**

**SQL Injection:** Exploits the concatenation of strings into a structured query language command (e.g., `' OR 1=1`).
    
**NoSQL Injection:** Often exploits the structure of the data itself. Instead of breaking syntax with characters, attackers inject **objects** or **arrays** (like `{"$ne": ""}`) that the application interprets as logic operators, changing the query's behavior.
    

### Q2: How does a "Not Equal" (`$ne`) bypass work in a login form?

Answer:

If the backend query is db.users.find({username: user, password: pass}), and I send {"$ne": ""} as the password, the query becomes: "Find a user where the password is NOT empty." Since valid users have passwords, this condition is always true, and the database returns the first match (often the Admin) without me knowing the real password.

### Q3: What is the `$where` operator and why is it dangerous?

Answer:

$where allows the database to evaluate arbitrary JavaScript expressions for each document to determine if it should be returned. If user input is passed into $where without sanitization, it leads to Syntax Injection, allowing an attacker to execute complex JavaScript logic to extract data or traverse the database.

### Q4: How would you perform Blind NoSQL Injection to extract a password?

Answer:

I would use the $regex operator. By iterating through characters (e.g., ^a..., ^b...), I can ask the database true/false questions. If the server returns a valid response (like a user profile), I know the character matches. I repeat this process to reconstruct the entire string character by character.

### Q5: Can sanitizing single quotes (`'`) prevent NoSQL Injection?

Answer:

No. While it might prevent Syntax Injection (which relies on breaking out of a string context), it does not prevent Operator Injection. Operator injection involves sending JSON objects or arrays, which doesn't require special characters like quotes to succeed.

### Q6: What is the `$nin` operator used for in an attack context?

Answer:

$nin stands for "Not In". It is used to enumerate users. If a bypass logs me in as "Admin" by default, I can inject username: {"$nin": ["Admin"]}. The database will skip the "Admin" document and return the next user it finds, allowing me to cycle through every account in the database.

### Q7: Why are API-based (JSON) applications more susceptible to Operator Injection?

Answer:

APIs often automatically parse JSON input into native objects (like Python dictionaries or PHP associative arrays) before passing them to the database driver. This means if I send {"password": {"$ne": 1}}, the backend treats it as a nested object (the operator) rather than a simple string string, triggering the logic flaw.

### Q8: What is the difference between a "Document" and a "Collection"?

**Answer:**

**Collection:** Equivalent to a **Table** in SQL. It is a grouping of records.
    
**Document:** Equivalent to a **Row** in SQL. It is a single record stored as a JSON-like object (BSON in MongoDB) containing key-value pairs.
 

### Q9: How do you fix NoSQL Injection?

**Answer:**

1. **Sanitize Input:** Ensure input is cast to a String before using it in a query (preventing Arrays/Objects).
    
2. **Avoid `$where`:** Use native operators instead of JavaScript evaluation whenever possible.
    
3. **Use Libraries:** Use ODMs (Object Document Mappers) like Mongoose (for Node.js) that define strict schemas, preventing the injection of unexpected operators.
    

### Q10: What is a "Tautology" in the context of NoSQL Syntax Injection?

Answer:

It is a statement that always evaluates to True. In a Syntax Injection (e.g., inside a $where clause), an attacker might inject ' || 1==1 || '. This forces the entire query condition to be True, causing the database to return all records instead of just the intended one.

---

## 🎭 7. Scenario-Based Questions

### 🎭 Scenario 1: The "Secure" Login

Context: A developer argues their login is secure because they use a NoSQL database and "SQL Injection doesn't work here." They show you a POST /login endpoint accepting JSON.

The Question: How do you demonstrate the vulnerability in 30 seconds?

The "Hired" Answer:

"I will send a JSON payload modifying the password field. Instead of a string, I will send {"password": {"$ne": "invalid_value"}}. If the application logs me in, it proves that I bypassed the authentication check by injecting a NoSQL operator, demonstrating that the code blindly trusts the input structure."

### 🎭 Scenario 2: The Hidden User

Context: You have bypassed the login using $ne and are logged in as 'Administrator'. The client wants to know if you can compromise other users.

The Question: How do you target the CEO's account specifically?

The "Hired" Answer:

"I will use the $regex operator on the username field. I will send a payload like username: {"$regex": "^ceo.*"}. This tells the database to find any user whose username starts with 'ceo'. If the account exists, the application will log me in as that user without needing the password."

### 🎭 Scenario 3: The Python Backend

Context: You are reviewing code for a Python app using PyMongo. You see: db.users.find("username == '" + user_input + "'").

The Question: What type of vulnerability is this, and how do you exploit it?

The "Hired" Answer:

"This is a Syntax Injection vulnerability because the developer is concatenating strings inside a query string (likely a $where clause implicitly). I would exploit it by injecting a payload like ' || 1==1 || '. This breaks the query syntax and adds a 'True' condition, forcing the database to dump all user records."

### 🎭 Scenario 4: The Blind Extraction

Context: You found a search bar vulnerable to NoSQLi, but it only returns "Result Found" or "No Result". It does not show you the data.

The Question: How do you steal the database version?

The "Hired" Answer:

"I will use Blind Injection techniques. I will script an attack that iterates through characters using regex.

Request 1: `search[$regex]=^1.*` (Is version 1.x?) -> No Result.
 
Request 2: search[$regex]=^4.* (Is version 4.x?) -> Result Found.
    
    I will continue appending characters (^4.0, ^4.1) until I determine the full string based on the server's boolean response."
    

### 🎭 Scenario 5: The Patch Verification

Context: The developer added a check: if (typeof username !== 'string') return Error.

The Question: Does this fix the vulnerability?

The "Hired" Answer:

"Yes, largely. This prevents Operator Injection because that attack relies on passing an Object or Array (e.g., {"$ne": 1}) instead of a String. By strictly enforcing the input type as a String, the database driver will treat any special characters as literal text, neutralizing the logic manipulation."
