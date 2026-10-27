**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:** Piranat Wadlom / 6732031621
**My role on the review board:** Site reliability engineer (SRE)

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

As SRE, I found AI strong at reading a design and listing risks, it correctly spotted that an order could be saved while its event is lost, and it checked the ADR against MailFast's limits. But it only partly covered what I raised, who gets paged when the queue backs up, how to replay failed events, and how to trace one order across the async boundary. That knowledge comes from running systems, and it wasn't written in the ADR. I would trust AI as an analyst for finding technical risks, but keep operational ownership, business priorities like loyalty refunds, and the final accept/reject decision for humans.
