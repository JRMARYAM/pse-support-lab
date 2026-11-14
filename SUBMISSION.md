# Submission

- **Name:** Aaisha Maryam
- **Date submitted:** 14 November 2026
- **Repository link:** https://github.com/JRMARYAM/pse-support-lab
- **Total time spent (real time):** 5 hours 25 minutes (actual working time)

> All customer replies and acknowledgements below are drafts. They were not sent. Exercise-clock times are proposed response times for Monday 16 November 2026, Asia/Dubai. Lab replay and database evidence was collected on 14 November 2026; it is not original production evidence from 16 November.

> The local Docker Compose web port was changed from 3000 to 3002 to work around the local environment's port-forwarding issue. This local configuration change is retained.

- **Items I fully investigated, and why I chose them:**
  - Incident 3 — Al Noor: reproducible ASP rejection with a code-level calculation discrepancy.
  - Incident 4 — Sahara: reproducible missing ASP references, rate-limit logs, database event history and a missing 429 handling path.
  - Incident 5 — Ghaf: reproducible portal validation errors with a clear documented date-format requirement.
  - Incidents 1 and 2 were partially investigated. The available local environment did not contain the original production attempts or incident-date telemetry, so I did not claim an unverified production root cause.

---

## Incident 1 — `inbox/01-oasis-builders-upload-stuck.md`

### Acknowledgement

- **Sent at (exercise clock):** Draft proposed for 09:20, Monday 16 November 2026. Not sent.
- **Reply:**

> Hello Rashid,
>
> We understand that your November workbook is stuck in Processing and that repeated attempts have resulted in duplicate document number messages. We will reconcile the original import attempts against the invoice records and ASP references before advising on resubmission.
>
> Please do not upload the complete workbook again for now, as this could create additional duplicate attempts. We will provide an update by 10:30 Dubai time with the invoice-level status and safe next steps.
>
> Regards,
> Support Team

### Priority and justification

- **Priority:** P2.
- **Why:** A large monthly upload is affected and the customer cannot establish which invoices reached the ASP. This is a significant customer workflow issue, but the supplied evidence does not establish a platform-wide outage.
- **SLA deadlines I am working to:** First response by 10:05; proposed acknowledgement 09:20. Next update 10:30. Resolution target within 8 hours of receipt, by 17:05, subject to investigation and an agreed workaround.

### Investigation log

| Time / source | What I checked / did | What I found |
|---|---|---|
| Lab test, 14 Nov | Uploaded the clean Desert Rose sample | Import completed; 5 invoices accepted. This confirmed the normal upload path worked in the local lab. |
| Lab replay, 14 Nov 08:33:56 UTC | Ran `make replay CUSTOMER=oasis` | Replay import `d6ce51a1-904d-4085-a7b6-1da48fd2bdf0` completed with 8 invoices accepted; duration approximately 121 seconds. |
| Database inspection | Queried Oasis imports and invoice events | The local database contained an October seeded import and the November replay import. It did not contain the customer's three original attempts. |
| Database inspection | Checked Oasis invoice history | October invoices and the replay-generated November invoices followed Parsed → Validated → Submitted → Accepted. Replay invoice numbers have a generated suffix and are not proof of the customer's original invoice numbers. |
| Code review | Reviewed `services/api/internal/importer/importer.go` | Some error-return paths after setting Processing do not visibly call `FinishImport` to mark the import Failed. This is a possible stuck-Processing defect, not a proven cause of this incident. |

### Root cause

- **What happened:** The original customer upload attempts could not be located in the available local database or logs. The replay of the supplied workbook succeeded, but it was a separate test import and does not establish what happened to the original attempts.
- **Where** (component, `file:line`): Import lifecycle in `services/api/internal/importer/importer.go`; import persistence and status updates in `services/api/internal/store/store.go`.
- **Why it happens:** Code review identified error paths after the import enters Processing that return without visibly finalising the import as Failed. If one of those errors occurs, an import could remain Processing. There is insufficient evidence to say that this occurred in the original customer upload.
- **Evidence:**
  - [Source code findings](evidence/source-code-findings.txt)
  - Database checks: Oasis import records and invoice events were inspected during the lab investigation. The original three customer attempts were not present.
- **Confidence and open questions:** Low confidence in the original incident's root cause. Need the original import IDs, upload timestamps, API logs, database import rows, and reverse-proxy logs for the three attempts. Need to confirm whether the customer saw a client-side timeout while the server continued processing.

