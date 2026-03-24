# MyChart Scraper API Catalog (from `scrapers/myChart`)

This document summarizes how auth/session state works and catalogs the Epic MyChart endpoints used by the scraper code under `scrapers/myChart/**`.

## Scope and method

- Parsed TypeScript files in `scrapers/myChart` and nested folders.
- Excluded test fixtures and mock files (`__tests__`, `mock_data`, `docs`, `clo-to-jpg-converter`) because they do not represent production API behavior.
- API details are reverse-engineered from request bodies, response parsing, and return types in this repository.

---

## 1) Authentication and session model

## 1.1 Transport/session container

`MyChartRequest` is the shared authenticated HTTP client:

- Wraps `fetch` with a cookie jar (`fetch-cookie` + `tough-cookie`).
- Stores:
  - `hostname`
  - `protocol` (`https` by default, `http` for localhost/dev)
  - `firstPathPart` (e.g. `MyChart` / `MyChart-PRD`)
- Supports serialized cookie/session persistence.
- Automatically follows 301/302 redirects unless disabled.

Auth implications:

- **Cookie jar is the primary auth state** for all downstream API calls.
- **`__RequestVerificationToken` (CSRF token)** is retrieved from HTML pages and sent on many POST APIs.

## 1.2 Discovering base path (`firstPathPart`)

Before login, the scraper probes the root hostname and determines the MyChart path prefix from:

- `Location` redirect header, or
- `meta http-equiv="REFRESH"` HTML tag.

All path-based requests are then built as:

`{protocol}://{hostname}/{firstPathPart}{path}`

## 1.3 Username/password login

Primary login flow (`myChartUserPassLogin`):

1. `GET /Authentication/Login` to get CSRF and hidden navigation fields.
2. Optional login-page JS inspection to detect credential field variant:
   - `LoginIdentifier` (newer), or
   - `Username` (older)
3. `POST /Authentication/Login/DoLogin` with form-encoded body:
   - `__RequestVerificationToken`
   - `LoginInfo` (JSON with base64-unicode encoded username/password)
   - navigation telemetry fields (`__NavigationRequestMetrics`, etc.)

Outcome states:

- `logged_in`
- `need_2fa`
- `invalid_login`
- `error`

## 1.4 Secondary validation (2FA)

When login lands on Secondary Validation:

- CSRF is parsed from secondary auth page.
- Delivery method inferred from page content (email/SMS, masked contact if present).
- If not TOTP mode:
  - optional consent check `GET /Authentication/SecondaryValidation/GetSMSConsentStrings`
  - trigger code via `POST /Authentication/SecondaryValidation/SendCode`
- Complete verification via `POST /Authentication/SecondaryValidation/Validate` with form payload:
  - `TwoFactorCode`
  - `RememberMe`
  - `Workflow`
  - `isTOTP`

2FA completion state:

- `logged_in`
- `invalid_2fa`
- `error`

## 1.5 Terms & Conditions gate

Some tenants force redirect to `/Authentication/TermsConditions` post-login.

`acceptTermsAndConditions`:

1. Loads T&C page.
2. Extracts hidden form inputs + CSRF.
3. Replays form POST to T&C action (or fallback path).
4. If needed, follows accept/agree/continue links.

Login and 2FA paths both auto-call this helper if T&C is detected.

## 1.6 Session validity and keep-alive

- `areCookiesValid` checks `/Home` with `followRedirects: false`; 200 means session is still valid.
- `sessionStore` keeps sessions alive periodically using:
  - `/Home/KeepAlive?cnt=...`
  - fallback `/keepalive.asp?cnt=...`

## 1.7 TOTP management APIs

TOTP setup (`setupTotp`) flow:

1. Get CSRF (`/Home/CSRFToken`) 
2. `POST /api/secondary-validation/GetTwoFactorInfo`
3. `POST /api/secondary-validation/VerifyPasswordAndUpdateContact` body `{ Password }`
4. `POST /api/secondary-validation/TotpQrCode` -> extract secret from response variants
5. Generate local code, `POST /api/secondary-validation/VerifyCode` body `{ Code }`
6. `POST /api/secondary-validation/UpdateTwoFactorTotpOptInStatus`

TOTP disable (`disableTotp`) uses:

- `VerifyPasswordAndUpdateContact`
- `VerifyCode`
- `UpdateTwoFactorTotpOptInStatus`

---

## 2) API catalog by feature

For each area: endpoint(s), request schema used by scraper, and output schema (typed return).

## 2.1 Activity feed

- Function: `getActivityFeed(mychartRequest)`
- Endpoints:
  - `GET /app/home` (token/bootstrap page)
  - `POST /api/item-feed/FetchItemFeed`
- Request schema: tokenized JSON POST (payload from scraper defaults)
- Output schema: `ActivityFeedItem[]`
  - `{ id, title, description, date, type, isRead }`

