# PDF File Analyser: Design Proposal

This design is based on a review of [jfwberg/Agentforce-File-Analyser](https://github.com/jfwberg/Agentforce-File-Analyser), version 0.3.0-1. It suggests a PDF analyser for Salesforce that keeps that repo's main ideas (Prompt Builder templates that take a `ContentDocument` as input) and reworks the parts that are demo-only so the result can go to production.

---

## 1. What the reference repo does

| Area | Implementation |
|---|---|
| **UI** | Four LWCs on App pages: `agentforceFileAnalyser` (free-form Q&A), `agentforceTableExtractor`, `agentforceProcessAnalyser`, `agentforceProcessQuestion`. A small `llmDatatable` renders extracted tables. |
| **Apex** | Controllers with one `@AuraEnabled` method each: `AgentforceFileAnalyserCtrl.analyseFile`, `AgentforceTableExtractorCtrl`, `AgentforceProcess{Extractor,Validator,Fixer,Question,RecordCreator}Ctrl`. `AgentforceProcess` is a typed JSON wrapper that includes `validateJson`. |
| **LLM call** | `ConnectApi.EinsteinLLM.generateMessagesForPromptTemplate(templateName, input)`. The file is passed as `Input:File` = `{Id: contentDocumentId}` (a `SOBJECT://ContentDocument` template input). |
| **Prompt templates** | Flex templates (`einstein_gpt__flex`): `File_Analyser`, `Table_Extractor`, and `Process_Extractor/Validator/Fixer/Question`. Each has a GPT-4o-mini and a Gemini 2.5 Pro variant. |
| **Output contract** | The table extractor returns minified JSON with the shape `{message, category, summary, tables:[{name, description, columns:[{fieldName,label,type}], data:[...]}]}`. That shape maps directly to `lightning-datatable`. |
| **Data model** | `Process__c`, then `Step__c`, then `Outcome__c`. Used only by the process-flow feature. |
| **Self-correction loop** | Extract, then validate (a second LLM call), then fix (a third LLM call), then persist. This is the best pattern in the repo. |

### Gaps to fix for a real PDF analyser

1. **Nothing is persisted.** The file is deleted right after analysis (`deleteFile`) and the answer exists only in the LWC. That leaves no audit trail, no reuse, and no link to a business record.
2. **Synchronous LLM call from `@AuraEnabled`.** Large PDFs can run past the request timeout, and the user has to wait with a spinner.
3. **Model-to-template mapping is hard-coded** in static Maps (`LLM_MODEL_TO_PROMPT_TEMPLATE_MAPPING_*`). Adding a model requires a deployment.
4. **`enablePiiMasking = false`.** PII masking in the Trust Layer is turned off.
5. **JSON is parsed in JavaScript** after `removeStart('```json')`, with no schema validation for file or table output.
6. **Tests rely on `Test.isRunningTest()` branches**, so the real code path is never covered. There is no seam for mocking the LLM.
7. **App-page only.** The LWCs are not record-aware, so a PDF cannot be analysed in the context of an Account, Case, or Opportunity.
8. **No document-type-specific extraction.** You can ask a question or pull tables, but you cannot say "this is an invoice, fill these fields".

---

## 2. Proposed solution: a PDF Analyser that is record-aware, async, and config-driven

### 2.1 Capabilities

1. **Classify** the PDF (Invoice, Contract, Purchase Order, KYC/ID, Bank Statement, Other) and summarise it.
2. **Extract structured fields** for that document type, driven by Custom Metadata. Each field comes with a confidence score and the page number it came from.
3. **Extract tables** using the reference repo's JSON contract, rendered with `lightning-datatable`.
4. **Validate and fix** the output with a second prompt, following the reference repo's pattern.
5. **Human review, then apply.** The user checks the extracted values and clicks **Apply** to write them to the parent record, such as `Invoice__c` or `Opportunity`.
6. **Q&A** on a stored analysis without re-uploading the file.
7. **Agentforce and Flow entry point** through an `@InvocableMethod`, so an agent action ("analyse the attached contract") reuses the same service.

### 2.2 Architecture

```
 Record Page (Case / Account / Opportunity / Invoice__c)
 └── LWC  pdfAnalyser  (lightning-file-upload, recordId-aware)
       │ 1. startAnalysis(contentDocumentId, recordId, docType?)
       ▼
 PdfAnalyserController (@AuraEnabled, with sharing)
       │ 2. insert Document_Analysis__c (Status = Queued)
       │ 3. System.enqueueJob(new PdfAnalysisQueueable(analysisId))
       ▼
 PdfAnalysisQueueable (Database.AllowsCallouts)
       │ 4. resolve config  ← PDF_Analyser_Config__mdt
       │ 5. ILlmService.generate(template, inputs)   ← Prompt Builder
       │ 6. parse + validate → PdfAnalysisResult (typed wrapper)
       │ 7. (optional) Validator / Fixer prompt loop (max N retries)
       │ 8. upsert Extracted_Field__c / Extracted_Table__c
       │ 9. EventBus.publish(PDF_Analysis_Event__e)
       ▼
 LWC subscribes via lightning/empApi → refreshes, shows review screen
       │ 10. applyToRecord(analysisId)  → field mapping from CMDT
```

Optional third-party path, useful for scanned or image-only PDFs, very large files, or when a deterministic OCR engine is required: `IDocumentExtractor` has a second implementation that calls AWS Textract, Azure Document Intelligence, or Google Document AI through a **Named Credential + External Credential**. The downstream data model stays the same.

---

## 3. Data model

### `Document_Analysis__c` (one record per PDF analysed)

| Field | Type | Notes |
|---|---|---|
| `Name` | Auto Number | `DA-{00000}` |
| `Content_Document_Id__c` | Text(18) | The file is kept, not deleted (retention policy is configurable) |
| `Content_Version_Id__c` | Text(18) | Pins the exact version that was analysed |
| `File_Name__c` | Text(255) | |
| `Related_Record_Id__c` | Text(18) | Polymorphic parent. Add typed lookups (`Case__c`, `Account__c`, …) if you need reporting |
| `Document_Type__c` | Picklist | Invoice, Contract, Purchase_Order, KYC, Bank_Statement, Other |
| `Status__c` | Picklist | Queued, Processing, Needs_Review, Completed, Applied, Failed |
| `Model_Name__c` | Text(80) | e.g. `sfdc_ai__DefaultOpenAIGPT4OmniMini` |
| `Prompt_Template__c` | Text(80) | Template API name used |
| `Category__c` | Text(80) | Returned by the LLM |
| `Summary__c` | Long Text(32k) | |
| `Overall_Confidence__c` | Percent | Minimum or average of the field confidences |
| `Validation_Result__c` | Long Text(32k) | Output of the validator prompt |
| `Raw_Response__c` | Long Text(131072) | Full JSON for debugging and replay |
| `Retry_Count__c` | Number(2,0) | Fixer-loop iterations |
| `Error_Message__c` | Long Text(32k) | |
| `Processing_Time_Ms__c` | Number | |

### `Extracted_Field__c` (Master-Detail to `Document_Analysis__c`)

| Field | Type | Notes |
|---|---|---|
| `Field_Key__c` | Text(80) | e.g. `invoice_number` |
| `Field_Label__c` | Text(255) | |
| `Value__c` | Long Text(32k) | Value as extracted |
| `Data_Type__c` | Picklist | text, number, currency, date, boolean |
| `Confidence__c` | Percent | |
| `Page_Number__c` | Number | Used to jump to the source page |
| `Source_Snippet__c` | Text(255) | Evidence text from the PDF |
| `Target_Field_API__c` | Text(80) | From the CMDT mapping |
| `Corrected_Value__c` | Long Text(32k) | Reviewer override |
| `Is_Verified__c` | Checkbox | |

### `Extracted_Table__c` (Master-Detail to `Document_Analysis__c`)

`Table_Index__c`, `Description__c`, `Columns_JSON__c` (Long Text), `Rows_JSON__c` (Long Text), `Row_Count__c`. The JSON uses the reference repo's table contract, so the `llmDatatable` component can be reused unchanged.

### `PDF_Analyser_Config__mdt` (Custom Metadata, one row per document type)

| Field | Purpose |
|---|---|
| `Document_Type__c` | Key |
| `Model_Name__c` | Replaces the hard-coded model map |
| `Extract_Template__c` / `Validate_Template__c` / `Fix_Template__c` | Prompt template API names |
| `Field_Schema_JSON__c` | Fields the LLM must return, e.g. `[{"key":"invoice_number","label":"Invoice #","type":"text","required":true}]` |
| `Target_Object__c` | e.g. `Invoice__c` |
| `Field_Mapping_JSON__c` | `{"invoice_number":"Invoice_Number__c","total":"Amount__c"}` |
| `Min_Confidence__c` | At or above this value, Status can go straight to Completed. Below it, Status is Needs_Review |
| `Max_Fix_Retries__c` | Limit for the validator/fixer loop |
| `Is_Active__c` | |

### `PDF_Analysis_Event__e` (Platform Event, Publish After Commit)

`Analysis_Id__c`, `Status__c`, `User_Id__c`. The LWC filters events on `User_Id__c`.

---

## 4. Apex design

```
classes/
  PdfAnalyserController.cls        @AuraEnabled: startAnalysis, getAnalysis, applyToRecord, askQuestion
  PdfAnalysisQueueable.cls         Queueable + AllowsCallouts; orchestration and retry loop
  PdfAnalysisService.cls           Business logic (classify → extract → validate → fix → persist)
  ILlmService.cls                  interface: String generate(String template, Map<String,Object> inputs)
  PromptTemplateLlmService.cls     ConnectApi.EinsteinLLM implementation (PII masking ON)
  PdfAnalysisResult.cls            Typed wrapper + validate() (same idea as AgentforceProcess.validateJson)
  PdfFieldMapper.cls               CMDT mapping → SObject, Schema.describe type coercion, FLS check
  PdfAnalyserInvocable.cls         @InvocableMethod for Flow / Agentforce agent actions
  PdfAnalyserSelector.cls          SOQL (WITH USER_MODE)
  tests/ MockLlmService.cls + *Test.cls
```

### Key snippets

**LLM seam.** This replaces the `Test.isRunningTest()` branches:

```apex
public interface ILlmService {
    String generate(String promptTemplateName, Map<String, Object> inputs);
}

public with sharing class PromptTemplateLlmService implements ILlmService {
    public String generate(String promptTemplateName, Map<String, Object> inputs) {
        Map<String, ConnectApi.WrappedValue> params = new Map<String, ConnectApi.WrappedValue>();
        for (String key : inputs.keySet()) {
            ConnectApi.WrappedValue wv = new ConnectApi.WrappedValue();
            wv.value = inputs.get(key);
            params.put(key, wv);
        }
        ConnectApi.EinsteinLlmAdditionalConfigInput cfg = new ConnectApi.EinsteinLlmAdditionalConfigInput();
        cfg.applicationName  = 'PromptTemplateGenerationsInvocable';
        cfg.enablePiiMasking = true;      // reference repo had this off
        cfg.temperature      = 0.1;       // extraction should be deterministic

        ConnectApi.EinsteinPromptTemplateGenerationsInput req = new ConnectApi.EinsteinPromptTemplateGenerationsInput();
        req.additionalConfig = cfg;
        req.isPreview        = false;
        req.inputParams      = params;

        ConnectApi.EinsteinPromptTemplateGenerationsRepresentation res =
            ConnectApi.EinsteinLLM.generateMessagesForPromptTemplate(promptTemplateName, req);
        return stripFences(res.generations[0].text);
    }

    @TestVisible
    private static String stripFences(String s) {
        return s == null ? null : s.trim().removeStart('```json').removeStart('```').removeEnd('```').trim();
    }
}
```

**Controller.** Returns immediately. The work happens in the Queueable:

```apex
public with sharing class PdfAnalyserController {
    @AuraEnabled
    public static Id startAnalysis(Id contentDocumentId, Id recordId, String documentType) {
        ContentVersion cv = [SELECT Id, Title, FileExtension, ContentSize
                             FROM ContentVersion
                             WHERE ContentDocumentId = :contentDocumentId AND IsLatest = true
                             WITH USER_MODE LIMIT 1];
        if (cv.FileExtension?.toLowerCase() != 'pdf') {
            throw new AuraHandledException('Only PDF files are supported.');
        }
        Document_Analysis__c da = new Document_Analysis__c(
            Content_Document_Id__c = contentDocumentId,
            Content_Version_Id__c  = cv.Id,
            File_Name__c           = cv.Title,
            Related_Record_Id__c   = recordId,
            Document_Type__c       = documentType,   // null → classify first
            Status__c              = 'Queued'
        );
        insert as user da;
        System.enqueueJob(new PdfAnalysisQueueable(da.Id));
        return da.Id;
    }
}
```

**Queueable orchestration** with a validate/fix loop:

```apex
public with sharing class PdfAnalysisQueueable implements Queueable, Database.AllowsCallouts {
    @TestVisible private static ILlmService llm = new PromptTemplateLlmService();
    private final Id analysisId;
    public PdfAnalysisQueueable(Id analysisId) { this.analysisId = analysisId; }

    public void execute(QueueableContext ctx) {
        new PdfAnalysisService(llm).run(analysisId); // classify → extract → validate → (fix ≤ N) → persist → publish event
    }
}
```

Callout budget: each Queueable allows up to 100 callouts and 120 s in total. Classify, extract, validate, and up to 2 fixes add up to about 5 calls, which fits in one job. If you need more than that, chain a new Queueable for each stage.

---

## 5. Prompt templates

Keep the reference repo's Flex-template approach (`SOBJECT://ContentDocument` input), with one template per stage:

