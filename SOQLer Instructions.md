# Comprehensive SOQL Instructions for Custom GPT

This guide provides a detailed framework for constructing, interpreting, and validating SOQL (Salesforce Object Query Language) queries. It covers syntax, functions, relationship queries, date handling, filtering techniques, operators, multi-select picklists, null handling, and advanced clauses—ensuring that all queries conform to Salesforce best practices and limitations.

---

## 1. Overview

- **Purpose & Scope:**  
  SOQL is designed to query data from Salesforce’s object database. Unlike SQL, it is object-oriented, working with Salesforce objects and relationships while respecting platform governor limits.

- **Key Characteristics:**  
  - **Object-Oriented:** Operates on Salesforce objects (e.g., `Account`, `Contact`) rather than relational tables.
  - **Relationship Queries:** Supports child-to-parent (dot notation) and parent-to-child (subqueries) traversals.
  - **Governor Limits:** Queries must obey limits on record counts, CPU time, and more.

---

## 2. Query Structure

Every SOQL query follows a structured format:

```sql
SELECT <fieldList> [subquery...]
FROM <object>
[WHERE <conditions>]
[ORDER BY <fields> [ASC|DESC] [NULLS {FIRST|LAST}]]
[LIMIT <numberOfRows>]
[OFFSET <numberOfRowsToSkip>]
[USING SCOPE <filterScope>]
```

Each clause has specific syntax and considerations, as detailed below.

---

## 3. The SELECT Clause

### A. Standard Field Listing

- **Field List:**  
  You must explicitly list one or more fields (e.g., `SELECT Id, Name FROM Account`). Wildcards (`*`) are not supported.

- **Relationship Traversal:**  
  - **Child-to-Parent:** Use dot notation to access parent fields.
    ```sql
    SELECT Id, Account.Name FROM Contact
    ```
  - **Parent-to-Child (Subqueries):** Retrieve child records via subqueries.
    ```sql
    SELECT Id, Name, (SELECT Id, LastName FROM Contacts) FROM Account
    ```
  - **Limitation:** Only one level of subquery is allowed.

### B. Using the FIELDS() Keyword

To simplify field selection—especially when you need to retrieve many fields or explore an object’s shape without prior knowledge of its fields—use the **FIELDS()** keyword. Introduced in API version 51.0 and later, it allows you to select groups of fields with a single call.

#### Supported Variants:

- **FIELDS(ALL):**  
  Selects all accessible fields for an object.
  
- **FIELDS(CUSTOM):**  
  Selects all custom fields (ending with `__c`).
  
- **FIELDS(STANDARD):**  
  Selects all standard fields.

*Note:* In each case, field-level security is respected, so only fields the user is permitted to see are returned.

#### Examples:

- Use as the entire field list:
  ```sql
  SELECT FIELDS(ALL) FROM Account LIMIT 200
  SELECT FIELDS(CUSTOM) FROM Account LIMIT 200
  SELECT FIELDS(STANDARD) FROM Account
  ```
- Combine with other fields:
  ```sql
  SELECT Name, Id, FIELDS(CUSTOM) FROM Account LIMIT 200
  SELECT someCustomField__c, FIELDS(STANDARD) FROM Account
  ```
- In subqueries:
  ```sql
  SELECT Account.Name, 
         (SELECT FIELDS(ALL) FROM Account.Contacts LIMIT 200)
  FROM Account
  ```

#### Bounded vs. Unbounded Queries

- **Bounded Queries:**  
  For example, `FIELDS(STANDARD)` returns a known set of fields.  
  - Supported in Apex (inline/dynamic), Bulk API 2.0, CLI, SOAP API, and REST API.
  
- **Unbounded Queries:**  
  `FIELDS(ALL)` and `FIELDS(CUSTOM)` are considered unbounded because the number of fields isn’t predetermined.  
  - They are supported only if result rows are limited (using a LIMIT clause or filtering on IDs) in Bulk API, SOAP API, and REST API.

#### Considerations When Using FIELDS()

- **Performance:**  
  If you already know which fields you need, listing them explicitly can improve performance by not retrieving unnecessary data.
  
- **Grouping and Aggregates:**  
  FIELDS() cannot be used with aggregation operators if it leads to duplicate field names or violates grouping rules.  
  *Example:*  
  ```sql
  -- Valid:
  SELECT Id, MIN(NumberOfEmployees) FROM Account GROUP BY Id
  -- Invalid:
  SELECT FIELDS(ALL), MIN(NumberOfEmployees) FROM Account GROUP BY Id LIMIT 200
  ```
  The latter produces an error because the expanded field list isn’t fully grouped or aggregated.

- **Result Pagination:**  
  If FIELDS() returns a large number of columns, consider using pagination (see OFFSET below) or queryMore() in SOAP API to retrieve additional pages.

- **Errors:**  
  If FIELDS(CUSTOM) is used on an object without custom fields, an error is returned unless at least one other field is specified:
  ```sql
  SELECT Id, FIELDS(CUSTOM) FROM User LIMIT 200
  ```
  In this case, no error occurs because there’s another field in the list.

---

## 4. The FROM Clause

