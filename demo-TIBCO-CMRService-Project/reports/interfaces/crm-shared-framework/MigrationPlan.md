# Migration Plan — CRMServices.SharedFramework

**Project:** Demo-TIBCO-CMRService-Project  
**Source platform:** TIBCO  
**Generated:** 2026-10-29T05:02:49.378Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | CRMServices.SharedFramework |
| Identifier | crm-shared-framework |
| Trigger | Listener |
| Starts with | No inbound data |
| Complexity | medium (45) |
| Components to build | 12 |
| Source artefacts | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process, legacy/tibco/CRMServices/CRMServices/Shared Resources/CRMServicesMessages.xml, legacy/tibco/CRMServices/CRMServices.archive, legacy/tibco/CRMServices/defaultVars/Common/JMS/defaultVars.substvar, legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Request/AddAccountReference.xsd, legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Response/AddAccountReference.xsd, legacy/tibco/CRMServices/XSDSchemas/CRMServices/Publish/UpsertAccountWithReferencing.xsd |
| Depends on | Nothing |

## 2. End-to-End Process Architecture

Two CRMServices sub-processes, 'Send Mail' and 'Sub_AddAccountReference', are invoked by Call Process from the business flows. Sub_AddAccountReference delegates the transport to Common Sub_Requester (Rendezvous request/reply, 5 s timeout) and parses the reply with validation. Error handling, trace logging, the exception queue and component registration live in the Common library (Sub_ThrowException, Sub_ErrorTrace, Sub_Trace, Sub_RegisterComponent), together with the JMS and RV shared resources and the CMH, ResultStatus and Errors schemas. Custom Java XPath functions in com.kpn.eai.component.common.CommonXPathFunctions supply message-ID extraction and trace switches. Message catalogues CRMServicesMessages.xml and EAIMessages.xml are deployed in the EAR's shared archive.

