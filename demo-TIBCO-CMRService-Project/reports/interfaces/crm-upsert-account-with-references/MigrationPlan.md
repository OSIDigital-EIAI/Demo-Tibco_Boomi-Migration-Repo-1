# Migration Plan — CRMServices.UpsertAccountWithReferences (1056)

**Project:** Demo-TIBCO-CMRService-Project  
**Source platform:** TIBCO  
**Generated:** 2026-10-29T05:02:49.480Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | CRMServices.UpsertAccountWithReferences (1056) |
| Identifier | crm-upsert-account-with-references |
| Trigger | Listener |
| Starts with | JMS |
| Complexity | complex (85) |
| Components to build | 15 |
| Source artefacts | legacy/tibco/CRMServices/CRMServices/Processes/Subscribe_UpsertAccountWithReferencesJMS.process, legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertAccountWithReferences.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process, legacy/tibco/CRMServices/CRMServices/Processes/_Subscribe_UpsertAccountWithReferencesJMS.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferences.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process, legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertAccount.process, legacy/tibco/CRMServices/XSDSchemas/CRMServices/Publish/FlatAccountHierarchy.xsd, legacy/tibco/CRMServices/XSDSchemas/SFAAdapter/Request/UpsertAccount.xsd, legacy/tibco/CRMServices/XSDSchemas/SFAAdapter/Response/UpsertAccount.xsd |
| Depends on | crm-shared-framework |

## 2. End-to-End Process Architecture

Two JMS listeners and six processes. Subscribe_UpsertAccountWithReferencesJMS calls Main_UpsertAccountWithReferences (parse), which calls Sub_UpserAccountWithReferencing (flatten, 50-account check, send to the internal queue). _Subscribe_UpsertAccountWithReferencesJMS calls _Sub_UpsertAccountWithReferences (parse, error mail, throw), which calls _Sub_UpsertAccountWithReferencing_RecursivePublish (filter, upsert, remove, republish). That calls _Sub_UpsertAccountIntoSFA_UCM (SFA upsert via Sub_UpsertAccount, then UCM AddAccountReference loops via Sub_AddAccountReference). Iteration over accounts is achieved by the worker republishing to its own queue. Failures in both listeners go to Common Sub_ThrowException, and the message is then acknowledged.