| Template | Inputs | Output |
|---|---|---|
| `PDF_Classifier` | `File` | `{"documentType":"Invoice","confidence":0.93,"summary":"..."}` |
| `PDF_Field_Extractor` | `File`, `FieldSchema` (String, from CMDT) | `{"fields":[{"key","value","type","confidence","page","snippet"}],"tables":[…ref-repo table contract…]}` |
| `PDF_Extraction_Validator` | `File`, `ExtractionJSON` | `{"isValid":bool,"issues":[{"key","problem"}]}` |
| `PDF_Extraction_Fixer` | `File`, `ExtractionJSON`, `Issues` | Same shape as the extractor output |
| `PDF_Question` | `File`, `Question`, `PriorSummary` | Free text answer with page citations |

Prompt rules carried over from the reference repo's `Table_Extractor`, which worked well there: "return a single minified JSON object", an explicit schema, `null` for missing values, and an explicit "no data found" response. Add these two rules: "**never invent values; if a value is not in the document, return null with confidence 0**" and "**always return the page number**".

---

## 6. LWC design

`pdfAnalyser` (targets: `lightning__RecordPage`, `lightning__AppPage`, `lightning__FlowScreen`)

- **Step 1, Upload:** `lightning-file-upload` with `record-id={recordId}` and `accept=".pdf"`, plus an optional document-type combobox (default: Auto-detect).
- **Step 2, Processing:** show a progress indicator (Queued, then Processing, then Validating) and subscribe to `/event/PDF_Analysis_Event__e` through `lightning/empApi`. Use `getAnalysis` polling as a fallback.
- **Step 3, Review:**
  - Summary card showing category, overall confidence, and validator notes.
  - An editable `lightning-datatable` of `Extracted_Field__c` rows. Rows below `Min_Confidence__c` are highlighted, and each row has a "page N" link that opens `/sfc/servlet.shepherd/version/renditionDownload?rendition=SVGZ&versionId=…` or the file preview through `NavigationMixin` (`standard__namedPage` → `filePreview`).
  - Extracted tables rendered with the reused `llmDatatable`.
