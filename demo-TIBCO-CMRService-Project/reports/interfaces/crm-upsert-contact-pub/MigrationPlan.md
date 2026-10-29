# Migration Plan — CRMServices.UpsertContactPub (0429/1062)

**Project:** Demo-TIBCO-CMRService-Project  
**Source platform:** TIBCO  
**Generated:** 2026-10-29T05:02:49.507Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | CRMServices.UpsertContactPub (0429/1062) |
| Identifier | crm-upsert-contact-pub |
| Trigger | Listener |
| Starts with | JMS |
| Complexity | medium (62) |
| Components to build | 11 |
| Source artefacts | legacy/tibco/CRMServices/CRMServices/Processes/Subscribe_UpsertContactPub.process, legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertContactPub.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process, legacy/tibco/CRMServices/CRMServices/Processes/_Subscribe_UpsertContactPub.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_CallAddAccountReferenceFromUpsertContactPub.process, legacy/tibco/CRMServices/XSDSchemas/CRMServices/Publish/0429_UpsertContactPub.xsd, legacy/tibco/CRMServices/XSDSchemas/SFAAdapter/Request/0431_UpsertContact.xsd, legacy/tibco/CRMServices/XSDSchemas/SFAAdapter/Response/0431_UpsertContact.xsd, legacy/tibco/CRMServices/defaultVars/CRMServices/UpsertContactPub/defaultVars.substvar |
| Depends on | crm-shared-framework |

## 2. End-to-End Process Architecture

Two JMS listeners and three processes. Subscribe_UpsertContactPub computes the redeliver flag and calls Main_UpsertContactPub (parse, tag RESEND_ALLOWED, throw on error). That calls Sub_UpsertContactPub, which runs a group: SFA call via Common Sub_Requester, parse, status and OPERATION decisions, then build and queue AddAccountReference via Common Sub_NotifierJMS. The group's error handler decides between retry (rethrow) and log plus mail. _Subscribe_UpsertContactPub computes its own redeliver flag and calls _CallAddAccountReferenceFromUpsertContactPub (parse, rebuild header, Sub_AddAccountReference, the same retry or log-and-mail decision). Both listeners send errors to Common Sub_ThrowException and then confirm.

