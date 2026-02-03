# MoeGo Growth Intelligence — Metrics Definition Contract v2.1

**Version:** 2.1
**Last Updated:** 2026-02-01
**Owner:** Growth Intelligence System
**Status:** Production (Documenting Existing Dashboard)

---

## Overview

This document reflects the **actual compute logic** in the existing MoeGo Growth Intelligence Dashboard (5,949 lines). No changes to dashboard code — documentation only.

---

## 1. Data Architecture

### 1.1 Global Data Objects

```javascript
// Primary data stores
const DATA = {
    leads: [],           // Raw lead records
    clients: [],         // Raw client records
    appointments: [],    // Raw appointment records
    transactions: []     // Raw transaction records (if available)
};

// Computed/derived data
const COMPUTED = {
    customerMap: new Map(),     // customer_id → enriched customer object
    channelMetrics: [],         // Array of channel performance objects
    segments: [],               // Customer value segments
    cohortData: [],             // Monthly cohort metrics
    funnelData: {},             // Funnel stage counts
    locationStats: []           // Location performance metrics
};

// User-applied filters
const FILTERS = {
    location: 'all',            // Selected location filter
    timeRange: 90               // Days to look back (30/90/180/365)
};
```

### 1.2 Customer Enrichment (from `processClients()`)

```javascript
// Each customer in COMPUTED.customerMap has:
{
    customer_id: string,
    total_spend_12m: number,       // Parsed from string
    visits_12m: number,
    days_since_last: number,       // days_since_last_visit_12m
    apv: number,                   // Average per visit
    had_boarding: boolean,         // had_boarding_12m === '1'
    had_daycare: boolean,
    had_grooming: boolean,
    location: string,              // business_name
    converted_on: string|null,     // From lead linkage
    channel: string|null           // From lead linkage
}
```

---

## 2. Channel Intelligence

### 2.1 Channel Normalization (`normalizeChannel()`)

| Raw Input | Normalized Output |
|-----------|-------------------|
| `'ob'`, `'online booking'`, `'online'` | `'Online Booking'` |
| `'call'`, `'phone call'`, `'text'`, `'phone'`, `'sms'` | `'Phone'` |
| `'walk-in'`, `'walkin'`, `'walk in'`, `'manual'` | `'Walk-in'` |
| `'if'`, `'instagram/facebook'`, `'instagram'`, `'facebook'` | `'Paid Social'` |
| `'google'`, `'search'`, `'sem'`, `'ppc'` | `'Google/Search'` |
| `'referral'`, `'ref'`, `'word of mouth'` | `'Referral'` |
| `null`, `undefined`, others | `'Unknown'` |

### 2.2 Channel Metrics Calculation (`computeChannelMetrics()`)

```javascript
// Per-channel computed metrics:
{
    channel: string,
    leads: number,                  // COUNT(leads WHERE channel = X)
    converted: number,              // COUNT(leads WHERE converted_status = 'Converted')
    totalFirstOrder: number,        // SUM(converted_first_service_dollar)
    total90DRevenue: number,        // SUM(customer.total_spend_12m) for converted leads
    totalLTV: number,               // Same as above (proxy)
    retained90d: number,            // COUNT(customers WHERE days_since_last <= 90)

    // Derived rates:
    convRate: number,               // converted / leads * 100
    retention90d: number,           // retained90d / converted * 100
    revPerLead: number,             // total90DRevenue / leads  ⭐ PRIMARY KPI
    avgLTV: number,                 // totalLTV / converted
    qualityScore: number            // Composite score (see below)
}
```

### 2.3 Quality Score Formula (`calculateQualityScore()`)

```javascript
function calculateQualityScore(stats, maxRevPerLead, maxConvRate, maxRetention) {
    if (stats.channel === 'Unknown') return '-';
    if (stats.leads === 0) return 0;

    // Normalize each metric to 0-100 scale relative to best performer
    const revScore = maxRevPerLead > 0 ? (stats.revPerLead / maxRevPerLead * 100) : 0;
    const convScore = maxConvRate > 0 ? (stats.convRate / maxConvRate * 100) : 0;
    const retScore = maxRetention > 0 ? (stats.retention90d / maxRetention * 100) : 0;

    // Weighted composite: 40% Rev/Lead + 30% Conv Rate + 30% Retention
    return Math.round(revScore * 0.4 + convScore * 0.3 + retScore * 0.3);
}
```

