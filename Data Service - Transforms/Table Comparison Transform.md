# SAP Data Services (BODS) --- Table Comparison Transform

> A practical guide to detecting inserts, updates, and deletes by
> comparing an incoming data set with an existing database table.

## 1. What is the Table Comparison transform?

The **Table_Comparison** transform in SAP BusinessObjects Data Services
(SAP Data Services / BODS) compares:

1.  An **input data set** --- usually rows extracted from a source
    system.
2.  A **comparison table** --- an existing database table, commonly the
    target table that was populated by a previous load.

It identifies differences and emits rows with operation codes:

-   **INSERT** --- the input row's primary key does not exist in the
    comparison table.
-   **UPDATE** --- the key exists, but one or more selected non-key
    columns differ.
-   **DELETE** --- a row exists in the comparison table but is absent
    from the current input set, when deleted-row detection is enabled.
-   **Ignored** --- the key exists and the compared values are
    unchanged.

The transform is commonly used for **target-based change data capture
(CDC)** when the source does not provide database CDC or a reliable
change log. It can prevent unchanged records from being sent to the
target on every batch run.

**Important:** Table Comparison detects and flags changes. It does not,
by itself, guarantee that the target database has been modified. The
downstream target/load configuration must apply the INSERT, UPDATE, and
DELETE operation codes.

## 2. Where it fits in an ETL data flow

A typical design looks like this:

``` text
Source database / file
        |
        v
     Query
 (map, cleanse, standardize,
  select comparison columns)
        |
        v
 Table_Comparison  <------ Existing target/comparison table
        |
        v
 Optional Key_Generation
 (generate keys for new rows)
        |
        v
 Target table
 (apply INSERT / UPDATE / DELETE row operations)
```

The comparison table must already exist and be accessible through a
configured datastore. In the common target-based CDC pattern, the
comparison table is the target table from the previous successful load.

### What is compared?

The transform uses the configured **input primary key columns** to find
matching rows in the comparison table. It then compares the selected
non-key **Compare columns**.

For example:

**Incoming source data**

    ID Name      City       Year
  ---- --------- -------- ------
     1 Raj       US         2010
     2 Mitul     UK         2011
     3 Shubham   Greece     2010

**Existing comparison/target table**

    ID Name      City      Year
  ---- --------- ------- ------
     1 Raj       US        2010
     2 Mitul     UK        2011
     3 Shubham   Paris     2010
     4 John      US        2012

With `ID` as the primary key and `Name`, `City`, `Year` as compare
columns:

-   ID 1: same key and same compared values → ignored.
-   ID 2: same key and same compared values → ignored.
-   ID 3: same key, but City changed from Paris to Greece → UPDATE.
-   ID 4: exists in comparison table but not in input → DELETE only if
    deleted-row detection is enabled.

If deleted-row detection is disabled, ID 4 is not emitted as a DELETE.

## 3. Understanding the operation codes

  -----------------------------------------------------------------------
  Situation               Resulting operation     Meaning
                          code                    
  ----------------------- ----------------------- -----------------------
  Input key is not found  INSERT                  New row
  in comparison table                             

  Input key is found and  UPDATE                  Changed row
  a compare column                                
  differs                                         

  Input key is found and  No output / ignored     No change
  all compare columns                             
  match                                           

  Comparison-table key is DELETE                  Row removed from
  absent from the                                 input/source snapshot
  complete input set,                             
  with delete detection                           
  enabled                                         
  -----------------------------------------------------------------------

### INSERT

An INSERT is generated when the incoming primary key does not exist in
the comparison table.

Example: input contains `ID = 5`, while the comparison table has no row
with `ID = 5`.

The output row is based on the input row. If the comparison table has
columns that are not present in the input schema, those output columns
can be populated with NULL for an INSERT.

### UPDATE

An UPDATE is generated when:

1.  The input primary key matches a row in the comparison table.
2.  At least one selected non-key compare column has a different value.

