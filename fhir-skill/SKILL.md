---
name: fhir-developer
description: >
  Use this skill when working with: FHIR, HL7, healthcare APIs, Patient, Observation,
  Medication, MedicationRequest, Encounter, Condition, Bundle, CapabilityStatement,
  OperationOutcome, LOINC, SNOMED, RxNorm, SMART on FHIR, OAuth scopes, or any 
  healthcare/medical API development.

  This skill provides FHIR domain knowledge: resource structures, cardinality rules
  (required vs optional fields), coding systems, HTTP status codes, SMART on FHIR
  authorization, pagination, and bundle operations.
---

# FHIR Developer Skill

## Quick Reference

### HTTP Status Codes
| Code | When to Use |
|------|-------------|
| `200 OK` | Successful read, update, or search |
| `201 Created` | Successful create |
| `204 No Content` | Successful delete |
| `400 Bad Request` | Missing required fields, malformed JSON |
| `401 Unauthorized` | Missing or invalid access token |
| `403 Forbidden` | Valid token but insufficient scopes |
| `404 Not Found` | Resource doesn't exist |
| `412 Precondition Failed` | If-Match ETag mismatch (NOT 400!) |
| `422 Unprocessable Entity` | Invalid enum values, business rule violations |

### Required Fields by Resource (FHIR R4)
| Resource | Required Fields | Everything Else |
|----------|-----------------|-----------------|
| Patient | *(none)* | All optional |
| Observation | `status`, `code` | Optional |
| Encounter | `status`, `class` | Optional (including `subject`, `period`) |
| Condition | `subject` | Optional (including `code`, `clinicalStatus`) |
| MedicationRequest | `status`, `intent`, `medication[x]`, `subject` | Optional |
| Medication | *(none)* | All optional |
| Bundle | `type` | Optional |

---

## Overview

FHIR (Fast Healthcare Interoperability Resources) is the HL7 standard for healthcare data exchange using:
- **Resources**: Modular building blocks (Patient, Observation, Medication, etc.)
- **RESTful API**: HTTP-based CRUD operations
- **JSON format**: Preferred over XML

**FHIR R4** (v4.0.1) is the most widely deployed version.

---

## Required vs Optional Fields (CRITICAL)

**Only validate fields with cardinality starting with "1" as required.** All other fields are optional.

| Cardinality | Meaning | Required? |
|-------------|---------|-----------|
| `0..1` | Optional, at most one | NO |
| `0..*` | Optional, any number | NO |
| `1..1` | Required, exactly one | YES |
| `1..*` | Required, at least one | YES |

**Common mistake**: Making `subject` or `period` required on Encounter. They are 0..1 (optional).

**When in doubt**: Check the official spec at `https://hl7.org/fhir/R4/[resourcetype].html` - look for fields with cardinality starting with "1" (required) vs "0" (optional).

---

## Value Sets (Required Enum Values)

When a field has a required binding to a value set, invalid values must return `422 Unprocessable Entity`.

### Patient.gender
```
male | female | other | unknown
```

### Observation.status
```
registered | preliminary | final | amended | corrected | cancelled | entered-in-error | unknown
```

### Encounter.status
```
planned | arrived | triaged | in-progress | onleave | finished | cancelled | entered-in-error | unknown
```

### Encounter.class (Common Codes)
| Code | Display | Use |
|------|---------|-----|
| `AMB` | ambulatory | Outpatient visits |
| `IMP` | inpatient encounter | Hospital admissions |
| `EMER` | emergency | Emergency department |
| `VR` | virtual | Telehealth |

### Condition.clinicalStatus
```
active | recurrence | relapse | inactive | remission | resolved
```

### Condition.verificationStatus
```
unconfirmed | provisional | differential | confirmed | refuted | entered-in-error
```

### MedicationRequest.status
```
active | on-hold | cancelled | completed | entered-in-error | stopped | draft | unknown
```

