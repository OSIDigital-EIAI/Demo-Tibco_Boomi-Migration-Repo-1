# Migration Plan — CRMServices.UpsertContact (1046)

**Project:** Demo-TIBCO-CMRService-Project  
**Source platform:** TIBCO  
**Generated:** 2026-10-29T05:02:49.446Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | CRMServices.UpsertContact (1046) |
| Identifier | crm-upsertcontact-rv |
| Trigger | Listener |
| Starts with | JMS |
| Complexity | medium (56) |
| Components to build | 17 |
| Source artefacts | legacy/tibco/CRMServices/CRMServices/Processes/Reply_UpsertContactRV.process, legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertContact.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_ValidateContact.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_sub_UpsertContact.process, legacy/tibco/CRMServices/XSDSchemas/CRMServices/Request/0426_UpsertContact.xsd, legacy/tibco/CRMServices/XSDSchemas/CRMServices/Response/0426_UpsertContact.xsd, legacy/tibco/CRMServices/XSDSchemas/CDCAdapter/Request/0425_ValidateContact.xsd, legacy/tibco/CRMServices/XSDSchemas/CDCAdapter/Response/0425_ValidateContact.xsd, legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Request/0430_UpsertContact.xsd, legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Response/0430_UpsertContact.xsd |
| Depends on | crm-shared-framework |

## 2. End-to-End Process Architecture

Five BW processes. Reply_UpsertContactRV is the RV starter (trace, reply, error reply). Main_UpsertContact parses and validates, rebuilds the header, builds error replies and sends the error mail. Sub_UpsertContact_UpsertContact maps to CDC, decides on the validation counts, maps to UCM, decides on UCM status and builds the response or the business error response. _Sub_ValidateContact makes the CDC request/reply call and applies overrides and totals. _sub_UpsertContact makes the UCM request/reply call. Both back-end calls go through Common Sub_Requester over Rendezvous.