Recommended: one Boomi listener process, 'CRM UpsertAccountWithReferences Listener' (uawr-main-process), in folder CRMServices/UpsertAccountWithReferences. Shapes: listen, Groovy flatten (one document per account), a Decision on count > 50, then Flow Control (1 thread, sequential) with a per-account Decision chain for the Residential filter. Each account goes through Process Call 'Sub: CRM Account UPSERT Into SFA and UCM', inside a Try/Catch that stops at the first failure and passes the remaining accounts to the error handler and mail. If per-account durability is required, keep the two-process design instead: the listener sends to the internal queue, and 'CRM UpsertAccountWithReferences Account Worker' (uawr-worker-process) processes one account and republishes the remainder. The shared EMS connection, AddAccountReference sub-process, error mail and error handler come from crm-shared-framework.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| Subscribe_UpsertAccountWithReferencesJMS | uawr-subscribe-listen-op + uawr-main-process | Listener acknowledges on completion. |
| FlattenAccountHierarchy mapper | uawr-flatten-fn (Groovy) |  |
| count > 50 transition + Generate Error | Decision + Exception |  |
| Sub_NotifierJMS to internal queue | uawr-internal-send-op | Only in the two-process design. |
| _Subscribe_UpsertAccountWithReferencesJMS | uawr-internal-listen-op + uawr-worker-process | Only in the two-process design. |
| SetFilter assign + transitions + Group over SFA reference | Decision shapes |  |
| CallUpsertAccountIntoSFAUCM input mapping | uawr-flat-to-sfa-map |  |
| _Sub_UpsertAccountIntoSFA_UCM | uawr-sub-upsert-into-sfa-ucm |  |
| Sub_UpsertAccount | uawr-sfa-upsertaccount-op + Decision on STATUS |  |
| AddAccountReference inputLoop groups | Data Process split + uawr-sfa-to-ucm-accountref-map / uawr-sfa-to-ucm-contactref-map + Process Call shared-sub-ucm-accountreference-add | Sequential. |
| RemoveProcessedAccount + Republish Message | uawr-remaining-accounts-map + uawr-internal-send-op, or the Flow Control loop |  |
| SendErrorMail + Throw + Sub_ThrowException | Catch: shared-sub-send-error-mail + shared-sub-error-handler | crm-shared-framework. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive publication | JMS Queue Receiver on %%Env%%.FIXED.Operations.%%CRMServices/ComponentId%%.Subscribe.UpsertAccountWithReferences.1; Text; CLIENT_ACKNOWLEDGE; non-transacted; max sessions JmsMaxSessions. | legacy/tibco/CRMServices/CRMServices/Processes/Subscribe_UpsertAccountWithReferencesJMS.process |
| 2 | Trace (optional) | If eai:isTraceOn('EAI-04052') is true, log destination, headers and body via Common Sub_Trace. | legacy/tibco/CRMServices/CRMServices/Processes/Subscribe_UpsertAccountWithReferencesJMS.process |
| 3 | Parse publication | Parse Body as UpsertAccountWithReferencingPublish. | legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertAccountWithReferences.process |
| 4 | Flatten hierarchy | Mapper FlattenAccountHierarchy: CMH rebuilt (ID 1056, NAME '_UpsertAccountWithReferences', MID 1056.20, PUBLISH, OBJECT FLAT_SIEBEL_ACCOUNT_HIERARCHY, TO = internal queue); RecursiveProcessing copied if present; one SiebelAccount per BODY//SiebelAccount. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process |
| 5 | Check size limit | If count(FlatAccountHierarchy/SiebelAccount) > 50, raise Errors CRMS-03001 (MsgId = MESSAGE_ID). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process |
| 6 | Hand off to internal queue | Common Sub_NotifierJMS sends the rendered FlatAccountHierarchy to %%Env%%.FIXED.Operations.CRMServices._UpsertAccountWithReferences.1. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process |
| 7 | Acknowledge publication | Confirm the JMS message on success, and also after an error has been passed to Common Sub_ThrowException. | legacy/tibco/CRMServices/CRMServices/Processes/Subscribe_UpsertAccountWithReferencesJMS.process |
| 8 | Receive flat list (worker) | JMS Queue Receiver on the internal queue (same settings), then parse as FlatAccountHierarchy. | legacy/tibco/CRMServices/CRMServices/Processes/_Subscribe_UpsertAccountWithReferencesJMS.process |
| 9 | Empty list? | If count(SiebelAccount) = 0, trace CRMS-03022 'No SiebelAccount found in received message' with the message, and end. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process |
| 10 | Apply Residential filter | sendAccountThrough = if RecursiveProcessing = 'true' then true else not(SiebelAccount[1]/Type = 'Residential'). If false, process only when SiebelAccount[1] has CifAccountReference[SystemName = 'SFA'] (loop over [1], so at most once). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process |
| 11 | Upsert account into SFA | Build UpserAccountRequest from SiebelAccount[1] and call SFAAdapter UpsertAccount (timeout 5000 ms); STATUS_RESULT/@STATUS != 0 raises CRMS-03021. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertAccount.process |
| 12 | Register account references | For each SFA response SiebelAccount[OCRMOperation = 'Create']/CifAccountReference, call UCM AddAccountReference (IntObjectName UCMAccountExternalID). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process |
| 13 | Register contact references | For each response Contact, then each CifContactReference of a Contact with OCRMOperation = 'Create', call UCM AddAccountReference (IntObjectName UCMContactExternalID). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process |
| 14 | Remove processed account | Copy SiebelAccount[position() > 1] into a new FlatAccountHierarchy, with TRACKING from the last successful back-end call (EVENT '_UpsertAccountWithReferencing'). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process |
| 15 | Republish remainder | If count > 0, send the remainder to $Start CMH/TO/@DESTINATION (the same internal queue), which repeats steps 8 to 15. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process |
| 16 | Worker error path | Mail 'Error in UpserAccountWithReferencing' with all Error Params and the rendered remaining list, then Throw Exception, then Common Sub_ThrowException, then acknowledge. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferences.process |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| CRM FlatAccountHierarchy PUBLISH XML Prof | xml | FlatAccountHierarchy/CMH | complex (Base.xsd:Base) | yes |
| CRM FlatAccountHierarchy PUBLISH XML Prof | xml | FlatAccountHierarchy/RecursiveProcessing | xs:boolean | no |
| CRM FlatAccountHierarchy PUBLISH XML Prof | xml | FlatAccountHierarchy/SiebelAccount | Imported:Type_SiebelAccount (0420), 0..unbounded | no |
| SFA Account UPSERT Req XML Prof | xml | UpserAccountRequest/CMH | complex (Base.xsd:Base) | yes |
| SFA Account UPSERT Req XML Prof | xml | BODY/SiebelAccount | complex, exactly one | yes |
| SFA Account UPSERT Req XML Prof | xml | SiebelAccount/{AccountHierarchyLevel, AccountStatus, AccountTypeCode, Alias, CAUpdateFlag, CAUpdateType, CSN, DisableMailings, DontCall, DontShare, HomePage, Id, IntegrationId, KPNBlockedTelephone, KPNBranchCode, KPNBranchFlag, KPNCOCNR, KPNCODRV, KPNFinancialStatus, KPNFraudIndicator, KPNMoveCKRnrOld, KPNMoveIndicator, KPNRELNUM, KPNSalesDistrictCode, Location, MainEmailAddress, MainFaxNumber, MainPhoneNumber, MarketClass, MarketType, MasterAccountIntegrationId, Name, OCRMOperation, ParentAccountId, ParentAccountIntegrationId, PrimaryContactUCMRowId, Type, UCMRowId, UUID} | xs:string, each 0..1 | no |
| SFA Account UPSERT Req XML Prof | xml | SiebelAccount/Account_Affiliation/{AffiliationId, AffiliationIntegrationId, AffiliationType, AffiliationUUID} | xs:string; 0..unbounded | no |
| SFA Account UPSERT Req XML Prof | xml | SiebelAccount/Contact/{BirthDate, EmailAddress, FaxPhone, FirstName, IntegrationId, KPNAcademicTitle, LastName, MiddleName, OCRMOperation, PrefferedName, Suffix, SuppressAllEmails, UCMRowId, UUID, WorkPhone} | xs:string; 0..unbounded | no |
| SFA Account UPSERT Req XML Prof | xml | SiebelAccount/BusinessAddress/{AddressDescription, AddressName, AddressType, City, Country, FuturePrimaryFlg, KPNEndDate, KPNPrimaryPOBoxFlag, KPNSpecialAddrDescription, KPNStartDate, KPNZipcodeSuffix, PostalCode, State, Status, StreetAddress, StreetNumber, StreetNumberExtension}, @IsPrimaryMVG | xs:string; 0..unbounded | no |
| SFA Account UPSERT Req XML Prof | xml | SiebelAccount/CifAccountReference/{AccountId, CIFSystemId, ExternalId1, SystemName, SystemNumber} | xs:string; 0..unbounded | no |
| SFA Account UPSERT Resp XML Prof | xml | UpserAccountResponse/CMH | complex (Base.xsd:Base) | yes |
| SFA Account UPSERT Resp XML Prof | xml | UpserAccountResponse/STATUS_RESULT | ResultStatus.xsd:ResultStatus | yes |
| SFA Account UPSERT Resp XML Prof | xml | BODY/SiebelAccount | complex | no |
| SFA Account UPSERT Resp XML Prof | xml | SiebelAccount/OCRMOperation | xs:string | no |
| SFA Account UPSERT Resp XML Prof | xml | SiebelAccount/Contact/CifContactReference/{AccountId, CIFSystemId, ExternalId1, SystemName, SystemNumber} | xs:string; 0..unbounded | no |
| SFA Account UPSERT Resp XML Prof | xml | SiebelAccount/CifAccountReference/{AccountId, CIFSystemId, ExternalId1, SystemName, SystemNumber} | xs:string; 0..unbounded | no |
| SFA Account UPSERT Resp XML Prof | xml | Contact/Error, BusinessAddress/Error, CifAccountReference/Error {Code, Description, Type} | xs:string; 0..unbounded | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | CMH/ID, NAME, MID, PARADIGM, OBJECT, VERSION | constants | hardcoded |  | '1056', '_UpsertAccountWithReferences', '1056.20', 'PUBLISH', 'FLAT_SIEBEL_ACCOUNT_HIERARCHY', '1.0' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process | Flatten hierarchy publication |
| MR-002 | CMH/TO/@DESTINATION | Env, ComponentId | calculated |  | concat(Env,'.FIXED.Operations.',ComponentId,'._UpsertAccountWithReferences.1') | low | The worker republishes to this value, so it acts as the iteration address. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process | Flatten hierarchy publication |
| MR-003 | CMH/TRACKING | publication CMH/TRACKING | transformed | EXTERNAL_REF if present | MESSAGE_ID, EXTERNAL_REF, ITEMs copied + new ITEM (APPINFO 'Operations/CRMServices', EVENT '_UpsertAccountWithReferencing') | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process | Flatten hierarchy publication |
| MR-004 | RecursiveProcessing | BODY/RecursiveProcessing | conditional | present | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process | Flatten hierarchy publication |
| MR-005 | SiebelAccount (one per source account) | BODY//SiebelAccount | looped | all depths, document order | descendant-axis for-each | high | Groovy in Boomi. Children are not nested in the output (Type_SiebelAccount). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process | Flatten hierarchy publication |
| MR-006 | SiebelAccount/{AccountHierarchyLevel ... UUID, OCRMOperation} | same-named elements | conditional | each copied if present |  | low | Includes OCRMOperation and DisableMailings, MasterAccountIntegrationId (presence-checked here). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process | Flatten hierarchy publication |
| MR-007 | SiebelAccount/{Account_Affiliation, Contact (incl. OCRMOperation), BusinessAddress (@IsPrimaryMVG), CifAccountReference (AccountId, CIFSystemId, ExternalId1, SystemName, SystemNumber)} | same-named children | looped | fields copied if present | for-each | medium |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process | Flatten hierarchy publication |
| MR-008 | CMH/ID, NAME, MID, PARADIGM, OBJECT, VERSION | constants | hardcoded |  | '1056', 'UpsertAccount', '1056.20', 'REQUEST', 'SIEBEL_ACCOUNT', '1.0' | low | Both call variants (filtered and unfiltered) use identical bindings. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process | Current account to SFA UpsertAccount request |
| MR-009 | CMH/TO/@DESTINATION | Env | calculated |  | concat(Env,'.FIXED.CRM.SFAAdapter.Request.UpsertAccount.1') | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process | Current account to SFA UpsertAccount request |
| MR-010 | CMH/TRACKING | FlatAccountHierarchy/CMH/TRACKING | transformed |  | MESSAGE_ID, EXTERNAL_REF, ITEMs + new ITEM (APPINFO 'Operations/CRMServices', EVENT 'UpsertAccount') | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process | Current account to SFA UpsertAccount request |
| MR-011 | BODY/SiebelAccount (scalar fields incl. OCRMOperation) | FlatAccountHierarchy/SiebelAccount[1] | conditional | fields copied if present |  | low | Only the first account of the list. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process | Current account to SFA UpsertAccount request |
| MR-012 | BODY/SiebelAccount/{Account_Affiliation, Contact (incl. OCRMOperation), BusinessAddress} | SiebelAccount[1] children | looped | fields copied if present | for-each | medium |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process | Current account to SFA UpsertAccount request |
| MR-013 | BODY/SiebelAccount/CifAccountReference | none | hardcoded |  | not mapped | low | The request schema allows it, but the source does not send it; SFA returns references in its response. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process | Current account to SFA UpsertAccount request |
| MR-014 | (iteration) | UpserAccountResponse/BODY/SiebelAccount[OCRMOperation='Create']/CifAccountReference | looped | account created by SFA | one request per reference | medium |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process | SFA response to UCM AddAccountReference (account) |
| MR-015 | CMH/ID, NAME, MID, PARADIGM, OBJECT | constants | hardcoded |  | '1056', 'AddAccountReference', '1056.20', 'REQUEST', 'SIEBEL_ACCOUNT_ID' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process | SFA response to UCM AddAccountReference (account) |
| MR-016 | CMH/FID | SiebelAccount/UUID | conditional | present |  | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process | SFA response to UCM AddAccountReference (account) |
| MR-017 | CMH/TO/@DESTINATION | Env | calculated |  | concat(Env,'.FIXED.CRM.UCMAdapter.Request.AddAccountReference.1') | low | UCMAdapter hard-coded (not the UCMAdapter/ComponentID variable). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process | SFA response to UCM AddAccountReference (account) |
| MR-018 | CMH/ATTRIBUTE IntObjectName | constant | hardcoded |  | 'UCMAccountExternalID' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process | SFA response to UCM AddAccountReference (account) |
| MR-019 | BODY/SIEBEL_ACCOUNT_ID | SiebelAccount/UUID | transformed |  | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process | SFA response to UCM AddAccountReference (account) |
| MR-020 | BODY/EXTENAL_REFERENCE/{REFERENCE, ORIGIN, ORIGIN_ID} | CifAccountReference/{ExternalId1, SystemName, SystemNumber} | transformed |  | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process | SFA response to UCM AddAccountReference (account) |
| MR-021 | (iteration) | BODY/SiebelAccount/Contact, then Contact[OCRMOperation='Create']/CifContactReference | looped | contact created by SFA | nested loop; one request per contact reference | medium |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process | SFA response to UCM AddAccountReference (contact) |
| MR-022 | CMH/FID | Contact/UUID | transformed |  | copy (always written) | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process | SFA response to UCM AddAccountReference (contact) |
| MR-023 | CMH/TRACKING | previous AddAccountReference response TRACKING, else start TRACKING | conditional | an account-reference response exists | copy + new ITEM EVENT 'AddAccountReference' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process | SFA response to UCM AddAccountReference (contact) |
| MR-024 | CMH/ATTRIBUTE IntObjectName | constant | hardcoded |  | 'UCMContactExternalID' | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process | SFA response to UCM AddAccountReference (contact) |
| MR-025 | BODY/SIEBEL_ACCOUNT_ID | Contact/UUID | conditional | present | copy | low | The contact's UUID, not the account's. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process | SFA response to UCM AddAccountReference (contact) |
| MR-026 | BODY/EXTENAL_REFERENCE/{REFERENCE, ORIGIN, ORIGIN_ID} | CifContactReference/{ExternalId1, SystemName, SystemNumber} | transformed |  | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process | SFA response to UCM AddAccountReference (contact) |
| MR-027 | CMH/{ID, NAME, FID, MID, PARADIGM, OBJECT, VERSION, FROM, TO} | input FlatAccountHierarchy CMH | passthrough | FID and MID if present | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process | Remaining accounts (iteration) |
| MR-028 | CMH/TRACKING | Tracking returned by the SFA/UCM sub-process (filtered or unfiltered call) | conditional | whichever call ran | MESSAGE_ID, EXTERNAL_REF, ITEMs + new ITEM EVENT '_UpsertAccountWithReferencing' | medium | xsl:choose with two branches and no otherwise: if the account was skipped by the filter, no TRACKING element is written, so MESSAGE_ID is lost for the rest of the hierarchy (and the republish MessageId is empty). If Base.xsd makes TRACKING mandatory, the next iteration fails to parse. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process | Remaining accounts (iteration) |
| MR-029 | RecursiveProcessing | input RecursiveProcessing | conditional | present | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process | Remaining accounts (iteration) |
| MR-030 | SiebelAccount | SiebelAccount[position() > 1] | looped |  | full field copy incl. children and CifAccountReference | medium |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process | Remaining accounts (iteration) |
| MR-031 | Subject | constant | hardcoded |  | 'Error in UpserAccountWithReferencing' | low | Misspelling as in source. | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferences.process | Worker error mail |
| MR-032 | Body | Errors/Error/Params, FlatAccountHierarchy | calculated |  | concat('Location :UpserAccountWithReferences', tib:concat-sequence-format(Params, newline), 'Message Causing Error is : ', render-xml(FlatAccountHierarchy)) | low | The message rendered is the list received by the failing iteration (failed account plus remaining). | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferences.process | Worker error mail |
| MR-033 | MessageId | FlatAccountHierarchy/CMH/TRACKING/MESSAGE_ID | transformed |  | copy | low |  | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferences.process | Worker error mail |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Maximum 50 accounts per hierarchy | After flattening, if count(SiebelAccount) > 50, reject the whole publication with CRMS-03001 'Account Hierarchy exceeds 50 accounts'. | High | Decision on document count after the flatten script, then Exception. |
| BR-002 | Residential account filter | sendAccountThrough = if RecursiveProcessing = 'true' then true else not(SiebelAccount[1]/Type = 'Residential'). If false, send only if SiebelAccount[1]/CifAccountReference[SystemName = 'SFA'] exists. Otherwise skip the account and continue. | High | Decision shapes (or a Set Properties + Route); the account-type value could come from a Cross Reference Table if more types are added. |
| BR-003 | Sequential per-account processing | Process SiebelAccount[1], remove it, republish the remainder to the internal queue while count > 0; parents before children. | High | Either keep the internal queue (worker process) or split into one document per account and process with Flow Control (1 thread), stopping at the first failure. |
| BR-004 | References only for created records | UCM AddAccountReference is called only for SFA response accounts and contacts with OCRMOperation = 'Create' (case-sensitive), once per CifAccountReference or CifContactReference. | High | Split by reference with a filter on OCRMOperation, then Process Call 'Sub: UCM AccountReference ADD' (crm-shared-framework). |
| BR-005 | SFA non-zero status is an error | UpserAccountResponse/STATUS_RESULT/@STATUS != 0 raises CRMS-03021 with ERROR_DESCRIPTION. | High | Decision plus Exception. |
| BR-006 | Stop hierarchy on first failure | Any failure in an iteration stops further accounts; the failing list is mailed and passed to the exception framework; the message is acknowledged. | High | Catch path with mail and error handler; no further iterations. |
| BR-007 | Ignore empty publications | A flat list with no SiebelAccount is traced as CRMS-03022 and ignored. | Low | Decision; Stop. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound JMS queue (TIBCO EMS) | %%Env%%.FIXED.Operations.%%CRMServices/ComponentId%%.Subscribe.UpsertAccountWithReferences.1 | EMS username/password (shared connection) | legacy/tibco/CRMServices/CRMServices/Processes/Subscribe_UpsertAccountWithReferencesJMS.process |
| inbound JMS queue (TIBCO EMS, private) | %%Env%%.FIXED.Operations.%%CRMServices/ComponentId%%._UpsertAccountWithReferences.1 | EMS username/password | legacy/tibco/CRMServices/CRMServices/Processes/_Subscribe_UpsertAccountWithReferencesJMS.process |
| outbound JMS queue send via Common Sub_NotifierJMS | concat(Env,'.FIXED.Operations.',ComponentId,'._UpsertAccountWithReferences.1') | EMS username/password | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process |
| outbound Rendezvous request/reply via Common Sub_Requester | concat(Env,'.FIXED.CRM.SFAAdapter.Request.UpsertAccount.1') | none visible | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process |
| outbound Rendezvous request/reply (via crm-shared-framework) | concat(Env,'.FIXED.CRM.UCMAdapter.Request.AddAccountReference.1') | none visible | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process |
| JMS Queue Receiver (com.tibco.plugin.jms.JMSQueueEventSource), x2 | PermittedMessageType Text; transacted false; acknowledgeMode 2 (client); maxSessions %%Common/JMS/JmsMaxSessions%%; JMSDeliveryMode PERSISTENT, JMSExpiration 0, JMSPriority 4; ConnectionReference /Common/Shared Resources/JMS Connection.sharedjmscon; ApplicationProperties /Common/Shared Resources/JMS Default Properties.sharedjmsapp (both not in export). | — | legacy/tibco/CRMServices/CRMServices/Processes/_Subscribe_UpsertAccountWithReferencesJMS.process |
| Common Sub_NotifierJMS | Inputs Destination, Body (rendered XML text), MessageId. | — | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process |
| Common Sub_Requester (SFA) | Destination = UpserAccountRequest/CMH/TO/@DESTINATION; Timeout = CRMServices/RV/Timeout; Body = request; reply parsed as UpserAccountResponse. | — | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertAccount.process |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| crm-shared-framework | Sub: UCM AccountReference ADD | synchronous |
| crm-shared-framework | Sub: CRM Send Error Mail | synchronous |
| crm-shared-framework | Sub: CRM Error Handler (replaces Sub_ThrowException) | synchronous |
| crm-getaccountdata-rv | Producer of the hierarchy publication | asynchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process | FlattenAccountHierarchy: for-each over BODY//SiebelAccount (descendant axis) copying around 80 optional fields per account with presence checks. | uawr-flatten-fn: Groovy (or XSLT from Groovy) over the whole document. | High |
| java | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferences.process | eai:getMessageText(code) for exception text; eai:getMessageId and eai:isTraceOn in the listeners (com.kpn.eai.component.common.CommonXPathFunctions). | Cross Reference Table 'CRM Message Catalogue CRT' and shared-cmh-header-fn (crm-shared-framework). | Medium |
| other | legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferences.process | tib:concat-sequence-format(Errors/Error/Params, newline) builds the mail body. | Groovy or string-concatenation map function. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Errors are raised as KPN Errors (CRMS-03001, CRMS-03021, requester errors, CRMS-03000) and propagate to the listener, which calls Common Sub_ThrowException and then acknowledges. The worker additionally mails the remaining list before throwing. In Boomi: Try/Catch per listener; Catch calls 'Sub: CRM Send Error Mail' (worker only) and 'Sub: CRM Error Handler' with the remaining list, then ends normally so the message is acknowledged. |
| Retries | None in this interface: no redelivery flag is computed and the message is always acknowledged. Whether Common Sub_ThrowException triggers redelivery is not in the export. |
| Compensation | None. SFA writes and earlier UCM references are not undone. Re-driving the remaining list re-sends the failed account to SFA (an upsert) and its references to UCM. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| /Common/Shared Resources/JMS Connection.sharedjmscon and JMS Default Properties.sharedjmsapp | other | no |
| Common Sub_NotifierJMS / Sub_Requester / Sub_ThrowException / Sub_Trace | library | no |
| EMS bridge Publish.UpsertAccountWithReferences.1 to Subscribe queue | other | no |
| XSDSchemas/Common/Base.xsd, ResultStatus.xsd | other | no |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| SFAAdapter UpsertAccount | TIBCO Rendezvous request/reply (target TBD) | %%Env%%.FIXED.CRM.SFAAdapter.Request.UpsertAccount.1 | Not stated in export |
| UCMAdapter AddAccountReference | TIBCO Rendezvous request/reply (target TBD) | %%Env%%.FIXED.CRM.UCMAdapter.Request.AddAccountReference.1 | Not stated in export |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| DEV | Env | XDVL (design-time default; environment it denotes not stated) | [redacted].substvar |