- **Object Specification:**  
  You must specify a valid Salesforce object. Custom objects require the `__c` suffix (e.g., `Invoice__c`).

---

## 5. The WHERE Clause

The WHERE clause filters records based on field values. It supports:

### A. Filtering with Dates and Date Literals

#### Date Values & Formats

- **Date Fields:**  
  Use the `YYYY-MM-DD` format (e.g., `1999-01-01`).

- **DateTime Fields:**  
  Use ISO-8601 formats:
  - `YYYY-MM-DDThh:mm:ss+hh:mm`
  - `YYYY-MM-DDThh:mm:ss-hh:mm`
  - `YYYY-MM-DDThh:mm:ssZ`  
  *Example:* `1999-01-01T23:01:01+01:00`  
  *Note:* Salesforce stores DateTime values as UTC; you may supply a time zone offset.

#### Date Literals

Date literals let you filter records relative to the current date. They adapt to the user’s locale.

**Examples:**

- **YESTERDAY:**  
  ```sql
  SELECT Id FROM Account WHERE CreatedDate = YESTERDAY
  ```
- **TODAY:**  
  ```sql
  SELECT Id FROM Account WHERE CreatedDate < TODAY
  ```
- **LAST_90_DAYS:**  
  ```sql
  SELECT Id FROM Account WHERE CreatedDate = LAST_90_DAYS
  ```
- **Parameterized Literals:**  
  ```sql
  SELECT Id FROM Account WHERE CreatedDate = LAST_N_DAYS:365
  SELECT Id FROM Opportunity WHERE CloseDate > NEXT_N_DAYS:15
  ```
  
*Additional literals exist for weeks, months, quarters, years, and fiscal periods. Note that whether the current day is included in the range depends on the specific literal used.*

#### B. Comparison and Logical Operators

**Comparison Operators:**

| Operator | Description                                                                           |
|----------|---------------------------------------------------------------------------------------|
| `=`      | Equals; true if the field value equals the specified value.                         |
| `!=`     | Not equals; true if the field value does not equal the specified value.               |
| `<`      | Less than.                                                                            |
| `<=`     | Less than or equal to.                                                                |
| `>`      | Greater than.                                                                         |
| `>=`     | Greater than or equal to.                                                             |
| `LIKE`   | Pattern matching; use `%` (multiple characters) and `_` (single character).           |
| `IN`     | Matches any value in a list or subquery.                                              |
| `NOT IN` | True if the field value does not match any value in the list or subquery.              |

*Note:* Comparisons on strings are case-sensitive for fields defined as case-sensitive.

**Logical Operators:**

| Operator | Description                                         |
|----------|-----------------------------------------------------|
| `AND`    | Both expressions must be true.                      |
| `OR`     | At least one expression must be true.               |
| `NOT`    | Inverts the truth value of the expression.          |

*Usage Tip:* Always use parentheses when mixing logical operators to enforce the intended evaluation order.

#### C. Semi-Joins and Anti-Joins (Subqueries in WHERE)

- **Semi-Join Example:**
  ```sql
  SELECT Id, Name 
  FROM Account 
  WHERE Id IN (
      SELECT AccountId FROM Opportunity WHERE StageName = 'Closed Lost'
  )
  ```
- **Anti-Join Example:**
  ```sql
  SELECT Id 
  FROM Account 
  WHERE Id NOT IN (
      SELECT AccountId FROM Opportunity WHERE IsClosed = false
  )
  ```
  
**Key Restrictions:**

- The left operand must be a single ID or reference field.
- Dot notation is not allowed in subqueries.
- Semi-joins/anti-joins cannot be nested or combined with OR.
- Functions like COUNT, ORDER BY, LIMIT, and FOR UPDATE aren’t allowed in subqueries.
- Certain objects (e.g., ActivityHistory, Attachments, Event, Note, Task) aren’t supported in subqueries.

#### D. Querying Multi-Select Picklists

For multi-select picklist fields, use these operators:

| Operator   | Description                                                  |
|------------|--------------------------------------------------------------|
| `=`        | Exact match (e.g., `'AAA;BBB'`).                             |
| `!=`       | Not an exact match.                                          |
| `includes` | Checks if the field contains the specified value.          |
| `excludes` | Ensures the field does not contain the specified value.      |
| `;`        | Delimiter when multiple values must all be selected.         |

*Examples:*

```sql
-- Exact match where both AAA and BBB are selected:
SELECT Id, MSP1__c FROM CustObj__c WHERE MSP1__c = 'AAA;BBB'

-- Records that include either "AAA;BBB" or "CCC":
SELECT Id, MSP1__c FROM CustObj__c WHERE MSP1__c includes ('AAA;BBB','CCC')
```

#### E. Handling NULL Values

- **Testing for NULL:**  
  ```sql
  SELECT AccountId FROM Event WHERE ActivityDate != null
  ```
- **Boolean Fields:**  
  Comparing a Boolean field to `null` evaluates as false:
  ```sql
  SELECT Id, Name FROM Account WHERE Test_c = null  -- equivalent to Test_c = false
  ```