### MedicationRequest.intent
```
proposal | plan | order | original-order | reflex-order | filler-order | instance-order | option
```

### Bundle.type
```
document | message | transaction | transaction-response | batch | batch-response | history | searchset | collection
```

---

## Validation Implementation (CRITICAL)

**You MUST validate required fields and reject invalid requests.** Do not simply accept any input.

### Missing Required Fields → Return 400

**Python/FastAPI Example:**
```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse

app = FastAPI()

def operation_outcome(severity: str, code: str, diagnostics: str):
    return {
        "resourceType": "OperationOutcome",
        "issue": [{"severity": severity, "code": code, "diagnostics": diagnostics}]
    }

@app.post("/Observation", status_code=201)
async def create_observation(data: dict):
    # VALIDATE REQUIRED FIELDS
    if not data.get("status"):
        return JSONResponse(status_code=400, content=operation_outcome(
            "error", "required", "Observation.status is required"
        ), media_type="application/fhir+json")
    
    if not data.get("code"):
        return JSONResponse(status_code=400, content=operation_outcome(
            "error", "required", "Observation.code is required"
        ), media_type="application/fhir+json")
    
    # VALIDATE ENUM VALUES
    valid_statuses = {"registered", "preliminary", "final", "amended", 
                      "corrected", "cancelled", "entered-in-error", "unknown"}
    if data["status"] not in valid_statuses:
        return JSONResponse(status_code=422, content=operation_outcome(
            "error", "value", f"Invalid status '{data['status']}'. Must be one of: {', '.join(valid_statuses)}"
        ), media_type="application/fhir+json")
    
    # ... proceed with creating the resource
```

**TypeScript/Express Example:**
```typescript
import express, { Request, Response } from 'express';

const app = express();
app.use(express.json());

const operationOutcome = (severity: string, code: string, diagnostics: string) => ({
  resourceType: 'OperationOutcome',
  issue: [{ severity, code, diagnostics }]
});

const VALID_OBSERVATION_STATUS = new Set([
  'registered', 'preliminary', 'final', 'amended',
  'corrected', 'cancelled', 'entered-in-error', 'unknown'
]);

app.post('/Observation', (req: Request, res: Response) => {
  const data = req.body;

  // VALIDATE REQUIRED FIELDS
  if (!data.status) {
    return res.status(400).contentType('application/fhir+json')
      .json(operationOutcome('error', 'required', 'Observation.status is required'));
  }
  if (!data.code) {
    return res.status(400).contentType('application/fhir+json')
      .json(operationOutcome('error', 'required', 'Observation.code is required'));
  }

  // VALIDATE ENUM VALUES
  if (!VALID_OBSERVATION_STATUS.has(data.status)) {
    return res.status(422).contentType('application/fhir+json')
      .json(operationOutcome('error', 'value', 
        `Invalid status '${data.status}'. Must be one of: ${[...VALID_OBSERVATION_STATUS].join(', ')}`));
  }

  // ... proceed with creating the resource
});
```

### When to Use 400 vs 422

- **400 Bad Request**: Missing required fields, malformed JSON structure, missing resourceType
- **422 Unprocessable Entity**: Invalid enum values, business rule violations, value set binding failures

---

## SMART on FHIR Authorization

SMART on FHIR is the standard authorization framework for FHIR APIs, built on OAuth 2.0.

### Discovery Endpoint

Servers MUST publish their authorization configuration at:
```
GET /.well-known/smart-configuration
```

**Response:**
```json
{
  "authorization_endpoint": "https://auth.example.org/authorize",
  "token_endpoint": "https://auth.example.org/token",
  "token_endpoint_auth_methods_supported": ["client_secret_basic", "private_key_jwt"],
  "scopes_supported": ["openid", "fhirUser", "launch", "launch/patient", 
                       "patient/*.rs", "user/*.cruds", "offline_access"],
  "capabilities": ["launch-ehr", "launch-standalone", "client-public", 
                   "client-confidential-symmetric", "permission-v2", "sso-openid-connect"]
}
```

