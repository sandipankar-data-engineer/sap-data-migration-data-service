* Map columns from Input Schema to Output Schema
* Transformation on source data
* It is similar to SELECT statement in SQL
* Sections:
  * Mapping: Here is where the transformation logic is written.
  * SELECT: Here we can enable Distinct rows.
  * FROM: Here we can design the Join logic.
  * WHERE: Here we can apply WHERE/filter condition.
  * GROUP BY: To group by output based on a input column.
  * ORDER BY: To sort output by input column.
* Advance:
  * Run DISTINCT as a separate process.
  * Run GROUP BY as a separate process.
  * Run JOIN as a separate process.
  * Run ORDER BY as a separate process.
* Schema Remapping
  * Remap schemas if a source is removed and added a different source in the dataflow.
  * Input required: "Specify obsolete schema" and "Choose correct schema".
  * Input:Output is n:1
* Joins:
  * LEFT OUTER JOIN
  * INNER JOIN
* In SAP BODS, a DTD (Document Type Definition) file can be used within a Query Transform to define the structure and data validation rules for an XML file.

<img width="422" height="112" alt="image" src="https://github.com/user-attachments/assets/031ea4c4-8653-4f5e-bef4-adad9d6e1679" />

<img width="1168" height="565" alt="image" src="https://github.com/user-attachments/assets/3805657c-f914-4b94-9ce4-dca019213135" />

Notes:
* To update a default value for multiple fields, select the fields needs to be updated, go to properties, select the metadata you want to update and click on ok.
* Create a job using DTD format (reference): https://community.sap.com/t5/technology-blogs-by-members/data-transfer-from-xml-file-to-database-by-using-dtd-format/ba-p/12972178