### Customer resolution reply OR escalation note

- **Which one and why:** Customer update draft. The original attempts must be reconciled before recommending any retry.

> Hello Rashid,
>
> We have reproduced the normal upload flow using a separate test import, but that does not confirm the status of your original November attempts. We need to match each original invoice number to its import record and ASP reference before confirming which documents were delivered.
>
> Please do not re-upload the full workbook yet. We will provide an invoice-by-invoice reconciliation and identify only those invoices that are safe to resubmit.
>
> Regards,
> Support Team

### Workaround offered

Do not retry the entire workbook. Obtain the original import IDs or timestamps and reconcile each invoice number against its current portal status and ASP message reference. Retry only invoices confirmed not to have been accepted or submitted, following the documented lifecycle rules.

### Prevention ideas

- Ensure every error path after Processing marks the import Failed with a useful error.
- Add monitoring for imports remaining in Processing beyond the expected duration.
- Make import processing resilient to client timeouts and expose a reliable import-status endpoint.
- Provide an invoice-level reconciliation report before allowing a customer to retry a large workbook.

---

## Incident 2 — `inbox/02-alert-api-restarted-marina-bay.md`

### Acknowledgement

- **Sent at (exercise clock):** Draft proposed for 09:55, Monday 16 November 2026. Not sent.
- **Sent to:** Priya (internal alert reporter); a separate customer update should be sent to affected customers.
- **Reply:**

> Hi Priya,
>
> We acknowledge the critical API restart alert and reports of intermittent 502 responses affecting multiple customers. We are treating this as P1 and escalating to Engineering immediately.
>
> We are checking API health, restart history, error rates and invoices remaining in Validated. We will provide the next update by 10:30 Dubai time. We will not confirm Marina Bay's go-live readiness until service stability and invoice delivery have been verified.
>
> Regards,
> Support Team

### Priority and justification

- **Priority:** P1.
- **Why:** The alert reports repeated production API restarts, intermittent 502s across multiple customers and invoices stuck in Validated. This meets the P1 production-impact criteria.
- **SLA deadlines I am working to:** Alert received 09:42. First response target within 15–30 minutes; proposed response 09:55. Next update by 10:30, within the 30–60-minute update interval. Resolution target within 4 hours of the alert, by 13:42. Engineering escalation should happen immediately.

### Investigation log

| Time / source | What I checked / did | What I found |
|---|---|---|
| Local environment inspection, 14 Nov | Ran `docker compose ps -a` | API, ASP mock, PostgreSQL, web and Grafana were running. Alloy, cAdvisor, Loki and Prometheus had exited with code 127. |
| Local API logs, 14 Nov | Reviewed the available API log tail | The displayed requests were successful GET requests with HTTP 200. These logs were from 14 November, not the reported 16 November incident. |
| Local ASP logs, 14 Nov | Reviewed ASP mock startup and request logs | Several historical `listening` entries were present. Current container inspection showed restart count 0 since its latest start. This does not establish the production API restart history. |
| Scope check | Compared local evidence with the incident report | The local lab is not the production environment and does not contain the incident-date production telemetry. |

### Root cause

- **What happened:** A production-impacting API incident was reported, but the available local environment did not contain the production telemetry needed to establish why the API restarted or why customers received 502 responses.
- **Where** (component, `file:line`): Production API service, deployment/container runtime, and upstream proxy/load balancer. The precise failing component is not established.
- **Why it happens:** Unknown. The local API was healthy during the later lab inspection, but that is not evidence about the reported production incident.
- **Evidence:**
  - Local API log output was inspected; it contains successful requests from 14 November, not 16 November.
  - Local container status showed monitoring services exited with code 127.
  - Production Grafana, API logs, container restart events and load-balancer logs were not available in the local lab.
- **Confidence and open questions:** High confidence in P1 classification based on the incident report; low confidence in any technical root cause. Engineering must inspect production restart timestamps, OOM events, CPU/memory, API error rates, 502 upstream details, deployment changes, ASP connectivity and the age of invoices in Validated.

### Customer resolution reply OR escalation note

- **Which one and why:** L3 Engineering escalation is required because the reported symptoms involve production infrastructure and multiple customers. L1 retains ownership of customer communications.

