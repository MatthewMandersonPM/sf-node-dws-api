**AI Prompt for Developer: Build a Salesforce Flow Node for Nutrient DWS API**

**Objective**: Develop a reusable, admin-friendly Salesforce Flow node that integrates the Nutrient Document Web Services (DWS) Processor API to perform OCR, AI Redaction, and PDF/UA processing on uploaded files, and package it as a Second-Generation Managed Package (2GP) for AppExchange distribution. The solution must use the provided Swagger specification and prioritize low-code configuration for Salesforce administrators.

**Swagger Specification**:  
- URL: https://dashboard.nutrient.io/assets/specs/document-engine/upstream@1.9.0-9c1fbfe4951c0b87d91eb33a6136dea8.yml?vsn=d  
- Key Endpoint: `/build` (supports chaining OCR, redaction, and PDF/UA operations via multipart/form-data requests).

**Requirements**:

1. **Core Functionality**:
   - Create a single invocable Flow action (`Nutrient DWS Processor`) that supports:
     - **OCR**: Convert scanned documents (PDF, PNG, JPG) into searchable PDFs or extract text, with configurable language (e.g., "english") and page selection (e.g., "all", "1,3-5").
     - **AI Redaction**: Redact sensitive data (e.g., PII) using patterns or natural language instructions (e.g., “redact all SSNs”).
     - **PDF/UA**: Generate accessible PDF/UA-compliant documents with machine-readable text and metadata.
   - Use the `/build` endpoint to chain operations in one API call when multiple operations are selected (e.g., OCR + Redaction + PDF/UA).
   - Inputs:
     - `fileData` (String, required): Base64-encoded file from `ContentVersion.VersionData`.
     - `fileName` (String, required): File name with extension (e.g., `document.pdf`).
     - `operations` (Picklist, required): Options: “OCR”, “Redaction”, “PDF/UA”, “OCR+Redaction”, “OCR+PDF/UA”, “All”.
     - `language` (String, optional): OCR language, default “english”.
     - `redactionRules` (String, optional): JSON (e.g., `{ "pattern": "PII" }`) or natural language (e.g., “redact all PII”).
     - `pageIndexes` (String, optional): Pages to process, default “all”.
   - Outputs:
     - `success` (Boolean): Operation success status.
     - `processedFile` (String): Base64-encoded output file (e.g., searchable PDF/UA).
     - `extractedData` (String): JSON with OCR-extracted text or metadata.
     - `errorMessage` (String): Error details if the operation fails.

2. **Implementation Approach**:
   - Use **Salesforce External Services** to register the Nutrient DWS API with the provided Swagger file, creating an invocable action for the `/build` endpoint.
   - Avoid custom Apex unless necessary for complex logic (e.g., advanced input validation or error handling).
   - Configure a **Named Credential** (`Nutrient_DWS_API`) with:
     - URL: `https://api.nutrient.io`.
     - Authentication: Bearer token (API key) or JWT, per the Swagger’s requirements.
   - Map Flow inputs to the `/build` endpoint’s `instructions` JSON payload, e.g.:
     ```json
     {
       "parts": [{ "file": "{!fileName}" }],
       "actions": [
         { "type": "ocr", "language": "{!language}", "pageIndexes": "{!pageIndexes}" },
         { "type": "redact", "pattern": "{!redactionRules}" },
         { "type": "pdfua" }
       ]
     }
     ```

3. **Sample Flow Template**:
   - Include a prebuilt Flow template to demonstrate usage:
     - **Screen**: File Upload component to collect input file (`ContentVersionIds`, `VersionData`, `PathOnClient`).
     - **Get Records**: Retrieve `ContentVersion` to get `VersionData` (base64-encoded).
     - **Screen (Optional)**: Allow admins to select `operations` (picklist), `language`, `redactionRules`, and `pageIndexes`.
     - **Action**: Call the `Nutrient DWS Processor` action with mapped inputs.
     - **Create Records**: Save `processedFile` as a new `ContentVersion`:
       - `Title`: `{!PathOnClient}_Processed`.
       - `PathOnClient`: `{!PathOnClient}_Processed.pdf`.
       - `VersionData`: `{!processedFile}` (base64-decoded).
       - `FirstPublishLocationId`: Link to the original record (e.g., Opportunity ID).
     - **Create Records (Optional)**: Create `ContentDocumentLink` for record association.
     - **Decision**: Check `success` and display `errorMessage` if needed.
   - Ensure the Flow is editable and triggerable via buttons, Lightning Pages, or record updates.