Two Boomi listener processes in folder CRMServices/UpsertContactPub, keeping the two independent retry limits. 'CRM ContactPub UPSERT Listener' (ucp-main-process): listen, then Set Properties (DDP delivery count from the JMS redelivery header, DPP_MessageId), then Try, then map to SFA, the SFA connector, a Decision on STATUS, a Decision on OPERATION, the map to AddAccountReference, and the internal send. Catch: a Decision on error class and the delivery count. Retry ends with an Exception (message not acknowledged, so EMS redelivers). Otherwise the Catch calls the error handler and error mail from crm-shared-framework and ends normally. 'CRM ContactPub AccountReference ADD Worker' (ucp-worker-process): listen, rebuild the header, Process Call 'Sub: UCM AccountReference ADD' (crm-shared-framework), and the same Catch pattern with limit 5.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| Subscribe_UpsertContactPub (+ redeliver computation) | ucp-listen-op + Set Properties in ucp-main-process | The delivery count header must be available from the Boomi JMS connector. |
| Main_UpsertContactPub parse + RESEND_ALLOWED attribute | Profile parse; document property instead of mutating the CMH |  |
| Call 0431-UpsertContact (Sub_Requester) | ucp-pub-to-sfa-map + ucp-sfa-upsertcontact-op |  |
| Status / OPERATION transitions | Decision shapes |  |
| Build AddAccountReference Request + Sub_NotifierJMS | ucp-sfa-to-ucm-accountref-map + ucp-internal-send-op |  |
| CheckErrorCondition / Generate Error / LogError / Send Error Mail | Catch with Decision; Exception or shared error handler + mail |  |
| _Subscribe_UpsertContactPub + _CallAddAccountReferenceFromUpsertContactPub | ucp-internal-listen-op + ucp-worker-process |  |
| Sub_AddAccountReference | shared-sub-ucm-accountreference-add (crm-shared-framework) |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive publication | JMS Queue Receiver on %%Env%%.FIXED.%%CRMServices/ComponentDomain%%.%%CRMServices/ComponentId%%.Subscribe.UpsertContactPub.1; Text; CLIENT_ACKNOWLEDGE; max sessions JmsMaxSessions. | legacy/tibco/CRMServices/CRMServices/Processes/Subscribe_UpsertContactPub.process |
| 2 | Compute redeliver flag | redeliver = if not(exists(OtherProperties/RedeliveryCount)) then true() else RedeliveryCount < CRMServices/UpsertContactPub/UpsertContact_redeliverycount (4). | legacy/tibco/CRMServices/CRMServices/Processes/Subscribe_UpsertContactPub.process |
| 3 | Parse and tag publication | Parse as UpsertContactPub; copy the CMH (ID, NAME, FID, MID, PARADIGM, OBJECT, VERSION, FROM, TO, TRACKING, ATTRIBUTE) and add ATTRIBUTE RESEND_ALLOWED = redeliver; copy BODY. | legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertContactPub.process |
| 4 | Call SFA UpsertContact | Build UpsertContactRequest (0431; CMH ID 1062, NAME 'UpsertContact', MID 1062.20); Common Sub_Requester to %%Env%%.FIXED.CRM.SFAAdapter.Request.UpsertContact.1, timeout CRMServices/RV/Timeout (5000 ms). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process |
| 5 | Parse SFA response | Parse as SFA UpsertContactResponse; failure raises CRMS-03002. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process |
| 6 | SFA status check | RESULT_STATUS/@STATUS != 0 raises Error MsgCode 'EAI-03004' with ActivityError = the SFA response. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process |
| 7 | Was the contact created? | If STATUS = 0 and lower-case(CMH/ATTRIBUTE[@NAME='OPERATION']/@VALUE) != 'create', end without publishing (DoNotPublish). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process |
| 8 | Build AddAccountReference request | Mapper 'Build AddAccountReference Request' from the SFA response. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process |
| 9 | Queue UCM step | Common Sub_NotifierJMS to concat(Env,'.FIXED.Operations.',ComponentId,'._UpsertContactPub.1') with the rendered request; MessageId = SFA response MESSAGE_ID. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process |
| 10 | Main error decision | On error from the group: if (Call error MsgCode = 'EAI-03004' or ErrorReport/Msg = 'BWENGINE-100067' or SFA RESULT_STATUS/@ERROR_CODE = '10000') and RESEND_ALLOWED = 'true', raise Generate Error (rethrow). Otherwise LogError (Sub_ErrorTrace), then Send Error Mail, then end normally. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process |
| 11 | Listener handling of rethrow | Main_UpsertContactPub throws (MsgCode from the error, else CRMS-03003 parse, else CRMS-03000; text from eai:getMessageText). The listener calls Common Sub_ThrowException, then Confirm. | legacy/tibco/CRMServices/CRMServices/Processes/Subscribe_UpsertContactPub.process |
| 12 | Receive internal request (worker) | JMS Queue Receiver on %%Env%%.FIXED.%%CRMServices/ComponentDomain%%.%%CRMServices/ComponentId%%._UpsertContactPub.1; redeliver = RedeliveryCount < AddAccountRefernce_redelivercount (5). | legacy/tibco/CRMServices/CRMServices/Processes/_Subscribe_UpsertContactPub.process |
| 13 | Parse and call UCM | Parse as AddAccountReferenceRequest (failure: CRMS-03003); rebuild the CMH (OBJECT 'SIEBEL_ADCCOUNT_ID', new ITEM EVENT 'AddAccountReferenceRequest'); call Sub_AddAccountReference (crm-shared-framework). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_CallAddAccountReferenceFromUpsertContactPub.process |
| 14 | Worker error decision | Code0 = first Error/ApplicationError/@ERROR_CODE else '10001'. If (first MsgCode = 'EAI-03004' or Msg = 'BWENGINE-100067' or Code0 = '10000') and redeliver = 'true', throw; otherwise LogError, then mail 'Error in 0429-UpsertContactPub', then end normally. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_CallAddAccountReferenceFromUpsertContactPub.process |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| CRM ContactPub PUBLISH XML Prof | xml | UpsertContactPub/CMH | complex (Base.xsd:Base) | yes |
| CRM ContactPub PUBLISH XML Prof | xml | BODY/CONTACT_PERSON | ContactPerson_Type (CDM_Structures.xsd) | yes |
| SFA Contact UPSERT Req XML Prof | xml | UpsertContactRequest/CMH | complex (Base.xsd:Base) | yes |
| SFA Contact UPSERT Req XML Prof | xml | BODY/CONTACT_PERSON | ContactPerson_Type | yes |
| SFA Contact UPSERT Resp XML Prof | xml | UpsertContactResponse/CMH | complex (Base.xsd:Base) | yes |
| SFA Contact UPSERT Resp XML Prof | xml | UpsertContactResponse/RESULT_STATUS | ResultStatus.xsd:ResultStatus | yes |
| SFA Contact UPSERT Resp XML Prof | xml | UpsertContactResponse/GD | resultStatus:GD (element ref into ResultStatus.xsd) | yes |
| SFA Contact UPSERT Resp XML Prof | xml | BODY/CONTACT_PERSON | ContactPerson_Type | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | CMH/ID, NAME, MID, PARADIGM, OBJECT, VERSION | constants | hardcoded |  | '1062', 'UpsertContact', '1062.20', 'REQUEST', 'CONTACT_PERSON', '1.0' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | Publication to SFA UpsertContactRequest |
| MR-002 | CMH/FID | UpsertContactPub/CMH/FID | transformed |  | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | Publication to SFA UpsertContactRequest |
| MR-003 | CMH/FROM/@COMPONENT_ID, @COMPONENT_VERSION | ComponentId, ComponentVersion | hardcoded |  |  | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | Publication to SFA UpsertContactRequest |
| MR-004 | CMH/TO/@DESTINATION | Env | calculated |  | concat(Env,'.FIXED.CRM.SFAAdapter.Request.UpsertContact.1') | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | Publication to SFA UpsertContactRequest |
| MR-005 | CMH/TRACKING | UpsertContactPub/CMH/TRACKING | transformed |  | MESSAGE_ID, EXTERNAL_REF, ITEMs copied + new ITEM (APPINFO 'Operations/CRMServices', EVENT 'UpsertContactRequest', TIMESTAMP) | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | Publication to SFA UpsertContactRequest |
| MR-006 | BODY/CONTACT_PERSON (all ContactPerson_Type fields incl. OVERRIDE_VALIDATION, validation counts, names, DATE_OF_BIRTH, CONTACT_MEDIUM/USAGE, VALIDATION_RESULT, ADDRESS with UNFORMATTED_ADDRESS/ADDRESS_LINE, RUNTIME_PARAMETERS, VALIDITY, VALIDATION_RESULT, SIEBEL_ACCOUNT_REFERENCE, RUNTIME_PARAMETERS, CROSS_REFERENCE) | UpsertContactPub/BODY/CONTACT_PERSON | looped | leaf fields copied if present (about 100 presence checks) | nested for-each | medium | Same ContactPerson_Type on both sides: a near-identity map. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | Publication to SFA UpsertContactRequest |
| MR-007 | CMH/ID, NAME, MID, PARADIGM, OBJECT, VERSION | constants | hardcoded |  | '1062', 'AddAccountReference', '1062.20', 'REQUEST', 'SIEBEL_ACCOUNT_ID', '1.0' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | SFA response to AddAccountReferenceRequest |
| MR-008 | CMH/FID | SFA UpsertContactResponse/CMH/FID | transformed |  | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | SFA response to AddAccountReferenceRequest |
| MR-009 | CMH/TO/@DESTINATION | Env | calculated |  | concat(Env,'.FIXED.CRM.UCMAdapter.Request.AddAccountReference.1') | low | Used later by the worker as the UCM destination. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | SFA response to AddAccountReferenceRequest |
| MR-010 | CMH/TRACKING | SFA response CMH/TRACKING | transformed |  | MESSAGE_ID, EXTERNAL_REF, ITEMs + new ITEM (APPINFO 'Operations/CRMServices', EVENT 'INTERNAL_PUBLISH_FOR_AddAccountReference') | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | SFA response to AddAccountReferenceRequest |
| MR-011 | CMH/ATTRIBUTE (IntMessageType, IntObjectName, IntObjectFormat) | constants | hardcoded |  | 'Integration Object', 'UCMContactExternalID', 'Siebel Hierarchical' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | SFA response to AddAccountReferenceRequest |
| MR-012 | BODY/SIEBEL_ACCOUNT_ID | CONTACT_PERSON/SIEBEL_ACCOUNT_REFERENCE[1]/SIEBEL_ACCOUNT_ID | transformed | first reference only | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | SFA response to AddAccountReferenceRequest |
| MR-013 | BODY/EXTENAL_REFERENCE/REFERENCE | CONTACT_PERSON/RUNTIME_PARAMETERS/ID | transformed |  | copy | low | The SFA contact ID. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | SFA response to AddAccountReferenceRequest |
| MR-014 | BODY/EXTENAL_REFERENCE/ORIGIN, ORIGIN_ID | constants | hardcoded |  | 'SFA', 'SFA' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | SFA response to AddAccountReferenceRequest |
| MR-015 | CMH/ID, NAME, FID, MID, PARADIGM, VERSION, FROM, TO | queued request CMH | passthrough | FID and MID if present | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_CallAddAccountReferenceFromUpsertContactPub.process | Worker header rebuild |
| MR-016 | CMH/OBJECT | constant | hardcoded |  | 'SIEBEL_ADCCOUNT_ID' | low | Typo in source; the queued request carried 'SIEBEL_ACCOUNT_ID'. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_CallAddAccountReferenceFromUpsertContactPub.process | Worker header rebuild |
| MR-017 | CMH/TRACKING | queued request TRACKING | transformed | EXTERNAL_REF if present | MESSAGE_ID, EXTERNAL_REF, ITEMs + new ITEM (APPINFO = last ITEM's APPINFO, EVENT 'AddAccountReferenceRequest') | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_CallAddAccountReferenceFromUpsertContactPub.process | Worker header rebuild |
| MR-018 | CMH/ATTRIBUTE, BODY | queued request | passthrough |  | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_CallAddAccountReferenceFromUpsertContactPub.process | Worker header rebuild |
| MR-019 | Subject (main) | constant | hardcoded |  | 'Error in service 0429-UpsertContactPub' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | Error notifications |
| MR-020 | Body (main) | SFA RESULT_STATUS/@ERROR_DESCRIPTION, publication | calculated |  | concat(CRLF, ERROR_DESCRIPTION, CRLF, 'Message causing error is:', CRLF, render-xml(UpsertContactPub)) | low | ERROR_DESCRIPTION is empty for transport errors. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | Error notifications |
| MR-021 | Subject / Body (worker) | ErrorReport/Msg, queued request | calculated |  | Subject 'Error in 0429-UpsertContactPub'; Body concat(CRLF, ErrorReport/Msg, CRLF, ' Message cousing error is:', CRLF, render-xml(AddAccountReferenceRequest)) | low | Typo 'cousing' as in source. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_CallAddAccountReferenceFromUpsertContactPub.process | Error notifications |
| MR-022 | Errors/Error (LogError, main) | SFA RESULT_STATUS or requester error | conditional | SFA STATUS != 0 -> CRMS-03004 + Params ERROR_DESCRIPTION + RESULT_STATUS; requester MsgCode exists -> copy; else CRMS-03000 + ErrorReport |  | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | Error notifications |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | UCM reference only for creates | After SFA UpsertContact: if RESULT_STATUS/@STATUS != 0, error. If STATUS = 0 and lower-case(CMH/ATTRIBUTE[@NAME='OPERATION']/@VALUE) != 'create', stop. Otherwise queue AddAccountReference. | High | Decision on STATUS, then Decision on lower-cased OPERATION attribute. |
| BR-002 | Timeout-only retry, SFA step | Retry (rethrow) if (requester MsgCode = 'EAI-03004' or ErrorReport/Msg = 'BWENGINE-100067' or SFA ERROR_CODE = '10000') and RESEND_ALLOWED = 'true'; RESEND_ALLOWED = (no RedeliveryCount) or RedeliveryCount < 4. | High | Catch, then Decision on error class and the delivery-count document property, then Exception (no ack) or the error handler + mail (ack). |
| BR-003 | Timeout-only retry, UCM step | Retry if (first MsgCode = 'EAI-03004' or Msg = 'BWENGINE-100067' or first ApplicationError/@ERROR_CODE (default '10001') = '10000') and redeliver = (no RedeliveryCount) or RedeliveryCount < 5. | High | Same pattern as BR-002 in the worker process. |
| BR-004 | Link the first Siebel account only | SIEBEL_ACCOUNT_ID = CONTACT_PERSON/SIEBEL_ACCOUNT_REFERENCE[1]/SIEBEL_ACCOUNT_ID; REFERENCE = RUNTIME_PARAMETERS/ID; ORIGIN = ORIGIN_ID = 'SFA'. | Medium | ucp-sfa-to-ucm-accountref-map. |
| BR-005 | Non-retryable failures are reported and consumed | Log via Sub_ErrorTrace and mail; the message is not redelivered. | High | shared-sub-error-handler + shared-sub-send-error-mail; end normally. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound JMS queue (TIBCO EMS) | %%Env%%.FIXED.%%CRMServices/ComponentDomain%%.%%CRMServices/ComponentId%%.Subscribe.UpsertContactPub.1 (default XDVL.FIXED.Operations.CRMServices.Subscribe.UpsertContactPub.1) | EMS username/password (shared connection) | legacy/tibco/CRMServices/CRMServices/Processes/Subscribe_UpsertContactPub.process |
| outbound Rendezvous request/reply via Common Sub_Requester | concat(Env,'.FIXED.CRM.SFAAdapter.Request.UpsertContact.1') | none visible | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process |
| outbound JMS queue send via Common Sub_NotifierJMS | concat(Env,'.FIXED.Operations.',ComponentId,'._UpsertContactPub.1') | EMS username/password | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process |
| inbound JMS queue (TIBCO EMS, private) | %%Env%%.FIXED.%%CRMServices/ComponentDomain%%.%%CRMServices/ComponentId%%._UpsertContactPub.1 | EMS username/password | legacy/tibco/CRMServices/CRMServices/Processes/_Subscribe_UpsertContactPub.process |
| outbound Rendezvous request/reply (via crm-shared-framework) | %%Env%%.FIXED.CRM.UCMAdapter.Request.AddAccountReference.1 (CMH/TO copied from the queued request) | none visible | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_CallAddAccountReferenceFromUpsertContactPub.process |
| JMS Queue Receiver x2 | Text; transacted false; acknowledgeMode 2; maxSessions %%Common/JMS/JmsMaxSessions%%; JMSDeliveryMode PERSISTENT; ConnectionReference /Common/Shared Resources/JMS Connection.sharedjmscon (not in export). The redelivery count is read from OtherProperties/RedeliveryCount. | — | legacy/tibco/CRMServices/CRMServices/Processes/Subscribe_UpsertContactPub.process |
| Common Sub_Requester (SFA) | Destination concat(Env,'.FIXED.CRM.SFAAdapter.Request.UpsertContact.1'); Timeout CRMServices/RV/Timeout; Body = UpsertContactRequest. | — | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process |
| Common Sub_NotifierJMS | Destination internal queue; Body tib:render-xml(AddAccountReferenceRequest); MessageId = SFA response MESSAGE_ID. | — | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| crm-shared-framework | Sub: UCM AccountReference ADD | synchronous |
| crm-shared-framework | Sub: CRM Send Error Mail | synchronous |
| crm-shared-framework | Sub: CRM Error Handler (replaces Sub_ErrorTrace / Sub_ThrowException) | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertContactPub.process | eai:getMessageText(varMsgCode) for the thrown message text; eai:getMessageId and eai:isTraceOn in the listeners. | Cross Reference Table 'CRM Message Catalogue CRT' and shared-cmh-header-fn (crm-shared-framework). | Medium |
| xslt | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process | Retry decision XPath combining requester MsgCode, BW engine error code, SFA ERROR_CODE and the RESEND_ALLOWED attribute. | Decision/Route shapes on the Try/Catch error message plus a document property holding the delivery count. | Medium |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Classify each failure: timeout-class and within the retry limit, then rethrow so the message is not completed and is redelivered (intended); anything else, then log, mail and complete. Implemented separately in the main flow (limit 4) and the worker (limit 5). In Boomi: Try/Catch; a Decision on error class and on a delivery-count document property; the retry path ends with an Exception so the JMS listener does not acknowledge; the non-retry path calls 'Sub: CRM Error Handler' and 'Sub: CRM Send Error Mail' (crm-shared-framework) and ends normally. |
| Retries | JMS redelivery: SFA step while RedeliveryCount < 4, UCM step while RedeliveryCount < 5; only for EAI-03004, BWENGINE-100067 or back-end ERROR_CODE 10000. No delay between attempts is configured in the processes (EMS redelivery delay, if any, is server configuration not in the export). |
| Compensation | None. If the UCM step ultimately fails, the SFA contact remains without its UCM link. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| Common Sub_ThrowException (redelivery behaviour) | library | no |
| Common Sub_Requester / Sub_NotifierJMS / Sub_ErrorTrace | library | no |
| XSDSchemas/Common/Base.xsd, ResultStatus.xsd (incl. GD element) | other | no |
| EMS bridge Publish.UpsertContactPub.1 to Subscribe queue | other | no |
| CDM_Structures.xsd (ContactPerson_Type) | other | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| SFAAdapter UpsertContact | TIBCO Rendezvous request/reply (target TBD) | %%Env%%.FIXED.CRM.SFAAdapter.Request.UpsertContact.1 | Not stated in export |
| UCMAdapter AddAccountReference | TIBCO Rendezvous request/reply (target TBD) | %%Env%%.FIXED.CRM.UCMAdapter.Request.AddAccountReference.1 | Not stated in export |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| DEV | Env | XDVL (design-time default; environment it denotes not stated) | [redacted].substvar |

## 14. Boomi Migration Readiness Summary

When a contact person changes in the customer master, a publication arrives and this interface applies the change to SFA. If SFA reports that it created a new contact, rather than updating an existing one, a second step tells UCM about the new SFA contact ID, so that UCM can link the two records. That second step runs separately, from its own queue, so it can be retried independently.

Retries are deliberately narrow. Only failures that look like timeouts (a transport timeout, the TIBCO engine timeout code, or a back-end error code 10000) are retried, by letting the message be redelivered: up to 4 times for the SFA step and 5 times for the UCM step. Every other failure is logged, e-mailed to integration support and dropped. If queuing the UCM step itself fails, the contact is in SFA but the UCM link is never made, and only an e-mail records it.

The flow is moderately complex. Its two listeners, two back-end calls and two maps are straightforward, apart from the large but 1:1 contact mapping. What needs care is reproducing the retry behaviour on Boomi's JMS connector: it depends on the message's redelivery count, and on a TIBCO framework component (Sub_ThrowException) that is not in the export. The Rendezvous transport to the adapters and the missing header schemas are shared blockers.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | CRM ContactPub PUBLISH XML Prof | ucp-pub-prof | profile.xml | — |
| 2 | SFA Contact UPSERT Req XML Prof | ucp-sfa-upsertcontact-req-prof | profile.xml | — |
| 3 | SFA Contact UPSERT Resp XML Prof | ucp-sfa-upsertcontact-resp-prof | profile.xml | — |
| 4 | EMS CRM ContactPub Internal LISTEN Op | ucp-internal-listen-op | connector-action | shared-ems-jms-conn, shared-ucm-accountreference-add-req-prof |
| 5 | EMS CRM ContactPub Internal SEND Op | ucp-internal-send-op | connector-action | shared-ems-jms-conn, shared-ucm-accountreference-add-req-prof |
| 6 | EMS CRM ContactPub LISTEN Op | ucp-listen-op | connector-action | shared-ems-jms-conn, ucp-pub-prof |
| 7 | SFA Contact UPSERT Op | ucp-sfa-upsertcontact-op | connector-action | shared-ems-jms-conn, ucp-sfa-upsertcontact-req-prof, ucp-sfa-upsertcontact-resp-prof |
| 8 | CRM ContactPub PUBLISH XML - SFA Contact UPSERT Req XML | ucp-pub-to-sfa-map | transform.map | ucp-pub-prof, ucp-sfa-upsertcontact-req-prof, shared-cmh-header-fn |
| 9 | SFA Contact UPSERT Resp XML - UCM AccountReference ADD Req XML | ucp-sfa-to-ucm-accountref-map | transform.map | ucp-sfa-upsertcontact-resp-prof, shared-ucm-accountreference-add-req-prof, shared-cmh-header-fn |
| 10 | CRM ContactPub AccountReference ADD Worker | ucp-worker-process | process | ucp-internal-listen-op, shared-sub-ucm-accountreference-add, shared-sub-send-error-mail, shared-sub-error-handler |
| 11 | CRM ContactPub UPSERT Listener | ucp-main-process | process | ucp-listen-op, ucp-pub-to-sfa-map, ucp-sfa-upsertcontact-op, ucp-sfa-to-ucm-accountref-map, ucp-internal-send-op, shared-sub-send-error-mail, shared-sub-error-handler |

## 15. Appendix

- legacy/tibco/CRMServices/CRMServices/Processes/Subscribe_UpsertContactPub.process
- legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertContactPub.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContactPub.process
- legacy/tibco/CRMServices/CRMServices/Processes/_Subscribe_UpsertContactPub.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_CallAddAccountReferenceFromUpsertContactPub.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Publish/0429_UpsertContactPub.xsd
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Imported/0429_UpsertContactPub_1.0_Publish_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/SFAAdapter/Request/0431_UpsertContact.xsd
- legacy/tibco/CRMServices/XSDSchemas/SFAAdapter/Response/0431_UpsertContact.xsd
- legacy/tibco/CRMServices/XSDSchemas/SFAAdapter/Imported/0431_UpsertContact_1.0_Request_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/SFAAdapter/Imported/0431_UpsertContact_1.0_Response_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/SFAAdapter/Imported/CDM_Structures.xsd
- legacy/tibco/CRMServices/defaultVars/CRMServices/UpsertContactPub/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/CRMServices/RV/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/CRMServices/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/Common/JMS/defaultVars.substvar
- legacy/tibco/CRMServices/.designtimelibs
- legacy/tibco/CRMServices/CRMServices/Test Processes/UpsertContactPub/Publisher FF.process
- legacy/tibco/CRMServices/CRMServices/Test Processes/UpsertContactPub/STUB 0431-UpsertContact.process

## Open questions

### Still outstanding

REVIEW-NEEDED: When a contact update fails because SFA or UCM timed out, should the migrated process retry it automatically, and how many times? (In BW the retry path rethrows the error to the listener, which calls Sub_ThrowException and then always confirms the message, so whether redelivery actually happens depends on Sub_ThrowException (missing). The Boomi design (do not acknowledge on retry, acknowledge otherwise) assumes redelivery was the intent.)
  - If this is not answered: Timeout-class failures are retried by redelivery up to 4 times for the SFA step and 5 times for the UCM step, as the configuration suggests.

REVIEW-NEEDED: Which system publishes the contact-update events that this flow receives? (Confirm the producer of the Publish.UpsertContactPub.1 topic and the EMS bridge to the Subscribe queue. The schema namespace suggests UCM, but neither is in the export.)
  - If this is not answered: UCM publishes to %%Env%%.FIXED.Operations.CRMServices.Publish.UpsertContactPub.1, and the existing EMS bridge to the Subscribe queue is kept unchanged.

REVIEW-NEEDED: When a contact is linked to several Siebel accounts, should UCM record the new SFA contact against each account, or only the first, as today? (Only the first SIEBEL_ACCOUNT_REFERENCE is used. Confirm that a contact linked to several accounts needs only one UCM reference.)
  - If this is not answered: Only the first account is linked, as today.

REVIEW-NEEDED: If a new SFA contact is created but its link in UCM cannot be queued, should that be retried or only reported by e-mail, as today? (A failure to queue the UCM step is not treated as retryable, so the SFA contact stays unlinked in UCM unless someone acts on the e-mail.)
  - If this is not answered: As today: reported by e-mail and the exception record, with no automatic retry.

REVIEW-NEEDED: Does UCM rely on the object label 'SIEBEL_ADCCOUNT_ID' (as currently sent, misspelt) in the AddAccountReference header, or may it be corrected to 'SIEBEL_ACCOUNT_ID'? (The worker re-maps the CMH OBJECT to 'SIEBEL_ADCCOUNT_ID' (a typo), while the main flow sends 'SIEBEL_ACCOUNT_ID'. Confirm with the UCMAdapter owners whether OBJECT is significant before choosing which value the Boomi build sends.)
  - If this is not answered: The current value 'SIEBEL_ADCCOUNT_ID' is reproduced.

REVIEW-NEEDED: Should error e-mails for this flow still include the full contact publication with personal details? (The SFA-step error mail embeds the rendered publication (names, date of birth, e-mail, phone, addresses).)
  - If this is not answered: The e-mail carries the message ID, error code and description only; the full message stays in the exception record.

REVIEW-NEEDED: Will the queue names keep the 'Operations' domain in every environment? (The sender hard-codes '.FIXED.Operations.' in the internal queue name, but the listener uses %%CRMServices/ComponentDomain%% (default 'Operations'). Keep them identical in Boomi (both from CRM_ComponentDomain) and confirm no environment overrides ComponentDomain.)
  - If this is not answered: Both names are built from one domain property defaulting to 'Operations'.

REVIEW-NEEDED: How many contact publications arrive per day, and what share create new contacts in SFA? (Volumes drive listener concurrency and the load on the UCM step.)
  - If this is not answered: Low volume; same concurrency as today (10 sessions per listener).

