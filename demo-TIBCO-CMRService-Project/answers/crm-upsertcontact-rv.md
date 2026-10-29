# Client questions — CRMServices.UpsertContact (1046)

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Which applications call UpsertContact today, and can they move from TIBCO Rendezvous to EMS request/reply or to an HTTP API?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Identify the callers of UpsertContact and confirm they can move from RV request/reply to EMS request/reply (or HTTP). This decides whether the listener and reply are JMS or a Web Services Server listener.

**If this is not answered:** Callers move to an EMS request queue named like today's subject (&lt;Env&gt;.FIXED.CRM.CRMServices.Request.UpsertContact.1), with the same decision as GetAccountData.

**The choices as we understand them:**

- EMS request/reply — Closest to today.
- HTTP API on Boomi — Modern API; callers change their integration.
- Keep Rendezvous via a bridge — No caller change; a bridge is operated in between.

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Should warnings or errors reported only on a contact's addresses (with none on the person) stop the contact from being saved in UCM?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** As written, the totals are computed only when the contact-level warning or error count is above 0, so warnings or errors reported only on addresses produce totals of 0 and the contact is upserted. Confirm whether this is intended behaviour to replicate or a defect to fix.

**If this is not answered:** Today's behaviour is replicated exactly: address-only findings do not stop the upsert.

**The choices as we understand them:**

- Keep today's behaviour — Address-only findings are ignored.
- Always count address findings — Any address warning or error stops the upsert unless overridden.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · When a contact is marked 'override validation', should the override also stop contact-level warnings from counting towards the total that blocks the save?

**Why we are asking:** Also, the contact-level override is applied to VALIDATION_WARNING_COUNT, but the totals test reads the raw response counts, so an overridden contact warning can still block the save.

**If this is not answered:** Today's behaviour is replicated: totals use the raw counts.

**The choices as we understand them:**

- Keep today's behaviour — The override changes the reported count only.
- Override also clears the total — Overridden warnings never block the save.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · When validation stops a contact but the validation service reports success status, should the caller receive the validation details instead of the generic 'General Un-Attribute Error' with no details that it gets today?

**Why we are asking:** In this case the reply has no BODY, so the caller cannot see which warning or error stopped the contact. This looks like a defect, and the fix changes the reply content.

**If this is not answered:** Today's reply is replicated exactly: STATUS 1, 'General Un-Attribute Error', no BODY.

**The choices as we understand them:**

- Keep today's reply — No change for callers.
- Return the validation details — Callers receive the CDC BODY with the findings.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · Does the validation service always return a contact's addresses in the same order as they were sent?

**Why we are asking:** The override uses the request ADDRESS at the same position as the response ADDRESS. If the order can change, the override applies to the wrong address.

**If this is not answered:** Order is preserved, and addresses are matched by position as today.

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->

### Q-006 · Should every address still be sent for validation as Dutch-language with the 'proprietary' street-naming convention, whatever the caller supplied?

**Why we are asking:** Language 'nl' and naming convention 'PROPRIETARY' are hard-coded, which overrides any convention the caller sends. Foreign addresses may be validated incorrectly.

**If this is not answered:** The hard-coded values are kept.

**Answer:**

<!-- answer:Q-006 -->
<!-- /answer:Q-006 -->

### Q-007 · Which system sits behind the CDC validation service, and who owns it?

**Why we are asking:** The system behind CDCAdapter and its owner are not named in the export. The transport choice and testing need a contact.

**If this is not answered:** CDCAdapter is reached the same way as UCMAdapter (the shared transport decision), with no change to its message format.

**Answer:**

<!-- answer:Q-007 -->
<!-- /answer:Q-007 -->

### Q-008 · Do any callers depend on the reply header fields as they are today (reply ID 1048 with type 'REQUEST'), or may they be corrected to a response type?

**Why we are asking:** The reply header identifies itself as service 1048 with PARADIGM 'REQUEST', and the UCM request is labelled 'ValidateContact'. Correcting these could affect callers' parsing or monitoring.

**If this is not answered:** Header values are reproduced exactly as today.

**Answer:**

<!-- answer:Q-008 -->
<!-- /answer:Q-008 -->

### Q-009 · How long do callers wait for an UpsertContact reply, and how many requests per day and per peak hour are expected?

**Why we are asking:** The service makes two back-end calls in sequence, each allowed 5 seconds. Any extra hop from the new transport must fit within the callers' timeout, and volumes drive sizing.

**If this is not answered:** Callers wait at least 15 seconds; low volume; low-latency mode on a single runtime.

**Answer:**

<!-- answer:Q-009 -->
<!-- /answer:Q-009 -->