**Sort Order:** Channels sorted by `revPerLead` descending, `Unknown` always last.

---

## 3. Customer Value Segments

### 3.1 Segment Definitions (`computeSegments()`)

| Segment | Condition | Color |
|---------|-----------|-------|
| **VIP ($500+)** | `total_spend_12m >= 500` | `#22c55e` (green) |
| **Regular ($100-500)** | `total_spend_12m >= 100 AND < 500` | `#3b82f6` (blue) |
| **Light ($1-100)** | `total_spend_12m > 0 AND < 100` | `#f59e0b` (amber) |
| **Converted Only** | `total_spend_12m === 0` | `#94a3b8` (slate) |

### 3.2 Service Mix Flags

```javascript
const had_boarding = customer.had_boarding_12m === '1';
const had_daycare = customer.had_daycare_12m === '1';
const had_grooming = customer.had_grooming_12m === '1';
```

### 3.3 Activity Status

| Status | Condition |
|--------|-----------|
| Active | `days_since_last <= 90` |
| At Risk | `days_since_last > 90 AND <= 120` |
| Churned | `days_since_last > 120` |

---

## 4. Risk Detection Engine

### 4.1 Risk Definitions (`computeGrowthRisks()`)

| Risk ID | Title | Trigger Condition | Impact Logic |
|---------|-------|-------------------|--------------|
| `channel-bottleneck` | Manual Channel Bottleneck | `manualChannelPct > 50%` | High if >60%, else Medium |
| `speed-to-lead` | Speed-to-Lead Failure | `staleLeadsPct > 15%` | High if >25%, else Medium |
| `retention-decay` | Single-Service Retention Decay | `groomingOnlyPct > 30% AND groomingRetention < 40%` | High |
| `conversion-gap` | Customer → Paid Conversion Gap | `customerToPaidRate < 70% AND convertedCount >= 10` | High if <50%, else Medium |
| `revenue-concentration` | Revenue Concentration Risk | `top20CustomersRevenue > 70%` | Medium |
| `location-variance` | Location Performance Variance | `maxConvRate - minConvRate > 20pp` | High if >30pp, else Medium |

### 4.2 Stale Lead Definition

```javascript
function isStale(lead) {
    if (lead.converted_status === 'Converted') return false;

    const age = Math.floor((new Date() - new Date(lead.lead_create_time)) / (1000 * 60 * 60 * 24));
    const hasContact = lead.log_type_count && (
        lead.log_type_count.includes('MESSAGE:') ||
        lead.log_type_count.includes('EMAIL:') ||
        lead.log_type_count.includes('CALL:')
    );

    return age > 3 && !hasContact;
}
```

### 4.3 Customer → Paid Rate Calculation (THREE-LAYER MODEL)

```javascript
// CRITICAL: This is the conversion gap detection
const convertedCount = leads.filter(l => l.converted_status === 'Converted').length;
const payingCount = customers.filter(c => c.total_spend_12m > 0).length;
const customerToPaidRate = convertedCount > 0 ? payingCount / convertedCount * 100 : 100;

// Trigger risk if < 70% and at least 10 converted customers
if (customerToPaidRate < 70 && convertedCount >= 10) {
    // Add 'conversion-gap' risk
}
```

---

## 5. Leverage Opportunities Engine

### 5.1 Opportunity Definitions (`computeLeverageOpportunities()`)

| Opportunity ID | Addresses Risk | Condition | Upside Calculation |
|----------------|----------------|-----------|-------------------|
| `channel-shift` | `channel-bottleneck` | Online conv > Manual conv | `manualLeads × convLift × avgLTV` |
| `speed-sla` | `speed-to-lead` | `staleLeads > 10` | `staleLeads × 15% recovery rate` |
| `cross-sell` | `retention-decay` | `groomingOnlyActive >= 10` | `pool × 20% target × $200 boarding` |

---

## 6. Location Metrics

### 6.1 Location Stats (`computeLocationStats()`)

