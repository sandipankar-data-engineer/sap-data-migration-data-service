# SAP BODS – Query Transform

## 1. Purpose

The **Query transform** is used to map input columns to an output schema and apply transformations to source data. It is similar to a SQL `SELECT` statement.

## 2. Main Sections

| Section | Purpose |
|---|---|
| **Mapping** | Define output columns, expressions, calculations, and column mappings. |
| **SELECT** | Enable `DISTINCT` to remove duplicate output rows. |
| **FROM** | Define joins between input schemas. |
| **WHERE** | Filter rows using conditions. |
| **GROUP BY** | Group rows by one or more input columns, typically with aggregate expressions. |
| **ORDER BY** | Sort the output using selected columns. |

## 3. Advanced Options

The Query transform can execute certain operations as separate processes:

- Run **DISTINCT** as a separate process.
- Run **GROUP BY** as a separate process.
- Run **JOIN** as a separate process.
- Run **ORDER BY** as a separate process.

These options affect how the operations are executed and may be useful when tuning dataflow performance. Validate performance with the actual data volume and database environment.

## 4. Schema Remapping

Use **Schema Remapping** when an input source is replaced and the new source has a different schema.

Typical steps:

1. Open the schema-remapping settings.
2. Specify the **obsolete schema** (the original schema being replaced).
3. Choose the **correct schema** (the replacement schema).
4. Verify the column mappings and output metadata.

## 5. Inputs, Outputs, and Joins

- A Query transform can accept multiple input schemas and produce one output schema (a common **many-to-one** pattern).
- Supported join types in the described setup:
  - **INNER JOIN** – returns only rows with matching join keys in both inputs.
  - **LEFT OUTER JOIN** – returns all rows from the left input and matching rows from the right; unmatched right-side columns are null.

## 6. XML and DTD

In SAP BODS, a **DTD (Document Type Definition)** describes the permitted structure of an XML document, including its elements and attributes. It can be used as a structural reference when configuring XML input and validation workflows involving a Query transform.

Reference walkthrough: [Data transfer from XML file to database using DTD format](https://community.sap.com/t5/technology-blogs-by-members/data-transfer-from-xml-file-to-database-by-using-dtd-format/ba-p/12972178)

## 7. Useful Notes

- To update a default value for multiple columns, select the required columns, open **Properties**, choose the metadata/default-value property to update, and click **OK**.
- After changing sources or schemas, check the Query transform's input and output metadata to ensure mappings and expressions still refer to valid columns.

## 8. Visual References

- [Query transform – mapping view](https://github.com/user-attachments/assets/031ea4c4-8653-4f5e-bef4-adad9d6e1679)
- [Query transform – configuration view](https://github.com/user-attachments/assets/3805657c-f914-4b94-9ce4-dca019213135)
