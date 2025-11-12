Extract & Validate Legal Citations from Documents with PDF Vector AI

https://n8nworkflows.xyz/workflows/extract---validate-legal-citations-from-documents-with-pdf-vector-ai-8504


# Extract & Validate Legal Citations from Documents with PDF Vector AI

### 1. Workflow Overview

This workflow automates the extraction, validation, and reporting of legal citations from documents stored on Google Drive. It targets legal professionals, researchers, and scholars who need to analyze large legal documents for case law, statutes, regulations, and academic citations efficiently. The workflow is structured into these logical blocks:

- **1.1 Input Reception:** Starting point for document retrieval from Google Drive.
- **1.2 Citation Extraction:** Using PDF Vector AI to extract structured citation data including case law, statutes, regulations, academic, and other citations.
- **1.3 Citation Analysis & Validation:** Processes extracted citations, validates required fields, identifies citation patterns, and collects academic DOIs.
- **1.4 Academic Papers Retrieval:** Conditional fetching of academic papers metadata using DOIs.
- **1.5 Report Generation:** Compiles a comprehensive citation report in Markdown format.
- **1.6 Export:** Saves the generated report as a Markdown file.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block starts the workflow manually and retrieves the specified legal document from Google Drive.
- **Nodes Involved:** Manual Trigger, Google Drive - Get Legal Document

##### Manual Trigger
- **Type & Role:** Manual trigger node; initiates the workflow upon user action.
- **Configuration:** No parameters; purely manual start.
- **Expressions:** None.
- **Connections:** Outputs to Google Drive - Get Legal Document.
- **Edge Cases:** User forgetting to provide the `fileId` in input JSON may cause Google Drive node to fail.
- **Notes:** Acts as entry point.

##### Google Drive - Get Legal Document
- **Type & Role:** Google Drive node; downloads the file specified by `fileId`.
- **Configuration:** Operation set to "download"; fileId is dynamically passed from input JSON (`{{$json.fileId}}`).
- **Expressions:** File ID expression dynamically set.
- **Connections:** Outputs binary file data to PDF Vector - Extract Citations.
- **Edge Cases:** 
  - Invalid or missing fileId will cause a download failure.
  - Permissions/authentication errors with Google Drive OAuth2 credentials.
- **Version:** Uses version 3 of the node.
- **Notes:** Requires configured Google Drive OAuth2 credentials.

---

#### 2.2 Citation Extraction

- **Overview:** Extracts detailed legal citations from the downloaded document using PDF Vector AI, leveraging OCR for scanned documents.
- **Nodes Involved:** PDF Vector - Extract Citations

##### PDF Vector - Extract Citations
- **Type & Role:** PDF Vector AI node; extracts structured citation data from document binary.
- **Configuration:** 
  - Operation: "extract"
  - Resource: "document"
  - Binary input property: "data" (uses binary data from Google Drive node)
  - Prompt: Detailed instruction to extract case citations, statutes, regulations, academic citations, and contextual info including page numbers.
  - Schema: Defines expected structured JSON output with citation types and their properties.
- **Expressions:** None; prompt and schema are static strings.
- **Connections:** Outputs JSON data with extracted citations to Analyze & Validate Citations.
- **Edge Cases:** 
  - OCR may fail or produce incomplete results on poor-quality scans.
  - Inaccurate extractions if document formatting is unusual.
  - Timeout or API errors.
- **Notes:** Critical node for AI-based citation detection.

---

#### 2.3 Citation Analysis & Validation

- **Overview:** Processes extracted citation data, validates required fields, builds a citation network summary, flags missing info, and prepares academic DOIs for further retrieval.
- **Nodes Involved:** Analyze & Validate Citations, Has Academic DOIs?

##### Analyze & Validate Citations
- **Type & Role:** Code node; runs JavaScript to analyze and validate citation data.
- **Configuration:** 
  - JavaScript reads citation data from previous node.
  - Calculates totals for each citation type.
  - Validates presence of required fields for case and statute citations.
  - Builds citation network mapping cases cited in the document.
  - Extracts DOIs for academic citations, flags missing DOIs with warnings.
  - Identifies primary (binding cases/statutes) and secondary (academic) authorities.