- **Relationship Fields:**  
  When filtering on parent fields, a missing parent may still return the record:
  ```sql
  SELECT Id FROM Case WHERE Contact.LastName = null
  ```

---

## 6. The GROUP BY Clause

- **Usage:**  
  GROUP BY is required when your query includes an aggregate function along with a LIMIT clause (unless all fields are aggregated).  
  *Example:*
  ```sql
  SELECT Name, MAX(CreatedDate)
  FROM Account
  GROUP BY Name
  LIMIT 5
  ```
  
- **Considerations:**
  - Only fields that support grouping (as indicated by the `groupable` attribute in the object’s describe call) can be included.
  - You cannot use child relationship expressions (using the __r syntax) in GROUP BY.
  - In SOAP API, queries with GROUP BY cannot use queryMore() to paginate results.
  - In REST API, queries with GROUP BY do not support query locators.

---

## 7. The ORDER BY Clause

ORDER BY is used to control the sort order of query results.

### Syntax

```sql
ORDER BY <field1> [ASC|DESC] [NULLS {FIRST|LAST}], <field2> [ASC|DESC] [NULLS {FIRST|LAST}], ...
```

- **ASC / DESC:**  
  Sort in ascending (default) or descending order.
  
- **NULLS FIRST / LAST:**  
  Controls whether null values appear at the beginning or end of the result set. (Default is usually nulls first, but can vary by locale.)

### Examples

- **Basic Ordering:**
  ```sql
  SELECT Name, Industry FROM Account ORDER BY Industry
  ```
- **Adding a Unique Field:**  
  To ensure a consistent order, include a unique field:
  ```sql
  SELECT Name, Industry FROM Account ORDER BY Industry, Id
  ```
- **Descending Order with Nulls Last:**
  ```sql
  SELECT Name FROM Account ORDER BY Name DESC NULLS LAST
  ```

### Special Considerations

- **Locale Sensitivity:**  
  For English locales, sorting is case insensitive and uses the UTF-8 values of uppercase letters. For non-English locales, Salesforce uses a natural, predefined order.
- **Data Type Limitations:**  
  ORDER BY is not supported for multi-select picklist, rich text area, long text area, encrypted fields (if enabled), or data category group references (if Salesforce Knowledge is enabled).
- **Relationship Fields:**  
  When ordering on a relationship field that might be null, the `NULLS FIRST`/`NULLS LAST` expressions may not be honored.

---

## 8. The LIMIT Clause

LIMIT restricts the number of rows returned by a query.

### Syntax

```sql
SELECT <fieldList>
FROM <object>
[WHERE <condition>]
LIMIT <numberOfRows>
```

### Example

```sql
SELECT Name FROM Account WHERE Industry = 'Media' LIMIT 125
```

- **Usage with Aggregates:**  
  When using aggregate functions with a GROUP BY clause, LIMIT can restrict the number of groups returned.  
  *Note:* A LIMIT clause cannot be used in a query that uses an aggregate function but lacks a GROUP BY.

---

## 9. The OFFSET Clause

OFFSET is used for paginating query results by skipping a set number of rows.

### Syntax

```sql
SELECT <fieldList>
FROM <object>
[WHERE <condition>]
ORDER BY <fieldList>
LIMIT <numberOfRowsToReturn>
OFFSET <numberOfRowsToSkip>
```

### Example

To skip the first 10 rows:
```sql
SELECT Name FROM Merchandise__c
WHERE Price__c > 5.0
ORDER BY Name
LIMIT 100
OFFSET 10
```

### Considerations When Using OFFSET

- **Maximum Offset:**  
  The maximum offset is 2,000 rows. Exceeding this limit results in a NUMBER_OUTSIDE_VALID_RANGE error.
  
- **Applicability:**  
  OFFSET is allowed in SOAP API, REST API, and Apex SOQL but not in Bulk APIs or Streaming API.
  
- **Subqueries:**  
  OFFSET is generally not permitted in subqueries (except as a pilot feature under specific conditions with LIMIT 1).
  
- **Stable Ordering:**  
  Always include an ORDER BY clause when using OFFSET to ensure a consistent result order.
  
- **Dynamic Data:**  
  Because OFFSET is applied at query time, changes in the underlying data may alter the result set for subsequent OFFSET queries.
  
- **Pagination Alternatives:**  
  For large result sets, consider using queryMore() (SOAP API), nextRecordsUrl (REST API), or Sforce-Locator in Bulk API 2.0 rather than multiple OFFSET queries.

---

## 10. The USING SCOPE Clause

The optional `USING SCOPE` clause limits query results based on predefined filters (filter scopes) provided by the object's metadata.

### Example

```sql
SELECT Id, Name FROM Account USING SCOPE mine
```

### Common Scope Values

| Scope                | Description                                                              |
|----------------------|--------------------------------------------------------------------------|
| **everything**       | All records.                                                             |
| **mine**             | Records owned by the user running the query.                             |
| **mine_and_my_groups** | Records assigned to the user and their queues (for applicable objects).   |
| **my_team_territory**| Records in the user’s team territory (if territory management is enabled).|
| **delegated**        | Records delegated to another user for action.                            |
| **scopingRule**      | Records filtered by an active scoping rule.                              |