## 2.2 Allergies

- Function: `getAllergies(mychartRequest)`
- Endpoints:
  - `GET /Clinical/Allergies`
  - `POST /api/allergies/LoadAllergies`
- Output schema: `AllergiesResult`
  - `allergies: Allergy[]`
  - `allergiesStatus: number`
  - `Allergy = { name, id, formattedDateNoted, type, reaction, severity }`

## 2.3 Medical history

- Function: `getMedicalHistory(mychartRequest)`
- Endpoints:
  - `GET /app/histories`
  - `POST /api/histories/LoadHistoriesViewModel`
- Output schema: `MedicalHistoryResult`
  - `medicalHistory: { diagnoses: Diagnosis[]; notes: string }`
  - `surgicalHistory: { surgeries: Surgery[]; notes: string }`
  - `familyHistory: FamilyMember[]`

## 2.4 Health summary

- Function: `getHealthSummary(mychartRequest)`
- Endpoints:
  - `GET /app/health-summary`
  - `POST /api/health-summary/FetchHealthSummary`
  - `POST /api/health-summary/FetchH2GHeader`
- Output schema: `HealthSummary`
  - demographics and vitals snapshot (`patientAge`, `height`, `weight`, `bloodType`, `lastVisit`, etc.)

## 2.5 Health issues

- Function: `getHealthIssues(mychartRequest)`
- Endpoints:
  - `GET /Clinical/HealthIssues`
  - `POST /api/HealthIssues/LoadHealthIssuesData`
- Output schema: `HealthIssue[]`
  - `{ name, id, formattedDateNoted, isReadOnly }`

## 2.6 Medications + refill

### Medications list
- Function: `getMedications(mychartRequest)`
- Endpoints:
  - `GET /Clinical/Medications`
  - `POST /api/medications/LoadMedicationsPage`
- Output schema: `MedicationsResult`
  - `patientFirstName`
  - `medications: Medication[]`

### Medication refill action
- Function: `requestMedicationRefill(mychartRequest, medicationKey)`
- Endpoints:
  - `GET /Clinical/Medications` (token bootstrap)
  - `POST /api/medications/RequestRefill`
- Request schema:
  - includes medication key identifier for the refill target
- Output schema: `RefillRequestResult`
  - `{ success: boolean, error?: string }`

## 2.7 Immunizations

- Function: `getImmunizations(mychartRequest)`
- Endpoints:
  - `GET /Clinical/Immunizations`
  - `POST /api/immunizations/LoadImmunizations`
- Output schema: `Immunization[]`
  - `{ name, id, administeredDates, organizationName }`

## 2.8 Vitals / flowsheets

- Function: `getVitals(mychartRequest)`
- Endpoints:
  - `GET /app/track-my-health`
  - `POST /api/track-my-health/GetFlowsheets`
- Output schema: `Flowsheet[]`
  - `Flowsheet = { name, flowsheetId, readings: VitalReading[] }`
  - `VitalReading = { date, value, units }`

## 2.9 Care team / journeys / goals / preventive care

### Care team
- Function: `getCareTeam`
- Endpoint: `GET /Clinical/CareTeam`
- Output: `CareTeamMember[]` (`name`, `role`, `specialty`)

### Care journeys
- Function: `getCareJourneys`
- Endpoints:
  - `GET /app/care-journeys`
  - `POST /api/care-journeys/GetCareJourneys`
- Output: `CareJourney[]`

### Goals
- Function: `getGoals`
- Endpoints:
  - `GET /app/goals`
  - `POST /api/goals/LoadCareTeamGoals`
  - `POST /api/goals/LoadPatientGoals`
- Output: `GoalsResult`
  - `careTeamGoals: Goal[]`
  - `patientGoals: Goal[]`

### Preventive care
- Function: `getPreventiveCare`
- Endpoint: `GET /HealthAdvisories`
- Output: `PreventiveCareItem[]`

## 2.10 Profile / contact / insurance / emergency contacts

### Profile and email
- Functions: `getMyChartProfile`, `getEmail`
- Endpoints:
  - `GET /Home`
  - `GET /PersonalInformation`
  - `GET /PersonalInformation/GetContactInformation?noCache=...`
- Output schema:
  - `ProfileData = { name, dob, mrn, pcp, email? }`
  - `getEmail` returns `string | null`

### Insurance
- Function: `getInsurance`
- Endpoint: `GET /Insurance`
- Output: `InsuranceResult`
  - `coverages: InsuranceCoverage[]`
  - `hasCoverages: boolean`

### Emergency contacts
- Functions: `getEmergencyContacts`, `addEmergencyContact`, `updateEmergencyContact`, `removeEmergencyContact`
- Endpoints:
  - `GET /app/personal-information`
  - `POST /api/personalInformation/GetRelationships`
  - `POST /api/personalInformation/AddRelationship`
  - `POST /api/personalInformation/UpdateRelationship`
  - `POST /api/personalInformation/RemoveRelationship`