Everything in this interface lives in the OSI universal shared folder ('#Shared/CRMServices'). A single TIBCO EMS JMS connection (Generic JNDI, EMS client JARs as a custom library) with Environment Extensions. A Mail connection and send operation, with recipients held in a Cross Reference Table and the sub-process reached through Process Route so recipients can change without redeploying callers. 'Sub: CRM Send Error Mail' has a Try/Catch that never propagates. 'Sub: CRM Error Handler' sends a structured error record to the EMS exception queue. 'Sub: UCM AccountReference ADD' is a request/reply connector call, a Decision on STATUS and an Exception carrying stable codes. Also: a user-defined map function for CMH stamping, a message-catalogue Cross Reference Table, and a 'CRMServices Settings' Process Property component for ComponentId, Domain, Version, Env and timeouts. Sub-processes start with Data Passthrough, per OSI guidance.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| Send Mail.process (MailPubActivity) | shared-sub-send-error-mail + shared-mail-send-op + shared-smtp-mail-conn | Try/Catch swallows mail errors; Process Route recommended. |
| Sub_AddAccountReference.process | shared-sub-ucm-accountreference-add + shared-ucm-accountreference-add-op | Decision on STATUS; Exception with CRMS-03021 / preserved transport codes. |
| Common Sub_Requester (RV request/reply) | JMS request/reply operation on shared-ems-jms-conn (assumed) | Depends on the transport decision; Boomi has no Rendezvous connector. |
| Common Sub_ThrowException / Sub_ErrorTrace | shared-sub-error-handler + shared-exception-queue-send-op | Format and behaviour not in export. |
| Common Sub_Trace + eai:isTraceOn | Process property trace flag; Notify shapes, or omitted | OSI guidance discourages logging inside production processes. |
| /Common/Shared Resources/JMS Connection.sharedjmscon + Common/JMS substvar | shared-ems-jms-conn with Environment Extensions |  |
| eai:getMessageId and CMH stamping in every mapper | shared-cmh-header-fn (user-defined map function) |  |
| CRMServicesMessages.xml / EAIMessages.xml | Cross Reference Table 'CRM Message Catalogue CRT' | EAIMessages.xml not in export. |
| UpsertAccountWithReferencing.xsd (+ Imported 0420) | shared-crm-uawr-publish-prof | Recursive type; script-based handling downstream. |
| CRMServices/Mail/* global variables | Mail connection extension + 'CRM Mail Distribution CRT' | Values missing. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive mail request | Sub-process start with MessageId, Body and Subject (all strings). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process |
| 2 | Optional trace of mail | If eai:isTraceOn('CRMS-04020') is true, log 'Sending Mail' with MessageId and Body via Common Sub_Trace. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process |
| 3 | Send e-mail | Mail activity to host %%CRMServices/Mail/Host%%. From/To/CC/BCC come from global variables; subject is the input; body is tib:render-xml(Body). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process |
| 4 | Optional trace of notification | If eai:isTraceOn('CRMS-04021') is true, log 'Error Mail Send'. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process |
| 5 | Mail failure | On error, raise an Error fault with the SendFailedException msg/msgCode. All callers route this fault to a Null activity and continue. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process |
| 6 | Receive AddAccountReference request | Sub-process start with AddAccountReferenceRequest (CMH incl. TO/@DESTINATION, BODY/SIEBEL_ACCOUNT_ID, EXTENAL_REFERENCE). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process |
| 7 | Request/reply to UCMAdapter | Call Common Sub_Requester with Destination = CMH/TO/@DESTINATION, Timeout = CRMServices/RV/Timeout (5000 ms), Body = the request. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process |
| 8 | Parse and validate reply | Parse XML (validateOutput = true) as AddAccountReferenceResponse. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process |
| 9 | Check UCM status | If STATUS_RESULT/@STATUS != 0, raise Errors/Error with MsgCode CRMS-03021, Params = ERROR_DESCRIPTION, MsgId = MESSAGE_ID, ApplicationError = STATUS_RESULT. Otherwise return the response. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process |
| 10 | Technical failure | Requester error: re-raise its Errors/Error unchanged. Parse or other error: raise CRMS-03000 with the BW ErrorReport message. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process |
| 11 | Hand failure to exception framework | JMS listeners pass ComponentId, ComponentVersion, Errors, the original text body and JMS headers and properties to Common Sub_ThrowException, then acknowledge the message. | legacy/tibco/CRMServices/CRMServices/Processes/_Subscribe_UpsertAccountWithReferencesJMS.process |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| UCM AccountReference ADD Req XML Prof | xml | AddAccountReferenceRequest/CMH | complex (Base.xsd:Base) | yes |
| UCM AccountReference ADD Req XML Prof | xml | AddAccountReferenceRequest/BODY/SIEBEL_ACCOUNT_ID | xs:string | no |
| UCM AccountReference ADD Req XML Prof | xml | AddAccountReferenceRequest/BODY/EXTENAL_REFERENCE | complex, 0..unbounded | no |
| UCM AccountReference ADD Req XML Prof | xml | AddAccountReferenceRequest/BODY/EXTENAL_REFERENCE/REFERENCE | xs:string | yes |
| UCM AccountReference ADD Req XML Prof | xml | AddAccountReferenceRequest/BODY/EXTENAL_REFERENCE/ORIGIN | xs:string | no |
| UCM AccountReference ADD Req XML Prof | xml | AddAccountReferenceRequest/BODY/EXTENAL_REFERENCE/ORIGIN_ID | xs:string | no |
| UCM AccountReference ADD Resp XML Prof | xml | AddAccountReferenceResponse/CMH | complex (Base.xsd:Base) | yes |
| UCM AccountReference ADD Resp XML Prof | xml | AddAccountReferenceResponse/STATUS_RESULT | ResultStatus.xsd:ResultStatus | yes |
| UCM AccountReference ADD Resp XML Prof | xml | AddAccountReferenceResponse/BODY/SIEBEL_ACCOUNT_ID | xs:string | no |
| UCM AccountReference ADD Resp XML Prof | xml | AddAccountReferenceResponse/BODY/EXTENAL_REFERENCE/REFERENCE | xs:string | yes |
| UCM AccountReference ADD Resp XML Prof | xml | AddAccountReferenceResponse/BODY/EXTENAL_REFERENCE/ORIGIN | xs:string | no |
| UCM AccountReference ADD Resp XML Prof | xml | AddAccountReferenceResponse/BODY/EXTENAL_REFERENCE/ORIGIN_ID | xs:string | no |
| CRM UpsertAccountWithReferencing PUBLISH XML Prof | xml | UpsertAccountWithReferencingPublish/CMH | complex (Base.xsd:Base) | yes |
| CRM UpsertAccountWithReferencing PUBLISH XML Prof | xml | UpsertAccountWithReferencingPublish/BODY/RecursiveProcessing | xs:boolean | no |
| CRM UpsertAccountWithReferencing PUBLISH XML Prof | xml | UpsertAccountWithReferencingPublish/BODY/SiebelAccount | Type_SiebelAccountHierarchy, 0..unbounded | no |
| CRM UpsertAccountWithReferencing PUBLISH XML Prof | xml | .../SiebelAccount/{AccountHierarchyLevel, AccountStatus, AccountTypeCode, Alias, CAUpdateFlag, CAUpdateType, CSN, DisableMailings, DontCall, DontShare, HomePage, Id, IntegrationId, KPNBlockedTelephone, KPNBranchCode, KPNBranchFlag, KPNCOCNR, KPNCODRV, KPNFinancialStatus, KPNFraudIndicator, KPNMoveCKRnrOld, KPNMoveIndicator, KPNRELNUM, KPNSalesDistrictCode, Location, MainEmailAddress, MainFaxNumber, MainPhoneNumber, MarketClass, MarketType, MasterAccountIntegrationId, Name, OCRMOperation, ParentAccountId, ParentAccountIntegrationId, PrimaryContactUCMRowId, Type, UCMRowId, UUID} | xs:string, each 0..1 | no |
| CRM UpsertAccountWithReferencing PUBLISH XML Prof | xml | .../SiebelAccount/Account_Affiliation/{AffiliationId, AffiliationIntegrationId, AffiliationType, AffiliationUUID} | xs:string; parent 0..unbounded | no |
| CRM UpsertAccountWithReferencing PUBLISH XML Prof | xml | .../SiebelAccount/Contact/{BirthDate, EmailAddress, FaxPhone, FirstName, IntegrationId, KPNAcademicTitle, LastName, MiddleName, OCRMOperation, PrefferedName, Suffix, SuppressAllEmails, UCMRowId, UUID, WorkPhone} | xs:string; parent 0..unbounded | no |
| CRM UpsertAccountWithReferencing PUBLISH XML Prof | xml | .../SiebelAccount/BusinessAddress/{AddressDescription, AddressName, AddressType, City, Country, FuturePrimaryFlg, KPNEndDate, KPNPrimaryPOBoxFlag, KPNSpecialAddrDescription, KPNStartDate, KPNZipcodeSuffix, PostalCode, State, Status, StreetAddress, StreetNumber, StreetNumberExtension}, @IsPrimaryMVG | xs:string; parent 0..unbounded | no |
| CRM UpsertAccountWithReferencing PUBLISH XML Prof | xml | .../SiebelAccount/CifAccountReference/{AccountId, CIFSystemId, ExternalId1, SystemName, SystemNumber} | xs:string; parent 0..unbounded | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | RequesterIn/Destination | AddAccountReferenceRequest/CMH/TO/@DESTINATION | transformed |  | direct copy | low | Callers set it to concat(Env,'.FIXED.CRM.UCMAdapter.Request.AddAccountReference.1'). In Boomi it becomes the operation destination or a dynamic document property. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process | AddAccountReference request to requester envelope |
| MR-002 | RequesterIn/Timeout | global variable CRMServices/RV/Timeout | hardcoded |  | 5000 | low | Operation timeout in ms. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process | AddAccountReference request to requester envelope |
| MR-003 | RequesterIn/Body | AddAccountReferenceRequest (whole document) | passthrough |  | xsl:copy-of | low | The request is sent unchanged. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process | AddAccountReference request to requester envelope |
| MR-004 | Errors/Error/MsgCode | constant | hardcoded | STATUS_RESULT/@STATUS != 0 | 'CRMS-03021' | low | Catalogue text 'Backend replied with error: %1'. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process | UCM status error to KPN Errors |
| MR-005 | Errors/Error/Params | STATUS_RESULT/@ERROR_DESCRIPTION | conditional | attribute present |  | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process | UCM status error to KPN Errors |
| MR-006 | Errors/Error/MsgId | AddAccountReferenceResponse/CMH/TRACKING/MESSAGE_ID | conditional | element present |  | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process | UCM status error to KPN Errors |
| MR-007 | Errors/Error/ApplicationError | STATUS_RESULT (all attributes and children) | passthrough |  | xsl:copy-of | low | Callers read ApplicationError/@ERROR_CODE (for example '10000' for timeout-class retry). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_CallAddAccountReferenceFromUpsertContactPub.process | UCM status error to KPN Errors |
| MR-008 | fault message / messageCode | STATUS_RESULT/@ERROR_DESCRIPTION, @ERROR_CODE | conditional | attribute present |  | low | In Boomi, put both in the Exception shape message in a parseable form (for example 'CRMS-03021\|<ERROR_CODE>\|<description>'). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process | UCM status error to KPN Errors |
| MR-009 | Errors/Error | $_error_Call-Process/Error | conditional | the requester call failed | copy unchanged | low | Preserves EAI-03004 (timeout). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process | Technical error to KPN Errors |
| MR-010 | Errors/Error/MsgCode | constant | conditional | any other error | 'CRMS-03000' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process | Technical error to KPN Errors |
| MR-011 | Errors/Error/Params | $_error/ErrorReport/Msg | conditional | any other error |  | low | Boomi: Try/Catch base error message. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process | Technical error to KPN Errors |
| MR-012 | from | global variable CRMServices/Mail/From | hardcoded |  |  | low | Value not in export. Cross Reference Table in Boomi. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process | Mail request to mail message |
| MR-013 | to, cc, bcc | global variables CRMServices/Mail/To, CC, BCC | hardcoded |  |  | low | Values not in export. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process | Mail request to mail message |
| MR-014 | subject | ServiceIn/Subject | transformed |  | direct copy | low | Caller-supplied, for example 'Error in GetAccountData'. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process | Mail request to mail message |
| MR-015 | bodyText | ServiceIn/Body | transformed |  | tib:render-xml(Body, false(), true()) | low | Plain text. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process | Mail request to mail message |
| MR-016 | CMH/FROM/@COMPONENT_ID | CRMServices/ComponentId | hardcoded |  | 'CRMServices' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | CMH header stamping (shared function) |
| MR-017 | CMH/FROM/@COMPONENT_VERSION | CRMServices/ComponentVersion | hardcoded |  | '2.0.3' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | CMH header stamping (shared function) |
| MR-018 | CMH/TRACKING/ITEM(new)/@APPINFO | ComponentDomain, ComponentId | calculated |  | concat(ComponentDomain,'/',ComponentId) = 'Operations/CRMServices' | low | The UpsertContact flow appends '/' + ComponentVersion; make the function take an optional version. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process | CMH header stamping (shared function) |
| MR-019 | CMH/TRACKING/ITEM(new)/@TIMESTAMP | current time | calculated |  | tib:timestamp() -> Boomi current date/time | medium | The exact output format of tib:timestamp() must be matched; it is not determinable from the export. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | CMH header stamping (shared function) |
| MR-020 | CMH/TRACKING/MESSAGE_ID | inbound CMH/TRACKING/MESSAGE_ID or eai:getMessageId(raw text) | transformed |  | custom Java -> profile element read | medium | Java behaviour on malformed input unknown. | legacy/tibco/CRMServices/CRMServices/Processes/Reply_GetAccountdataRV.process | CMH header stamping (shared function) |
| MR-021 | CMH/TRACKING/ITEM (existing) | inbound CMH/TRACKING/ITEM | looped |  | copy all, then append the new ITEM | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | CMH header stamping (shared function) |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Non-zero UCM status is a back-end error | If AddAccountReferenceResponse/STATUS_RESULT/@STATUS != 0, raise CRMS-03021 with UCM's ERROR_CODE and ERROR_DESCRIPTION; otherwise return the response. | High | Decision shape on STATUS_RESULT/@STATUS, then Exception shape with a code-bearing message. |
| BR-002 | Preserve transport error codes | Requester errors (including timeout EAI-03004) are passed to the caller unchanged so that callers can decide to retry. | High | Try/Catch around the connector; map the connector's timeout to a stable code (EAI-03004 equivalent) in the Exception message. |
| BR-003 | Reply must be schema-valid | The UCM reply is parsed with validation; an invalid reply raises CRMS-03000. | Medium | Optional Groovy XSD validation step; otherwise profile parse failure caught by Try/Catch. |
| BR-004 | Mail failure never affects the business outcome | Send Mail raises an error on SendFailedException, but every caller catches it with a Null activity and continues its own error path. | High | Try/Catch inside 'Sub: CRM Send Error Mail' that logs and ends normally. |
| BR-005 | Trace switch per message code | Mail sending (CRMS-04020) and mail notification (CRMS-04021) are logged only when eai:isTraceOn(code) is true. | Low | Process property CRM_TraceEnabled gating Notify shapes (or omit, per OSI guidance to keep logging out of production processes). |
| BR-006 | Message catalogue | Error codes and texts: CRMS-03000 Technical Error; 03001 Account Hierarchy exceeds 50 accounts; 03002 Error Parsing response; 03003 Error parsing request; 03004 Called service returned error %1; 03011 Output data invalid; 03020 General Un-attributable Error; 03021 Backend replied with error: %1; 03022 No SiebelAccount found in received message; 04020 Sending Mail; 04021 Mail Notification. Application code CRMS, locale en_UK. | Medium | Cross Reference Table 'CRM Message Catalogue CRT' (code -> description, role, category). |
| BR-007 | Failed listener messages go to the exception framework and are acknowledged | On an unhandled error, a JMS listener passes Errors (or EAI-03057), the text body and JMS headers to Common Sub_ThrowException, then confirms the message. | High | 'Sub: CRM Error Handler' sends to the exception queue; the listener then ends normally (acknowledge) unless the caller explicitly chose retry. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| outbound TIBCO Rendezvous request/reply (via Common Sub_Requester) | %%Env%%.FIXED.CRM.UCMAdapter.Request.AddAccountReference.1 (subject supplied by caller in CMH/TO/@DESTINATION) | none visible (network and subject segregation) | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process |
| outbound SMTP | %%CRMServices/Mail/Host%% (value not in export) | unknown | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process |
| outbound JMS (TIBCO EMS) | %%Env%%.%%Common/ErrorHandling/ExceptionQueue%% (name not in export) | username/password (EMS) | legacy/tibco/CRMServices/CRMServices/Test Processes/UpsertAccountwithReferences/WithOutFiltering/SubscribeErrorHandler.process |
| JMS connection (TIBCO EMS via JNDI) | Shared resource '/Common/Shared Resources/JMS Connection.sharedjmscon' (not in export). Global variables: JndiContextFactory com.tibco.tibjms.naming.TibjmsInitialContextFactory, JndiContextUrl tibjmsnaming://localhost:7222, JmsProviderUrl tcp://localhost:7222, [redacted], JmsClientId BWClientid, JmsMaxSessions 10, JmsTibcoCompress true, JmsTibcoPreserveUndelivered true; usernames and passwords redacted. | — | legacy/tibco/CRMServices/defaultVars/Common/JMS/defaultVars.substvar |
| Mail (TIBCO Mail palette, MailPubActivity) | host %%CRMServices/Mail/Host%%; newMimeSupport true; inputOutputVersion 5.2.0; From/To/CC/BCC from CRMServices/Mail/* global variables; plain-text body. | — | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process |
| Rendezvous request/reply (Common Sub_Requester) | Inputs Destination, Timeout (CRMServices/RV/Timeout = 5000) and Body; output RequesterOut/Body (XML text). The implementation is in the Common library and not in the export. | — | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process |

## 8. Sub-Process Call Hierarchy

> **Not available.** This interface calls no sub-processes.

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/tibco/CRMServices/CRMServices/Processes/Reply_GetAccountdataRV.process | Custom XPath functions from class com.kpn.eai.component.common.CommonXPathFunctions (namespace java://com.kpn.eai.component.common.CommonXPathFunctions): getMessageId(xmlText) extracts CMH/TRACKING/MESSAGE_ID from raw XML; isTraceOn(code) switches trace logging per message code; getMessageText(code) looks up catalogue text. The JAR and source are not in the export. | getMessageId: read MESSAGE_ID from the parsed profile in shared-cmh-header-fn. isTraceOn: process property CRM_TraceEnabled. getMessageText: Cross Reference Table 'CRM Message Catalogue CRT'. | Medium |
| other | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process | tib:render-xml(Body, false(), true()) renders the body for the mail text. | Message shape / document text, no script needed. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Errors are carried as a KPN Errors/Error structure (MsgCode, Params, MsgId, ApplicationError, ErrorReport) defined in /Common/Shared Resources/Errors.xsd (not in export). Sub-processes raise faults, callers decide whether to mail, retry or pass the error to the exception framework, and mail failures are always swallowed. |
| Retries | None inside this interface. Retry is decided by callers (crm-upsert-contact-pub retries timeout-class errors via JMS redelivery, up to 4 or 5 times), so this interface must preserve the timeout error codes. |
| Compensation | None. A successful AddAccountReference is not undone if a later step fails. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| Common_2.2.2.projlib (Sub_Requester, Sub_Publisher, Sub_NotifierJMS, Sub_Trace, Sub_ErrorTrace, Sub_ThrowException, Sub_RegisterComponent, Sub_CommonBundle) | library | no |
| /Common/Shared Resources/JMS Connection.sharedjmscon | other | no |
| /Common/Shared Resources/RVCMQ Transport.rvtransport | other | no |
| XSDSchemas/Common/Base.xsd and ResultStatus.xsd | other | no |
| /Common/Shared Resources/Errors.xsd | other | no |
| /Common/Shared Resources/EAIMessages.xml | other | no |
| CRMServicesMessages.xml (message catalogue) | crossref | yes |
| com.kpn.eai.component.common.CommonXPathFunctions (Java) | library | no |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| UCMAdapter AddAccountReference | TIBCO Rendezvous request/reply (target: JMS request/reply, to be decided) | %%Env%%.FIXED.CRM.UCMAdapter.Request.AddAccountReference.1 | Not stated in export (separate KPN EAI component) |
| SMTP server | SMTP | %%CRMServices/Mail/Host%% | Not stated in export |
| EAI exception queue | JMS (TIBCO EMS) | %%Env%%.%%Common/ErrorHandling/ExceptionQueue%% | KPN EAI Common framework |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| DEV | Env | XDVL (design-time default; which environment it denotes is not stated) | [redacted].substvar |
| DEV | Common/JMS/JmsProviderUrl | tcp://localhost:7222 (design-time default) | [redacted].substvar |

## 14. Boomi Migration Readiness Summary

This interface collects the shared plumbing used by the four CRMServices integrations: the EMS connection, the 'something failed' e-mail, the hand-off of failures to the enterprise exception queue, the standard message header, and one reusable back-end call that tells UCM about an external reference for an account or contact.

The parts that exist in the export are small and clear. The e-mail sub-process sends a subject and body to a configured list and never lets a mail failure stop the business flow. The AddAccountReference sub-process sends one request, checks the returned status, and raises a 'back-end replied with error' (CRMS-03021) when the status is not zero.

Much of what this interface replaces lives in a shared TIBCO library (Common_2.2.2.projlib) that was not included in the export. That library holds the transport that actually reaches UCM, the exception-queue handler, the header schemas, the JMS connection definition and some custom Java helper functions. The mail server settings were also not exported. Until those are supplied, and the transport question for reaching UCM is decided, the profiles, connection and error handler cannot be finished.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | CRM UpsertAccountWithReferencing PUBLISH XML Prof | shared-crm-uawr-publish-prof | profile.xml | — |
| 2 | UCM AccountReference ADD Req XML Prof | shared-ucm-accountreference-add-req-prof | profile.xml | — |
| 3 | UCM AccountReference ADD Resp XML Prof | shared-ucm-accountreference-add-resp-prof | profile.xml | — |
| 4 | KPN SMTP Mail Conn | shared-smtp-mail-conn | connector-settings | — |
| 5 | TIBCO EMS Conn | shared-ems-jms-conn | connector-settings | — |
| 6 | EMS CRM Exception SEND Op | shared-exception-queue-send-op | connector-action | shared-ems-jms-conn |
| 7 | KPN Mail ErrorNotification SEND Op | shared-mail-send-op | connector-action | shared-smtp-mail-conn |
| 8 | UCM AccountReference ADD Op | shared-ucm-accountreference-add-op | connector-action | shared-ems-jms-conn, shared-ucm-accountreference-add-req-prof, shared-ucm-accountreference-add-resp-prof |
| 9 | CRM CMH Header Functions | shared-cmh-header-fn | transform.function | — |
| 10 | Sub: CRM Send Error Mail | shared-sub-send-error-mail | process | shared-mail-send-op |
| 11 | Sub: CRM Error Handler | shared-sub-error-handler | process | shared-exception-queue-send-op, shared-sub-send-error-mail |
| 12 | Sub: UCM AccountReference ADD | shared-sub-ucm-accountreference-add | process | shared-ucm-accountreference-add-op, shared-sub-error-handler |

## 15. Appendix

- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Send Mail.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process
- legacy/tibco/CRMServices/CRMServices/Shared Resources/CRMServicesMessages.xml
- legacy/tibco/CRMServices/CRMServices.archive
- legacy/tibco/CRMServices/.designtimelibs
- legacy/tibco/CRMServices/defaultVars/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/Common/JMS/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/CRMServices/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/CRMServices/RV/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/CRMServices/Mail/.copyarea.db
- legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Request/AddAccountReference.xsd
- legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Response/AddAccountReference.xsd
- legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Imported/0422_AddAccountReference_1.0_Request1_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Imported/0422_AddAccountReference_1.0_Response_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Publish/UpsertAccountWithReferencing.xsd
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Imported/0420_UpsertAccountWithReferencing_1.0_Publish_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/CRMServices/Processes/Reply_GetAccountdataRV.process
- legacy/tibco/CRMServices/CRMServices/Processes/_Subscribe_UpsertAccountWithReferencesJMS.process
- legacy/tibco/CRMServices/CRMServices/Processes/Main_GetAccountdata.process
- legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertContact.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertContact_UpsertContact.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_CallAddAccountReferenceFromUpsertContactPub.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferences.process
- legacy/tibco/CRMServices/CRMServices/Test Processes/UpsertAccountwithReferences/WithOutFiltering/SubscribeErrorHandler.process

## Open questions

### Still outstanding

REVIEW-NEEDED: Can you provide the KPN EAI Common library (Common_2.2.2.projlib), including Base.xsd, ResultStatus.xsd, Errors.xsd, EAIMessages.xml and the source of the CommonXPathFunctions Java class? (The message header and status schemas, the transport to the back-end adapters, the exception-queue handler and the custom helper functions all live in this library, and it was not included in the export. The three shared message profiles and the error handler cannot be completed without it.)
  - If this is not answered: Header, status and error structures are reconstructed from how the CRMServices mappings use them; they are validated against real messages captured from the current system before go-live.

REVIEW-NEEDED: How should the migrated integrations reach UCMAdapter for AddAccountReference, now that TIBCO Rendezvous will not be available to Boomi? (Today the call goes over TIBCO Rendezvous request/reply, which Boomi cannot connect to. The choice decides how the UCM AccountReference ADD operation and its message formats are built, and the same decision applies to every back-end call in the project.)
  - If this is not answered: EMS (JMS) request/reply to the same subject name, with an RV-to-EMS bridge or the adapter re-pointed to EMS by the client's middleware team.

REVIEW-NEEDED: Which mail server should the migrated error notifications use, and who should receive them (To, CC, BCC) in each environment? (SMTP host, port, authentication, TLS and the From/To/CC/BCC lists are unknown: the mail settings file was not included in the export. Without them the mail connection and distribution list cannot be configured.)
  - If this is not answered: Notifications are built against a placeholder SMTP connection and an empty distribution table that must be filled in before testing.

REVIEW-NEEDED: Should error e-mails keep including the full failing message, which contains contact personal data, or only the message ID, error code and description? (Error mails currently embed the full failing message (contact personal data). The answer decides the notification content in all four business interfaces.)
  - If this is not answered: Mails carry the service name, message ID, error code and error description only; the full message stays in the exception queue and in Boomi process reporting.

REVIEW-NEEDED: What is the name and expected message format of the enterprise exception queue that failed CRMServices messages are sent to, and which team or tool reads it? (The message format and destination of the exception queue, and whether failed messages are also rolled back or redelivered, are defined in the missing Common library. The Boomi error handler must produce records the existing exception tooling accepts.)
  - If this is not answered: Failures are sent to a queue named from the environment prefix plus a configurable name, in a simple XML record holding the error codes, message ID, original body and JMS properties. Downstream tooling may need adjusting.

REVIEW-NEEDED: Does any system or support process depend on the specific CRMServices and EAI error codes (for example CRMS-03021 'Backend replied with error' or EAI-03004 for a timeout)? (The error codes produced by the transport on timeout (EAI-03004) and the engine timeout code (BWENGINE-100067) are framework codes, and the retry rules of the contact-publication flow depend on them. Keeping the same codes keeps those rules and any monitoring working.)
  - If this is not answered: The existing CRMS codes are kept, and a Boomi timeout is reported as EAI-03004 so that retry rules behave as today. BWENGINE codes are not reproduced.

REVIEW-NEEDED: Do any systems that read CRMServices messages rely on the exact format of the tracking timestamp or the 'Operations/CRMServices' application tag in the message header? (The exact output format of the TIBCO timestamp function must be matched so that consumers parsing the header timestamp are unaffected; the format cannot be determined from the export.)
  - If this is not answered: ISO 8601 date-time with milliseconds and time-zone offset is used, and the application tag is kept as 'Operations/CRMServices'.

REVIEW-NEEDED: Which EMS server addresses, and which dedicated service account, should the Boomi runtime use in each environment (development, test, production)? (The export holds only localhost defaults and a generic account name. Real values are needed to set up the shared EMS connection, and a least-privilege account is recommended.)
  - If this is not answered: The connection is built with environment extensions left empty, to be filled in by the client's EMS administrators at deployment.

REVIEW-NEEDED: Should traffic between Boomi and EMS be encrypted (SSL/TLS)? (An SSL address is defined in the configuration, but the defaults use unencrypted connections and nothing shows which one production uses.)
  - If this is not answered: The same transport as today's production configuration is used once the EMS administrators confirm it; otherwise SSL is recommended.

REVIEW-NEEDED: Roughly how many failures, and how many new account or contact references, do the CRMServices integrations handle per day, and what is the busiest hour? (No volumes, rates or SLAs are stated anywhere in the export; they drive runtime sizing and the concurrency of the shared components.)
  - If this is not answered: Low volume (thousands per day or fewer), served by a single Boomi runtime with the same concurrency as today (10 sessions per listener).

