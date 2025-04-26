---
# Front‑matter

title: "Advanced‑Materials Procurement Electronic Logs – REST API"
version: "0.2.0‑draft"
status: "In‑Work"
date:   "2025‑04‑26"
domain: "GAIA‑Platforms :: AMPEL‑3ELDB"
author: "Amedeo Pelliccia"
license: "Apache‑2.0"
---

## 1 Overview

The **AMPEL API** provides machine‑readable end‑points for managing advanced‑materials procurement, full traceability and sustainability metrics across **GAIA** projects (**AIR · SPACE · GREENTECH**).

| Key Item | Value |
|----------|-------|
| **Protocol** | REST + JSON over HTTPS |
| **Base URL** | `https://api.gaia-platforms.com/v1` |
| **Auth** | OAuth 2.1 (Bearer) · *or* GitHub‑App JWT |
| **Rate‑limit** | 900 req / h / token (burst 100) |

> All payloads conform to the canonical **COAFI JSON schema** (see § 6) and reference the common **`DES‑ID` / `PN‑ID` / `UUID`** keys used in CD‑, GT‑ and SD‑COATFI metadata files.

---

## 2 Authentication

```http
POST /v1/oauth/token
Content‑Type: application/x-www-form-urlencoded

grant_type=client_credentials&
client_id=GAIA_CLIENT_ID&
client_secret=GAIA_CLIENT_SECRET&
scope=ampel.read ampel.write
```

Successful response:

```json
{
  "access_token": "eyJhbGciOiJSUzI1...",
  "token_type"  : "Bearer",
  "expires_in"  : 3600,
  "scope"       : "ampel.read ampel.write"
}
```

Attach the token to every request:

```http
Authorization: Bearer <token>
```

---

## 3 Key Resources & End‑points

| Resource                | Verb  | Path                              | Purpose |
|-------------------------|-------|-----------------------------------|---------|
| **Materials**           | GET   | `/materials`                      | List / filter materials |
|                         | POST  | `/materials`                      | Create material record |
|                         | GET   | `/materials/{matId}`              | Retrieve material |
|                         | PATCH | `/materials/{matId}`              | Update material |
| **Purchase Orders**     | GET   | `/pos`                            | List POs |
|                         | POST  | `/pos`                            | Create PO + lines |
|                         | GET   | `/pos/{poId}`                     | Detailed PO |
| **Sustainability**      | GET   | `/lca/{uuid}`                     | Fetch cradle‑to‑gate CO₂e & water footprint |
| **Certificates**        | POST  | `/certificates`                   | Upload mill/REACH/RoHS PDF (SHA‑256 return) |
| **Webhooks**            | POST  | `/webhooks`                       | Register callback (e.g. `po.statusChanged`) |

### 3.1 Example – Register a new material

```http
POST /v1/materials
Content‑Type: application/json

{
  "uuid"       : "4794c8b0-93ca-4fd2-8d9c-51f084e66dfe",
  "pnId"       : "PN-6501-BIO-01",
  "title"      : "Flax‑fibre / bio‑epoxy laminate panel",
  "category"   : "BIO-MAT",
  "spec"       : "GT-COMP-6501-03",
  "supplier"   : "GreenMat S.p.A.",
  "unit"       : "m^2",
  "unitCost"   : 42.50,
  "currency"   : "EUR",
  "co2eKg"     : 5.8,
  "recycledPct": 30,
  "status"     : "qualified"
}
```

`201 Created` → `Location: /materials/4794c8b0-93ca-...`

---

## 4 Query Parameters & Filtering

| Parameter       | Type            | Example                           | Notes |
|-----------------|-----------------|-----------------------------------|-------|
| `q`             | full‑text       | `q=flax`                          | Search title / spec |
| `category`      | enum            | `category=BIO-MAT`                | See schema enum |
| `status`        | enum            | `status=qualified`                | `draft | qualified | obsolete` |
| `createdAfter`  | ISO 8601 date   | `2025-01-01T00:00:00Z`            | Filter by creation date |
| `limit` / `offset` | integer      | `limit=50&offset=0`               | Pagination |

---

## 5 Webhooks