## 14. Boomi Migration Readiness Summary

This interface takes a customer's account hierarchy, published by GetAccountData, and writes it into SFA. It flattens the tree into a list, refuses anything over 50 accounts, and then handles the accounts strictly one at a time. For each account it calls SFA's 'upsert account'. For every account and contact that SFA reports as newly created, it asks UCM to record the external reference, so that both systems know the same customer by the same identifiers.

The one-at-a-time processing is done in an unusual way. After each account, the remaining list is put back on a private message queue and picked up again. That makes each account its own unit of work, but if one account fails, all accounts after it in that hierarchy are abandoned. The failing list is sent to integration support by e-mail and to the exception queue. There is no rollback: an account already written to SFA stays written, even if its UCM references fail.

This is the most complex interface in the project. It combines a tree-flattening step that needs a script, a choice about whether to keep the private-queue iteration in Boomi, a filtering rule for residential accounts, nested loops over accounts and contacts, and writes to two systems with no compensation. It also shares the project-wide blockers: Rendezvous transport to the adapters, missing header schemas, and the missing exception handler.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | CRM FlatAccountHierarchy PUBLISH XML Prof | uawr-flat-hierarchy-prof | profile.xml | — |
| 2 | SFA Account UPSERT Req XML Prof | uawr-sfa-upsertaccount-req-prof | profile.xml | — |
| 3 | SFA Account UPSERT Resp XML Prof | uawr-sfa-upsertaccount-resp-prof | profile.xml | — |
| 4 | EMS CRM UpsertAccountWithReferences Internal LISTEN Op | uawr-internal-listen-op | connector-action | shared-ems-jms-conn, uawr-flat-hierarchy-prof |
| 5 | EMS CRM UpsertAccountWithReferences Internal SEND Op | uawr-internal-send-op | connector-action | shared-ems-jms-conn, uawr-flat-hierarchy-prof |
| 6 | EMS CRM UpsertAccountWithReferences LISTEN Op | uawr-subscribe-listen-op | connector-action | shared-ems-jms-conn, shared-crm-uawr-publish-prof |
| 7 | SFA Account UPSERT Op | uawr-sfa-upsertaccount-op | connector-action | shared-ems-jms-conn, uawr-sfa-upsertaccount-req-prof, uawr-sfa-upsertaccount-resp-prof |
| 8 | CRM SiebelAccount Hierarchy Flatten | uawr-flatten-fn | transform.function | shared-crm-uawr-publish-prof, uawr-flat-hierarchy-prof |
| 9 | CRM FlatAccountHierarchy XML - CRM FlatAccountHierarchy XML (Remaining) | uawr-remaining-accounts-map | transform.map | uawr-flat-hierarchy-prof, shared-cmh-header-fn |
| 10 | CRM FlatAccountHierarchy XML - SFA Account UPSERT Req XML | uawr-flat-to-sfa-map | transform.map | uawr-flat-hierarchy-prof, uawr-sfa-upsertaccount-req-prof, shared-cmh-header-fn |
| 11 | SFA Account UPSERT Resp XML - UCM AccountReference ADD Req XML (Account) | uawr-sfa-to-ucm-accountref-map | transform.map | uawr-sfa-upsertaccount-resp-prof, shared-ucm-accountreference-add-req-prof, shared-cmh-header-fn |
| 12 | SFA Account UPSERT Resp XML - UCM AccountReference ADD Req XML (Contact) | uawr-sfa-to-ucm-contactref-map | transform.map | uawr-sfa-upsertaccount-resp-prof, shared-ucm-accountreference-add-req-prof, shared-cmh-header-fn |
| 13 | Sub: CRM Account UPSERT Into SFA and UCM | uawr-sub-upsert-into-sfa-ucm | process | uawr-sfa-upsertaccount-op, uawr-sfa-to-ucm-accountref-map, uawr-sfa-to-ucm-contactref-map, shared-sub-ucm-accountreference-add |
| 14 | CRM UpsertAccountWithReferences Account Worker | uawr-worker-process | process | uawr-internal-listen-op, uawr-flat-to-sfa-map, uawr-sub-upsert-into-sfa-ucm, uawr-remaining-accounts-map, uawr-internal-send-op, shared-sub-send-error-mail, shared-sub-error-handler |
| 15 | CRM UpsertAccountWithReferences Listener | uawr-main-process | process | uawr-subscribe-listen-op, uawr-flatten-fn, uawr-internal-send-op, shared-sub-error-handler |