- **Expressions:** Uses `$input.first().json.data` to access citations.
- **Connections:** Outputs enriched JSON to conditional node Has Academic DOIs?.
- **Edge Cases:** 
  - Citation data missing expected arrays or fields may cause errors.
  - Logic assumes some fields like court names or DOIs may be null or missing.
  - Large citation arrays may impact performance.
- **Notes:** Core validation and enrichment logic.

##### Has Academic DOIs?
- **Type & Role:** If node; routes based on presence of academic DOIs.
- **Configuration:** Checks if `doisToFetch` string is not empty.
- **Connections:**
  - True: PDF Vector - Fetch Papers (to retrieve academic metadata)
  - False: Generate Citation Report (skip fetching)
- **Edge Cases:** DOI string formatting must be correct; empty string leads to skipping fetch.

---

#### 2.4 Academic Papers Retrieval

- **Overview:** Fetches metadata of academic papers via PDF Vector API using DOIs extracted previously.
- **Nodes Involved:** PDF Vector - Fetch Papers

##### PDF Vector - Fetch Papers
- **Type & Role:** PDF Vector AI node; fetches academic paper metadata.
- **Configuration:** 
  - Operation: "fetch"
  - Resource: "academic"
  - IDs: comma-separated DOIs from previous node.
  - Fields: title, abstract, authors, year, doi, pdfURL, totalCitations
- **Connections:** Outputs fetched paper data to Generate Citation Report.
- **Edge Cases:** 
  - API rate limits or errors.
  - Missing metadata for some DOIs.
  - Invalid DOIs may cause partial failures.
- **Notes:** Optional enrichment step for academic citations.

---

#### 2.5 Report Generation

- **Overview:** Generates a detailed Markdown report summarizing citations, validation issues, case law, statutes, academic citations with enriched metadata, and citation patterns.
- **Nodes Involved:** Generate Citation Report

##### Generate Citation Report
- **Type & Role:** Code node; creates final textual report.
- **Configuration:** 
  - Uses citation data from Has Academic DOIs? or PDF Vector - Fetch Papers (depending on path).
  - Iterates over citation types to generate sections.
  - Includes validation issues as bullet points.
  - Enriches academic citations with fetched metadata if available.
  - Lists primary and secondary authorities.
  - Outputs JSON with the report string and metadata.
- **Expressions:** Accesses nodes dynamically; uses JavaScript templating for Markdown.
- **Edge Cases:** 
  - Missing fields gracefully handled.
  - Large reports may cause performance issues.
- **Notes:** Produces exportable Markdown report.

---

#### 2.6 Export

- **Overview:** Saves the generated Markdown report as a timestamped file.
- **Nodes Involved:** Save Citation Report

##### Save Citation Report
- **Type & Role:** Write Binary File node; exports report to local filesystem.
- **Configuration:** 
  - File name uses dynamic timestamp format: `citation_report_YYYY-MM-DD_HH-mm.md`.
  - File content is the `report` string from previous node.
- **Connections:** Terminal node.
- **Edge Cases:** 
  - File system write permissions.
  - Filename conflicts or invalid characters.
- **Notes:** Final output step.

---

### 3. Summary Table

| Node Name                    | Node Type                  | Functional Role                     | Input Node(s)                 | Output Node(s)                      | Sticky Note                                                |
|------------------------------|----------------------------|-----------------------------------|------------------------------|-----------------------------------|------------------------------------------------------------|
| Manual Trigger               | manualTrigger              | Workflow start                    | —                            | Google Drive - Get Legal Document  | Start citation extraction                                  |
| Google Drive - Get Legal Document | googleDrive                | Retrieve document from Drive       | Manual Trigger               | PDF Vector - Extract Citations     | Retrieve document from Drive                               |
| PDF Vector - Extract Citations | pdfVector                  | Extract all citations              | Google Drive - Get Legal Document | Analyze & Validate Citations       | Extract all citations                                      |
| Analyze & Validate Citations | code                       | Process citation data              | PDF Vector - Extract Citations | Has Academic DOIs?                 | Process citation data                                      |
| Has Academic DOIs?           | if                         | Check if academic DOIs exist      | Analyze & Validate Citations | PDF Vector - Fetch Papers (true), Generate Citation Report (false) |                                                           |
| PDF Vector - Fetch Papers     | pdfVector                  | Retrieve academic papers           | Has Academic DOIs? (true)     | Generate Citation Report           | Retrieve academic papers                                  |
| Generate Citation Report      | code                       | Create final report                | Has Academic DOIs? (false), PDF Vector - Fetch Papers | Save Citation Report          | Create final report                                       |
| Save Citation Report          | writeBinaryFile            | Export report                     | Generate Citation Report       | —                                 | Export report                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**
   - Name: "Manual Trigger"
   - No parameters.
   - Position: (250,300).