### Scope Syntax

**SMART v2 (Current Standard):**
```
<context>/<resource>.<permissions>

context: patient | user | system
resource: Patient | Observation | * (wildcard)
permissions: c (create) | r (read) | u (update) | d (delete) | s (search)
```

**Examples:**
| Scope | Meaning |
|-------|---------|
| `patient/Patient.rs` | Read and search Patient for current patient context |
| `patient/Observation.rs` | Read and search Observations for current patient |
| `patient/*.rs` | Read and search all resources for current patient |
| `user/Encounter.cruds` | Full CRUD + search on Encounters the user can access |
| `system/*.rs` | Backend service: read/search all resources |

**SMART v1 (Legacy - still widely used):**
```
<context>/<resource>.<read|write|*>
```
| v1 Scope | Equivalent v2 |
|----------|---------------|
| `patient/Patient.read` | `patient/Patient.rs` |
| `patient/Patient.write` | `patient/Patient.cud` |
| `patient/Patient.*` | `patient/Patient.cruds` |

### Scope Types

| Scope Type | Use Case | Context Required |
|------------|----------|------------------|
| `patient/` | Patient-facing apps | `patient` launch context |
| `user/` | Provider-facing apps | User's access permissions |
| `system/` | Backend services | Pre-configured policy |

### Launch Context Scopes

| Scope | Provides | In Token Response |
|-------|----------|-------------------|
| `launch` | EHR launch context | `patient`, `encounter` |
| `launch/patient` | Standalone patient selection | `patient` |
| `launch/encounter` | Standalone encounter selection | `encounter` |
| `openid fhirUser` | User identity | `id_token` with `fhirUser` claim |
| `offline_access` | Refresh token (persistent) | `refresh_token` |
| `online_access` | Refresh token (session-bound) | `refresh_token` |

### Authorization Flow (EHR Launch)

```
1. EHR redirects to app with launch parameter:
   GET https://app.example.com/launch?iss=https://fhir.hospital.org&launch=abc123

2. App discovers authorization endpoints:
   GET https://fhir.hospital.org/.well-known/smart-configuration

3. App redirects to authorization:
   GET https://auth.hospital.org/authorize?
     response_type=code&
     client_id=my-app&
     redirect_uri=https://app.example.com/callback&
     scope=launch patient/Patient.rs patient/Observation.rs&
     state=xyz789&
     aud=https://fhir.hospital.org&
     launch=abc123

4. User authenticates & authorizes

5. Authorization server redirects back:
   GET https://app.example.com/callback?code=auth_code_here&state=xyz789

6. App exchanges code for token:
   POST https://auth.hospital.org/token
   Content-Type: application/x-www-form-urlencoded

   grant_type=authorization_code&
   code=auth_code_here&
   redirect_uri=https://app.example.com/callback&
   client_id=my-app

7. Token response includes context:
   {
     "access_token": "eyJ...",
     "token_type": "Bearer",
     "expires_in": 3600,
     "scope": "launch patient/Patient.rs patient/Observation.rs",
     "patient": "123",
     "encounter": "456"
   }
```

### Backend Services (System-to-System)

For automated systems without user interaction:

```python
import jwt
import time
import requests

# 1. Create signed JWT assertion
now = int(time.time())
claims = {
    "iss": CLIENT_ID,
    "sub": CLIENT_ID,
    "aud": TOKEN_ENDPOINT,
    "exp": now + 300,
    "jti": str(uuid.uuid4())
}
assertion = jwt.encode(claims, PRIVATE_KEY, algorithm="RS384")

# 2. Exchange for access token
response = requests.post(TOKEN_ENDPOINT, data={
    "grant_type": "client_credentials",
    "scope": "system/*.rs",
    "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
    "client_assertion": assertion
})
token = response.json()["access_token"]

# 3. Use token in FHIR requests
headers = {"Authorization": f"Bearer {token}"}
patients = requests.get(f"{FHIR_BASE}/Patient", headers=headers)
```