## 15. Appendix

- legacy/tibco/CRMServices/CRMServices/Processes/Subscribe_UpsertAccountWithReferencesJMS.process
- legacy/tibco/CRMServices/CRMServices/Processes/Main_UpsertAccountWithReferences.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpserAccountWithReferencing.process
- legacy/tibco/CRMServices/CRMServices/Processes/_Subscribe_UpsertAccountWithReferencesJMS.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferences.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountWithReferencing_RecursivePublish.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/_Sub_UpsertAccountIntoSFA_UCM.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_UpsertAccount.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_AddAccountReference.process
- legacy/tibco/CRMServices/CRMServices/Processes/SubProcesses/Sub_GetAccountdata.process
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Publish/FlatAccountHierarchy.xsd
- legacy/tibco/CRMServices/XSDSchemas/CRMServices/Imported/0420_UpsertAccountWithReferencing_1.0_Publish_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/SFAAdapter/Request/UpsertAccount.xsd
- legacy/tibco/CRMServices/XSDSchemas/SFAAdapter/Response/UpsertAccount.xsd
- legacy/tibco/CRMServices/XSDSchemas/SFAAdapter/Imported/0421_UpsertAccount_1.0_Request_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/XSDSchemas/SFAAdapter/Imported/0421_UpsertAccount_1.0_Response_EAI_BUS_SIDE.xsd
- legacy/tibco/CRMServices/CRMServices/Shared Resources/CRMServicesMessages.xml
- legacy/tibco/CRMServices/defaultVars/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/CRMServices/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/Common/JMS/defaultVars.substvar
- legacy/tibco/CRMServices/defaultVars/CRMServices/RV/defaultVars.substvar
- legacy/tibco/CRMServices/.designtimelibs
- legacy/tibco/CRMServices/CRMServices/Test Processes/UpsertAccountwithReferences/STUB_UpsertAccount_FullyFilled.process
- legacy/tibco/CRMServices/CRMServices/Test Processes/UpsertAccountwithReferences/STUB_AddAccountReference_FullyFilled.process