The UPDATE row contains the incoming values for changed input columns.
If the comparison table has columns absent from the input data set, the
transform can carry those values from the comparison table into the
output.

Primary key columns identify the row; they should not normally be placed
in the Compare columns list.

### DELETE

DELETE detection is **not enabled by default**.

When **Detect deleted row(s) from comparison table** is selected, the
transform identifies comparison-table rows that are not represented in
the input data set and flags them as DELETE.

This setting makes an important assumption: **the input represents the
complete set of rows that should exist** for the comparison scope. If
the input is only a partial extract, a filtered subset, or an
incremental feed, absence from the input does not prove deletion.
Enabling delete detection in that situation can delete valid target
rows.

Delete detection can also add processing overhead. For row-by-row and
cached comparison methods, deleted rows are processed at the end of the
data flow; with sorted input they are processed as the ordered
comparison proceeds.

## 4. Table Comparison editor options

### 4.1 Table name

Specifies the fully qualified database table used as the comparison
table, commonly in the form:

``` text
DATASOURCE_OR_DATASTORE.OWNER.TABLE_NAME
```

This table must exist and be accessible to the datastore connection. In
a common target-based CDC design, this is the existing target table.

### 4.2 Generated key column

An optional column in the comparison table used to distinguish multiple
rows that share the same business/input primary key.

For example, a customer table might contain:

    Customer_ID (business key)   Customer_SK (generated key) City
  ---------------------------- ----------------------------- --------
                           100                           501 London
                           100                           508 Paris

Here, `Customer_ID` is not unique in the comparison table, but
`Customer_SK` is. Set `Customer_SK` as the Generated key column if this
is the intended duplicate-resolution rule.

Key points:

-   It must be an existing column in the comparison table and should
    contain unique values.
-   For an UPDATE, the transform uses the row with the **largest
    generated-key value** for a repeated primary key.
-   For an INSERT, the generated-key output is NULL when the transform
    is reserving that column for a downstream Key_Generation transform.
    If the generated key is already supplied in the input, its value is
    preserved.
-   For DELETE detection with duplicate comparison-table keys, the
    editor provides a choice to detect all duplicate rows or only the
    row with the largest generated key value.
-   The generated key column is not itself compared as a normal compare
    column.

**Do not assume every surrogate key should automatically be selected
here.** It is useful when the comparison table can contain multiple
records for the configured input key and you need a deterministic
row-selection rule.

### 4.3 Input contains duplicate keys

Select this option when the incoming data set can contain more than one
row for the configured input primary key.

Without this option, duplicate input keys can lead the transform to
choose rows during data-flow processing rather than deliberately
processing all duplicate-key records.

With this option enabled, Data Services supports processing duplicate
input keys. It requires additional memory to track them, so enable it
only when duplicate keys are genuinely possible and their handling is
understood.

**Best practice:** If the input is supposed to contain one row per
business key, investigate and resolve duplicates upstream rather than
relying on this checkbox to hide a data-quality issue.

### 4.4 Detect deleted row(s) from comparison table

Enables detection of rows present in the comparison table but missing
from the input data set.

When enabled, an additional choice may appear for handling deleted rows
with duplicate keys (depending on the other selected options):

-   **Detect all rows** --- emit DELETE rows for all matching
    duplicate-key rows.
-   **Detect row with largest generated key value** --- emit a DELETE
    only for the duplicate-key row with the largest generated-key value.

Use this only when the input is a complete snapshot for the comparison
scope. It is generally unsafe for a partial extract or incremental-only
feed.

### 4.5 Comparison method