One Boomi listener process, 'CRM Contact UPSERT Listener' (uc-main-process), in folder CRMServices/UpsertContact, running in low-latency mode with simultaneous executions allowed. Listen, then Set Properties (DPP_MessageId, reply destination, correlation ID), then Document Cache add of the request, then Try. Inside the Try: map to CDC, the CDC connector, the post-process map, a Decision, then (if clean) map to UCM, the UCM connector, a Decision, and the success or business-error map. The reply goes through uc-reply-op. Catch: a Decision on error type, Process Call to the error mail (crm-shared-framework) except for parse errors, the technical-error map, then reply. Shared EMS connection, CMH function and error handler come from crm-shared-framework.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| Reply_UpsertContactRV RV Subscriber + RV Reply | uc-listen-op + uc-reply-op (JMS, assumed) | Transport decision required. |
| Main_UpsertContact Parse Request + header rebuild | Profile parse in Try; Set Properties | The intermediate header rebuild is not needed as a separate step. |
| Call CDCAdapter input mapping | uc-req-to-cdc-map |  |
| _Sub_ValidateContact (Sub_Requester + Parse XML) | uc-cdc-validate-op |  |
| _Sub_ValidateContact Map Data | uc-cdc-postprocess-map + uc-validation-totals-fn | Request from Document Cache. |
| 'No Error or Warning' transition | Decision shape |  |
| Map Data (CDC to UCM) | uc-cdc-to-ucm-map |  |
| _sub_UpsertContact (Sub_Requester + Parse XML) | uc-ucm-upsert-op |  |
| Map Error Response | uc-error-response-map | Two source profiles: two map variants or a Route on origin. |
| Sub_UpsertContact_UpsertContact return bindings | uc-ucm-to-response-map |  |
| Main_UpsertContact Throw Exception + Send Error Mail | Catch path: shared-sub-send-error-mail + technical-error map |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive request | RV Subscriber on %%Env%%.FIXED.CRM.%%CRMServices/ComponentId%%.Request.UpsertContact.1 (RVCMQ); body in RV field 'xml'; reply subject captured. | legacy/tibco/CRMServices/CRMServices/Processes/Reply_UpsertContactRV.process |
| 2 | Trace request (optional) | If eai:isTraceOn('EAI-04005') is true, log via Common Sub_Trace. | legacy/tibco/CRMServices/CRMServices/Processes/Reply_UpsertContactRV.process |
| 3 | Parse and validate request | Parse as UpsertContactRequest (0426) with validateOutput = true. | legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertContact.process |
| 4 | Rebuild header for implementation | Main_UpsertContact rebuilds the CMH (ID '1046', NAME 'UpsertContact', MID '1046.20', TO copied from request, new ITEM EVENT 'UpsertContact'), copies BODY and passes ReplyDestination. | legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertContact.process |
| 5 | Map to CDC ValidateContact | Build ValidateContactRequest: CMH ID '1046', NAME 'ValidateContact'; ADDRESS/@language 'nl'; STREET_NAME/@namingConvention 'PROPRIETARY'; OVERRIDE_VALIDATION default false. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process |
| 6 | Call CDC ValidateContact | Common Sub_Requester to the CMH TO destination, timeout CRMServices/RV/Timeout (5000 ms); reply parsed and validated as ValidateContactResponse. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_ValidateContact.process |
| 7 | Apply overrides and totals | Copy CMH and RESULT_STATUS. If BODY exists: apply contact-level and per-address OVERRIDE_VALIDATION to warning counts; compute VALIDATION_TOTAL_WARNING_COUNT and VALIDATION_TOTAL_ERROR_COUNT. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_ValidateContact.process |
| 8 | Validation passed? | Proceed only if TOTAL_WARNING_COUNT exists and = 0, TOTAL_ERROR_COUNT exists and = 0, and RESULT_STATUS/@STATUS = 0; otherwise build the error response from CDC. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process |
| 9 | Map to UCM UpsertContact | Build UCM UpsertContactRequest from the validated CDC CONTACT_PERSON; DESTINATION %%Env%%.FIXED.CRM.UCMAdapter.Request.UpsertContact.1. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process |
| 10 | Call UCM UpsertContact | Common Sub_Requester, timeout 5000 ms; reply parsed as UCM UpsertContactResponse. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_sub_UpsertContact.process |
| 11 | UCM status = 0? | If RESULT_STATUS/@STATUS != 0, build the error response from UCM; otherwise build the success response. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process |
| 12 | Build response | UpsertContactResponse: CMH (ID 1048), RESULT_STATUS (STATUS '0' 'No Error', or the error-response status) and BODY (error-response BODY, or UCM CONTACT_PERSON). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process |
| 13 | Reply | Optional trace (EAI-04006), then RV reply with the rendered response. | legacy/tibco/CRMServices/CRMServices/Processes/Reply_UpsertContactRV.process |
| 14 | Technical error path | Implementation failure: mail 'Error in UpsertContact', then ErrorReply. Parse failure: ErrorReply without mail. The starter logs the error and replies with ErrorReply/Body when present. | legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertContact.process |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| CRM Contact UPSERT Req XML Prof | xml | UpsertContactRequest/CMH | complex (Base.xsd:Base) | yes |
| CRM Contact UPSERT Req XML Prof | xml | BODY/CONTACT_PERSON | ContactPerson_Type (CDM_Structures.xsd) | yes |
| CRM Contact UPSERT Req XML Prof | xml | CONTACT_PERSON/OVERRIDE_VALIDATION | xsd:boolean | no |
| CRM Contact UPSERT Req XML Prof | xml | CONTACT_PERSON/{VALIDATION_WARNING_COUNT, VALIDATION_ERROR_COUNT, VALIDATION_TOTAL_WARNING_COUNT, VALIDATION_TOTAL_ERROR_COUNT} | xsd:nonNegativeInteger | no |
| CRM Contact UPSERT Req XML Prof | xml | CONTACT_PERSON/{CONTACT_PERSON_ID, INITIALS, FAMILY_NAME_PREFIX, FAMILY_NAME, FAMILY_NAME_POSTFIX, PREFERRED_NAME, ACADEMIC_TITLE} | xsd:string | no |
| CRM Contact UPSERT Req XML Prof | xml | CONTACT_PERSON/DATE_OF_BIRTH | xsd:date | no |
| CRM Contact UPSERT Req XML Prof | xml | CONTACT_PERSON/CONTACT_MEDIUM/{TELEPHONE, FAX, EMAIL_ADDRESS, USAGE/SUPPRESS_ALL_EMAILS (boolean)} | xsd:string / boolean | no |
| CRM Contact UPSERT Req XML Prof | xml | CONTACT_PERSON/VALIDATION_RESULT/{SEVERITY, CODE, DESCRIPTION} | xsd:string; 0..unbounded | no |
| CRM Contact UPSERT Req XML Prof | xml | CONTACT_PERSON/ADDRESS (PropertyAddress_Type, 0..unbounded, @language) | complex | no |
| CRM Contact UPSERT Req XML Prof | xml | CONTACT_PERSON/SIEBEL_ACCOUNT_REFERENCE/{SIEBEL_ACCOUNT_ID, UCM_ROW_ID, INTEGRATION_ID} | xsd:string; 0..unbounded | no |
| CRM Contact UPSERT Req XML Prof | xml | CONTACT_PERSON/RUNTIME_PARAMETERS/{IS_PRIMARY_MULTI_VALUE_GROUP, INTEGRATION_ID, UCM_ROW_ID, ID} | boolean / xsd:string | no |
| CRM Contact UPSERT Req XML Prof | xml | CONTACT_PERSON/CROSS_REFERENCE/{CONTACT_ID, CIF_SYSTEM_ID, EXTERNAL_ID1, SYSTEM_NAME, SYSTEM_NUMBER} | xsd:string; 0..unbounded | no |
| CRM Contact UPSERT Resp XML Prof | xml | UpsertContactResponse/CMH | complex (Base.xsd:Base) | yes |
| CRM Contact UPSERT Resp XML Prof | xml | UpsertContactResponse/RESULT_STATUS | ResultStatus.xsd:ResultStatus | yes |
| CRM Contact UPSERT Resp XML Prof | xml | UpsertContactResponse/BODY/CONTACT_PERSON | ContactPerson_Type | no |
| CDC Contact VALIDATE Req XML Prof | xml | ValidateContactRequest/CMH | complex (Base.xsd:Base) | yes |
| CDC Contact VALIDATE Req XML Prof | xml | BODY/CONTACT_PERSON | ContactPerson_Type | yes |
| CDC Contact VALIDATE Resp XML Prof | xml | ValidateContactResponse/CMH | complex (Base.xsd:Base) | yes |
| CDC Contact VALIDATE Resp XML Prof | xml | ValidateContactResponse/RESULT_STATUS | ResultStatus.xsd:ResultStatus | yes |
| CDC Contact VALIDATE Resp XML Prof | xml | BODY/CONTACT_PERSON | ContactPerson_Type | no |
| UCM Contact UPSERT Req XML Prof | xml | UpsertContactRequest/CMH | complex (Base.xsd:Base) | yes |
| UCM Contact UPSERT Req XML Prof | xml | BODY/CONTACT_PERSON | ContactPerson_Type | yes |
| UCM Contact UPSERT Resp XML Prof | xml | UpsertContactResponse/CMH | complex (Base.xsd:Base) | yes |
| UCM Contact UPSERT Resp XML Prof | xml | UpsertContactResponse/RESULT_STATUS | ResultStatus.xsd:ResultStatus | yes |
| UCM Contact UPSERT Resp XML Prof | xml | BODY/CONTACT_PERSON | ContactPerson_Type | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | CMH/ID, NAME, MID, PARADIGM, OBJECT, VERSION | constants | hardcoded |  | '1046', 'ValidateContact', '1046.20', 'REQUEST', 'CONTACT_PERSON', '1.0' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | UpsertContactRequest to CDC ValidateContactRequest |
| MR-002 | CMH/TO/@DESTINATION | Env | calculated |  | concat(Env,'.FIXED.CRM.CDCAdapter.Request.ValidateContact.1') | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | UpsertContactRequest to CDC ValidateContactRequest |
| MR-003 | CMH/TRACKING | UpsertContactRequest/CMH/TRACKING | transformed |  | MESSAGE_ID copied; ITEMs copied + new ITEM (APPINFO concat(ComponentDomain,'/',ComponentId,'/',ComponentVersion), EVENT 'ValidateContact', TIMESTAMP); SEQUENCE_NO for-each | low | APPINFO includes the version in this flow. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | UpsertContactRequest to CDC ValidateContactRequest |
| MR-004 | BODY/CONTACT_PERSON (scalar fields) | UpsertContactRequest/BODY/CONTACT_PERSON | passthrough |  | copy | low | Same ContactPerson_Type in both namespaces. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | UpsertContactRequest to CDC ValidateContactRequest |
| MR-005 | BODY/CONTACT_PERSON/ADDRESS/@language | constant | hardcoded | for each ADDRESS | 'nl' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | UpsertContactRequest to CDC ValidateContactRequest |
| MR-006 | BODY/CONTACT_PERSON/ADDRESS/OVERRIDE_VALIDATION | ADDRESS/OVERRIDE_VALIDATION | conditional | absent in request | if (exists(OVERRIDE_VALIDATION)) then OVERRIDE_VALIDATION else false() | low | Boomi: Default Value 'false'. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | UpsertContactRequest to CDC ValidateContactRequest |
| MR-007 | BODY/CONTACT_PERSON/ADDRESS/STREET_NAME/@namingConvention | constant | hardcoded |  | 'PROPRIETARY' | low | Overrides any convention sent by the caller. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | UpsertContactRequest to CDC ValidateContactRequest |
| MR-008 | BODY/CONTACT_PERSON/ADDRESS/UNFORMATTED_ADDRESS/ADDRESS_LINE (@sequenceNumber) | ADDRESS/UNFORMATTED_ADDRESS/ADDRESS_LINE | looped |  | for-each | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | UpsertContactRequest to CDC ValidateContactRequest |
| MR-009 | BODY/CONTACT_PERSON/SIEBEL_ACCOUNT_REFERENCE | CONTACT_PERSON/SIEBEL_ACCOUNT_REFERENCE | looped |  | for-each | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | UpsertContactRequest to CDC ValidateContactRequest |
| MR-010 | CMH, RESULT_STATUS | ValidateContactResponse/CMH, RESULT_STATUS | passthrough |  | copy-of | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_ValidateContact.process | CDC response post-processing (override and totals) |
| MR-011 | BODY/CONTACT_PERSON/VALIDATION_WARNING_COUNT | response VALIDATION_WARNING_COUNT; request CONTACT_PERSON/OVERRIDE_VALIDATION | conditional | request OVERRIDE_VALIDATION = 'true' | '0' else response value (if present) | medium |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_ValidateContact.process | CDC response post-processing (override and totals) |
| MR-012 | BODY/CONTACT_PERSON/VALIDATION_TOTAL_WARNING_COUNT | contact and ADDRESS VALIDATION_WARNING_COUNT | calculated | (contact WARNING_COUNT > 0) or (contact ERROR_COUNT > 0); else '0' | number(contact VALIDATION_WARNING_COUNT) + sum(ADDRESS/VALIDATION_WARNING_COUNT) | high | Uses the raw response counts, not the overridden ones. Groovy map function. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_ValidateContact.process | CDC response post-processing (override and totals) |
| MR-013 | BODY/CONTACT_PERSON/VALIDATION_TOTAL_ERROR_COUNT | contact and ADDRESS VALIDATION_ERROR_COUNT | calculated | same condition as MR-012 | number(contact VALIDATION_ERROR_COUNT) + sum(ADDRESS/VALIDATION_ERROR_COUNT) | high |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_ValidateContact.process | CDC response post-processing (override and totals) |
| MR-014 | BODY/CONTACT_PERSON/ADDRESS[n]/VALIDATION_WARNING_COUNT | response ADDRESS[n]; request ADDRESS[n]/OVERRIDE_VALIDATION | conditional | request ADDRESS at the same position has OVERRIDE_VALIDATION = 'true' | '0' else response value | medium | Positional match ($IDX = position()). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_ValidateContact.process | CDC response post-processing (override and totals) |
| MR-015 | BODY/CONTACT_PERSON/{CONTACT_MEDIUM, VALIDATION_RESULT, ADDRESS (other fields, ADDRESS_LINE, RUNTIME_PARAMETERS, VALIDITY, VALIDATION_RESULT), SIEBEL_ACCOUNT_REFERENCE} | same paths in response | passthrough |  | copy-of / for-each | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_ValidateContact.process | CDC response post-processing (override and totals) |
| MR-016 | CMH/ID, NAME, MID, PARADIGM, OBJECT, VERSION | constants | hardcoded |  | '1046', 'ValidateContact', '1046.20', 'REQUEST', 'CONTACT_PERSON', '1.0' | low | NAME is 'ValidateContact' even though this is the UCM UpsertContact request; reproduce unless the client agrees to change it. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | CDC validated contact to UCM UpsertContactRequest |
| MR-017 | CMH/TO/@DESTINATION | Env | calculated |  | concat(Env,'.FIXED.CRM.UCMAdapter.Request.UpsertContact.1') | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | CDC validated contact to UCM UpsertContactRequest |
| MR-018 | CMH/TRACKING | ValidateContactResponse/CMH/TRACKING | transformed |  | MESSAGE_ID, ITEMs, SEQUENCE_NO copied + new ITEM (APPINFO concat(ComponentDomain,'/',ComponentId) without version, EVENT 'ValidateContact') | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | CDC validated contact to UCM UpsertContactRequest |
| MR-019 | BODY/CONTACT_PERSON/{VALIDATION_WARNING_COUNT, VALIDATION_ERROR_COUNT, VALIDATION_TOTAL_WARNING_COUNT, VALIDATION_TOTAL_ERROR_COUNT, CONTACT_PERSON_ID, INITIALS, FAMILY_NAME_PREFIX, FAMILY_NAME, FAMILY_NAME_POSTFIX, PREFERRED_NAME, DATE_OF_BIRTH, ACADEMIC_TITLE} | same-named CDC response elements | conditional | copied if present |  | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | CDC validated contact to UCM UpsertContactRequest |
| MR-020 | BODY/CONTACT_PERSON/CONTACT_MEDIUM/{TELEPHONE, FAX, EMAIL_ADDRESS, USAGE/SUPPRESS_ALL_EMAILS} | CDC CONTACT_MEDIUM | conditional | copied if present |  | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | CDC validated contact to UCM UpsertContactRequest |
| MR-021 | BODY/CONTACT_PERSON/VALIDATION_RESULT/{SEVERITY, CODE, DESCRIPTION} | CDC VALIDATION_RESULT | looped |  | for-each | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | CDC validated contact to UCM UpsertContactRequest |
| MR-022 | BODY/CONTACT_PERSON/ADDRESS (@language, VALIDATION_WARNING_COUNT, VALIDATION_ERROR_COUNT, STREET_NAME/@namingConvention, HOUSE_NUMBER, HOUSE_NUMBER_EXTENSION, POSTCODE, CITY_NAME, COUNTRY, COUNTRY_CODE, ADDRESS_TYPE, LOCATION_DESCRIPTION, UNFORMATTED_ADDRESS/ADDRESS_LINE, RUNTIME_PARAMETERS/{SPECIAL_DESCRIPTION, POSTCODE_SUFFIX, IS_FUTURE_PRIMARY_ADDRESS, IS_PRIMARY_LOGICAL_ADDRESS, IS_PRIMARY_MULTI_VALUE_GROUP}, VALIDITY/{ADDRESS_STATUS, VALID_FROM_DATE, VALID_UNTIL_DATE}, VALIDATION_RESULT) | CDC ADDRESS | looped | fields copied if present; STREET_NAME and COUNTRY_CODE always written | for-each | medium |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | CDC validated contact to UCM UpsertContactRequest |
| MR-023 | BODY/CONTACT_PERSON/SIEBEL_ACCOUNT_REFERENCE/{SIEBEL_ACCOUNT_ID, UCM_ROW_ID, INTEGRATION_ID} | CDC SIEBEL_ACCOUNT_REFERENCE | looped | fields copied if present | for-each | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | CDC validated contact to UCM UpsertContactRequest |
| MR-024 | BODY/CONTACT_PERSON/RUNTIME_PARAMETERS/{INTEGRATION_ID, ID} | CDC RUNTIME_PARAMETERS | conditional | copied if present |  | low | UCM_ROW_ID and IS_PRIMARY_MULTI_VALUE_GROUP are not observed in this mapping; confirm during build. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | CDC validated contact to UCM UpsertContactRequest |
| MR-025 | CMH/ID, NAME, MID, PARADIGM, OBJECT, VERSION | constants | hardcoded |  | 1048, 'UpsertContact', '1046.20', 'REQUEST', 'CONTACT_PERSON', '1.0' | low | Reply ID 1048 and PARADIGM 'REQUEST' as in source. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | Response to caller |
| MR-026 | CMH/FID | UCM UpsertContactResponse/CMH/FID | conditional | present |  | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | Response to caller |
| MR-027 | CMH/FROM/@COMPONENT_ID, @COMPONENT_VERSION | ComponentId, ComponentVersion | hardcoded |  |  | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | Response to caller |
| MR-028 | CMH/TO/@DESTINATION | ReplyDestination (RV reply subject) | transformed |  | copy | low | JMSReplyTo in the target design. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | Response to caller |
| MR-029 | CMH/TRACKING | original request CMH/TRACKING | transformed | EXTERNAL_REF if present | MESSAGE_ID, EXTERNAL_REF, ITEMs, SEQUENCE_NO copied + new ITEM (APPINFO with version, EVENT 'UpsertContact') | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | Response to caller |
| MR-030 | RESULT_STATUS | error-response RESULT_STATUS or constants | conditional | error response exists -> copy its RESULT_STATUS; else success | success: STATUS '0', ERROR_DESCRIPTION 'No Error' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | Response to caller |
| MR-031 | BODY | error-response BODY, else UCM response BODY/CONTACT_PERSON | conditional | error-response BODY exists -> copy; else if UCM BODY exists -> field-by-field copy of CONTACT_PERSON (incl. RUNTIME_PARAMETERS INTEGRATION_ID/UCM_ROW_ID/ID and CROSS_REFERENCE CONTACT_ID/CIF_SYSTEM_ID/EXTERNAL_ID1/SYSTEM_NAME/SYSTEM_NUMBER) |  | medium | No BODY when neither exists. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | Response to caller |
| MR-032 | CMH | constants and request TRACKING | hardcoded |  | ID 1048, NAME 'UpsertContact', MID '1046.20', PARADIGM 'REQUEST', OBJECT 'CONTACT_PERSON', TO = ReplyDestination, EVENT 'UpsertContact' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | Business error response (Map Error Response) |
| MR-033 | RESULT_STATUS | CDC or UCM RESULT_STATUS | conditional | CDC BODY exists and CDC STATUS != 0 -> CDC; else UCM BODY exists and UCM STATUS != 0 -> UCM; else STATUS 1 'General Un-Attribute Error' | copy STATUS; MSG_CODE, ERROR_CODE, ERROR_DESCRIPTION if present | medium |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | Business error response (Map Error Response) |
| MR-034 | BODY | CDC response BODY or UCM response BODY | conditional | CDC BODY exists and CDC STATUS != 0 -> CDC BODY; else UCM BODY exists and UCM STATUS != 0 -> UCM BODY; no otherwise branch | field-by-field copy | medium | When CDC status is 0 but warnings or errors stop the contact, no BODY is produced, so the caller does not see which validation failed. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | Business error response (Map Error Response) |
| MR-035 | CMH | constants + parsed request TRACKING | hardcoded |  | ID 1048, NAME 'UpsertContact', MID '1046.20', PARADIGM 'REQUEST', OBJECT 'CONTACT_PERSON', VERSION '1.0', EVENT 'UpsertContact' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertContact.process | Technical error reply |
| MR-036 | RESULT_STATUS | constants | conditional | parse error -> STATUS 1 'Technical Error'; otherwise STATUS 1 'General Un-Attribute Error' |  | low |  | legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertContact.process | Technical error reply |
| MR-037 | Errors/Error/MsgCode | constants / fault | conditional | parse error -> CRMS-03011 (Params = raw request); implementation error -> copy fault Errors, else CRMS-03000 |  | low | Errors go to the error log only, not to the caller. | legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertContact.process | Technical error reply |
| MR-038 | Error mail Subject / Body | error, parsed request | calculated | implementation error | Subject 'Error in UpsertContact'; Body concat('Location :upsertContact', first Errors/MsgCode else Result_Status/@ERROR_DESCRIPTION, 'Message Causing Error is : ', render-xml(request)) | low | Sent via crm-shared-framework. | legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertContact.process | Technical error reply |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Contact must pass CDC validation before UCM upsert | Proceed to UCM only if VALIDATION_TOTAL_WARNING_COUNT exists and = 0, VALIDATION_TOTAL_ERROR_COUNT exists and = 0, and RESULT_STATUS/@STATUS = 0. Otherwise return the CDC result to the caller. | High | Decision shape on the post-processed CDC document. |
| BR-002 | Validation override | CONTACT_PERSON/OVERRIDE_VALIDATION = 'true' forces the contact VALIDATION_WARNING_COUNT to 0. ADDRESS[n]/OVERRIDE_VALIDATION = 'true' forces the n-th returned address's warning count to 0. Errors are never overridden. | Medium | uc-validation-totals-fn with the request from the Document Cache. |
| BR-003 | Validation totals | TOTAL_WARNING = contact WARNING + sum(address WARNING); TOTAL_ERROR = contact ERROR + sum(address ERROR). Computed only when the contact-level warning or error count > 0; otherwise both totals are 0. Totals use the raw (not overridden) response counts. | High | Groovy map function; replicate exactly unless the business decides otherwise (see questions). |
| BR-004 | Default address attributes | Every address sent to CDC is tagged language 'nl' and street-name convention 'PROPRIETARY'; a missing address override defaults to false. | Low | Default values in uc-req-to-cdc-map. |
| BR-005 | UCM rejection returned to caller | If UCM RESULT_STATUS/@STATUS != 0, reply with UCM's status and BODY; not treated as a technical error (no mail). | High | Decision on UCM STATUS. |
| BR-006 | Error mail only on technical failure after parse | Mail 'Error in UpsertContact' is sent when the implementation sub-process faults; not for parse errors, nor for CDC or UCM business rejections. | Medium | Process Call to Sub: CRM Send Error Mail in the Catch path. |
| BR-007 | Request must be schema-valid | Parse with validation; failure gives STATUS 1 'Technical Error' (CRMS-03011). | Medium | Profile parse in Try; optional Groovy XSD validation. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound TIBCO Rendezvous request/reply (RVCMQ) | %%Env%%.FIXED.CRM.%%CRMServices/ComponentId%%.Request.UpsertContact.1 (default XDVL.FIXED.CRM.CRMServices.Request.UpsertContact.1) | none visible (network and subject segregation) | legacy/tibco/CRMServices/CRMServices/Processes/Reply_UpsertContactRV.process |
| outbound TIBCO Rendezvous request/reply via Common Sub_Requester | %%Env%%.FIXED.CRM.CDCAdapter.Request.ValidateContact.1 | none visible | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process |
| outbound TIBCO Rendezvous request/reply via Common Sub_Requester | %%Env%%.FIXED.CRM.UCMAdapter.Request.UpsertContact.1 | none visible | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process |
| RV Subscriber + RV Reply | Identical to GetAccountData's starter except the subject and service name: sharedChannel /Common/Shared Resources/RVCMQ Transport.rvtransport (not in export), body field 'xml' (tibrvXml). | — | legacy/tibco/CRMServices/CRMServices/Processes/Reply_UpsertContactRV.process |
| Common Sub_Requester (CDC) | Destination = ValidateContactRequest/CMH/TO/@DESTINATION; Timeout = CRMServices/RV/Timeout; Body = request. | — | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_ValidateContact.process |
| Common Sub_Requester (UCM) | Destination = UpsertContactRequest/CMH/TO/@DESTINATION; Timeout = CRMServices/RV/Timeout; Body = request. | — | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_sub_UpsertContact.process |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| crm-shared-framework | Sub: CRM Send Error Mail | synchronous |
| crm-shared-framework | Sub: CRM Error Handler (replaces Sub_ErrorTrace) | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/tibco/CRMServices/CRMServices/Processes/Reply_UpsertContactRV.process | eai:getMessageId and eai:isTraceOn (com.kpn.eai.component.common.CommonXPathFunctions) for trace gating and error-log MsgId. | shared-cmh-header-fn and a trace process property (crm-shared-framework). | Medium |
| xslt | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_ValidateContact.process | Totals: number(contact count) + sum(ADDRESS/count); positional override lookup via xsl:variable $IDX = position() and $SFA_ADDRESS = request ADDRESS[$IDX]. | uc-validation-totals-fn (Groovy map function) with the request read from a Document Cache keyed by MESSAGE_ID. | Medium |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Business rejections (CDC or UCM) are normal replies with a non-zero RESULT_STATUS. Technical failures in the sub-processes raise an Errors fault, copying the requester error or CRMS-03000. Main_UpsertContact mails, then throws an ErrorReply with a ready-made response body, and the starter replies with it. In Boomi: decisions for business outcomes; Try/Catch for technical errors; error mail via crm-shared-framework; error reply in the Catch path. |
| Retries | None; the synchronous caller decides. |
| Compensation | None. If UCM is updated and the reply then fails, the update stands. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| /Common/Shared Resources/RVCMQ Transport.rvtransport | other | no |
| Common Sub_Requester / Sub_Trace / Sub_ErrorTrace | library | no |
| XSDSchemas/Common/Base.xsd, ResultStatus.xsd | other | no |
| CDM_Structures.xsd (ContactPerson_Type; identical copy in four folders) | other | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| CDCAdapter ValidateContact | TIBCO Rendezvous request/reply (target TBD) | %%Env%%.FIXED.CRM.CDCAdapter.Request.ValidateContact.1 | Not stated in export |
| UCMAdapter UpsertContact | TIBCO Rendezvous request/reply (target TBD) | %%Env%%.FIXED.CRM.UCMAdapter.Request.UpsertContact.1 | Not stated in export |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| DEV | Env | XDVL (design-time default; environment it denotes not stated) | [redacted].substvar |