## Open questions

### Still outstanding

REVIEW-NEEDED: If one account in a hierarchy fails, should the migrated process be able to restart from that account, or is it acceptable to re-run the whole hierarchy? (Decide whether to keep the self-republishing internal queue (durable per-account checkpoint; each account is its own Boomi execution) or replace it with an in-process split of the flat list and sequential processing in one execution. This changes how many processes and queues are built.)
  - If this is not answered: In-process sequential processing in one execution; on failure, the remaining accounts (including the failed one) are sent to the exception queue and the error mail, so they can be re-driven.

REVIEW-NEEDED: Is it safe to send the same account to SFA, and the same external reference to UCM, a second time when a failed hierarchy is re-processed? (Confirm that SFA UpsertAccount is idempotent (it is an upsert) and that re-adding an existing UCM reference is harmless or rejected predictably. Without that, the re-drive procedure is unsafe, because SFA and UCM are updated in sequence with no rollback.)
  - If this is not answered: Both operations are treated as safe to repeat, and re-drive re-sends the failed account in full.

REVIEW-NEEDED: Apart from GetAccountData, which systems publish account hierarchies to this flow, and should their residential accounts still be skipped unless they already have an SFA reference? (Confirm the business meaning of the Residential filter. All publications from GetAccountData set RecursiveProcessing = true, so the filter only applies to publications from other producers, and none are visible in the export.)
  - If this is not answered: The filter is kept exactly as today.

