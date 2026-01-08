# Healthcare Marketplace for Claude Code

Skills for healthcare workflows including clinical trials, prior authorization review, and FHIR API development.

## Quick Start

```bash
# Add the marketplace
/plugin marketplace add https://github.com/anthropics/healthcare.git

# Install skills
/plugin install clinical-trial-protocol@healthcare
/plugin install prior-auth-review@healthcare
/plugin install fhir-developer@healthcare
```

## Available Skills

### Clinical Trial Protocol Generator
**Plugin ID**: `clinical-trial-protocol@healthcare`

Generate FDA/NIH-compliant clinical trial protocols for medical devices or drugs. Uses a waypoint-based architecture that guides through intervention setup, clinical research, and protocol development.

**Features:**
- Research-only mode for clinical research analysis
- Full protocol mode for complete protocol generation
- Waypoint-based architecture for resume capability
- Sample size calculations

**Requirements**: Python with scipy and numpy for statistical calculations

---

### Prior Authorization Review
**Plugin ID**: `prior-auth-review@healthcare`

Automate payer review of prior authorization requests by synthesizing clinical documentation, validating medical necessity against coverage policies, and generating approval/denial determinations.

**Target Users**: Health insurance payer organizations (Medicare Advantage, Commercial, Medicaid MCOs)

**Features:**
- 2-subskill workflow (Intake & Assessment → Decision & Notification)
- Provider credential validation
- Diagnosis code validation
- Coverage policy lookup
- Auto-approval capability for clear-cut cases

**Requirements**: None

---

### FHIR Developer
**Plugin ID**: `fhir-developer@healthcare`

FHIR API development guide for building healthcare endpoints. Provides guidance on FHIR R4 resource structures, validation rules, SMART on FHIR authorization, and healthcare API best practices.

**Use when working with:**
- FHIR resources (Patient, Observation, Encounter, Condition, MedicationRequest)
- Healthcare APIs and HL7 standards
- SMART on FHIR authorization and OAuth scopes
- Medical coding systems (LOINC, SNOMED, RxNorm, ICD-10)

**Requirements**: None - provides domain knowledge for development

---

## Skill Requirements Summary

| Skill | Requirements |
|-------|-------------|
| Clinical Trial Protocol | Python (scipy, numpy) |
| Prior Authorization Review | None |
| FHIR Developer | None |

## License

Skills are provided under Anthropic's terms of service.
