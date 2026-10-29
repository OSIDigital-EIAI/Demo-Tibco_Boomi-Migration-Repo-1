# Client questions — CRMServices.UpsertContactPub (0429/1062)

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · When a contact update fails because SFA or UCM timed out, should the migrated process retry it automatically, and how many times?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** In BW the retry path rethrows the error to the listener, which calls Sub_ThrowException and then always confirms the message, so whether redelivery actually happens depends on Sub_ThrowException (missing). The Boomi design (do not acknowledge on retry, acknowledge otherwise) assumes redelivery was the intent.

**If this is not answered:** Timeout-class failures are retried by redelivery up to 4 times for the SFA step and 5 times for the UCM step, as the configuration suggests.

**The choices as we understand them:**

- Retry 4 / 5 times — Matches the configured limits.
- No automatic retry — Every failure is logged, e-mailed and re-driven by support.
- Different limits or delay — State the number of attempts and wait between them.

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Which system publishes the contact-update events that this flow receives?

**Why we are asking:** Confirm the producer of the Publish.UpsertContactPub.1 topic and the EMS bridge to the Subscribe queue. The schema namespace suggests UCM, but neither is in the export.

**If this is not answered:** UCM publishes to %%Env%%.FIXED.Operations.CRMServices.Publish.UpsertContactPub.1, and the existing EMS bridge to the Subscribe queue is kept unchanged.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · When a contact is linked to several Siebel accounts, should UCM record the new SFA contact against each account, or only the first, as today?

**Why we are asking:** Only the first SIEBEL_ACCOUNT_REFERENCE is used. Confirm that a contact linked to several accounts needs only one UCM reference.

**If this is not answered:** Only the first account is linked, as today.

**The choices as we understand them:**

- First account only — Same as today.
- Every linked account — One AddAccountReference per account reference.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · If a new SFA contact is created but its link in UCM cannot be queued, should that be retried or only reported by e-mail, as today?

**Why we are asking:** A failure to queue the UCM step is not treated as retryable, so the SFA contact stays unlinked in UCM unless someone acts on the e-mail.

**If this is not answered:** As today: reported by e-mail and the exception record, with no automatic retry.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · Does UCM rely on the object label 'SIEBEL_ADCCOUNT_ID' (as currently sent, misspelt) in the AddAccountReference header, or may it be corrected to 'SIEBEL_ACCOUNT_ID'?

**Why we are asking:** The worker re-maps the CMH OBJECT to 'SIEBEL_ADCCOUNT_ID' (a typo), while the main flow sends 'SIEBEL_ACCOUNT_ID'. Confirm with the UCMAdapter owners whether OBJECT is significant before choosing which value the Boomi build sends.

**If this is not answered:** The current value 'SIEBEL_ADCCOUNT_ID' is reproduced.

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->

### Q-006 · Should error e-mails for this flow still include the full contact publication with personal details?

**Why we are asking:** The SFA-step error mail embeds the rendered publication (names, date of birth, e-mail, phone, addresses).

**If this is not answered:** The e-mail carries the message ID, error code and description only; the full message stays in the exception record.

**Answer:**

<!-- answer:Q-006 -->
<!-- /answer:Q-006 -->

### Q-007 · Will the queue names keep the 'Operations' domain in every environment?

**Why we are asking:** The sender hard-codes '.FIXED.Operations.' in the internal queue name, but the listener uses %%CRMServices/ComponentDomain%% (default 'Operations'). Keep them identical in Boomi (both from CRM_ComponentDomain) and confirm no environment overrides ComponentDomain.

**If this is not answered:** Both names are built from one domain property defaulting to 'Operations'.

**Answer:**

<!-- answer:Q-007 -->
<!-- /answer:Q-007 -->

### Q-008 · How many contact publications arrive per day, and what share create new contacts in SFA?

**Why we are asking:** Volumes drive listener concurrency and the load on the UCM step.

**If this is not answered:** Low volume; same concurrency as today (10 sessions per listener).

**Answer:**

<!-- answer:Q-008 -->
<!-- /answer:Q-008 -->