> **P1 — Production API instability / invoice submission delays**
>
> **Customer/business impact:** Multiple customers report intermittent HTTP 502 responses. Some invoices have remained in Validated since approximately 09:30 Dubai time. Marina Bay Hospitality has a planned go-live on 18 November.
>
> **Reporter/time:** Priya; alert received 16 November 2026 at 09:42 Dubai time.
>
> **Symptoms:** API reportedly restarted three times in 15 minutes; intermittent 502s; invoices delayed in Validated.
>
> **Evidence available:** Incident alert and customer reports. Local lab logs are from 14 November and cannot establish the production cause.
>
> **Checks still required:** Production API restart and OOM events, resource utilisation, deployment history, upstream proxy logs, API 5xx metrics, queue/worker health, ASP connectivity and affected invoice IDs.
>
> **Suspected cause/confidence:** Unknown; no production root cause established.
>
> **Workaround:** Identify affected invoice IDs and monitor their statuses. Do not blindly re-upload invoices. Confirm whether each invoice reached the ASP before any retry.
>
> **Exact ask:** Engineering to investigate production restart events and 502s immediately, restore service stability, identify affected invoice IDs and confirm safe recovery/retry steps.
>
> **Next update due:** 10:30 Dubai time.
>
> **Customer ownership:** L1 Support retains customer communication and will update affected customers every 30–60 minutes.

Customer update draft:

> Hello,
>
> We are investigating an API service issue that may be delaying invoice submissions. Please do not resend invoices that are already visible in the portal until their ASP delivery status has been checked. We are reconciling affected invoices and will provide another update by 10:30 Dubai time.
>
> We will confirm Marina Bay's go-live readiness only after service stability and successful invoice delivery have been verified.
>
> Regards,
> Support Team

### Workaround offered

Identify invoices stuck in Validated and check their submission status and ASP references. Avoid blind re-uploading or retrying invoices that may already have reached the ASP. Any recovery action must be coordinated with Engineering.

### Prevention ideas

- Add alerts for repeated API restarts, OOM kills and elevated 502 rates.
- Monitor age and count of invoices remaining in Validated.
- Add queue/worker health and ASP delivery success dashboards.
- Maintain a production incident runbook and go-live readiness checklist.
- Test recovery and duplicate-safe retry procedures before customer go-live.

---

## Incident 3 — `inbox/03-al-noor-fuel-br-co-15.md`

### Acknowledgement

- **Sent at (exercise clock):** Draft proposed for 10:30, Monday 16 November 2026. Not sent.
- **Reply:**

> Hello Fatima,
>
> We acknowledge the BR-CO-15 rejection affecting some invoices. We will compare the generated invoice totals against the line-level net and VAT amounts and identify the exact invoices affected.
>
> We will keep accepted invoices separate from rejected invoices and provide the findings and next steps by 13:15 Dubai time.
>
> Regards,
> Support Team

### Priority and justification

- **Priority:** P2.
- **Why:** A subset of invoices is rejected by the ASP and the customer has an active VAT audit. The issue affects financial compliance and invoice delivery, but the evidence does not indicate a complete service outage.
- **SLA deadlines I am working to:** Received 10:15. First response by 11:15; proposed response 10:30. Next update 13:15. Resolution target within 8 hours, by 18:15.

### Investigation log

| Time / source | What I checked / did | What I found |
|---|---|---|
| Lab replay, 14 Nov 09:13:40 UTC | Ran `make replay CUSTOMER=al-noor` | Six invoices processed: three Accepted and three Rejected by ASP. |
| Database inspection | Queried invoice status and ASP errors | Rejected invoices had BR-CO-15. Differences between reported gross and net-plus-VAT were 1–2 fils. |
| Payload inspection | Inspected generated PINT payload for `ANF-2611-0042-HIYR` | Tax exclusive amount 14,307.08; VAT 715.35; tax inclusive amount 15,022.42. Net plus VAT equals 15,022.43, a 0.01 difference. |
| Code review | Reviewed `services/api/internal/pint/pint.go:159-185` | Net and VAT are accumulated from line amounts, while gross is calculated separately from quantity × unit price × VAT rate. |
| Comparison | Compared the three rejected invoices with the three accepted invoices | The replay reproduced the mismatch on three invoices; the other three were accepted. |

### Root cause