### Enforcing Scopes in Your Server

**Python/FastAPI Middleware:**
```python
from fastapi import Request, HTTPException
import re

def parse_smart_scopes(token_scopes: str) -> dict:
    """Parse SMART scopes into a permissions structure."""
    permissions = {}
    for scope in token_scopes.split():
        # Match: patient/Patient.rs, user/*.cruds, system/Observation.r
        match = re.match(r'(patient|user|system)/(\w+|\*)\.([cruds]+|read|write|\*)', scope)
        if match:
            context, resource, perms = match.groups()
            # Convert v1 to v2
            if perms == 'read': perms = 'rs'
            elif perms == 'write': perms = 'cud'
            elif perms == '*': perms = 'cruds'
            
            if resource not in permissions:
                permissions[resource] = set()
            permissions[resource].update(perms)
    return permissions

async def check_scope(request: Request, resource_type: str, action: str):
    """Verify the token has required scope for the action."""
    token_scopes = request.state.token_scopes  # Set by auth middleware
    permissions = parse_smart_scopes(token_scopes)
    
    action_map = {'create': 'c', 'read': 'r', 'update': 'u', 'delete': 'd', 'search': 's'}
    required = action_map.get(action, action)
    
    # Check specific resource or wildcard
    allowed = permissions.get(resource_type, set()) | permissions.get('*', set())
    
    if required not in allowed:
        raise HTTPException(
            status_code=403,
            detail=f"Insufficient scope. Required: {resource_type}.{required}"
        )

# Usage in endpoint
@app.get("/Patient/{id}")
async def read_patient(id: str, request: Request):
    await check_scope(request, "Patient", "read")
    # ... return patient
```

### Common Authorization Errors

| Status | Error | When |
|--------|-------|------|
| `401` | `invalid_token` | Token expired, malformed, or revoked |
| `403` | `insufficient_scope` | Valid token but missing required scope |
| `403` | `access_denied` | Resource outside patient compartment |

---

## Pagination

All search results MUST support pagination. Return a Bundle with navigation links.

### Search Response with Pagination
```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 150,
  "link": [
    {
      "relation": "self",
      "url": "https://fhir.example.org/Patient?name=Smith&_count=10"
    },
    {
      "relation": "next",
      "url": "https://fhir.example.org/Patient?name=Smith&_count=10&_offset=10"
    }
  ],
  "entry": [
    {
      "fullUrl": "https://fhir.example.org/Patient/123",
      "resource": { "resourceType": "Patient", "id": "123" },
      "search": { "mode": "match" }
    }
  ]
}
```

### Pagination Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `_count` | Number of results per page | Server-defined (often 10-100) |
| `_offset` | Starting index (0-based) | 0 |
| `_page` | Alternative to offset (1-based page number) | 1 |

### Link Relations

| Relation | Description |
|----------|-------------|
| `self` | Current page URL |
| `first` | First page |
| `previous` | Previous page (if not on first) |
| `next` | Next page (if more results exist) |
| `last` | Last page |

### Implementation Example

```python
@app.get("/Patient")
async def search_patients(
    name: str = None,
    _count: int = 10,
    _offset: int = 0
):
    # Query with limit + 1 to detect if there are more results
    results = db.query_patients(name=name, limit=_count + 1, offset=_offset)
    has_more = len(results) > _count
    results = results[:_count]  # Trim to requested count
    
    base_url = f"{BASE_URL}/Patient"
    params = f"name={name}&_count={_count}" if name else f"_count={_count}"
    
    links = [
        {"relation": "self", "url": f"{base_url}?{params}&_offset={_offset}"}
    ]
    
    if _offset > 0:
        prev_offset = max(0, _offset - _count)
        links.append({"relation": "previous", "url": f"{base_url}?{params}&_offset={prev_offset}"})
    
    if has_more:
        links.append({"relation": "next", "url": f"{base_url}?{params}&_offset={_offset + _count}"})
    
    return {
        "resourceType": "Bundle",
        "type": "searchset",
        "total": db.count_patients(name=name),
        "link": links,
        "entry": [
            {"fullUrl": f"{base_url}/{p['id']}", "resource": p, "search": {"mode": "match"}}
            for p in results
        ]
    }
```