## 14. Boomi Migration Readiness Summary

UpsertContact is the service front-end applications use to create or update a contact person in UCM. Each request is first checked by a validation service (CDCAdapter), which reports warnings and errors for the person and for each address. Only a clean result lets the contact through to UCM. The caller can mark the contact, or individual addresses, with an override flag to accept warnings. Errors cannot be overridden.

The caller always receives a reply. On success, it contains the contact as stored by UCM. If validation stops the contact, the reply carries the validation status and details. If UCM rejects it, the reply carries UCM's status. Only technical failures (an unreadable request, a timeout or an unexpected error) produce a 'Technical Error' reply and, after a successful parse, an e-mail to integration support.

The business logic is moderate: two calls in sequence, some decisions and a counting rule. The mappings are large but mostly straight copies. As with the rest of CRMServices, the main obstacles are the TIBCO Rendezvous transport, which Boomi cannot use, and the missing header schemas. One counting rule also behaves in a way that may be a defect, and it needs a business decision before it is copied.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | CDC Contact VALIDATE Req XML Prof | uc-cdc-validate-req-prof | profile.xml | — |
| 2 | CDC Contact VALIDATE Resp XML Prof | uc-cdc-validate-resp-prof | profile.xml | — |
| 3 | CRM Contact UPSERT Req XML Prof | uc-request-prof | profile.xml | — |
| 4 | CRM Contact UPSERT Resp XML Prof | uc-response-prof | profile.xml | — |
| 5 | UCM Contact UPSERT Req XML Prof | uc-ucm-upsert-req-prof | profile.xml | — |
| 6 | UCM Contact UPSERT Resp XML Prof | uc-ucm-upsert-resp-prof | profile.xml | — |
| 7 | CDC Contact VALIDATE Op | uc-cdc-validate-op | connector-action | shared-ems-jms-conn, uc-cdc-validate-req-prof, uc-cdc-validate-resp-prof |
| 8 | EMS CRM Contact UPSERT LISTEN Op | uc-listen-op | connector-action | shared-ems-jms-conn, uc-request-prof |
| 9 | EMS CRM Contact UPSERT REPLY Op | uc-reply-op | connector-action | shared-ems-jms-conn, uc-response-prof |
| 10 | UCM Contact UPSERT Op | uc-ucm-upsert-op | connector-action | shared-ems-jms-conn, uc-ucm-upsert-req-prof, uc-ucm-upsert-resp-prof |
| 11 | CDC Validation Totals and Override | uc-validation-totals-fn | transform.function | — |
| 12 | CDC Contact VALIDATE Resp XML - CDC Contact VALIDATE Resp XML (Override) | uc-cdc-postprocess-map | transform.map | uc-cdc-validate-resp-prof, uc-cdc-validate-req-prof, uc-validation-totals-fn |
| 13 | CDC Contact VALIDATE Resp XML - UCM Contact UPSERT Req XML | uc-cdc-to-ucm-map | transform.map | uc-cdc-validate-resp-prof, uc-ucm-upsert-req-prof, shared-cmh-header-fn |
| 14 | CDC UCM Error - CRM Contact UPSERT Resp XML | uc-error-response-map | transform.map | uc-cdc-validate-resp-prof, uc-ucm-upsert-resp-prof, uc-response-prof |
| 15 | CRM Contact UPSERT Req XML - CDC Contact VALIDATE Req XML | uc-req-to-cdc-map | transform.map | uc-request-prof, uc-cdc-validate-req-prof, shared-cmh-header-fn |
| 16 | UCM Contact UPSERT Resp XML - CRM Contact UPSERT Resp XML | uc-ucm-to-response-map | transform.map | uc-ucm-upsert-resp-prof, uc-response-prof, shared-cmh-header-fn |
| 17 | CRM Contact UPSERT Listener | uc-main-process | process | uc-listen-op, uc-reply-op, uc-req-to-cdc-map, uc-cdc-validate-op, uc-cdc-postprocess-map, uc-cdc-to-ucm-map, uc-ucm-upsert-op, uc-ucm-to-response-map, uc-error-response-map, shared-sub-send-error-mail, shared-sub-error-handler |

