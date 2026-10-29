# Client questions — CRMServices.SharedFramework

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Can you provide the KPN EAI Common library (Common_2.2.2.projlib), including Base.xsd, ResultStatus.xsd, Errors.xsd, EAIMessages.xml and the source of the CommonXPathFunctions Java class?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The message header and status schemas, the transport to the back-end adapters, the exception-queue handler and the custom helper functions all live in this library, and it was not included in the export. The three shared message profiles and the error handler cannot be completed without it.

**If this is not answered:** Header, status and error structures are reconstructed from how the CRMServices mappings use them; they are validated against real messages captured from the current system before go-live.

**The choices as we understand them:**

- Provide the full Common library — Most accurate; profiles are imported directly from the schemas.
- Provide only the schemas and message catalogue — Profiles can be built; transport and error-handler behaviour are specified by the client instead.
- Provide captured sample messages instead — Profiles are reverse-engineered from samples; higher risk of missing optional fields.

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · How should the migrated integrations reach UCMAdapter for AddAccountReference, now that TIBCO Rendezvous will not be available to Boomi?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Today the call goes over TIBCO Rendezvous request/reply, which Boomi cannot connect to. The choice decides how the UCM AccountReference ADD operation and its message formats are built, and the same decision applies to every back-end call in the project.

**If this is not answered:** EMS (JMS) request/reply to the same subject name, with an RV-to-EMS bridge or the adapter re-pointed to EMS by the client's middleware team.

**The choices as we understand them:**

- EMS request/reply via an RV-EMS bridge — No change to UCMAdapter; requires bridge configuration.
- Re-point UCMAdapter to listen on EMS — Removes Rendezvous from this path; requires a change to the adapter.
- Call Siebel UCM web services directly — Removes the adapter layer; changes message formats and increases scope.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Which mail server should the migrated error notifications use, and who should receive them (To, CC, BCC) in each environment?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** SMTP host, port, authentication, TLS and the From/To/CC/BCC lists are unknown: the mail settings file was not included in the export. Without them the mail connection and distribution list cannot be configured.

**If this is not answered:** Notifications are built against a placeholder SMTP connection and an empty distribution table that must be filled in before testing.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Should error e-mails keep including the full failing message, which contains contact personal data, or only the message ID, error code and description?

**Why we are asking:** Error mails currently embed the full failing message (contact personal data). The answer decides the notification content in all four business interfaces.

**If this is not answered:** Mails carry the service name, message ID, error code and error description only; the full message stays in the exception queue and in Boomi process reporting.

**The choices as we understand them:**

- Keep the full message in the mail — Same as today.
- Message ID and error only — No personal data in e-mail.
- Masked message — Message included with personal fields masked.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · What is the name and expected message format of the enterprise exception queue that failed CRMServices messages are sent to, and which team or tool reads it?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The message format and destination of the exception queue, and whether failed messages are also rolled back or redelivered, are defined in the missing Common library. The Boomi error handler must produce records the existing exception tooling accepts.

**If this is not answered:** Failures are sent to a queue named from the environment prefix plus a configurable name, in a simple XML record holding the error codes, message ID, original body and JMS properties. Downstream tooling may need adjusting.

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->

### Q-006 · Does any system or support process depend on the specific CRMServices and EAI error codes (for example CRMS-03021 'Backend replied with error' or EAI-03004 for a timeout)?

**Why we are asking:** The error codes produced by the transport on timeout (EAI-03004) and the engine timeout code (BWENGINE-100067) are framework codes, and the retry rules of the contact-publication flow depend on them. Keeping the same codes keeps those rules and any monitoring working.

**If this is not answered:** The existing CRMS codes are kept, and a Boomi timeout is reported as EAI-03004 so that retry rules behave as today. BWENGINE codes are not reproduced.

**Answer:**

<!-- answer:Q-006 -->
<!-- /answer:Q-006 -->

### Q-007 · Do any systems that read CRMServices messages rely on the exact format of the tracking timestamp or the 'Operations/CRMServices' application tag in the message header?

**Why we are asking:** The exact output format of the TIBCO timestamp function must be matched so that consumers parsing the header timestamp are unaffected; the format cannot be determined from the export.

**If this is not answered:** ISO 8601 date-time with milliseconds and time-zone offset is used, and the application tag is kept as 'Operations/CRMServices'.

**Answer:**

<!-- answer:Q-007 -->
<!-- /answer:Q-007 -->

### Q-008 · Which EMS server addresses, and which dedicated service account, should the Boomi runtime use in each environment (development, test, production)?

**Why we are asking:** The export holds only localhost defaults and a generic account name. Real values are needed to set up the shared EMS connection, and a least-privilege account is recommended.

**If this is not answered:** The connection is built with environment extensions left empty, to be filled in by the client's EMS administrators at deployment.

**Answer:**

<!-- answer:Q-008 -->
<!-- /answer:Q-008 -->

### Q-009 · Should traffic between Boomi and EMS be encrypted (SSL/TLS)?

**Why we are asking:** An SSL address is defined in the configuration, but the defaults use unencrypted connections and nothing shows which one production uses.

**If this is not answered:** The same transport as today's production configuration is used once the EMS administrators confirm it; otherwise SSL is recommended.

**The choices as we understand them:**

- Use SSL/TLS — Requires the EMS certificates to be installed on the Boomi runtime.
- Keep plain TCP — As in the design-time defaults.

**Answer:**

<!-- answer:Q-009 -->
<!-- /answer:Q-009 -->

### Q-010 · Roughly how many failures, and how many new account or contact references, do the CRMServices integrations handle per day, and what is the busiest hour?

**Why we are asking:** No volumes, rates or SLAs are stated anywhere in the export; they drive runtime sizing and the concurrency of the shared components.

**If this is not answered:** Low volume (thousands per day or fewer), served by a single Boomi runtime with the same concurrency as today (10 sessions per listener).

**Answer:**

<!-- answer:Q-010 -->
<!-- /answer:Q-010 -->
