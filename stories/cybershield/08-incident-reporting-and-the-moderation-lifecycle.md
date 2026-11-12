# Chapter 8: Incident Reports Are a Workflow, Not a Form

TrustScan is the newest center of gravity in CyberShield, but the older incident-reporting system solves a different problem. A scan begins with a URL and produces a technical assessment. An incident report begins with a person's account of something that happened, potentially accompanied by evidence, contact details, anonymity preferences, and a sensitivity flag. Treating these as interchangeable would erase the difference between a signal and an allegation.

## From browser form to persisted record

The client form in [CreateReport.jsx](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/client/src/pages/reports/CreateReport.jsx) collects a title, description, category, subcategory, severity, source channel, optional contact email, evidence file, and the anonymous/sensitive flags. Category changes reset the subcategory to a valid option for that category. The AI detector can also prefill the form with a classification result and the text that was analyzed. That shortens the path from suspicion to a report, but the prefill is a convenience, not proof that the content is malicious.

The API's [createReport controller](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/controllers/reportController.js) validates the request, interprets the flags, encrypts the description when marked sensitive, validates and persists an optional evidence file, then creates a record with an initial `SUBMITTED` status and history entry. Anonymous reports omit the user reference. That is a data-model decision with consequences for attribution and follow-up, not merely a checkbox's visual state.

After the report is created, the controller attempts several side effects: a notification, XP and coin rewards for an authenticated submitter, and an analytics increment. These are handled with `Promise.allSettled`, so a failure in one side effect does not automatically undo the primary report. That is a useful availability trade-off: the incident record is the main operation, while rewards and counters are secondary. It also means the system needs logs and reconciliation if a side effect fails, because the response can still represent a successfully created report.

If an error occurs after evidence has been persisted but before the request completes, the controller attempts to delete the uploaded file. Cleanup itself can fail, and the failure is logged. This is a small but important lifecycle detail: uploads are external side effects around a database write, and the two do not form one atomic transaction.

## Public visibility versus private detail

The public reports feed applies a dedicated serializer rather than returning the database record verbatim. Sensitive descriptions are replaced with a generic message, and the public projection omits fields such as the submitter reference, contact email, and evidence path. The report model also records anonymity and sensitivity separately: anonymity concerns attribution, while sensitivity concerns the content.

The distinction matters. A report can be attributed but sensitive, or anonymous without necessarily containing sensitive material. Public sharing should not be implemented as a single boolean that accidentally grants access to every field.

The API reference documents public listing, personal report listing, and administrative report management as separate paths. Those boundaries need to be reviewed alongside the actual serializers and authorization middleware; a route label alone is not a privacy guarantee.

## Moderation and status history

Reports carry status and history fields so the product can represent a lifecycle beyond submission. The admin controller contains a set of states considered pending for dashboard counts, while the report model records status history. The available evidence supports a moderation workflow, but does not establish that every status transition is enforced by a single formal state machine. That distinction should remain explicit rather than upgrading a collection of status strings into a stronger guarantee than the code provides.

The admin report list also has a bounded page size and fetches report data and total count in parallel. Pagination reduces response size, but the controller comments note that some filters still need to move into the MongoDB query instead of being applied after retrieval. This is a useful example of a system evolving from correctness toward scale: limiting the response is not the same as limiting all database work.

## The engineering lesson

Incident reporting crosses several trust boundaries at once: user-provided narratives, potentially dangerous files, personal information, public projections, moderation decisions, and reward side effects. The design is strongest when each boundary has an explicit policy and a test. The important question is not just whether a report can be submitted, but what survives a partial failure, who can see each field, and how a moderator's decision is represented.

**Evidence limits:** This chapter describes the reviewed source snapshot, not a live privacy audit. The documentation pass did not upload files, exercise report creation end-to-end, or verify every public/private API response.

**Sources:** [Report controller](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/controllers/reportController.js), [report model](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/models/Report.js), [report creation UI](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/client/src/pages/reports/CreateReport.jsx), [API reference](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/engineering/API_REFERENCE.md).