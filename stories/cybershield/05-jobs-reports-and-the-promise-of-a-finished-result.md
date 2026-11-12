# Chapter 5: Jobs, Reports, and the Promise of a Finished Result

TrustScan separates a job record from a completed report artifact. The architecture describes job state and progress markers in `trustscan_jobs`, while `trustscan_reports` stores the result, factors, evidence, metadata, and summary. The public API supports protected job submission and status retrieval, authenticated history, and a public read-only report endpoint.

That separation gives the interface a lifecycle to represent. A submitted URL is not yet a completed report; the UI can poll, show progress, and then display the report. The design documents identify retries for transient external lookups, timeout guards, and crash-recovery-oriented state persistence as reliability considerations. Those statements are not all equivalent to independently verified guarantees.

## The report-generation path

The reviewed controller's completion path gathers passive signals, builds factor evidence, calculates score and confidence, generates a summary, and persists the report. The public route retrieves a report by identifier and exposes a read-only representation. The authenticated history endpoint is a different access path from public sharing and should not be conflated with it.

The current public API reference documents:
- `POST /api/trustscan` for protected job creation;
- `GET /api/trustscan/:id` for protected status retrieval;
- `GET /api/trustscan/history` for authenticated history;
- `GET /api/trustscan/report/:id/public` for a public read-only report.

## PDF export is a rendering technique

The client PDF helper dynamically imports `html2canvas` and `jsPDF`, captures the report DOM as a PNG image, and embeds that image into a PDF. That means the PDF is image-based rather than a structured document with selectable text. Layout, cross-origin assets, long-page sizing, and rendering failures can affect the output. The presence of the helper proves that an export path exists in source; it does not prove that every report exports correctly in deployed browsers.

The known concern about report generation should therefore be tested end-to-end: submit a valid URL, poll to completion, verify the persisted report and public view, then export a long report and inspect the PDF's contents and layout. Route-level tests alone cannot validate browser rendering or live signal providers.

**Sources:** [API reference](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/engineering/API_REFERENCE.md), [TrustScan controller](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/controllers/trustScanController.js), [TrustScan report model](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/models/TrustScanReport.js), [job model](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/models/TrustScanJob.js), [PDF helper](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/client/src/utils/trustscanPdf.js), [route tests](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/tests/routes/trustscan.routes.test.js).