This controls how the transform accesses the comparison table.

  ------------------------------------------------------------------------
  Method            How it works       Suitable          Main trade-off
                                       situation         
  ----------------- ------------------ ----------------- -----------------
  Row-by-row select Executes a         Relatively few    Can cause many
                    database lookup    input rows        database round
                    for each input row compared with a   trips
                                       very large target 

  Cached comparison Loads              Comparing a large Cache memory/disk
  table             comparison-table   proportion of the usage; large
                    data into a cache  target,           caches can page
                    and performs       especially when   and become slower
                    lookups against    the comparison    
                    the cache          table can be      
                                       cached            
                                       efficiently       

  Sorted input      Reads the          Input can be      Requires
                    comparison table   sorted to exactly correctly ordered
                    sequentially in    match the         input; sorting
                    primary-key order  configured        itself has a cost
                                       primary-key order 
  ------------------------------------------------------------------------

#### Row-by-row select

For each input row, the transform queries the comparison table using the
key.

Conceptually:

``` sql
SELECT ...
FROM target_table
WHERE ID = ?;
```

This can be appropriate when only a small number of source rows need to
be checked against a large target. It is often the slowest choice for
large input volumes because it can result in many individual lookups.

#### Cached comparison table

The comparison table is loaded into memory/cache. Lookups are then
served from the cache rather than issuing a database query for every
input row.

Use it when a substantial portion of the comparison table will be
checked and the table can be cached efficiently. SAP Data Services uses
pageable cache by default; if the table comfortably fits in available
memory, an in-memory cache may be considered in the data-flow
properties.

Do not assume cached comparison is always faster: cache construction,
memory pressure, paging, and data volume all matter.

#### Sorted input

The transform reads the comparison table sequentially in primary-key
order. This can be the fastest option because the comparison table is
read once in order rather than repeatedly looked up.

Requirements:

1.  The input data must be sorted by all configured primary-key columns.
2.  The sort-column sequence must match the order of the primary-key
    columns configured in Table Comparison.
3.  If necessary, add a Query transform before Table Comparison and
    place the primary-key columns in its `ORDER BY` configuration.

For a composite key `(Country_ID, Customer_ID)`, both the input ordering
and configured key order must be consistent.

**Rule of thumb:** Row-by-row select is useful for a small delta against
a large target; cached comparison is useful when much of the target is
compared and cache resources are adequate; sorted input is attractive
when you can guarantee the required ordering.

### 4.6 Input primary key columns

Specify the input columns that uniquely identify a row in the comparison
table.

Requirements and considerations:

-   The key columns must exist in both the input data set and comparison
    table.
-   Their names and data types must match as required by the transform.
-   Use multiple columns for a composite key when one column is not
    sufficient.
-   Do not add nested-schema columns.
-   The configured key should identify the intended comparison row.
    Duplicate-key handling must be deliberate.

Example:

``` text
Input primary key columns:
    Country_ID
    Customer_ID
```

This represents a composite key: a customer is identified by the
combination of country and customer ID.

### 4.7 Compare columns

Specify which non-key columns should be checked for changes.

Example:

``` text
Input primary key columns:
    Customer_ID

Compare columns:
    Customer_Name
    City
    Status
```

Only changes in the selected compare columns cause an UPDATE. If no
compare columns are specified, Data Services compares the applicable
columns shared by the input and comparison schemas.

Why select columns explicitly?

-   Avoid unnecessary comparisons.
-   Improve performance where many columns exist.
-   Avoid triggering updates for fields that are not meaningful to the
    downstream process.

Do not add primary-key columns to Compare columns. Long and BLOB data
types are not supported as ordinary compare columns. The generated-key
column is also excluded from normal comparison.

### 4.8 Filter

A SQL-style filter expression limits which rows from the comparison
table are considered.

Examples:

``` sql
TC.Department_ID = 10
```

``` sql
TC.Status IN ('ACTIVE', 'PENDING')
```

``` sql
TC.Created_Year >= 2024 AND TC.Created_Year < 2026
```

The filter applies to the comparison table, not as a general-purpose
filter on the input pipeline.