2. **Add Google Drive node**
   - Name: "Google Drive - Get Legal Document"
   - Credentials: Configure Google Drive OAuth2 credentials.
   - Operation: "download"
   - File ID: Expression `{{$json.fileId}}` (expects input JSON with `fileId`).
   - Connect output of Manual Trigger to this node.
   - Position: (450,300).

3. **Add PDF Vector node for extraction**
   - Name: "PDF Vector - Extract Citations"
   - Credentials: Configure PDF Vector API credentials.
   - Resource: "document"
   - Operation: "extract"
   - Input Type: "file"
   - Binary Property Name: "data"
   - Prompt: Use detailed prompt asking for extraction of all legal citations including cases, statutes, regulations, academic, with context and page number; enable OCR for scanned docs.
   - Schema: Paste JSON schema defining structured citation extraction with fields for documentInfo, caseCitations, statuteCitations, regulatoryCitations, academicCitations, otherCitations.
   - Connect output of Google Drive node to this node.
   - Position: (650,300).

4. **Add Code node for analysis & validation**
   - Name: "Analyze & Validate Citations"
   - Code: Paste the provided JavaScript code that:
     - Calculates citation counts.
     - Validates required fields in case and statute citations.
     - Builds citation network.
     - Collects academic DOIs and flags warnings.
     - Identifies primary and secondary authorities.
   - Connect output of PDF Vector extraction node to this node.
   - Position: (850,300).

5. **Add If node for DOI check**
   - Name: "Has Academic DOIs?"
   - Condition: Check if `{{$json.doisToFetch}}` is not empty string.
   - Connect output of Analyze & Validate Citations node to this node.
   - Position: (1050,300).

6. **Add PDF Vector node to fetch academic papers**
   - Name: "PDF Vector - Fetch Papers"
   - Credentials: Use PDF Vector credentials.
   - Resource: "academic"
   - Operation: "fetch"
   - IDs: Expression `{{$json.doisToFetch}}` (comma-separated DOIs)
   - Fields to retrieve: title, abstract, authors, year, doi, pdfURL, totalCitations
   - Connect "true" output of If node to this node.
   - Position: (1250,250).

7. **Add Code node to generate report**
   - Name: "Generate Citation Report"
   - Code: Paste provided JavaScript that generates a Markdown report including:
     - Document info.
     - Citation summary.
     - Validation issues.
     - Detailed case law, statute, academic sections.
     - Academic paper metadata if fetched.
     - Citation patterns.
   - Connect "false" output of If node and output of PDF Vector - Fetch Papers node to this node (merge).
   - Position: (1450,300).

8. **Add Write Binary File node to save report**
   - Name: "Save Citation Report"
   - File Name: Expression `citation_report_{{ $now.format('yyyy-MM-dd_HH-mm') }}.md`
   - File Content: Expression `{{$json.report}}`
   - Connect output of Generate Citation Report node to this node.
   - Position: (1650,300).

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow uses PDF Vector AI for advanced citation extraction and academic paper metadata retrieval.            | PDF Vector API documentation                         |
| Google Drive node requires OAuth2 credentials with read access to files.                                       | Google Drive API & OAuth2 setup                      |
| The citation extraction prompt is detailed to ensure inclusion of various legal citation types and context.    | Prompt engineering best practices                    |
| Academic DOI detection enables enrichment of citation data with paper abstracts and citation counts.           | Crossref or DOI metadata APIs                         |
| The generated report is Markdown formatted, suitable for further processing or sharing in legal workflows.    | Markdown formatting standards                         |
| Potential enhancements include error handling for API rate limits and retry logic for robustness.              | n8n workflow error handling documentation            |

---

**Disclaimer:** The text above is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.