- **What happened:** The local replay produced three ASP rejections for BR-CO-15 because the generated tax-inclusive total differed from the sum of tax-exclusive amount and VAT by one or two fils.
- **Where** (component, `file:line`): `services/api/internal/pint/pint.go:159-185`, particularly the gross calculation and returned totals.
- **Why it happens:** The code sums line net amounts and rounded VAT amounts, but calculates gross independently using quantity × unit price × (1 + VAT rate / 100). These different calculation paths can produce rounding differences. In the observed payload, net 14,307.08 plus VAT 715.35 equals 15,022.43, while the generated gross is 15,022.42.
- **Evidence:**
  - [Al Noor database findings](evidence/incident-3-al-noor-db.txt)
  - [Source code findings](evidence/source-code-findings.txt)
  - Code reference: `services/api/internal/pint/pint.go:159-185`
  - Replay import ID: `99f0ef5f-96bf-485f-a0b1-828c4587f04a`
- **Confidence and open questions:** High confidence that the local replay has an inconsistent total calculation path and that it explains the observed replay rejections. Engineering should verify the expected PINT AE rounding rules and test all VAT categories and decimal-price cases before applying a fix to production.

### Customer resolution reply OR escalation note

- **Which one and why:** L3 Engineering escalation is appropriate because the issue appears to be in the generated invoice calculation, not a customer workbook formatting error.

> **P2 — BR-CO-15 total calculation discrepancy**
>
> **Customer:** Al Noor Fuel & Lubricants.
>
> **Impact:** Three of six invoices in the local replay were rejected by the ASP with BR-CO-15. The other three were accepted.
>
> **Evidence:** Replay import `99f0ef5f-96bf-485f-a0b1-828c4587f04a`; rejected invoice records and errors are in `evidence/incident-3-al-noor-db.txt`. Payload inspection for `ANF-2611-0042-HIYR` shows net 14,307.08 + VAT 715.35 = 15,022.43, but tax inclusive amount is 15,022.42.
>
> **Suspected cause:** `pint.go:159-185` calculates gross independently from the accumulated net and rounded VAT amounts.
>
> **Exact ask:** Review the total calculation and implement consistent currency-safe rounding. Add regression tests covering fuel prices with three decimal places and all applicable VAT categories.
>
> **Workaround:** Keep accepted invoices untouched. Correct and resubmit only rejected invoices after confirming the corrected totals and invoice status.
>
> **Next update due:** 13:15 Dubai time.

Customer update draft:

> Hello Fatima,
>
> Our test reproduced the BR-CO-15 issue. The generated invoice total can differ from the sum of the net amount and VAT by one or two fils because those totals are calculated using different rounding paths.
>
> We identified three rejected invoices in the test replay. Accepted invoices should not be resubmitted. We are escalating the calculation issue for correction and will confirm the affected production invoice numbers before recommending any resubmission.
>
> Regards,
> Support Team

### Workaround offered

Do not resend the complete workbook. Reconcile the original production invoice statuses first. Leave accepted invoices unchanged. Correct and resubmit only confirmed rejected invoices after validating that the generated tax-inclusive total equals net plus VAT under the applicable rounding rules.

### Prevention ideas

- Use a consistent decimal-based currency calculation rather than independent floating-point calculations.
- Derive tax-inclusive totals from the same rounded net and VAT totals used in the payload.
- Add regression tests for one-fils and two-fils rounding boundaries.
- Test fuel prices with three decimal places and all VAT categories.
- Log the net, VAT and gross totals before ASP submission for easier diagnosis.

---

## Incident 4 — `inbox/04-bluepeak-sahara-not-received.md`

### Acknowledgement

- **Sent at (exercise clock):** Draft proposed for 11:45, Monday 16 November 2026. Not sent.
- **Reply:**

> Hello,
>
> We acknowledge that some invoices show Submitted without an ASP reference and that the buyer has received only part of the batch. We will reconcile invoice statuses and ASP responses before advising on any retry.
>
> Please do not resend the full batch. We will provide an update by 14:30 Dubai time with the confirmed accepted and pending invoice counts and the safe next steps.
>
> Regards,
> Support Team

### Priority and justification

- **Priority:** P2.
- **Why:** A batch of 40 invoices is affected and missing buyer delivery may affect the customer's payment run. The local replay demonstrates a specific rate-limit handling defect, although the original production requests still require verification.
- **SLA deadlines I am working to:** Received 11:30. First response by 12:30; proposed response 11:45. Next update 14:30. Resolution target within 8 hours, by 19:30.

