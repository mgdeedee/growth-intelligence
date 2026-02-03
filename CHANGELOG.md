# Changelog

## 2026-02-03

- Hardened CSV ingestion with normalization helpers, field alias resolution, numeric/date parsing, and text sanitization.
- Fixed schema drift between demo data and analytics logic by generating linked leads, clients, appointments, and transactions with expected keys.
- Added operational enrichment via appointments and transactions (`computeOperationalMetrics`) and integrated it into processing flow.
- Replaced vulnerable location dropdown `innerHTML` option rendering with safe DOM node creation.
- Escaped user-derived fields in churn-risk table rendering to prevent HTML/script injection.
- Upgraded funnel metrics to use appointment-derived booked counts and transaction-derived paid counts when available.
- Updated 90-day channel revenue KPI to use transaction-derived first-90-day revenue with fallback logic.
- Unified channel parsing to support both `channel` and `lead_source` inputs across the dashboard.