4. **Custom Metadata for Configuration**:
   - Create a **Custom Metadata Type** (`Nutrient_DWS_Settings`) with:
     - `Default_Language__c` (String): Default OCR language (e.g., “english”).
     - `Supported_Languages__c` (String): List of supported languages (e.g., “english,deu,fra”).
     - `Default_Redaction_Rules__c` (String): Default redaction rules (e.g., “PII”).
     - `Max_File_Size__c` (Number): Max file size in MB (e.g., 50 for Salesforce REST API limit).
   - Use in Flow to pre-populate defaults or validate inputs.

5. **Packaging as 2GP**:
   - Create a **Second-Generation Managed Package** (`NutrientDWSFlowBundle`) with:
     - **External Service**: Nutrient DWS API configuration (from Swagger).
     - **Flow Template**: Sample Flow demonstrating the action.
     - **Custom Metadata Type**: `Nutrient_DWS_Settings` with default records.
     - **Permission Set**: Grant access to External Service and Custom Metadata.
     - **Documentation**: PDF or URL with setup guide, use cases, and troubleshooting.
   - Use Salesforce CLI:
     ```bash
     sfdx force:package2:create -n NutrientDWSFlowBundle -r managed -t Managed
     sfdx force:package2:version:create -p NutrientDWSFlowBundle -d force-app -x
     ```
   - Namespace: `nutrient_dws`.
   - Test in a sandbox to ensure compatibility across Salesforce editions (Professional, Enterprise, Unlimited).

6. **AppExchange Compliance**:
   - Prepare for **security review**:
     - Use Named Credentials to secure API authentication.
     - Validate inputs to prevent injection attacks (e.g., sanitize `redactionRules`).
     - Handle errors gracefully without exposing sensitive data.
     - Run Checkmarx or Salesforce’s security scanner.
   - Create an AppExchange listing:
     - Name: “Nutrient DWS Flow Bundle”.
     - Description: “Low-code Salesforce Flow action for OCR, AI Redaction, and PDF/UA using Nutrient’s Document Engine API.”
     - Documentation: Include setup (Named Credential, External Service, API key), use cases (OCR for invoices, redaction for GDPR, PDF/UA for accessibility), and troubleshooting (e.g., “Invalid Token”).
     - Demo: 2–3 minute video showing an admin configuring and running the Flow.
   - Licensing: Offer free (using Nutrient’s 100 free credits) or paid versions via License Management App.
   - Submit for review via Salesforce Partner Community (partner.salesforce.com).

7. **Best Practices for Admin Usability**:
   - Use picklists for `operations` and `language` to simplify configuration.
   - Provide clear error messages (e.g., “File exceeds 50 MB limit” or “Invalid redaction rule”).
   - Optimize for Salesforce’s 50 MB REST API limit and Nutrient’s file size restrictions.
   - Include guidance for large files (e.g., split files or use Nutrient’s Zapier integration).
   - Highlight PDF/UA compliance for accessibility in regulated industries.
   - Document setup and usage in a PDF or URL, referencing https://nutrient.io/api for pricing and API details.

8. **Testing**:
   - Test with sample files (PDF, JPG) for OCR, redaction, and PDF/UA.
   - Verify outputs (searchable PDF, redacted PDF, PDF/UA-compliant file) are saved correctly.
   - Test error handling for common issues (e.g., invalid API key, file size limits).
   - Ensure compatibility with guest users and various Salesforce editions.

9. **Notes**:
   - The Swagger (version 1.9.0) supports multi-operation chaining via `/build`.
   - Avoid relying on Nutrient’s MCP Server for redaction (macOS-only); use direct API calls.
   - Warn admins about file size limits and provide workarounds.
   - Ensure the package is upgradeable without disrupting existing Flows.

**Deliverables**:
- A 2GP managed package (`NutrientDWSFlowBundle`) with all components.
- A sample Flow template demonstrating the action.
- Documentation (PDF or URL) for admins.
- A test org for security review with sample files.

**Resources**:
- Swagger: https://dashboard.nutrient.io/assets/specs/document-engine/upstream@1.9.0-9c1fbfe4951c0b87d91eb33a6136dea8.yml?vsn=d
- Nutrient API Docs: https://nutrient.io/api
- Salesforce ISVforce Guide: https://developer.salesforce.com/docs/atlas.en-us.packagingGuide
- AppExchange Publishing: https://www.salesforceben.com/a-guide-to-publishing-your-first-appexchange-app/

**Timeline**:
- Development: 2–3 weeks.
- Testing: 1 week.
- Security Review: 4–6 weeks (start early).

**Next Steps**:
- Confirm Nutrient API key or JWT setup.
- Register External Service and test `/build` endpoint.
- Provide feedback on any specific customization needs (e.g., additional inputs or error handling).