**Caution:** An overly restrictive or incorrect filter can make an
existing comparison row appear not to exist, causing a false INSERT.
Ensure the filter describes the intended comparison population and is
consistent with the input data scope.

## 5. Worked example

<img width="1203" height="606" alt="image" src="https://github.com/user-attachments/assets/5d9fbe2d-8321-43fb-bbfc-a2034c51c5b0" />
<img width="1203" height="606" alt="image" src="https://github.com/user-attachments/assets/4b58666b-2ca1-4268-9bd7-069591739311" />


Assume the source extract is the current snapshot and the target table
stores the previous snapshot.

### Source input

    ID Name      City       Year
  ---- --------- -------- ------
     1 Raj       US         2010
     2 Mitul     UK         2011
     3 Shubham   Greece     2010
     5 Sunny     Italy      2010

### Existing target / comparison table

    ID Name      City      Year
  ---- --------- ------- ------
     1 Raj       US        2010
     2 Mitul     UK        2011
     3 Shubham   Paris     2010
     4 John      US        2012

Configure:

``` text
Table name:             Existing target table
Input primary key:      ID
Compare columns:        Name, City, Year
Comparison method:      Choose based on volume/order
Detect deleted rows:    Enabled for this complete-snapshot example
```

### Expected differences

  ------------------------------------------------------------------------
                            ID Reason                Operation
  ---------------------------- --------------------- ---------------------
                             3 City changed from     UPDATE
                               Paris to Greece       

                             5 ID does not exist in  INSERT
                               target                

                             4 ID exists in target   DELETE
                               but is absent from    
                               complete input        
                               snapshot              

                             1 No compared values    Ignored
                               changed               

                             2 No compared values    Ignored
                               changed               
  ------------------------------------------------------------------------

The output contains only the detected changes, not every input row.

### What happens downstream?

The operation codes must be preserved and correctly applied by the
target load. Conceptually:

``` text
INSERT  -> insert a new target row
UPDATE  -> update the existing target row identified by its key
DELETE  -> delete the corresponding target row
```

For a target using surrogate keys, a common design is to send new INSERT
rows through Key_Generation before loading them. Existing rows being
updated must retain their existing generated key.

## 6. Important corrections and clarifications

### Does a missing key generate INSERT or DELETE?

It depends on which side is missing:

-   **Input key is missing from the comparison table** → INSERT.
-   **Comparison-table key is missing from the complete input set** →
    DELETE, but only when deleted-row detection is enabled.

These are opposite situations and should not be confused.

### Does Table Comparison execute INSERT, UPDATE, and DELETE SQL itself?

Think of it primarily as a **change-detection and operation-flagging
transform**. It emits rows marked with operation codes. The downstream
target/load configuration is responsible for applying the corresponding
database actions.

### Is the input-to-output relationship always 1:1?

No. The statement "Input:Output is 1:1" is not generally correct.

-   Unchanged input rows are ignored and do not appear in the difference
    output.
-   Changed and new input rows generate UPDATE and INSERT rows.
-   Delete detection can generate DELETE rows from the comparison table
    that are absent from the input snapshot.
-   Duplicate-key settings can affect the number and handling of emitted
    rows.

Therefore, the output is generally a **subset of changed input rows plus
any detected deleted comparison rows**, not a guaranteed
one-output-row-per-input-row result.

### Does it prevent duplicate records in the target?

It can help avoid reloading unchanged rows and can identify new rows by
key. However, it is not a replacement for:

-   A primary-key or unique constraint in the database.
-   Source duplicate validation.
-   Correct key mapping.
-   Idempotent load design and recovery handling.

The target should still enforce its intended uniqueness rules.

### Does the transform require the target/comparison table to exist?

The configured comparison table must exist and be accessible before the
transform can compare against it. In the usual target-based CDC design,
the target is first initialized with an initial load and then reused as
the comparison table for subsequent delta runs.

## 7. Practical implementation checklist

Before running the data flow, verify:

