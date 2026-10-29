# Migration Plan — CRMServices.GetAccountData (1054)

**Project:** Demo-TIBCO-CMRService-Project  
**Source platform:** TIBCO  
**Generated:** 2026-10-29T05:02:49.410Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | CRMServices.GetAccountData (1054) |
| Identifier | crm-getaccountdata-rv |
| Trigger | Listener |
| Starts with | JMS |
| Complexity | complex (72) |
| Components to build | 13 |
| Source artefacts | legacy/tibco/CRMServices/CRMServices/Processes/Reply_GetAccountdataRV.process, legacy/tibco/CRMServices/CRMServices/Processes/Main_GetAccountdata.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountHierarchy.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/RecursiveMappers/SiebelAccountMapper.1.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/RecursiveMappers/SiebelAccountMapper.2.process, legacy/tibco/CRMServices/XSDSchemas/CRMServices/Request/GetAccountData.xsd, legacy/tibco/CRMServices/XSDSchemas/CRMServices/Response/GetAccountData.xsd, legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Request/GetAccountHierarchy.xsd, legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Response/GetAccountHierarchy.xsd |
| Depends on | crm-shared-framework |

## 2. End-to-End Process Architecture

Four BW layers. Reply_GetAccountdataRV is the RV starter: trace, reply and error reply. Main_GetAccountdata parses and validates the request, builds error replies and sends the error mail. Sub_GetAccountdata orchestrates the UCM call, the status check, the recursive conversion loop and the publish. Sub_GetAccountHierarchy makes the request/reply call via Common Sub_Requester and parses the reply. The tree conversion uses SiebelAccountMapper.1 and .2 calling each other. Publishing uses Common Sub_Publisher. Logging and error trace use Common Sub_Trace and Sub_ErrorTrace, gated by the custom Java eai:isTraceOn.