### Additional Search Modifiers

| Parameter | Description | Example |
|-----------|-------------|---------|
| `_sort` | Sort by field (prefix `-` for descending) | `_sort=-date,name` |
| `_elements` | Return only specified fields | `_elements=name,birthDate` |
| `_summary` | Return summary view | `_summary=true` |
| `_include` | Include referenced resources | `_include=Observation:patient` |
| `_revinclude` | Include resources that reference this | `_revinclude=Observation:patient` |

---

## Bundle Operations: Batch vs Transaction

### Key Difference

| Feature | Transaction | Batch |
|---------|-------------|-------|
| Atomicity | **All-or-nothing** | Independent entries |
| Failure handling | Entire bundle fails | Partial success allowed |
| Use case | Related data that must succeed together | Independent operations |
| Response | Single success/failure | Per-entry status |

### Transaction Bundle

All entries succeed together, or all fail. Use for related data.

**Request:**
```json
{
  "resourceType": "Bundle",
  "type": "transaction",
  "entry": [
    {
      "fullUrl": "urn:uuid:patient-1",
      "resource": {
        "resourceType": "Patient",
        "name": [{"family": "Smith", "given": ["John"]}]
      },
      "request": {
        "method": "POST",
        "url": "Patient"
      }
    },
    {
      "fullUrl": "urn:uuid:observation-1",
      "resource": {
        "resourceType": "Observation",
        "status": "final",
        "code": {"coding": [{"system": "http://loinc.org", "code": "8480-6"}]},
        "subject": {"reference": "urn:uuid:patient-1"}
      },
      "request": {
        "method": "POST",
        "url": "Observation"
      }
    }
  ]
}
```

**Success Response (200 OK):**
```json
{
  "resourceType": "Bundle",
  "type": "transaction-response",
  "entry": [
    {
      "fullUrl": "https://fhir.example.org/Patient/123",
      "response": {
        "status": "201 Created",
        "location": "Patient/123/_history/1",
        "etag": "W/\"1\""
      }
    },
    {
      "fullUrl": "https://fhir.example.org/Observation/456",
      "response": {
        "status": "201 Created",
        "location": "Observation/456/_history/1",
        "etag": "W/\"1\""
      }
    }
  ]
}
```

**Failure Response (400 Bad Request):** Entire transaction rolled back
```json
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "required",
    "diagnostics": "Observation.status is required",
    "expression": ["Bundle.entry[1].resource"]
  }]
}
```

### Batch Bundle

Each entry processed independently. Partial success allowed.

**Request:**
```json
{
  "resourceType": "Bundle",
  "type": "batch",
  "entry": [
    {
      "request": {"method": "GET", "url": "Patient/123"}
    },
    {
      "request": {"method": "GET", "url": "Patient/999"}
    },
    {
      "resource": {
        "resourceType": "Observation",
        "status": "final",
        "code": {"coding": [{"system": "http://loinc.org", "code": "8480-6"}]}
      },
      "request": {"method": "POST", "url": "Observation"}
    }
  ]
}
```

**Response (200 OK with mixed results):**
```json
{
  "resourceType": "Bundle",
  "type": "batch-response",
  "entry": [
    {
      "resource": {"resourceType": "Patient", "id": "123"},
      "response": {"status": "200 OK"}
    },
    {
      "response": {
        "status": "404 Not Found",
        "outcome": {
          "resourceType": "OperationOutcome",
          "issue": [{"severity": "error", "code": "not-found"}]
        }
      }
    },
    {
      "response": {
        "status": "201 Created",
        "location": "Observation/789/_history/1"
      }
    }
  ]
}
```