### Investigation log

| Time / source | What I checked / did | What I found |
|---|---|---|
| Database inspection | Checked existing Sahara records | The pre-existing October import contained 12 Accepted invoices with ASP message IDs. |
| Lab replay, 14 Nov 09:16:28 UTC | Ran `make replay CUSTOMER=sahara` | The 40-invoice replay completed with 8 Accepted and 32 Submitted. |
| Database inspection | Queried all 40 invoice records and ASP references | Eight invoices had ASP message IDs. Thirty-two were Submitted with empty ASP references. |
| Event history | Queried invoice events | The 32 pending invoices had Parsed → Validated → Submitted events only; no Accepted event was recorded. |
| ASP mock logs | Reviewed the replay's ASP logs | Eight documents were accepted. Thirty-two requests received rate-limit warnings with `retry_after_s: 1`. |
| Code review | Reviewed `services/asp-mock/main.go:35-37,134-150,274-283` | The mock ASP allows a burst of 8 and refills at 2 requests per second; excess requests receive HTTP 429 and Retry-After. |
| Code review | Reviewed `services/api/internal/submit/submit.go:105-155` | The worker handles HTTP 200, 422 and selected 5xx responses, but has no explicit 429 handling. The invoice is marked Submitted before the ASP call. |

### Root cause

- **What happened:** In the local replay, 8 of 40 invoices were accepted by the ASP. The remaining 32 requests were rate-limited with HTTP 429 and Retry-After: 1. Those 32 invoices remained Submitted without ASP references or Accepted events.
- **Where** (component, `file:line`):
  - `services/asp-mock/main.go:35-37` — sender burst and rate configuration.
  - `services/asp-mock/main.go:134-150` — token-bucket limiter.
  - `services/asp-mock/main.go:274-283` — HTTP 429 response and Retry-After.
  - `services/api/internal/submit/submit.go:105-155` — submission status and response handling.
- **Why it happens:** The worker marks an invoice Submitted before sending it to the ASP. The local ASP accepts a burst of 8 requests and rate-limits excess requests. The worker does not explicitly handle HTTP 429 or honour Retry-After, so the rate-limited requests are not retried and remain Submitted without an ASP message ID.
- **Evidence:**
  - [Sahara database findings](evidence/incident-4-sahara-db.txt)
  - [Sahara ASP mock logs](evidence/incident-4-sahara-asp-logs.txt)
  - [Source code findings](evidence/source-code-findings.txt)
  - Replay import ID: `74cb0153-350c-4654-b44c-d284e3b820f2`
  - Local replay result: 8 Accepted, 32 Submitted.
- **Confidence and open questions:** High confidence in the local replay reproduction and code-path gap. This does not prove the original production batch received the same 429 responses. Production ASP logs, request IDs, response codes and invoice-level references are required to confirm the original incident.

### Customer resolution reply OR escalation note

- **Which one and why:** L3 Engineering escalation is required because the submission worker needs explicit rate-limit handling and safe retry behaviour.

> **P2 — ASP rate limiting leaves invoices Submitted without references**
>
> **Customer:** Sahara Medical Supplies, reported through Bluepeak ERP Integrations.
>
> **Impact:** 40 invoices reported; some buyer deliveries missing and payment run at risk.
>
> **Local reproduction:** Replay import `74cb0153-350c-4654-b44c-d284e3b820f2` produced 8 Accepted and 32 Submitted without ASP references.
>
> **Evidence:** ASP mock logs show eight accepted documents followed by 32 rate-limit warnings with Retry-After 1. Database event history shows the 32 invoices have no Accepted event. See `evidence/incident-4-sahara-db.txt` and `evidence/incident-4-sahara-asp-logs.txt`.
>
> **Suspected cause:** Submission worker lacks explicit HTTP 429 handling and retry scheduling.
>
> **Exact ask:** Implement explicit 429 handling, parse and honour Retry-After, add bounded retries/backoff, and ensure exhausted retries become a visible failure state rather than remaining Submitted indefinitely.
>
> **Workaround:** Reconcile every original invoice number against ASP message IDs and buyer receipt. Do not resend the complete batch.
>
> **Next update due:** 14:30 Dubai time.

Customer update draft:

> Hello,
>
> Our local replay reproduced a case where the ASP accepted the initial eight documents and rate-limited the remaining requests. The affected test invoices remained Submitted without ASP references because the submission worker did not retry the rate-limited requests.
>
> This is a local reproduction, not confirmation of the original production batch's exact ASP responses. We are checking the production invoice references and delivery status. Please do not resend the full batch until we confirm which invoices were accepted and which are safe to retry.
>
> Regards,
> Support Team

### Workaround offered

Reconcile all 40 original invoice numbers against portal status, ASP message IDs and buyer receipt. Do not resend the full batch. Only retry invoices confirmed not accepted, after Engineering verifies that the retry path is safe. A Submitted invoice without an ASP reference must not be assumed accepted.

### Prevention ideas

- Explicitly handle HTTP 429 and honour Retry-After.
- Use bounded exponential backoff with jitter for retryable responses.
- Avoid marking a request as successfully submitted before an ASP acceptance response is received.
- Introduce a clear pending/retry state and a terminal failure state after retry exhaustion.
- Add monitoring for Submitted invoices without ASP references beyond a defined threshold.
- Test batches larger than the ASP burst limit and verify eventual delivery.

---

## Incident 5 — `inbox/05-ghaf-cafe-date-error.md`

### Acknowledgement

- **Sent at (exercise clock):** Draft proposed for 12:20, Monday 16 November 2026. Not sent.
- **Reply:**

> Hello Hessa,
>
> We acknowledge that your invoices were rejected with AE-BR-04 because of the issue-date format. We will verify the uploaded workbook and the portal's required date format and provide clear correction steps.
>
> We will update you by 16:10 Dubai time. The rejected invoices have not been sent to the ASP.
>
> Regards,
> Support Team

### Priority and justification

- **Priority:** P3.
- **Why:** The issue affects one customer and all invoices in the supplied workbook, but it is a portal validation rejection rather than a production outage. The customer can correct the workbook and re-upload.
- **SLA deadlines I am working to:** Received 12:10. First response within 4 business hours, by 16:10; proposed response 12:20. Next update by 16:10. Resolution target within 3 days.

### Investigation log

| Time / source | What I checked / did | What I found |
|---|---|---|
| Lab replay, 14 Nov 09:22:16 UTC | Ran `make replay CUSTOMER=ghaf` | Four invoices were processed and all four were rejected by the portal. |
| Database inspection | Queried the four invoice records | All four were Rejected by portal with AE-BR-04. |
| Payload/database inspection | Checked issue and due dates | Dates were supplied in DD/MM/YYYY format, including `13/11/2026`, `14/11/2026` and `15/11/2026`. |
| Documentation review | Checked `docs/template-field-guide.md` | The required date format is YYYY-MM-DD, either as text or as a real Excel date cell. Ambiguous date text must not be guessed. |
| Lifecycle review | Checked `docs/status-lifecycle.md` | Portal-rejected invoices are not sent to the ASP. Rejected invoice numbers can be reused after correction. |

### Root cause

- **What happened:** All four invoices in the local replay were rejected by portal validation with AE-BR-04 because their issue and due dates were supplied in DD/MM/YYYY format.
- **Where** (component, `file:line`): Portal validation flow invoked in `services/api/internal/importer/importer.go:98-125`; required date format documented in `docs/template-field-guide.md`.
- **Why it happens:** The portal requires YYYY-MM-DD dates. It rejects ambiguous DD/MM/YYYY date text rather than guessing the intended date.
- **Evidence:**
  - [Ghaf database findings](evidence/incident-5-ghaf-db.txt)
  - [Source code findings](evidence/source-code-findings.txt)
  - Replay import ID: `b780ce07-64b7-4b31-b2e5-7667227156d6`
  - Four invoices rejected by portal with AE-BR-04.
- **Confidence and open questions:** High confidence in the date-format cause for the local replay. The original customer's workbook should still be checked to confirm the exact cells and whether the dates are text values or Excel date cells.

### Customer resolution reply OR escalation note

- **Which one and why:** Customer resolution reply is appropriate because the portal is applying the documented date-format rule and the workbook can be corrected without a code change.