```http
POST /webhooks
Content‑Type: application/json

{
  "event" : "po.statusChanged",
  "target": "https://build.server/hooks/gaia-air",
  "secret": "SUP3R-S3CR3T"
}
```

### Payload example

```json
{
  "event"    : "po.statusChanged",
  "timestamp": "2025-04-26T14:12:00Z",
  "poId"     : "PO-2025-00432",
  "oldStatus": "OPEN",
  "newStatus": "RECEIVED",
  "signature": "sha256=7a9d3f..."
}
```

Verify `X‑GAIA‑Signature` (HMAC‑SHA‑256) with your stored secret.

---

## 6 Canonical JSON Schema (Extract)

```jsonc
{
  "$schema" : "https://json-schema.org/draft/2020-12/schema",
  "$id"     : "https://gaia-platforms.com/schema/ampel/material.json",
  "title"   : "Material",
  "type"    : "object",
  "required": ["uuid","pnId","title","category","status"],
  "properties": {
    "uuid"       : {"type":"string","format":"uuid"},
    "pnId"       : {"type":"string","pattern":"^PN-[0-9A-Z-]+$"},
    "title"      : {"type":"string","maxLength":120},
    "category"   : {"enum":["AI-ML","ADV-MAT","SUST-TECH","DIG-TWIN","IOT-SENS","ENERGY","BIO-MAT","HYDROGEN","BAT-SOLID","CCUS"]},
    "status"     : {"enum":["draft","qualified","obsolete"]},
    "unit"       : {"type":"string"},
    "unitCost"   : {"type":"number","minimum":0},
    "currency"   : {"type":"string","pattern":"^[A-Z]{3}$"},
    "co2eKg"     : {"type":"number","minimum":0},
    "recycledPct": {"type":"number","minimum":0,"maximum":100}
  }
}
```

---

## 7 Error Handling

| Code | Meaning            | Typical Cause                          |
|------|--------------------|----------------------------------------|
| 400  | Bad request        | Invalid JSON / missing field           |
| 401  | Unauthorized       | Missing / expired token                |
| 403  | Forbidden          | Scope does not include resource        |
| 404  | Not found          | Unknown `uuid` / `id`                  |
| 409  | Conflict           | Duplicate `uuid` or immutable field    |
| 429  | Rate‑limit exceeded| Respect `Retry‑After` header           |
| 500  | Server error       | Internal log reference in body         |

---

## 8 Changelog

| Version | Date         | Notes                                                 |
|---------|--------------|-------------------------------------------------------|
| 0.2.0‑d | 2025‑04‑26   | Initial draft – resources, schema extract & webhooks |

---

**Feedback / MR →** <https://github.com/Robbbo-t/GAIA-Platforms/issues>

---

# Appendix A – Aerospace **Material** Printable Electronics Lot *(concept outline)*

*(Only a concise, corrected abstract is retained here; move in‑depth content to a separate white‑paper if required.)*

**Goal:** Embed printable electronics (sensors, actuators, antennas) directly onto/into aerospace‑grade substrates, creating "smart" lightweight structures with real‑time SHM, adaptive control and autonomous comms.

* **Printable electronics:** inkjet / aerosol‑jet conductive inks, polymer semiconductors; encapsulated for – 55 °C ➔ +120 °C, UV & vibration.
* **Primary use‑cases:**
  * Structural‑health‑monitoring meshes in CFRP wings.
  * Conformal L‑band antennas on fuselage skins.
  * Thin‑film thermoelectric harvesters for waste‑heat recovery.
* **Key hurdles:** substrate/ink adhesion, thermo‑mechanical mismatch, qualification (DO‑160G / CS‑25.609).
* **Next steps:** joint ASTM F42 + SAE AMS round‑robin to define acceptance metrics.

---

# Appendix B – Minimum Metadata Fields (Extended)

```
technologyReadinessLevel   # TRL 0‑9
commercialReadinessIndex   # CRI 0‑5
riskLikelihood            : low | medium | high
riskImpact                : minor | major | catastrophic
certificationChallenge    : string
mitigationOwner           : email / GitHub @handle
```

---

*Document auto‑generated via GenAI assistant — review before commit.*