## 15. Appendix

- legacy/tibco/CRMServices/CRMServices/Processes/Reply_UpsertContactRV.process
- legacy/tibco/CRMServices/CRMServices/Processes/Reply_GetAccountdataRV.process
- legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertContact.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_ValidateContact.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_sub_UpsertContact.process
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Request/0426_UpsertContact.xsd
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Response/0426_UpsertContact.xsd
- legacy/tibco/CRMServices/XSDSchemas/CDCAdapter/Request/0425_ValidateContact.xsd
- legacy/tibco/CRMServices/XSDSchemas/CDCAdapter/Response/0425_ValidateContact.xsd
- legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Request/0430_UpsertContact.xsd
- legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Response/0430_UpsertContact.xsd
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Imported/0426_UpsertContact_1.0_Request_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Imported/0426_UpsertContact_1.0_Response_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/CDCAdapter/Imported/0425_ValidateContact_1.0_Request_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/CDCAdapter/Imported/0425_ValidateContact_1.0_Response_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Imported/0430_UpsertContact_1.0_Request_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Imported/0430_UpsertContact_1.0_Response_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Imported/CDM_Structures.xsd
- legacy/tibco/CRMServices/XSDSchemas/CDCAdapter/Imported/CDM_Structures.xsd
- legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Imported/CDM_Structures.xsd
- legacy/tibco/CRMServices/defaultVars/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/CRMServices/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/CRMServices/RV/defaultVars.substvar
- legacy/tibco/CRMServices/.designtimelibs

