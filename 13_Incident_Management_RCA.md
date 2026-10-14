# Incident Management and RCA
# Severity, updates, timeline, root cause write-up

Use this on a Sev1 or Sev2, and when a problem record asks why it happened.
Restoring service comes first. The write-up is part of done, not optional paperwork.

---

## 1. Severity in practice

| Level | Examples | How you behave |
|---|---|---|
| Sev1 | Platform down, payments or provisioning stopped for many customers | Bridge, frequent updates, restore first |
| Sev2 | One product path failing, workaround exists, or a subset of customers | Own it, update at the agreed interval |
| Sev3 / Sev4 | Single record, cosmetic, or a next-business-day defect | Normal queue |

Do not wait for a perfect root cause before saying what you know. "We see 400s from subscriber-add since 12:07, one error code, investigating that service" is a valid update.

---

## 2. Roles

- Incident lead: timeline, decisions, when to escalate.
- Technical owner: logs, DB, fixes.
- Communications: stakeholder text in plain language.
- Resolver group: the team that can change code, data, or infra.

On a small bridge you may hold two of these. Say which hat you are wearing when you speak.

---

## 3. Update rhythm

For a Sev1, post on a timer (for example every 15 minutes) even if the news is "no change, still checking the subscriber service".

Each update has four lines:

```text
Impact: who is affected and what they cannot do
Status: investigating / mitigating / monitoring
What we know: one fact with a time
Next: the check you will finish before the next update
```

Avoid blame, internal tool names the business does not know, and "maybe". If you are unsure, say what you have ruled out.

Example:

```text
Impact: new prepaid adds are failing for a subset of subscribers since 12:07.
Status: mitigating.
What we know: gateway returns 500 wrapping subscriber error CPG-SUBS-04000021 (stored data does not match the request). Platform CPU and error rate outside this code are normal.
Next: compare IMSI and external ids for the failing MSISDNs. Next update 12:40.
```

---

## 4. Timeline

Keep it in the ticket as you go. Memory after a 2-hour bridge is unreliable.

```text
12:07 first error in gateway log (host, correlation id)
12:18 paged
12:22 confirmed not a pod outage (ready 1/1, restarts 0)
12:31 inner code identified
12:55 data fix applied for MSISDN ...
13:10 same MSISDN retried successfully
13:20 error count flat
```

Times in one zone. Say which zone.

---

## 5. RCA structure

Write this after service is stable.

```text
Summary:
 one paragraph a non-engineer can read

Impact:
 start, end, number of customers or orders, financial system if known

Detection:
 alert, customer report, or a person noticed a log

Timeline:
 as above

Technical cause:
 the actual defect (data, code, config, capacity)

Why it was not stopped earlier:
 missing alert, bad timeout, no idempotency, test gap

What we did:
 workaround and permanent fix

What will prevent recurrence:
 owner and date for each action

Evidence:
 log extract, SQL, dashboard window
```

Root cause is the reason the failure happened, not the trigger. "The job was rerun" is a trigger. "The load inserts without deleting the partition" is the cause of the duplicates.

---

## 6. Prioritize when several things are on fire

Order by:

1. Customers who cannot complete a money or service action now.
2. A failing path that is still accepting traffic and making bad data.
3. A batch that can wait until the online path is stable.
4. A noisy alert that is not customer-facing.

Say the order out loud so two people do not "fix" different things.

---

## 7. Handover

If your shift ends mid-incident, the next person needs:

- current impact
- what is already ruled out
- the next command or query
- links to the dashboard, the pod, and the ticket
- any change already made (restart, data fix, scale)

A handover that says only "it's the database" forces the next person to start over.

---

## 8. Aftercare

- Leave the monitoring window open until the error rate has been flat for an agreed period.
- Create problem or defect tickets for the permanent fix. Do not leave them as a sentence inside the incident.
- If you restarted something, note it. The next RCA should not claim the process "recovered on its own".