### Processing Order

Transactions are processed in this order (regardless of entry order):
1. DELETE
2. POST
3. PUT/PATCH
4. GET

This allows references to work correctly within a transaction.

---

## Data Type Patterns

### Coding vs CodeableConcept

**Coding** - Direct code at top level:
```json
{
  "system": "http://terminology.hl7.org/CodeSystem/v3-ActCode",
  "code": "AMB",
  "display": "ambulatory"
}
```
Used by: `Encounter.class`, `Identifier.type`

**CodeableConcept** - Wrapper with `coding` array:
```json
{
  "coding": [{
    "system": "http://loinc.org",
    "code": "8480-6",
    "display": "Systolic blood pressure"
  }],
  "text": "Systolic BP"
}
```
Used by: `Observation.code`, `Condition.code`, `Medication.code`

### Reference Pattern
```json
{
  "reference": "Patient/123",
  "display": "John Smith"
}
```

### Identifier Pattern
```json
{
  "system": "http://hospital.example.org/mrn",
  "value": "12345"
}
```

---

## Coding Systems (URLs)

| System | URL | Use |
|--------|-----|-----|
| LOINC | `http://loinc.org` | Lab tests, vital signs |
| SNOMED CT | `http://snomed.info/sct` | Clinical terms |
| RxNorm | `http://www.nlm.nih.gov/research/umls/rxnorm` | Medications |
| ICD-10 | `http://hl7.org/fhir/sid/icd-10` | Diagnoses |
| v3-ActCode | `http://terminology.hl7.org/CodeSystem/v3-ActCode` | Encounter class |
| Observation Category | `http://terminology.hl7.org/CodeSystem/observation-category` | Observation categories |
| Condition Clinical | `http://terminology.hl7.org/CodeSystem/condition-clinical` | Condition clinical status |
| Condition Ver Status | `http://terminology.hl7.org/CodeSystem/condition-ver-status` | Condition verification status |

### Common LOINC Codes (Vital Signs)
| Code | Description |
|------|-------------|
| `8867-4` | Heart rate |
| `8480-6` | Systolic blood pressure |
| `8462-4` | Diastolic blood pressure |
| `8310-5` | Body temperature |
| `2708-6` | Oxygen saturation (SpO2) |
| `85354-9` | Blood pressure panel |

---

## RESTful Endpoints

```
POST   /[ResourceType]              # Create (returns 201)
GET    /[ResourceType]/[id]         # Read
PUT    /[ResourceType]/[id]         # Update
DELETE /[ResourceType]/[id]         # Delete (returns 204)
GET    /[ResourceType]?param=value  # Search
GET    /metadata                    # CapabilityStatement
GET    /.well-known/smart-configuration  # SMART discovery
POST   /                            # Bundle transaction/batch
```

---

## Resource Structures

### Patient
```json
{
  "resourceType": "Patient",
  "id": "example",
  "meta": {
    "versionId": "1",
    "lastUpdated": "2024-01-15T10:30:00Z"
  },
  "identifier": [{
    "system": "http://hospital.example.org/mrn",
    "value": "12345"
  }],
  "name": [{
    "family": "Smith",
    "given": ["John"]
  }],
  "gender": "male",
  "birthDate": "1990-05-15"
}
```

### Observation (Vital Signs)
Required: `status`, `code`
```json
{
  "resourceType": "Observation",
  "status": "final",
  "code": {
    "coding": [{
      "system": "http://loinc.org",
      "code": "8480-6",
      "display": "Systolic blood pressure"
    }]
  },
  "subject": {
    "reference": "Patient/123"
  },
  "valueQuantity": {
    "value": 120,
    "unit": "mmHg",
    "system": "http://unitsofmeasure.org",
    "code": "mm[Hg]"
  }
}
```