One Boomi listener process, 'CRM AccountData GET Listener' (gad-main-process), in folder CRMServices/GetAccountData, running in low-latency mode with simultaneous executions allowed. Shapes: JMS listen (gad-listen-op), then Set Properties (DPP_MessageId, reply destination, correlation ID), then Try. Inside the Try: map to the UCM request, the UCM request/reply connector, a Decision on STATUS, the Groovy recursive conversion, the EMS publish, the success map and the JMS reply. Catch: a Decision on error type, Process Call to 'Sub: CRM Send Error Mail' (crm-shared-framework) except for parse errors, the error-response map, then the JMS reply. Shared EMS connection and CMH function come from crm-shared-framework.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| Reply_GetAccountdataRV RV Subscriber + RV Reply | gad-listen-op + gad-reply-op (JMS, assumed) | Transport decision required. |
| Main_GetAccountdata Parse Request | Profile on the listener; Try/Catch | Optional XSD validation script. |
| Sub_GetAccountdata input mapping | gad-req-to-ucm-map |  |
| Sub_GetAccountHierarchy (Sub_Requester + Parse XML) | gad-ucm-hierarchy-op connector shape | Timeout 5000 ms. |
| STATUS != 0 transition + Generate-Error-1 | Decision + Exception |  |
| Group inputLoop + SiebelAccountMapper.1/.2 | gad-hierarchy-recursive-fn (Data Process, Groovy) | Whole-document recursion. |
| Common Sub_Publisher | gad-publish-op (JMS topic send) |  |
| Sub_GetAccountdata return bindings | gad-ucm-to-response-map |  |
| Main_GetAccountdata Throw Exception (3 variants) | gad-error-response-map in the Catch path |  |
| SendMail call | Process Call shared-sub-send-error-mail | crm-shared-framework. |
| Sub_Trace / Sub_ErrorTrace | shared-sub-error-handler / process logging | crm-shared-framework. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive request | RV Subscriber on %%Env%%.FIXED.CRM.%%CRMServices/ComponentId%%.Request.GetAccountData.1 via RVCMQ transport. The body is the XML text in RV field 'xml'; the reply subject is captured. | legacy/tibco/CRMServices/CRMServices/Processes/Reply_GetAccountdataRV.process |
| 2 | Trace request (optional) | If eai:isTraceOn('EAI-04005') is true, log the send subject, reply subject and body via Common Sub_Trace (MsgId from eai:getMessageId). | legacy/tibco/CRMServices/CRMServices/Processes/Reply_GetAccountdataRV.process |
| 3 | Parse and validate request | Parse the text as GetAccountDataRequest with validateOutput = true. | legacy/tibco/CRMServices/CRMServices/Processes/Main_GetAccountdata.process |
| 4 | Build UCM request | Build GetAccountHierarchyRequest: CMH ID '1054', NAME 'GetAccountHierarchy', MID '1054.20', DESTINATION from Env and UCMAdapter ComponentID; BODY/SIEBEL_ACCOUNT_ID copied. Main_GetAccountdata first builds an intermediate CMH with ID '1055', which Sub_GetAccountdata replaces. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process |
| 5 | Call UCM GetAccountHierarchy | Common Sub_Requester request/reply, timeout CRMServices/RV/Timeout (5000 ms); reply parsed and validated as GetAccountHierarchyResponse. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountHierarchy.process |
| 6 | Check UCM status | If STATUS_RESULT/@STATUS != 0, raise a ResultStatus fault (message = ERROR_DESCRIPTION, messageCode = ERROR_CODE) and stop. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process |
| 7 | Convert hierarchy | Loop over BODY/SiebelAccount; for each, call SiebelAccountMapper.1, which copies the account and recurses into child SiebelAccounts via SiebelAccountMapper.2. Outputs are accumulated. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/RecursiveMappers/SiebelAccountMapper.1.process |
| 8 | Publish hierarchy | Common Sub_Publisher to concat(Env,'.FIXED.Operations.',ComponentId,'.Publish.UpsertAccountWithReferences.1') with UpsertAccountWithReferencingPublish (CMH ID '1054', MID '1054.10', PARADIGM 'PUBLISH', OBJECT 'SIEBEL_ACCOUNT_HIERARCHY'; BODY/RecursiveProcessing = true). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process |
| 9 | Build response | GetAccountDataResponse: CMH from the UCM response (NAME 'GetAccountData', PARADIGM 'RESPONSE', TO = reply destination) and STATUS_RESULT copied from UCM. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process |
| 10 | Trace and reply | Optional trace (EAI-04006), then RV reply with tib:render-xml(ServiceOut/Body/*) in field 'xml'. | legacy/tibco/CRMServices/CRMServices/Processes/Reply_GetAccountdataRV.process |
| 11 | Error path | On an implementation failure, send mail 'Error in GetAccountData', then throw ErrorReply. The starter logs the error (Sub_ErrorTrace) and sends ErrorReply/Body as the reply when present; otherwise it sends no reply. | legacy/tibco/CRMServices/CRMServices/Processes/Main_GetAccountdata.process |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| CRM AccountData GET Req XML Prof | xml | GetAccountDataRequest/CMH | complex (Base.xsd:Base) | yes |
| CRM AccountData GET Req XML Prof | xml | GetAccountDataRequest/BODY/SIEBEL_ACCOUNT_ID | xs:string | yes |
| CRM AccountData GET Resp XML Prof | xml | GetAccountDataResponse/CMH | complex (Base.xsd:Base) | yes |
| CRM AccountData GET Resp XML Prof | xml | GetAccountDataResponse/STATUS_RESULT | ResultStatus.xsd:ResultStatus | yes |
| UCM AccountHierarchy GET Req XML Prof | xml | GetAccountHierarchyRequest/CMH | complex (Base.xsd:Base) | yes |
| UCM AccountHierarchy GET Req XML Prof | xml | GetAccountHierarchyRequest/BODY/SIEBEL_ACCOUNT_ID | xs:string | yes |
| UCM AccountHierarchy GET Resp XML Prof | xml | GetAccountHierarchyResponse/CMH | complex (Base.xsd:Base) | yes |
| UCM AccountHierarchy GET Resp XML Prof | xml | GetAccountHierarchyResponse/STATUS_RESULT | ResultStatus.xsd:ResultStatus | yes |
| UCM AccountHierarchy GET Resp XML Prof | xml | GetAccountHierarchyResponse/BODY | complex | no |
| UCM AccountHierarchy GET Resp XML Prof | xml | BODY/SiebelAccount | TSiebelAccount, 0..unbounded | no |
| UCM AccountHierarchy GET Resp XML Prof | xml | SiebelAccount/{AccountHierarchyLevel ... UUID} (the MR-010 list plus DisableMailings and MasterAccountIntegrationId) | xs:string, each 0..1 | no |
| UCM AccountHierarchy GET Resp XML Prof | xml | SiebelAccount/Account_Affiliation/{AffiliationId, AffiliationIntegrationId, AffiliationType, AffiliationUUID} | xs:string; parent 0..unbounded | no |
| UCM AccountHierarchy GET Resp XML Prof | xml | SiebelAccount/Contact/{BirthDate, EmailAddress, FaxPhone, FirstName, IntegrationId, KPNAcademicTitle, LastName, MiddleName, PrefferedName, Suffix, SuppressAllEmails, UCMRowId, UUID, WorkPhone} | xs:string; parent 0..unbounded | no |
| UCM AccountHierarchy GET Resp XML Prof | xml | SiebelAccount/BusinessAddress/{AddressDescription, AddressName, AddressType, City, Country, FuturePrimaryFlg, KPNEndDate, KPNPrimaryPOBoxFlag, KPNSpecialAddrDescription, KPNStartDate, KPNZipcodeSuffix, PostalCode, State, Status, StreetAddress, StreetNumber, StreetNumberExtension}, @IsPrimaryMVG | xs:string; parent 0..unbounded | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | CMH/ID | constant | hardcoded |  | '1054' | low | Main_GetAccountdata's intermediate CMH uses '1055'; this value is what is sent. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | GetAccountDataRequest to GetAccountHierarchyRequest |
| MR-002 | CMH/NAME | constant | hardcoded |  | 'GetAccountHierarchy' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | GetAccountDataRequest to GetAccountHierarchyRequest |
| MR-003 | CMH/MID, PARADIGM, OBJECT, VERSION | constants | hardcoded |  | '1054.20', 'REQUEST', '', '1.0' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | GetAccountDataRequest to GetAccountHierarchyRequest |
| MR-004 | CMH/FROM/@COMPONENT_ID, @COMPONENT_VERSION | CRMServices/ComponentId, ComponentVersion | hardcoded |  | process properties | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | GetAccountDataRequest to GetAccountHierarchyRequest |
| MR-005 | CMH/TO/@DESTINATION | Env, CRMServices/UCMAdapter/ComponentID | calculated |  | concat(Env,'.FIXED.CRM.',UCMAdapter/ComponentID,'.Request.GetAccountHierarchy.1') | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | GetAccountDataRequest to GetAccountHierarchyRequest |
| MR-006 | CMH/TRACKING/MESSAGE_ID, EXTERNAL_REF | GetAccountDataRequest/CMH/TRACKING/MESSAGE_ID, EXTERNAL_REF | transformed |  | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | GetAccountDataRequest to GetAccountHierarchyRequest |
| MR-007 | CMH/TRACKING/ITEM | GetAccountDataRequest/CMH/TRACKING/ITEM | looped |  | copy all + append ITEM (APPINFO concat(ComponentDomain,'/',ComponentId), EVENT 'GetAccountDataRequest', TIMESTAMP tib:timestamp()) | low | Uses shared-cmh-header-fn. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | GetAccountDataRequest to GetAccountHierarchyRequest |
| MR-008 | CMH/ATTRIBUTE | GetAccountDataRequest/CMH/ATTRIBUTE | looped |  | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | GetAccountDataRequest to GetAccountHierarchyRequest |
| MR-009 | BODY/SIEBEL_ACCOUNT_ID | GetAccountDataRequest/BODY/SIEBEL_ACCOUNT_ID | transformed |  | copy | low | Mandatory in both schemas. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | GetAccountDataRequest to GetAccountHierarchyRequest |
| MR-010 | SiebelAccount/{AccountHierarchyLevel, AccountStatus, AccountTypeCode, Alias, CAUpdateFlag, CAUpdateType, CSN, DontCall, DontShare, HomePage, Id, IntegrationId, KPNBlockedTelephone, KPNBranchCode, KPNBranchFlag, KPNCOCNR, KPNCODRV, KPNFinancialStatus, KPNFraudIndicator, KPNMoveCKRnrOld, KPNMoveIndicator, KPNRELNUM, KPNSalesDistrictCode, Location, MainEmailAddress, MainFaxNumber, MainPhoneNumber, MarketClass, MarketType, Name, ParentAccountId, ParentAccountIntegrationId, PrimaryContactUCMRowId, Type, UCMRowId, UUID} | same-named TSiebelAccount elements | conditional | copied only if present in source |  | low | Namespace changes from the 0419 Response to the 0420 Publish namespace. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/RecursiveMappers/SiebelAccountMapper.1.process | UCM hierarchy to publication (recursive) |
| MR-011 | SiebelAccount/DisableMailings, MasterAccountIntegrationId | same-named elements | transformed | always written (empty element if absent) |  | low | Reproduce the empty-element behaviour unless the client says otherwise. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/RecursiveMappers/SiebelAccountMapper.1.process | UCM hierarchy to publication (recursive) |
| MR-012 | SiebelAccount/Account_Affiliation/{AffiliationId, AffiliationIntegrationId, AffiliationType, AffiliationUUID} | Account_Affiliation | looped |  | for-each; all four fields always written | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/RecursiveMappers/SiebelAccountMapper.1.process | UCM hierarchy to publication (recursive) |
| MR-013 | SiebelAccount/Contact/{BirthDate, EmailAddress, FaxPhone, FirstName, IntegrationId, KPNAcademicTitle, LastName, MiddleName, PrefferedName, Suffix, SuppressAllEmails, UCMRowId, UUID, WorkPhone} | Contact | looped |  | for-each; fields always written | low | Contact/OCRMOperation exists in the target but is not mapped (no source). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/RecursiveMappers/SiebelAccountMapper.1.process | UCM hierarchy to publication (recursive) |
| MR-014 | SiebelAccount/BusinessAddress/@IsPrimaryMVG | BusinessAddress/@IsPrimaryMVG | conditional | attribute present |  | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/RecursiveMappers/SiebelAccountMapper.1.process | UCM hierarchy to publication (recursive) |
| MR-015 | SiebelAccount/BusinessAddress/{AddressDescription, AddressName, AddressType, City, Country, FuturePrimaryFlg, KPNEndDate, KPNPrimaryPOBoxFlag, KPNSpecialAddrDescription, KPNStartDate, KPNZipcodeSuffix, PostalCode, State, Status, StreetAddress, StreetNumber, StreetNumberExtension} | BusinessAddress | looped |  | for-each; fields always written | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/RecursiveMappers/SiebelAccountMapper.1.process | UCM hierarchy to publication (recursive) |
| MR-016 | SiebelAccount/SiebelAccount (children) | SiebelAccount/SiebelAccount | looped | recursive, any depth | self-call via SiebelAccountMapper.2; accumulated output copied | high | Groovy recursion in Boomi. Child order is preserved. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/RecursiveMappers/SiebelAccountMapper.2.process | UCM hierarchy to publication (recursive) |
| MR-017 | SiebelAccount/OCRMOperation, SiebelAccount/CifAccountReference | none | hardcoded |  | not mapped (left absent) | low | Present in the 0420 target type but absent from the 0419 source type and not written by the mapper. | legacy/tibco/CRMServices/XSDSchemas/CRMServices/Imported/0420_UpsertAccountWithReferencing_1.0_Publish_EAI_BUS_SIDE.xsd | UCM hierarchy to publication (recursive) |
| MR-018 | CMH/ID, NAME, MID, PARADIGM, OBJECT, VERSION | constants | hardcoded |  | '1054', 'UpsertAccountWithReferences', '1054.10', 'PUBLISH', 'SIEBEL_ACCOUNT_HIERARCHY', '1.0' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | Publication envelope |
| MR-019 | CMH/TO/@DESTINATION | Env, ComponentId | calculated |  | concat(Env,'.FIXED.Operations.',ComponentId,'.Publish.UpsertAccountWithReferences.1') | low | Same value as the publish destination. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | Publication envelope |
| MR-020 | CMH/TRACKING | GetAccountHierarchyResponse/CMH/TRACKING | transformed | EXTERNAL_REF only if present | copy MESSAGE_ID, EXTERNAL_REF, ITEMs + append ITEM EVENT 'UpsertAccountWithReferencing' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | Publication envelope |
| MR-021 | BODY/RecursiveProcessing | constant | hardcoded |  | true() | low | Disables the Residential filter downstream. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | Publication envelope |
| MR-022 | BODY/SiebelAccount | accumulated output of MAP-002 | looped |  | copy | medium |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | Publication envelope |
| MR-023 | JMS message id hint (PublisherIn/MessageId) | GetAccountDataRequest/CMH/TRACKING/MESSAGE_ID | transformed |  | copy | low | How Sub_Publisher uses it (JMSCorrelationID or a property) is not in export. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | Publication envelope |
| MR-024 | CMH/ID, NAME, MID, PARADIGM, VERSION | constants | hardcoded |  | '1054', 'GetAccountData', '1054.20', 'RESPONSE', '1.0' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | UCM response to GetAccountDataResponse |
| MR-025 | CMH/FID | GetAccountHierarchyResponse/CMH/FID | conditional | present |  | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | UCM response to GetAccountDataResponse |
| MR-026 | CMH/OBJECT, CMH/FROM | GetAccountHierarchyResponse/CMH/OBJECT, FROM | transformed |  | copy (FROM is UCM's, not CRMServices') | low | Unusual: the reply carries the adapter's FROM. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | UCM response to GetAccountDataResponse |
| MR-027 | CMH/TO/@DESTINATION | ReplyDestination (RV replySubject) | transformed |  | copy | low | JMSReplyTo in the target design. | legacy/tibco/CRMServices/CRMServices/Processes/Reply_GetAccountdataRV.process | UCM response to GetAccountDataResponse |
| MR-028 | CMH/TRACKING | GetAccountHierarchyResponse/CMH/TRACKING | transformed |  | copy MESSAGE_ID, EXTERNAL_REF (if present), ITEMs, SEQUENCE_NO + append ITEM EVENT 'GetAccountDataResponse' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | UCM response to GetAccountDataResponse |
| MR-029 | STATUS_RESULT/@STATUS | GetAccountHierarchyResponse/STATUS_RESULT/@STATUS | transformed |  | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | UCM response to GetAccountDataResponse |
| MR-030 | STATUS_RESULT/@MSG_CODE, @ERROR_CODE, @ERROR_DESCRIPTION | same attributes on UCM STATUS_RESULT | conditional | each copied only when present |  | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process | UCM response to GetAccountDataResponse |
| MR-031 | CMH/ID, NAME, MID, PARADIGM, OBJECT | constants | hardcoded | all variants | '1054', 'GetAccountData', '1054.20', 'RESPONSE', '' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/Main_GetAccountdata.process | Error to GetAccountDataResponse |
| MR-032 | CMH/VERSION | constant | conditional | parse error and other: '1.0'; implementation error: '' |  | low |  | legacy/tibco/CRMServices/CRMServices/Processes/Main_GetAccountdata.process | Error to GetAccountDataResponse |
| MR-033 | CMH/TRACKING/MESSAGE_ID | eai1:getMessageId(raw request) or parsed MESSAGE_ID | conditional | implementation error uses the parsed request; others use the raw text | custom Java -> profile read / regex on raw text | medium |  | legacy/tibco/CRMServices/CRMServices/Processes/Main_GetAccountdata.process | Error to GetAccountDataResponse |
| MR-034 | CMH/TRACKING/ITEM(new)/@EVENT | constant | conditional | 'other' variant uses 'GetAccountDateRespone'; others 'GetAccountDataResponse' |  | low | Typo in source. | legacy/tibco/CRMServices/CRMServices/Processes/Main_GetAccountdata.process | Error to GetAccountDataResponse |
| MR-035 | STATUS_RESULT (parse error) | constants | hardcoded | request parse failed | STATUS 1, ERROR_CODE '1', ERROR_DESCRIPTION 'Technical Error' | low | Errors/MsgCode CRMS-03011, Params = raw request. | legacy/tibco/CRMServices/CRMServices/Processes/Main_GetAccountdata.process | Error to GetAccountDataResponse |
| MR-036 | STATUS_RESULT (implementation error) | fault ResultStatus | conditional | fault carries ResultStatus (UCM functional error) | copy STATUS, MSG_CODE, ERROR_CODE, ERROR_DESCRIPTION if present; else STATUS 1 + 'General Un-Attributable error' | medium | Errors copied, else CRMS-03000 with Params = ERROR_DESCRIPTION. | legacy/tibco/CRMServices/CRMServices/Processes/Main_GetAccountdata.process | Error to GetAccountDataResponse |
| MR-037 | STATUS_RESULT (other) | constants | hardcoded | any other error | STATUS 1, MSG_CODE 'CRMS-03011', ERROR_CODE '1', ERROR_DESCRIPTION 'Technical Error' | low | Errors/MsgCode CRMS-03000 + ErrorReport. | legacy/tibco/CRMServices/CRMServices/Processes/Main_GetAccountdata.process | Error to GetAccountDataResponse |
| MR-038 | Error mail Subject / Body | error, parsed request | calculated | implementation error | Subject 'Error in GetAccountData'; Body concat('Location :GetAccountData', first Errors/MsgCode else ResultStatus/@ERROR_DESCRIPTION, 'Message Causing Error is : ', render-xml(request)) | low | Sent via crm-shared-framework. | legacy/tibco/CRMServices/CRMServices/Processes/Main_GetAccountdata.process | Error to GetAccountDataResponse |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Publish only on UCM success | If GetAccountHierarchyResponse/STATUS_RESULT/@STATUS != 0, raise a functional error; nothing is published. Otherwise convert and publish. | High | Decision on STATUS; true path to an Exception carrying ResultStatus into the error path. |
| BR-002 | Status-only reply | The reply contains CMH and STATUS_RESULT copied from UCM; no account data. | High | gad-ucm-to-response-map. |
| BR-003 | Recursive processing flag | Publications from GetAccountData always carry RecursiveProcessing = true, so every account is sent to SFA downstream regardless of Type = 'Residential'. | High | Constant in the publication map. |
| BR-004 | Publish before reply | The publish completes before the reply is built. A publish failure becomes an error reply; a reply failure after a successful publish still leaves the publication in place. | Medium | Sequential branch: publish first, then build and send the reply. |
| BR-005 | Error mail on implementation failure | Any failure after successful parsing (UCM functional or technical error, publish error) sends 'Error in GetAccountData' before the error reply. Parse failures do not send mail. | Medium | Process Call 'Sub: CRM Send Error Mail' (crm-shared-framework) in the Catch path. |
| BR-006 | Request must be schema-valid | Request parsed with validation; failure produces CRMS-03011 'Technical Error'. | Medium | Profile parse inside Try; optional Groovy XSD validation. |
| BR-007 | Reply only when an error body exists | On error, an RV error reply is sent only if the ErrorReply carries a Body; otherwise no reply is sent. | Low | Decision in the top-level Catch. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound TIBCO Rendezvous request/reply (RVCMQ) | %%Env%%.FIXED.CRM.%%CRMServices/ComponentId%%.Request.GetAccountData.1 (default XDVL.FIXED.CRM.CRMServices.Request.GetAccountData.1) | none visible (network and subject segregation) | legacy/tibco/CRMServices/CRMServices/Processes/Reply_GetAccountdataRV.process |
| outbound TIBCO Rendezvous request/reply via Common Sub_Requester | concat(Env,'.FIXED.CRM.',CRMServices/UCMAdapter/ComponentID,'.Request.GetAccountHierarchy.1') (default XDVL.FIXED.CRM.UCMAdapter.Request.GetAccountHierarchy.1) | none visible | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountHierarchy.process |
| outbound JMS topic (TIBCO EMS) via Common Sub_Publisher | concat(Env,'.FIXED.Operations.',ComponentId,'.Publish.UpsertAccountWithReferences.1') | EMS username/password (shared connection) | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process |
| RV Subscriber (com.tibco.plugin.tibrv.RVEventSource) and RV Reply | sharedChannel /Common/Shared Resources/RVCMQ Transport.rvtransport (not in export); wantsValidationForOutput true; body schema RawRVMsg2.xsd (not in export), field 'xml' of type tibrvXml. | — | legacy/tibco/CRMServices/CRMServices/Processes/Reply_GetAccountdataRV.process |
| Common Sub_Requester (request/reply) | Destination = CMH/TO/@DESTINATION; Timeout = CRMServices/RV/Timeout; Body = request. | — | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountHierarchy.process |
| Common Sub_Publisher | Destination, MessageId (request MESSAGE_ID), Body ([redacted]). | — | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| crm-shared-framework | Sub: CRM Send Error Mail | synchronous |
| crm-shared-framework | Sub: CRM Error Handler (replaces Sub_ErrorTrace) | synchronous |
| crm-upsert-account-with-references | Consumer of the published hierarchy | asynchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/tibco/CRMServices/CRMServices/Processes/Reply_GetAccountdataRV.process | eai:getMessageId(text) and eai:isTraceOn(code) from com.kpn.eai.component.common.CommonXPathFunctions, used for trace gating and MsgId in logs and error replies (also as eai1:getMessageId in Main_GetAccountdata). | shared-cmh-header-fn (crm-shared-framework) and a trace process property. | Medium |
| xslt | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/RecursiveMappers/SiebelAccountMapper.1.process | Recursive tree copy implemented as two BW processes calling each other (BW cannot self-call); output accumulated from an inputLoop group. | gad-hierarchy-recursive-fn: Groovy recursive copy (or XSLT run from Groovy) over the whole document. | High |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Layered BW error transitions. Sub_GetAccountHierarchy and Sub_GetAccountdata raise Errors or ResultStatus faults. Main_GetAccountdata sends an error mail and throws an ErrorReply containing a ready-made error response body. Reply_GetAccountdataRV logs and replies with that body. In Boomi: one Try/Catch around the implementation, a Decision on error type, error mail via crm-shared-framework, then the error reply. |
| Retries | None. A synchronous caller decides whether to retry. |
| Compensation | None. If the reply fails after a successful publish, the synchronisation still proceeds. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| /Common/Shared Resources/RVCMQ Transport.rvtransport | other | no |
| Common Sub_Requester / Sub_Publisher / Sub_Trace / Sub_ErrorTrace | library | no |
| XSDSchemas/Common/Base.xsd, ResultStatus.xsd; /Common/Shared Resources/Errors.xsd, RawRVMsg2.xsd | other | no |
| com.kpn.eai.component.common.CommonXPathFunctions | library | no |
| CRMServicesMessages.xml | crossref | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| UCMAdapter GetAccountHierarchy | TIBCO Rendezvous request/reply (target TBD) | %%Env%%.FIXED.CRM.UCMAdapter.Request.GetAccountHierarchy.1 | Not stated in export |
| EMS topic Publish.UpsertAccountWithReferences.1 | JMS topic | %%Env%%.FIXED.Operations.CRMServices.Publish.UpsertAccountWithReferences.1 | KPN EAI (EMS) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| DEV | Env | XDVL (design-time default; environment it denotes not stated) | [redacted].substvar |

## 14. Boomi Migration Readiness Summary

GetAccountData is the starting point of the account synchronisation. A calling application sends a Siebel account ID. CRMServices asks UCM, through UCMAdapter, for the account's whole hierarchy (the account, its child accounts at any depth, their contacts, affiliations and business addresses), converts it into the CRMServices publication format, and publishes it on the message bus for the UpsertAccountWithReferences flow to process. The caller receives a reply that reflects UCM's status: success, or UCM's error code and description.

If UCM reports an error, nothing is published and the caller receives UCM's error. A technical failure (bad request, timeout, publishing error) produces a 'Technical Error' reply and an e-mail to integration support.

Three things make this interface complex to move to Boomi. The service and its call to UCM both run over TIBCO Rendezvous, which Boomi cannot connect to, so a transport decision is needed first. The account hierarchy is recursive (accounts contain accounts) and must be converted by a script, because Boomi maps cannot recurse. And the message header and status schemas, plus the helper code that reads message IDs, are in a TIBCO library that was not exported.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | CRM AccountData GET Req XML Prof | gad-request-prof | profile.xml | — |
| 2 | CRM AccountData GET Resp XML Prof | gad-response-prof | profile.xml | — |
| 3 | UCM AccountHierarchy GET Req XML Prof | gad-ucm-hierarchy-req-prof | profile.xml | — |
| 4 | UCM AccountHierarchy GET Resp XML Prof | gad-ucm-hierarchy-resp-prof | profile.xml | — |
| 5 | EMS CRM AccountData GET LISTEN Op | gad-listen-op | connector-action | shared-ems-jms-conn, gad-request-prof |
| 6 | EMS CRM AccountData GET REPLY Op | gad-reply-op | connector-action | shared-ems-jms-conn, gad-response-prof |
| 7 | EMS CRM UpsertAccountWithReferences PUBLISH Op | gad-publish-op | connector-action | shared-ems-jms-conn, shared-crm-uawr-publish-prof |
| 8 | UCM AccountHierarchy GET Op | gad-ucm-hierarchy-op | connector-action | shared-ems-jms-conn, gad-ucm-hierarchy-req-prof, gad-ucm-hierarchy-resp-prof |
| 9 | CRM SiebelAccount Hierarchy Recursive Mapper | gad-hierarchy-recursive-fn | transform.function | gad-ucm-hierarchy-resp-prof, shared-crm-uawr-publish-prof |
| 10 | CRM AccountData GET Req XML - UCM AccountHierarchy GET Req XML | gad-req-to-ucm-map | transform.map | gad-request-prof, gad-ucm-hierarchy-req-prof, shared-cmh-header-fn |
| 11 | CRM Error - CRM AccountData GET Resp XML | gad-error-response-map | transform.map | gad-response-prof, shared-cmh-header-fn |
| 12 | UCM AccountHierarchy GET Resp XML - CRM AccountData GET Resp XML | gad-ucm-to-response-map | transform.map | gad-ucm-hierarchy-resp-prof, gad-response-prof, shared-cmh-header-fn |
| 13 | CRM AccountData GET Listener | gad-main-process | process | gad-listen-op, gad-reply-op, gad-ucm-hierarchy-op, gad-req-to-ucm-map, gad-hierarchy-recursive-fn, gad-publish-op, gad-ucm-to-response-map, gad-error-response-map, shared-sub-send-error-mail, shared-sub-error-handler |

## 15. Appendix

- legacy/tibco/CRMServices/CRMServices/Processes/Reply_GetAccountdataRV.process
- legacy/tibco/CRMServices/CRMServices/Processes/Main_GetAccountdata.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountHierarchy.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/RecursiveMappers/SiebelAccountMapper.1.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/RecursiveMappers/SiebelAccountMapper.2.process
- legacy/tibco/CRMServices/CRMServices/Processes/Subscribe_UpsertAccountWithReferencesJMS.process
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Request/GetAccountData.xsd
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Response/GetAccountData.xsd
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Imported/0418_GetAccountData_1.0_Request1_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Request/GetAccountHierarchy.xsd
- legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Response/GetAccountHierarchy.xsd
- legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Imported/0419_GetAccountHierarchy_1.0_Request1_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/UCMAdapter/Imported/0419_GetAccountHierarchy_1.0_Response_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Imported/0420_UpsertAccountWithReferencing_1.0_Publish_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/CRMServices/Shared Resources/CRMServicesMessages.xml
- legacy/tibco/CRMServices/defaultVars/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/CRMServices/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/CRMServices/RV/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/CRMServices/UCMAdapter/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/Common/RV/defaultVars.substvar
- legacy/tibco/CRMServices/.designtimelibs
- legacy/tibco/CRMServices/CRMServices/Test Processes/UpsertAccountwithReferences/Driver_FullyFilled Topic.process
- legacy/tibco/CRMServices/CRMServices/Test Processes/GetAccountData/STUB_1_GetAccountHierarchy.process

## Open questions

### Still outstanding

REVIEW-NEEDED: Which applications call GetAccountData today, and can they move from TIBCO Rendezvous to EMS request/reply or to an HTTP API? (Identify the GetAccountData callers and confirm the target inbound transport (EMS request/reply or Boomi API). Boomi cannot receive Rendezvous requests, so the listener and reply cannot be built until this is decided.)
  - If this is not answered: Callers move to an EMS request queue named like today's subject (<Env>.FIXED.CRM.CRMServices.Request.GetAccountData.1) and receive the reply on their JMSReplyTo destination.

REVIEW-NEEDED: How long do callers wait for a GetAccountData reply before giving up, and how many requests per day and per peak hour should we expect? (The service makes a call to UCM (up to 5 seconds) and a publish before replying. Any extra hop added by the new transport must fit inside the callers' timeout, and volumes drive runtime sizing.)
  - If this is not answered: Callers wait at least 10 seconds; volume is low (thousands per day or fewer); the service runs in low-latency mode on a single runtime.

REVIEW-NEEDED: What is the largest account hierarchy (number of accounts and number of levels) that UCM can return for one customer? (Confirm the maximum realistic hierarchy depth and size. The conversion handles any depth, but hierarchies of more than 50 accounts are published here and then rejected by the UpsertAccountWithReferences flow.)
  - If this is not answered: No depth limit is enforced here; a safety limit of 20 levels raises an explicit error; the 50-account limit stays downstream as today.

REVIEW-NEEDED: Should GetAccountData reject hierarchies of more than 50 accounts immediately, instead of publishing them for the UpsertAccountWithReferences flow to reject later? (Decide whether to reject them earlier (in GetAccountData) or keep the current behaviour. Today the caller is told the request succeeded even though the synchronisation is later rejected with CRMS-03001.)
  - If this is not answered: Current behaviour is kept: publish regardless of size; the limit is enforced downstream.

REVIEW-NEEDED: Must the publish of the account hierarchy still happen before the caller receives its reply, so that a failed publish is reported to the caller as an error? (In BW the publish happens before the reply, inside the same job. If the publish fails, the caller gets an error reply. If the reply fails after a successful publish, the synchronisation still happens. Confirm whether the Boomi build must keep 'publish, then reply' ordering.)
  - If this is not answered: The order is kept: publish first, then reply.

REVIEW-NEEDED: Do any callers depend on the exact content of error replies, such as the empty version field or the event name 'GetAccountDateRespone' (misspelt), or may these be corrected? (Confirm whether consumers depend on the empty CMH/VERSION and the misspelt EVENT 'GetAccountDateRespone' in error replies, or whether the Boomi build may normalise them.)
  - If this is not answered: Error replies are reproduced exactly, including the empty version and the misspelling.

REVIEW-NEEDED: When UCM reports an error for an account (for example, account not found), should integration support still receive an error e-mail, as happens today? (Today any UCM error, including business errors, sends an 'Error in GetAccountData' e-mail as well as returning the error to the caller. This may produce e-mails for routine 'not found' cases.)
  - If this is not answered: Current behaviour is kept: every UCM error sends an e-mail.

REVIEW-NEEDED: Should empty 'DisableMailings' and 'MasterAccountIntegrationId' fields still be sent in the published hierarchy when UCM does not supply them? (Today these two fields are always written, even when empty, while every other optional field is omitted when absent. The consumer may treat an empty value differently from a missing one.)
  - If this is not answered: The current behaviour is reproduced: both fields are always written.

REVIEW-NEEDED: Which access control should protect GetAccountData once it is no longer on the Rendezvous network (for example EMS queue permissions or API credentials)? (Today no caller authentication exists in the service; access relied on network and Rendezvous subject segregation.)
  - If this is not answered: EMS queue permissions restrict who can send to the request queue; no message-level authentication is added.