## Open questions

### Still outstanding

REVIEW-NEEDED: Which applications call UpsertContact today, and can they move from TIBCO Rendezvous to EMS request/reply or to an HTTP API? (Identify the callers of UpsertContact and confirm they can move from RV request/reply to EMS request/reply (or HTTP). This decides whether the listener and reply are JMS or a Web Services Server listener.)
  - If this is not answered: Callers move to an EMS request queue named like today's subject (<Env>.FIXED.CRM.CRMServices.Request.UpsertContact.1), with the same decision as GetAccountData.

REVIEW-NEEDED: Should warnings or errors reported only on a contact's addresses (with none on the person) stop the contact from being saved in UCM? (As written, the totals are computed only when the contact-level warning or error count is above 0, so warnings or errors reported only on addresses produce totals of 0 and the contact is upserted. Confirm whether this is intended behaviour to replicate or a defect to fix.)
  - If this is not answered: Today's behaviour is replicated exactly: address-only findings do not stop the upsert.

REVIEW-NEEDED: When a contact is marked 'override validation', should the override also stop contact-level warnings from counting towards the total that blocks the save? (Also, the contact-level override is applied to VALIDATION_WARNING_COUNT, but the totals test reads the raw response counts, so an overridden contact warning can still block the save.)
  - If this is not answered: Today's behaviour is replicated: totals use the raw counts.

