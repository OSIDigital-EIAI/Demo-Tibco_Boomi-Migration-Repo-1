# Client questions — CRMServices.GetAccountData (1054)

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Which applications call GetAccountData today, and can they move from TIBCO Rendezvous to EMS request/reply or to an HTTP API?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Identify the GetAccountData callers and confirm the target inbound transport (EMS request/reply or Boomi API). Boomi cannot receive Rendezvous requests, so the listener and reply cannot be built until this is decided.

**If this is not answered:** Callers move to an EMS request queue named like today's subject (&lt;Env&gt;.FIXED.CRM.CRMServices.Request.GetAccountData.1) and receive the reply on their JMSReplyTo destination.

**The choices as we understand them:**

- EMS request/reply — Closest to today; callers change their transport library only.
- HTTP API on Boomi — Modern API with authentication; callers change their integration code.
- Keep Rendezvous via a bridge — No caller change; an RV-to-EMS bridge is operated in the middle.

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · How long do callers wait for a GetAccountData reply before giving up, and how many requests per day and per peak hour should we expect?

**Why we are asking:** The service makes a call to UCM (up to 5 seconds) and a publish before replying. Any extra hop added by the new transport must fit inside the callers' timeout, and volumes drive runtime sizing.

**If this is not answered:** Callers wait at least 10 seconds; volume is low (thousands per day or fewer); the service runs in low-latency mode on a single runtime.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · What is the largest account hierarchy (number of accounts and number of levels) that UCM can return for one customer?

**Why we are asking:** Confirm the maximum realistic hierarchy depth and size. The conversion handles any depth, but hierarchies of more than 50 accounts are published here and then rejected by the UpsertAccountWithReferences flow.

**If this is not answered:** No depth limit is enforced here; a safety limit of 20 levels raises an explicit error; the 50-account limit stays downstream as today.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Should GetAccountData reject hierarchies of more than 50 accounts immediately, instead of publishing them for the UpsertAccountWithReferences flow to reject later?

**Why we are asking:** Decide whether to reject them earlier (in GetAccountData) or keep the current behaviour. Today the caller is told the request succeeded even though the synchronisation is later rejected with CRMS-03001.

**If this is not answered:** Current behaviour is kept: publish regardless of size; the limit is enforced downstream.

**The choices as we understand them:**

- Keep the check downstream — Same as today.
- Reject here with CRMS-03001 — The caller learns immediately that the hierarchy is too large; nothing is published.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · Must the publish of the account hierarchy still happen before the caller receives its reply, so that a failed publish is reported to the caller as an error?

**Why we are asking:** In BW the publish happens before the reply, inside the same job. If the publish fails, the caller gets an error reply. If the reply fails after a successful publish, the synchronisation still happens. Confirm whether the Boomi build must keep 'publish, then reply' ordering.

**If this is not answered:** The order is kept: publish first, then reply.

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->

### Q-006 · Do any callers depend on the exact content of error replies, such as the empty version field or the event name 'GetAccountDateRespone' (misspelt), or may these be corrected?

**Why we are asking:** Confirm whether consumers depend on the empty CMH/VERSION and the misspelt EVENT 'GetAccountDateRespone' in error replies, or whether the Boomi build may normalise them.

**If this is not answered:** Error replies are reproduced exactly, including the empty version and the misspelling.

**The choices as we understand them:**

- Reproduce exactly — Zero risk to existing callers.
- Correct them — Cleaner contract; callers confirm they do not depend on these values.

**Answer:**

<!-- answer:Q-006 -->
<!-- /answer:Q-006 -->

### Q-007 · When UCM reports an error for an account (for example, account not found), should integration support still receive an error e-mail, as happens today?

**Why we are asking:** Today any UCM error, including business errors, sends an 'Error in GetAccountData' e-mail as well as returning the error to the caller. This may produce e-mails for routine 'not found' cases.

**If this is not answered:** Current behaviour is kept: every UCM error sends an e-mail.

**The choices as we understand them:**

- Mail every UCM error — Same as today.
- Mail technical errors only — UCM business errors are returned to the caller without an e-mail.

**Answer:**

<!-- answer:Q-007 -->
<!-- /answer:Q-007 -->

### Q-008 · Should empty 'DisableMailings' and 'MasterAccountIntegrationId' fields still be sent in the published hierarchy when UCM does not supply them?

**Why we are asking:** Today these two fields are always written, even when empty, while every other optional field is omitted when absent. The consumer may treat an empty value differently from a missing one.

**If this is not answered:** The current behaviour is reproduced: both fields are always written.

**Answer:**

<!-- answer:Q-008 -->
<!-- /answer:Q-008 -->

### Q-009 · Which access control should protect GetAccountData once it is no longer on the Rendezvous network (for example EMS queue permissions or API credentials)?

**Why we are asking:** Today no caller authentication exists in the service; access relied on network and Rendezvous subject segregation.

**If this is not answered:** EMS queue permissions restrict who can send to the request queue; no message-level authentication is added.

**Answer:**

<!-- answer:Q-009 -->
<!-- /answer:Q-009 -->