-   [ ] The comparison table exists and the datastore can read it.
-   [ ] The input and comparison schemas have compatible key columns.
-   [ ] The configured primary key represents the correct business
    grain.
-   [ ] Duplicate keys in the source and comparison table have been
    investigated.
-   [ ] Generated key handling is configured only when required.
-   [ ] Compare columns include the attributes that should trigger an
    update.
-   [ ] Primary-key columns are not unnecessarily included in Compare
    columns.
-   [ ] Sorted input is selected only when the input is sorted in the
    exact configured key order.
-   [ ] Cached comparison is sized with memory/cache capacity in mind.
-   [ ] Delete detection is enabled only for a complete source snapshot
    and correct comparison scope.
-   [ ] Filter expressions cannot incorrectly exclude valid comparison
    rows.
-   [ ] The target load is configured to apply INSERT, UPDATE, and
    DELETE operation codes.
-   [ ] A test run validates new, changed, unchanged, and deleted
    records.

## 8. Common real-world use cases

1.  **Daily source-to-target synchronization:** Compare today's full
    source extract with yesterday's target state and load only changes.
2.  **Dimension table maintenance:** Detect new and changed customer,
    product, or location records.
3.  **Target-based CDC without source CDC:** Identify changes by
    comparing snapshots when transaction logs or native CDC are
    unavailable.
4.  **Reducing unnecessary updates:** Avoid rewriting records whose
    relevant attributes have not changed.
5.  **Source deletion propagation:** Identify records removed from a
    complete source snapshot and propagate deletes, when the business
    rules permit hard deletes.

## 9. Limitations, risks, and design considerations

-   **Snapshot dependency for deletes:** A missing row is only
    meaningful when the input is a complete and correctly scoped
    snapshot.
-   **Comparison cost:** A full comparison can be expensive for large
    tables. Choose the access method based on input volume, target size,
    sorting, and available cache resources.
-   **Key quality:** Null, non-unique, inconsistent, or incorrectly
    mapped keys can lead to incorrect matching.
-   **Data normalization:** Differences in trimming, case, null
    handling, date/time precision, decimal scale, and source-to-target
    conversions can affect whether rows are considered changed.
    Standardize data before comparing where appropriate.
-   **Floating-point values:** SAP documentation warns that comparisons
    involving the `REAL` data type can produce unpredictable results.
    Avoid relying on it for exact change detection; use an appropriate
    exact numeric representation or a defined tolerance-based comparison
    upstream.
-   **LOB columns:** Long and BLOB data types are not supported as
    ordinary Compare columns.
-   **Schema drift:** Added, removed, or differently typed columns
    should be handled deliberately in the mapping and comparison design.
-   **Concurrent target writes:** If another process changes the
    comparison table during the comparison/load, results may be
    inconsistent. Coordinate writers or use an appropriate
    batch/transaction strategy.
-   **Recovery:** Design restart behavior so that a failed load does not
    leave the comparison table in a partially updated state that causes
    the next run to mis-detect changes.

## 10. Table Comparison vs. other change-detection approaches

  -----------------------------------------------------------------------
  Approach                How changes are found   Best fit
  ----------------------- ----------------------- -----------------------
  Table Comparison        Compares incoming rows  Snapshot-based
                          with an existing        detection when source
                          comparison table        CDC is unavailable

  Source-native CDC       Reads database change   High-volume changes
                          records/logs            with reliable source
                                                  CDC support

  Timestamp / watermark   Reads rows newer than a Sources with
                          stored timestamp or     trustworthy change
                          increasing key          timestamps or monotonic
                                                  keys

  Hash comparison         Compares a hash of      Custom snapshot-diff
                          selected business       patterns or
                          columns                 cross-platform
                                                  comparisons

  Full reload             Replaces the target     Small datasets or cases
                          with a fresh source     where full replacement
                          extract                 is acceptable
  -----------------------------------------------------------------------