REVIEW-NEEDED: Should a hierarchy with more than 50 accounts still be rejected completely, and should integration support be e-mailed when that happens? (Today such hierarchies are rejected (CRMS-03001) and only recorded in the exception framework, with no e-mail. A large customer can therefore silently fail to synchronise.)
  - If this is not answered: The 50-account limit is kept, and rejection is recorded in the exception framework without e-mail, as today.

REVIEW-NEEDED: After a residential account is skipped, should the remaining accounts keep the original message tracking ID, which today is lost from that point on? (When an account is skipped by the filter, the message passed to the next iteration has no tracking information, so later logs and exception records cannot be tied to the original request, and processing may fail.)
  - If this is not answered: The original tracking ID is kept for all accounts (fixing the defect).

REVIEW-NEEDED: When a hierarchy fails, should the error e-mail still list all remaining accounts with their contacts' personal details? (The mail body today contains the full remaining account list, including names, birth dates, e-mail addresses and phone numbers of contacts.)
  - If this is not answered: The e-mail carries the message ID, error and failing account ID only; the full list goes to the exception queue.

REVIEW-NEEDED: After a failure is recorded, should the failed publication ever be retried automatically, or only re-driven by support? (Today every message is acknowledged after the error is handed to the exception framework. Whether that framework re-queues it is not in the export.)
  - If this is not answered: No automatic retry; support re-drives from the exception record.

REVIEW-NEEDED: How many hierarchy publications arrive per day, and how many accounts does a typical hierarchy contain? (Each account costs one SFA call plus one UCM call per new reference, each allowed 5 seconds, so volumes decide whether sequential processing in one run is fast enough.)
  - If this is not answered: Low volume; typical hierarchies of fewer than 10 accounts.