> Hello Hessa,
>
> We reproduced the date validation rejection. The workbook uses dates such as 13/11/2026, while the portal requires YYYY-MM-DD, for example 2026-11-13.
>
> Please correct both the issue-date and due-date columns to YYYY-MM-DD and upload the corrected workbook. If the cells contain text, convert them to the required format; if they are actual Excel dates, apply the custom display format yyyy-mm-dd.
>
> These invoices were rejected by portal validation and were not sent to the ASP. You can reuse the same invoice numbers after correcting the dates. Please do not change the invoice numbers just to bypass the validation.
>
> Regards,
> Support Team

### Workaround offered

Correct the issue-date and due-date columns to YYYY-MM-DD and re-upload the workbook. Reuse the same invoice numbers because portal-rejected invoices were not submitted to the ASP and do not consume the invoice number.

### Prevention ideas

- Improve the upload template instructions with an explicit date example.
- Validate date formats before upload and show the affected row and column.
- Distinguish actual Excel date cells from text values.
- Do not silently interpret ambiguous date text using an assumed locale.
- Add a downloadable validation report that identifies invalid date cells.

---

## Overall

### Prioritisation

I acknowledged the incidents in received-time order while applying the priority levels and response targets from the SLA matrix.

1. **Oasis Builders — P2:** Received 09:05. A large monthly upload was stuck and duplicate messages appeared. I proposed an acknowledgement at 09:20 and an update by 10:30.
2. **API restart alert — P1:** Received 09:42. Despite being the second item received, it had the highest severity because multiple customers reported 502s and invoices were delayed. I proposed a response at 09:55 and immediate L3 escalation.
3. **Al Noor — P2:** Received 10:15. ASP rejections affected a subset of invoices and the customer had an audit concern.
4. **Sahara — P2:** Received 11:30. A 40-invoice batch had missing ASP references and potential buyer/payment-run impact.
5. **Ghaf Café — P3:** Received 12:10. A single-customer workbook issue caused portal validation rejections, with a documented correction available.

The order of technical investigation was influenced by which incidents could be reproduced and supported with evidence. I fully investigated incidents 3, 4 and 5. I did not claim that local test results proved the original production incidents for Oasis, the API alert or Sahara.

### What I would do differently

- Capture original import IDs, invoice numbers and timestamps at the start of each investigation.
- Separate production evidence from local replay evidence in every incident note.
- Check portal status, ASP message ID and buyer receipt separately rather than treating Submitted as Accepted.
- Prioritise P1 escalation immediately while continuing customer communication as L1.
- Capture the exact payload and calculation inputs for financial validation errors.
- Record the next update deadline when acknowledging each incident.

### What I learned / what I would need to learn

- The invoice lifecycle: Parsed, Validated, Submitted, Accepted, Rejected and Failed, and why these states must not be treated as interchangeable.
- The difference between portal validation rejection and ASP rejection.
- How Peppol/ASP message references establish downstream acceptance.
- How HTTP 429 and Retry-After should be handled by a background submission worker.
- How floating-point and rounding differences can cause tax-total validation failures.
- How to use PostgreSQL queries, invoice event history, Docker Compose logs and source-code line references to build an evidence-backed incident report.
- I would next study production observability, distributed retry/idempotency patterns, PINT AE validation rules and safe invoice reconciliation procedures.

### Time log (real time)

| Date | From–to | What I worked on |
|---|---|---|
| 14 Nov 2026 | 06:00–07:30 | Environment setup, Docker/WSL troubleshooting and normal sample upload |
| 14 Nov 2026 | 10:00–10:40 | Incident 1 and 2 review, prioritisation and investigation |
| 14 Nov 2026 | 11:00–11:15 | Initial review of incidents 3, 4 and 5 |
| 14 Nov 2026 | 12:30–15:30 | Incident reproductions, database and source-code investigation, evidence capture and submission preparation |

**Total time:** 5 hours 25 minutes (actual working time). Do not estimate or invent this figure.

### AI usage disclosure

I used ChatGPT as an assistant to help structure the investigation, explain the repository's documented workflow, suggest PostgreSQL queries and terminal commands, interpret code paths and organise the incident findings into this report.

I checked the suggested findings against the local repository, Docker Compose output, replay results, PostgreSQL records, ASP mock logs and source-code line references. I distinguished local replay evidence from original production evidence and did not treat AI-generated hypotheses as confirmed root causes without supporting evidence.

No customer emails were sent by the AI or by these investigation steps. All customer acknowledgements and replies in this document are drafts.
