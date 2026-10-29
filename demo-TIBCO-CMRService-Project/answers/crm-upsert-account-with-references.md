# Client questions — CRMServices.UpsertAccountWithReferences (1056)

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · If one account in a hierarchy fails, should the migrated process be able to restart from that account, or is it acceptable to re-run the whole hierarchy?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Decide whether to keep the self-republishing internal queue (durable per-account checkpoint; each account is its own Boomi execution) or replace it with an in-process split of the flat list and sequential processing in one execution. This changes how many processes and queues are built.

**If this is not answered:** In-process sequential processing in one execution; on failure, the remaining accounts (including the failed one) are sent to the exception queue and the error mail, so they can be re-driven.

**The choices as we understand them:**

- Process the hierarchy in one run — Simpler; fewer queues; re-drive restarts from the failed account using the exception record.
- Keep one account per message — Same as today; each account is a separate durable unit of work.

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Is it safe to send the same account to SFA, and the same external reference to UCM, a second time when a failed hierarchy is re-processed?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Confirm that SFA UpsertAccount is idempotent (it is an upsert) and that re-adding an existing UCM reference is harmless or rejected predictably. Without that, the re-drive procedure is unsafe, because SFA and UCM are updated in sequence with no rollback.

**If this is not answered:** Both operations are treated as safe to repeat, and re-drive re-sends the failed account in full.

**The choices as we understand them:**

- Yes, repeats are harmless — Simple re-drive.
- No, duplicates cause problems — The re-drive must skip references already registered; extra design is needed.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Apart from GetAccountData, which systems publish account hierarchies to this flow, and should their residential accounts still be skipped unless they already have an SFA reference?

**Why we are asking:** Confirm the business meaning of the Residential filter. All publications from GetAccountData set RecursiveProcessing = true, so the filter only applies to publications from other producers, and none are visible in the export.

**If this is not answered:** The filter is kept exactly as today.

**The choices as we understand them:**

- Keep the filter — Same as today.
- Drop the filter — Only if GetAccountData is the sole producer.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Should a hierarchy with more than 50 accounts still be rejected completely, and should integration support be e-mailed when that happens?

**Why we are asking:** Today such hierarchies are rejected (CRMS-03001) and only recorded in the exception framework, with no e-mail. A large customer can therefore silently fail to synchronise.

**If this is not answered:** The 50-account limit is kept, and rejection is recorded in the exception framework without e-mail, as today.

**The choices as we understand them:**

- Reject, no e-mail — Same as today.
- Reject and e-mail — Support is told immediately.
- Change the limit — State the new maximum.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · After a residential account is skipped, should the remaining accounts keep the original message tracking ID, which today is lost from that point on?

**Why we are asking:** When an account is skipped by the filter, the message passed to the next iteration has no tracking information, so later logs and exception records cannot be tied to the original request, and processing may fail.

**If this is not answered:** The original tracking ID is kept for all accounts (fixing the defect).

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->

### Q-006 · When a hierarchy fails, should the error e-mail still list all remaining accounts with their contacts' personal details?

**Why we are asking:** The mail body today contains the full remaining account list, including names, birth dates, e-mail addresses and phone numbers of contacts.

**If this is not answered:** The e-mail carries the message ID, error and failing account ID only; the full list goes to the exception queue.

**Answer:**

<!-- answer:Q-006 -->
<!-- /answer:Q-006 -->

### Q-007 · After a failure is recorded, should the failed publication ever be retried automatically, or only re-driven by support?

**Why we are asking:** Today every message is acknowledged after the error is handed to the exception framework. Whether that framework re-queues it is not in the export.

**If this is not answered:** No automatic retry; support re-drives from the exception record.

**Answer:**

<!-- answer:Q-007 -->
<!-- /answer:Q-007 -->

### Q-008 · How many hierarchy publications arrive per day, and how many accounts does a typical hierarchy contain?

**Why we are asking:** Each account costs one SFA call plus one UCM call per new reference, each allowed 5 seconds, so volumes decide whether sequential processing in one run is fast enough.

**If this is not answered:** Low volume; typical hierarchies of fewer than 10 accounts.

**Answer:**

<!-- answer:Q-008 -->
<!-- /answer:Q-008 -->