REVIEW-NEEDED: When validation stops a contact but the validation service reports success status, should the caller receive the validation details instead of the generic 'General Un-Attribute Error' with no details that it gets today? (In this case the reply has no BODY, so the caller cannot see which warning or error stopped the contact. This looks like a defect, and the fix changes the reply content.)
  - If this is not answered: Today's reply is replicated exactly: STATUS 1, 'General Un-Attribute Error', no BODY.

REVIEW-NEEDED: Does the validation service always return a contact's addresses in the same order as they were sent? (The override uses the request ADDRESS at the same position as the response ADDRESS. If the order can change, the override applies to the wrong address.)
  - If this is not answered: Order is preserved, and addresses are matched by position as today.

REVIEW-NEEDED: Should every address still be sent for validation as Dutch-language with the 'proprietary' street-naming convention, whatever the caller supplied? (Language 'nl' and naming convention 'PROPRIETARY' are hard-coded, which overrides any convention the caller sends. Foreign addresses may be validated incorrectly.)
  - If this is not answered: The hard-coded values are kept.

REVIEW-NEEDED: Which system sits behind the CDC validation service, and who owns it? (The system behind CDCAdapter and its owner are not named in the export. The transport choice and testing need a contact.)
  - If this is not answered: CDCAdapter is reached the same way as UCMAdapter (the shared transport decision), with no change to its message format.

REVIEW-NEEDED: Do any callers depend on the reply header fields as they are today (reply ID 1048 with type 'REQUEST'), or may they be corrected to a response type? (The reply header identifies itself as service 1048 with PARADIGM 'REQUEST', and the UCM request is labelled 'ValidateContact'. Correcting these could affect callers' parsing or monitoring.)
  - If this is not answered: Header values are reproduced exactly as today.

REVIEW-NEEDED: How long do callers wait for an UpsertContact reply, and how many requests per day and per peak hour are expected? (The service makes two back-end calls in sequence, each allowed 5 seconds. Any extra hop from the new transport must fit within the callers' timeout, and volumes drive sizing.)
  - If this is not answered: Callers wait at least 15 seconds; low volume; low-latency mode on a single runtime.