Table Comparison is a **target-based comparison technique**. It should
not be confused with source-log-based CDC or watermark-based incremental
extraction.

## 11. Interview questions to prepare

1.  What is the purpose of the Table Comparison transform in SAP Data
    Services?
2.  What is the difference between the input data set and the comparison
    table?
3.  When does the transform generate INSERT and UPDATE operation codes?
4.  How is DELETE detection enabled, and what assumption does it make?
5.  Why is delete detection unsafe for an incremental-only source feed?
6.  Explain Row-by-row select, Cached comparison table, and Sorted
    input.
7.  What are the sorting requirements for Sorted input?
8.  What is the Generated key column, and how does it handle duplicate
    keys?
9.  What does Input contains duplicate keys do?
10. What is the purpose of Compare columns?
11. How can a Filter cause false INSERT rows?
12. Does Table Comparison itself execute DML against the target?
13. Is the transform's input/output relationship always 1:1?
14. How would you design a daily full-snapshot source-to-target
    synchronization?
15. How would you handle a very large target table when only a small
    number of source rows change?

## 12. Quick revision sheet

  -----------------------------------------------------------------------
  Item                                Remember
  ----------------------------------- -----------------------------------
  Purpose                             Detect differences between input
                                      data and an existing comparison
                                      table

  Input                               Rows from source or an upstream
                                      transform; only NORMAL rows are
                                      considered

  Comparison table                    Existing database table used as the
                                      reference

  Key not found in comparison table   INSERT

  Key found, compare values differ    UPDATE

  Key found, values match             Ignored

  Comparison row missing from         DELETE only when enabled
  complete input                      

  Generated key column                Helps resolve duplicate
                                      comparison-table keys

  Input contains duplicate keys       Enables handling of duplicate keys
                                      in the input

  Row-by-row select                   Per-input-row database lookup

  Cached comparison table             Comparison table is cached for
                                      lookups

  Sorted input                        Sequential comparison; requires
                                      matching key order

  Compare columns                     Non-key columns checked for changes

  Filter                              Restricts comparison-table rows;
                                      incorrect filters can cause false
                                      INSERTs

  Output                              Difference rows flagged with
                                      operation codes; not necessarily
                                      1:1

  Target DML                          Applied by downstream target/load
                                      configuration
  -----------------------------------------------------------------------

## 13. Reference documentation

The following SAP sources were used to validate and expand the notes:

-   [SAP Help --- Table_Comparison
    transform](https://help.sap.com/docs/SAP_DATA_SERVICES/8092b085a68941f6aaa6708685a62b0d/5770ef146d6d1014b3fc9283b0e91070.html)
-   [SAP Help --- Table_Comparison transform editor
    options](https://help.sap.com/docs/SAP_DATA_SERVICES/e54136ab6a4a43e6a370265bf0a2d744/577100086d6d1014b3fc9283b0e91070.html)
-   [SAP Help --- Table_Comparison transform data
    outputs](https://help.sap.com/docs/SAP_DATA_SERVICES/e54136ab6a4a43e6a370265bf0a2d744/5770fa516d6d1014b3fc9283b0e91070.html)
-   [SAP Help --- Cache table
    comparisons](https://help.sap.com/docs/SAP_DATA_SERVICES/e54136ab6a4a43e6a370265bf0a2d744/5756b0186d6d1014b3fc9283b0e91070.html)
-   [SAP Learning --- Using Target-Based Changed Data
    Capture](https://learning.sap.com/courses/applying-sap-data-services-transformation-techniques/using-target-based-changed-data-capture-cdc_f4ba243e-7272-478f-96b5-1b43882bd6d1)

------------------------------------------------------------------------

## Appendix A. Original Designer screenshots

The screenshots supplied with the original notes are included below as a
visual reference to the Data Services Designer flow and Table Comparison
editor.

![SAP Data Services Table Comparison transform
screenshots](table_comparison_screenshots.png)