- Request schemas:
  - Add: `EmergencyContactInput { name, relationshipType, phoneNumber }`
  - Update: `EmergencyContactUpdateInput { id, name?, relationshipType?, phoneNumber? }`
  - Remove: identifier payload for contact record
- Output schemas:
  - list: `EmergencyContact[]`
  - mutations: `EmergencyContactResult { success, error? }`

## 2.11 Referrals / orders / questionnaires

### Referrals
- Function: `getReferrals`
- Endpoints:
  - `GET /app/referrals`
  - `POST /api/referrals/listReferrals`
- Output: `Referral[]`

### Upcoming orders
- Function: `getUpcomingOrders`
- Endpoints:
  - `GET /app/upcoming-orders`
  - `POST /api/upcoming-orders/GetUpcomingOrders`
- Output: `UpcomingOrder[]`

### Questionnaires
- Function: `getQuestionnaires`
- Endpoints:
  - `GET /Questionnaire`
  - `POST /Questionnaire/GetQuestionnaireList`
- Output: `Questionnaire[]`

## 2.12 Documents / letters / education materials / EHI export

### Documents
- Function: `getDocuments`
- Endpoints:
  - `GET /app/documents`
  - `POST /api/documents/viewer/LoadOtherDocuments`
- Output: `Document[]`

### Letters
- Functions: `getLetters`, `getLetterDetails`
- Endpoints:
  - `GET /app/letters`
  - `POST /api/letters/GetLettersList`
  - `POST /api/letters/GetLetterDetails`
- Request schema for details includes identifiers (`hnoId`, `csn`)
- Output schemas:
  - list: `Letter[]`
  - details: `LetterDetailsResponse { bodyHTML }`

### Education materials
- Function: `getEducationMaterials`
- Endpoints:
  - `GET /app/education`
  - `POST /api/education/GetPatEducationTitles`
- Output: `EducationMaterial[]`

### EHI export templates
- Function: `getEhiExportTemplates`
- Endpoints:
  - `GET /app/release-of-information`
  - `POST /api/release-of-information/GetEHIETemplates`
- Output: `EhiTemplate[]`

## 2.13 Visits

- Functions:
  - `upcomingVisits(mychartRequest)`
  - `pastVisits(mychartRequest, oldestRenderedDate)`
- Endpoints:
  - `GET /Home/CSRFToken?noCache=...`
  - `POST /Visits/VisitsList/LoadUpcoming?...`
  - `POST /Visits/VisitsList/LoadPast?...`
- Request schema:
  - CSRF token header
  - Past visits includes `oldestRenderedDate` + form field `serializedIndex`
- Output schemas:
  - upcoming: `VisitListContainer`
  - past: `PastVisitsContainer`
  - both contain detailed `Visit` records (appointment metadata, provider/facility, telemedicine/check-in/payment/status fields)

## 2.14 Billing and statements

- Functions:
  - `getBillingHistory`
  - `getStatementList`
  - `getEncBillingId`
  - `saveStatementPdf`
  - `getBillingStatementPDFs`
- Endpoints:
  - `GET /Billing/Summary`
  - `GET /Billing/Details/GetVisits?...`
  - `GET /Billing/Details/GetStatementList?...`
  - `GET /Billing/Details?ID=...&Context=...`
  - `GET /Billing/Details/DownloadFromBlob/?...` (PDF binary)
- Request schema:
  - billing account IDs/context extracted from billing HTML and payment URL
  - date range params converted to Epic DTE integers
- Output schemas:
  - account list: `BillingAccount[]`
  - visit details: `BillingDetails`
  - statements: `StatementListResponse`
  - PDF: binary `Buffer`

## 2.15 Labs, procedures, and imaging

### Lab results
- Function: `listLabResults`
- Endpoints:
  - `GET /app/test-results`
  - `POST /api/test-results/GetList`
  - `POST /api/test-results/GetDetails`
  - `POST /api/past-results/GetMultipleHistoricalResultComponents`
  - `POST /api/report-content/LoadReportContent`
- Output schema:
  - `LabTestResultWithHistory[]`
  - includes nested `LabResult`, result components, provider comments, report metadata/content, and historical trend series.

### Imaging results + viewer bootstrap
- Function: `getImagingResults`
- Endpoints:
  - lab/report content endpoints above
  - `GET /Extensibility/Redirection/FdiData?...` (SAML URL broker)
- Output schema:
  - `ImagingResult[]`
  - extends lab result with:
    - `fdiContext { fdi, ord }`
    - `samlUrl`
    - optional `viewerUrl`
    - extracted `reportText`