- **Step 4, Apply:** calls `applyToRecord`. `PdfFieldMapper` handles type coercion and FLS/CRUD checks through `Security.stripInaccessible`, then sets Status to Applied.
- **Q&A tab:** reuses the flow of `agentforceProcessQuestion`, but works against a stored `Document_Analysis__c`.

A related list, `Document_Analysis__c`, on the parent record gives you the audit history.

---

## 7. Security and governance

- Enable **PII masking** in the Einstein Trust Layer, and turn on Data Cloud audit/feedback if your org uses it.
- Use `with sharing` everywhere, `WITH USER_MODE` in SOQL, and `insert as user`/`update as user`.
- Create a permission set `PDF_Analyser_User` that grants the objects, the Apex classes, and the Prompt Template user permission. The reference repo required `EinsteinGPTPromptTemplateManager`, which is a manager-level permission. Give end users the lower **Prompt Template User** permission instead.
- Keep files and apply a retention policy through a scheduled batch, rather than deleting them immediately. Deleting is an option on the CMDT row for sensitive document types.
- Check file size and page count before enqueueing, and reject files above the current Prompt Builder file-input limit for your org's model.

---

## 8. Testing strategy

- `MockLlmService implements ILlmService` returns canned JSON for each template name: valid output, invalid output followed by fixed output, and malformed output.
- Inject the mock through `PdfAnalysisQueueable.llm = new MockLlmService();` and wrap `Test.startTest()`/`stopTest()` around the enqueue.
- Assert on status transitions, child record counts, retry counts, Platform Event publishing (`Test.getEventBus().deliver()`), and field mapping with type coercion.
- Add Jest tests for the LWC that mock `empApi`, the Apex imports, and the review/apply UI states. The reference repo already has a Jest setup to follow.

---

## 9. Delivery plan

| Phase | Scope |
|---|---|
| **1: MVP** | Data model, CMDT, `ILlmService`, Queueable, the Classifier and Field Extractor templates, and an LWC with upload, status, and read-only results. |
| **2: Trust** | Validator/Fixer loop, confidence scores, the review UI with corrections, and Apply to record. |
| **3: Scale** | `@InvocableMethod` and an Agentforce agent action, Flow screen support, a batch for re-analysis and retention, and a reporting dashboard. |
| **4: Integration** | Named Credential connector to an external OCR/IDP service for scanned PDFs, selected per document type in the CMDT. |

---

## 10. Reuse from the reference repo

| Reuse as-is | Adapt |
|---|---|
| `llmDatatable` LWC | `AgentforceFileAnalyserCtrl.analyseFile`, split into Controller, Queueable, and `ILlmService` |
| Table JSON contract from `Table_Extractor` | `AgentforceProcess.validateJson`, generalised into `PdfAnalysisResult.validate()` |
| Extract, validate, fix pattern | Model/template maps, moved to `PDF_Analyser_Config__mdt` |
| Flex prompt with `SOBJECT://ContentDocument` input | `agentforceFileAnalyser` LWC, made record-aware and async |