```javascript
// Per-location computed metrics:
{
    location: string,
    leads: number,
    converted: number,
    customers: number,
    revenue: number,                // SUM(customer.total_spend_12m)
    avgLTV: number,                 // revenue / customers
    convRate: number,               // converted / leads * 100
    atRiskCount: number,            // customers with days_since > 60
    activeRate: number              // customers with days_since <= 90 / total
}
```

---

## 7. Cohort Analysis

### 7.1 Cohort Computation (`computeCohorts()`)

```javascript
// Group customers by conversion month
const cohortMonth = customer.converted_on
    ? customer.converted_on.substring(0, 7)  // 'YYYY-MM'
    : 'Unknown';

// Per-cohort metrics:
{
    month: string,                  // 'YYYY-MM'
    count: number,                  // Customers in cohort
    avgLTV: number,                 // Average total_spend_12m
    retention: number               // % with days_since_last <= 90
}
```

---

## 8. Filtering System

### 8.1 Filter Application

```javascript
function getFilteredLeads() {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - FILTERS.timeRange);

    return DATA.leads.filter(l => {
        // Location filter
        if (FILTERS.location !== 'all' && l.company_name !== FILTERS.location) return false;

        // Time filter on lead_create_time
        if (l.lead_create_time) {
            if (new Date(l.lead_create_time) < cutoffDate) return false;
        }

        return true;
    });
}

function getFilteredCustomers() {
    return Array.from(COMPUTED.customerMap.values()).filter(c => {
        // Location filter
        if (FILTERS.location !== 'all' && c.location !== FILTERS.location) return false;

        // Time filter on converted_on
        if (c.converted_on) {
            if (new Date(c.converted_on) < cutoffDate) return false;
        }

        return true;
    });
}
```

---

## 9. Executive Scorecard Metrics

### 9.1 Overview Metrics (rendered in `renderOverview()`)

| Metric | Calculation | Source |
|--------|-------------|--------|
| New Leads | `filteredLeads.length` | Leads data |
| Converted | `filteredLeads.filter(l => l.converted_status === 'Converted').length` | Leads data |
| Paying Customers | `customers.filter(c => c.total_spend_12m > 0).length` | Clients data |
| Lead → Customer Rate | `converted / leads × 100` | Derived |
| Customer → Paid Rate | `paying / converted × 100` | Derived |
| Total Revenue (12M) | `SUM(customer.total_spend_12m)` | Clients data |
| 90D Active Rate | `customers.filter(c => c.days_since_last <= 90).length / total × 100` | Clients data |

---

## 10. Benchmark Thresholds

| Metric | Healthy | Watch | At Risk |
|--------|---------|-------|---------|
| Lead → Customer Rate | ≥30% | 20-30% | <20% |
| Customer → Paid Rate | ≥70% | 50-70% | <50% |
| 90D Active Rate | ≥50% | 30-50% | <30% |
| Channel Quality Score | ≥70 | 40-70 | <40 |
| Speed-to-Lead (3 days) | ≥85% | 75-85% | <75% |

---

## 11. Data Field Reference

### leads table (from CSV)
```
lead_id, lead_name, phone, lead_create_time, converted_status,
life_cycle, action_state_name, converted_on, converted_via,
referral_source, channel, pet_name, pet_type, pet_breed,
task_count, log_type_count, convert_to_customer_id,
converted_trigger, converted_source_id, first_order_id,
first_order_source_type, converted_first_service_type,
converted_first_service_dollar, appts_till_today_cnt,
total_appt_revenue, company_id, company_name
```

### clients table (from CSV)
```
business_id, business_name, customer_id, first_name, last_name,
is_active, last_service_time, utc_create_time, had_boarding_12m,
had_daycare_12m, had_grooming_12m, had_evaluation_12m,
total_spend_12m, visits_12m, add_on_gross_sale_total_12m,
add_on_attach_rate_12m, visits_last_30_days, visits_last_90_days,
apv_12m, first_visit_date_12m, last_visit_date_12m,
days_since_last_visit_12m
```

---

## Appendix: Change Log

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-01-27 | Initial release |
| 2.0 | 2026-01-27 | Three-layer model, dual conversion rates, AI constraints |
| 2.1 | 2026-02-01 | **Documentation only:** Aligned contract with actual dashboard code. No code changes. |

---

**Document End**

This contract documents the existing implementation. Dashboard code remains unchanged.