### eUnity viewer + direct download helpers
- Functions:
  - `getImageViewerSamlUrl`
  - `followSamlChain`
  - `downloadImagingJpegs`
  - `probeDicomWeb`
  - `downloadImagingDirect` / `downloadImagingStudyDirect`
- Endpoint patterns include:
  - `/Home/CSRFToken`
  - `/api/report-content/LoadReportContent`
  - `/Extensibility/Redirection/FdiData`
  - plus discovered eUnity/DICOM endpoints during SAML chain traversal.
- Output schemas:
  - session metadata: `ImagingViewerSession`
  - JPEG download bundle: `ImageDownloadResult`
  - direct DICOM bundle: `DirectDownloadResult`

## 2.16 Messaging / communication center

### Conversation listing and threads
- Functions:
  - `listConversations`
  - `listConversationsWithFullHistory`
  - `getConversationMessages`
- Endpoints:
  - `GET /app/communication-center`
  - `POST /api/conversations/GetConversationList`
  - `POST /api/conversations/GetConversationMessages`
- Output schemas:
  - list: `ConversationListResponse`
  - one thread: `ConversationThread`
  - expanded: `ConversationsWithFullHistory`

### Send new message
- Functions:
  - `getVerificationToken`
  - `getMessageTopics`
  - `getMessageRecipients`
  - `sendNewMessage`
- Endpoints:
  - `POST /api/medicaladvicerequests/GetSubtopics`
  - `POST /api/medicaladvicerequests/GetMedicalAdviceRequestRecipients`
  - `POST /api/medicaladvicerequests/GetViewers`
  - `POST /api/conversations/GetComposeId`
  - `POST /api/medicaladvicerequests/SendMedicalAdviceRequest`
  - `POST /api/conversations/RemoveComposeId`
- Request schema (`SendMedicalAdviceRequest`):
  - `recipient { displayName, userId, poolId, providerId, departmentId }`
  - `topic { title, value }`
  - `conversationId: ''`
  - `organizationId`
  - `viewers: [{ wprId }]`
  - `messageBody: string[]`
  - `messageSubject`
  - `documentIds: []`
  - `includeOtherViewers: false`
  - `composeId`
- Output schema: `SendNewMessageResult { success, conversationId?, error? }`

### Send reply
- Function: `sendReply`
- Endpoints:
  - `POST /api/medicaladvicerequests/GetViewers`
  - `POST /api/conversations/GetComposeId`
  - `POST /api/conversations/SendReply`
  - `POST /api/conversations/RemoveComposeId`
- Request schema (`SendReply`):
  - `conversationId`
  - `organizationId`
  - `viewers: [{ wprId }]`
  - `messageBody: string[]`
  - `documentIds: []`
  - `includeOtherViewers: false`
  - `composeId`
- Output schema: `SendReplyResult { success, conversationId?, error? }`

### Drafts and delete
- Functions:
  - `saveReplyDraft`
  - `saveNewMessageDraft`
  - `deleteDraft`
  - `deleteMessage`
- Endpoints:
  - `POST /api/conversations/SaveReplyDraft`
  - `POST /api/medicaladvicerequests/SaveMedicalAdviceRequestDraft`
  - `POST /api/conversations/DeleteDraft`
  - `POST /api/conversations/DeleteConversation`
- Request schemas:
  - reply draft: `{ conversationId, messageBody: string[] }`
  - new message draft: `{ messageBody: string[], messageSubject }`
  - delete draft: `{ conversationId }`
  - delete conversation: `{ conversationId }`
- Output schemas:
  - `DraftResult { success, error? }`
  - `DeleteMessageResult { success, error? }`

## 2.17 Linked MyChart accounts

- Function: `getLinkedMyChartAccounts`
- Endpoints:
  - `GET /Community/Manage`
  - `POST /Community/Shared/LoadCommunityLinks?noCache=...`
- Output schema: `LinkedMyChart[]`
  - `{ name, logoUrl, lastEncounter }`

---

## 3) Security mechanics used across APIs

Common patterns in request auth/authorization:

- **Session cookie auth** (maintained in cookie jar).
- **CSRF token** in `__RequestVerificationToken` header for most POSTs.
- **Bootstrap page first** (often `GET /app/...` or clinical page) to retrieve CSRF and context.
- **Compose/session IDs** for communication-center send flows.
- **Tenant-specific path prefix** (`firstPathPart`) discovered dynamically.

---

## 4) Notes / caveats

- The scraper supports varying MyChart deployments; field names and endpoint behavior can differ between Epic versions and tenant customizations.
- Many response payloads are large; this catalog documents **stable fields that the scraper actually reads/returns**.
- `messages/exploreSendMessage*.ts` and related exploration scripts are investigative helpers; production tool behavior should rely on `sendMessage.ts`, `sendReply.ts`, drafts, threads, and conversations modules.