### Encounter
Required: `status`, `class`

**Note**: `class` uses Coding (NOT CodeableConcept)

```json
{
  "resourceType": "Encounter",
  "status": "in-progress",
  "class": {
    "system": "http://terminology.hl7.org/CodeSystem/v3-ActCode",
    "code": "AMB",
    "display": "ambulatory"
  },
  "subject": {
    "reference": "Patient/123"
  }
}
```

### Condition
Required: `subject`
```json
{
  "resourceType": "Condition",
  "clinicalStatus": {
    "coding": [{
      "system": "http://terminology.hl7.org/CodeSystem/condition-clinical",
      "code": "active"
    }]
  },
  "verificationStatus": {
    "coding": [{
      "system": "http://terminology.hl7.org/CodeSystem/condition-ver-status",
      "code": "confirmed"
    }]
  },
  "code": {
    "coding": [{
      "system": "http://snomed.info/sct",
      "code": "73211009",
      "display": "Diabetes mellitus"
    }]
  },
  "subject": {
    "reference": "Patient/123"
  }
}
```

### MedicationRequest
Required: `status`, `intent`, `medication[x]`, `subject`
```json
{
  "resourceType": "MedicationRequest",
  "status": "active",
  "intent": "order",
  "medicationReference": {
    "reference": "Medication/123"
  },
  "subject": {
    "reference": "Patient/456"
  }
}
```

### OperationOutcome (Errors)
```json
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "not-found",
    "details": {
      "text": "Patient not found"
    }
  }]
}
```

Severity: `fatal`, `error`, `warning`, `information`
Code: `invalid`, `structure`, `required`, `value`, `not-found`, `conflict`, `lock-error`, `exception`

### CapabilityStatement (/metadata)
```json
{
  "resourceType": "CapabilityStatement",
  "status": "active",
  "date": "2024-01-15",
  "kind": "instance",
  "fhirVersion": "4.0.1",
  "format": ["json"],
  "rest": [{
    "mode": "server",
    "security": {
      "service": [{
        "coding": [{
          "system": "http://terminology.hl7.org/CodeSystem/restful-security-service",
          "code": "SMART-on-FHIR"
        }]
      }],
      "extension": [{
        "url": "http://fhir-registry.smarthealthit.org/StructureDefinition/oauth-uris",
        "extension": [
          {"url": "authorize", "valueUri": "https://auth.example.org/authorize"},
          {"url": "token", "valueUri": "https://auth.example.org/token"}
        ]
      }]
    },
    "resource": [{
      "type": "Patient",
      "interaction": [
        {"code": "read"},
        {"code": "create"},
        {"code": "update"},
        {"code": "delete"},
        {"code": "search-type"}
      ]
    }]
  }]
}
```

---

## Conditional Operations

**If-Match header** for optimistic locking:
- Client sends: `If-Match: W/"1"`
- Server compares with current `meta.versionId`
- Mismatch returns `412 Precondition Failed`

**If-None-Exist header** for conditional create:
- Client sends: `If-None-Exist: identifier=http://mrn|12345`
- If match exists, return existing resource (200)
- If no match, create new (201)

---

## Common Implementation Notes

1. Always set `Content-Type: application/fhir+json`
2. Return `meta.versionId` and `meta.lastUpdated` on all resources
3. Return `Location` header on create: `/Patient/{id}`
4. Return `ETag` header: `W/"{versionId}"`
5. Implement `/metadata` endpoint returning CapabilityStatement
6. Implement `/.well-known/smart-configuration` for SMART authorization
7. Use OperationOutcome for all error responses
8. Search results return Bundle with `type: "searchset"`
9. Validate required fields and return 400 for missing fields
10. Validate enum values and return 422 for invalid values
11. Support pagination with `_count` and `_offset` parameters
12. Enforce scopes - return 403 for insufficient permissions
