---
name: fhir-software
description: Comprehensive FHIR (Fast Healthcare Interoperability Resources) software development assistant. Use when working with FHIR APIs, implementations, or healthcare data exchange. Supports FHIR R4, R4B, R5, Implementation Guides (IGs), FHIR Shorthand (FSH) authoring, SUSHI, GoFSH, validation, terminology, and SMART on FHIR. Ideal for building FHIR servers, clients, validators, IG authors, or healthcare applications that need to process FHIR resources.
---

# FHIR Software Development Skill

Expert guidance for building robust FHIR (Fast Healthcare Interoperability Resources) software systems with comprehensive package management, spec knowledge, and development workflows.

This file is the overview and router. Detailed code patterns live in the reference files and runnable scaffolds below — load them on demand for the task at hand rather than reading everything up front.

## Reference Files & Assets

| Topic | Load when… | File |
|-------|------------|------|
| **Validation** — structure, data types, cardinality, profile, terminology, slicing, extensions | implementing or debugging resource/profile validation | [references/validation_patterns.md](references/validation_patterns.md) |
| **Search** — all parameter types, modifiers, chaining, `_include`/`_revinclude`, pagination, server APIs | building or querying a FHIR search endpoint | [references/search_parameters.md](references/search_parameters.md) |
| **SMART on FHIR** — discovery, auth code flow, scopes, client, backend services, security, React example | implementing SMART app launch or OAuth | [references/smart_on_fhir.md](references/smart_on_fhir.md) |
| **Testing** — unit, integration/CRUD, profile validation, load, conformance, test data, CI | writing FHIR tests or CI | [references/testing_patterns.md](references/testing_patterns.md) |
| **FSH / IG authoring** — full SUSHI/GoFSH reference | authoring FSH beyond the quick reference in §6 | [references/fsh_ig_authoring.md](references/fsh_ig_authoring.md) |
| **Package manager** — download/cache/index FHIR packages (runnable CLI) | resolving packages or building a resource index | [scripts/fhir_package_manager.py](scripts/fhir_package_manager.py) |
| **Validator CLI** — TS resource validator/formatter | validating from Node/TS | [scripts/fhir_validator.ts](scripts/fhir_validator.ts) |
| **FastAPI server scaffold** — Patient/Observation/Bundle CRUD, CapabilityStatement | starting a Python FHIR server | [assets/fhir_server.py](assets/fhir_server.py) |
| Resource & response templates | need a starting JSON shape | [assets/](assets/) (`patient-`, `observation-vitals-`, `bundle-searchset-`, `capability-statement-template.json`) |

## Core Architecture

### 1. Package / Specification Management

- **Local cache:** `~/.fhir/packages/` with version-specific directories.
- **Loaders:** `@fhir/package-loader` (TS/Node) or `fhir-package-loader` (Python). The runnable [scripts/fhir_package_manager.py](scripts/fhir_package_manager.py) downloads from `https://packages.fhir.org`, caches, and builds a searchable index.
- **Core packages:** `hl7.fhir.r4.core`, `hl7.fhir.r5.core`, plus IGs like `hl7.fhir.us.core`.
- **Index these artifact kinds** from package contents: StructureDefinitions (resources/profiles/extensions), SearchParameters, ValueSets/CodeSystems, OperationDefinitions, CapabilityStatements, and example instances.

### 2. Development Workflows

**Resource modeling** — prefer the `fhir.resources` (Python) or `fhir/r4` (TS) typed models over hand-rolled classes; constructing the typed model validates structure and data types. Use Pydantic only when you need a deliberately partial/lenient shape (`extra = "allow"`).

**Server implementation** — [assets/fhir_server.py](assets/fhir_server.py) is a working FastAPI scaffold (Patient/Observation/Bundle CRUD + CapabilityStatement). Express + `fhir/r4` is the common Node equivalent. Key contract for any FHIR endpoint:
- Reject mismatched `resourceType` with an `OperationOutcome` (see below), not a plain error.
- Return `201` on create, `200` on read/update, `204` on delete, `404` when absent.
- Validate by constructing the typed model; surface failures as `OperationOutcome`.

### 3. Validation & Quality

Validation has layers — structure → data types → cardinality → profile (must-support, constraints, fixed values, bindings) → terminology → slicing → extensions. Full implementations of each layer, plus a `FHIRValidator` orchestrator and caching, are in [references/validation_patterns.md](references/validation_patterns.md). A runnable TS validator is [scripts/fhir_validator.ts](scripts/fhir_validator.ts).

For authoritative, spec-complete validation, defer to the official **HL7 FHIR Validator** rather than hand-rolled checks — hand-rolled validators are useful for fast pre-checks and custom rules, not as the source of truth:
```bash
java -jar validator_cli.jar resource.json -version 4.0.1 -ig hl7.fhir.us.core#5.0.1
```
A HAPI FHIR server's `$validate` operation is the equivalent service call.

### 4. Search Implementation

FHIR search parameter types: string, token (`[system]|[code]`), date (prefixes `eq/ne/gt/lt/ge/le/sa/eb/ap`), number, reference (with chaining), quantity, and composite. Plus result controls: `_include`/`_revinclude`, `_sort`, `_count`/`_offset`, `_elements`/`_summary`, and reverse chaining (`_has`). Parameter parsing, query translation, search bundles, compartments, indexing, and per-server notes (HAPI, Microsoft, Google, AWS) are in [references/search_parameters.md](references/search_parameters.md).

### 5. SMART on FHIR Integration

SMART App Launch = OAuth 2.0 authorization-code flow with FHIR-specific discovery (`.well-known/smart-configuration`), scopes (`patient/*.read`, `user/*.rw`, `launch`, `openid`), and the `aud` parameter bound to the FHIR base URL. Backend Services uses the client-credentials flow with a signed JWT assertion. Full flows, a typed client, scope reference, token-storage/CSRF security guidance, and a React example are in [references/smart_on_fhir.md](references/smart_on_fhir.md).

### 6. Custom Profiles & Implementation Guides

Author profiles in **FSH** (see §7) rather than hand-writing StructureDefinition JSON — SUSHI compiles FSH to the JSON. The raw JSON shape is still worth recognizing: a constraint profile sets `derivation: "constraint"`, a `baseDefinition` pointing at the parent, and a `differential.element` list carrying the `min`/`max`/`mustSupport` constraints.

## Common Patterns

### OperationOutcome (error reporting)
```python
def create_operation_outcome(severity: str, code: str, details: str) -> dict:
    return {
        'resourceType': 'OperationOutcome',
        'issue': [{'severity': severity, 'code': code, 'details': {'text': details}}],
    }
```
`severity` ∈ `fatal|error|warning|information`; `code` is from the issue-type value set (`invalid`, `not-found`, `not-supported`, …).

### Batch & Transaction Bundles
A `batch` Bundle processes each entry independently; a `transaction` Bundle is all-or-nothing — wrap entries in a DB transaction and roll back on any failure. Respond with a `batch-response` / `transaction-response` Bundle whose entries carry per-entry status. Reference and `_include` resolution patterns are expanded in the search/validation references.

## Quick Reference Commands

### Package Management
```bash
npm install @types/fhir @fhir/package-loader
pip install fhir.resources fhir-package-loader
fhir-package-loader install hl7.fhir.r4.core 4.0.1
fhir-package-loader install hl7.fhir.us.core 5.0.1
```

### Validation Tools
```bash
java -jar validator_cli.jar resource.json -version 4.0.1        # HL7 Java validator
curl -X POST "http://localhost:8080/fhir/$validate" \           # HAPI $validate
  -H "Content-Type: application/fhir+json" -d @patient.json
```

### Development Server
```bash
uvicorn main:app --reload --port 8000   # Python / FastAPI
npm start                               # Node / Express
```

## Integration Points

- **EHR Systems**: Epic, Cerner, AllScripts FHIR APIs
- **Cloud Platforms**: AWS HealthLake, Azure FHIR, Google Healthcare API
- **Terminology Services**: UMLS, SNOMED CT, LOINC
- **Security**: OAuth 2.0, JWT, SMART on FHIR scopes
- **Interoperability**: HL7 v2 to FHIR conversion, CDA to FHIR

Spec: https://hl7.org/fhir/ · IG registry: https://fhir.org/guides/registry/

---

## 7. FHIR Shorthand (FSH) & IG Authoring

### Toolchain

| Tool | Role | Install |
|------|------|---------|
| **SUSHI** | Compiles `.fsh` files → FHIR JSON | `npm install -g fsh-sushi` |
| **GoFSH** | Converts FHIR JSON → FSH source | `npm install -g gofsh` |
| **IG Publisher** | Renders IG website from SUSHI output | Downloaded via `_updatePublisher.sh` |

### SUSHI Commands
```bash
sushi init                          # Scaffold new IG project
sushi build                         # Compile FSH → FHIR JSON (from project root)
sushi build . --snapshot            # Include full StructureDefinition snapshot
sushi build . --log-level debug     # Verbose output
sushi build . --preprocessed        # Debug: dump resolved aliases/rulesets
sushi update-dependencies           # Update deps to latest
```

Output: `fsh-generated/resources/{ResourceType}-{id}.json`. **SUSHI clears this folder on every build — never edit it manually.**

### GoFSH Commands
```bash
gofsh ./fhir-resources                          # Convert JSON artifacts to FSH
gofsh ./definitions -d hl7.fhir.us.core@6.1.0  # With extra dependencies
gofsh ./definitions --style group-by-profile    # Organize output by profile
gofsh ./definitions --indent                    # Use indented rule style
gofsh ./definitions --fshing-trip               # Validate round-trip accuracy
```

### Project Structure
```
my-ig/
├── sushi-config.yaml              # Required: IG configuration
├── ig.ini                         # Required for IG Publisher
├── _genonce.sh / _genonce.bat     # Run IG Publisher
├── _updatePublisher.sh / .bat     # Download latest IG Publisher jar
├── input/
│   ├── fsh/                       # All FSH source files
│   │   ├── aliases.fsh            # Alias: $LNC = http://loinc.org
│   │   ├── profiles.fsh
│   │   ├── extensions.fsh
│   │   ├── valuesets.fsh
│   │   ├── codesystems.fsh
│   │   ├── instances.fsh
│   │   └── rulesets.fsh
│   ├── pagecontent/               # IG narrative (Markdown)
│   │   ├── index.md               # Home page
│   │   ├── 1_background.md        # Numbered = TOC order
│   │   ├── {resource-id}-intro.md # Content before artifact
│   │   └── {resource-id}-notes.md # Content after artifact
│   ├── images/                    # Images, PDFs, spreadsheets
│   └── ignoreWarnings.txt
└── fsh-generated/                 # SUSHI output (auto-generated, do not edit)
```

### sushi-config.yaml (Minimal Required Fields)
```yaml
id: hl7.fhir.us.example
canonical: http://hl7.org/fhir/us/example
name: ExampleIG
title: "Example Implementation Guide"
status: draft                     # draft | active | retired | unknown
version: 0.1.0
fhirVersion: 4.0.1
copyrightYear: 2024+
releaseLabel: ci-build
publisher:
  name: My Organization
  url: http://example.org
dependencies:
  hl7.fhir.us.core: 6.1.0
menu:
  Home: index.html
  Artifacts: artifacts.html
parameters:
  show-inherited-invariants: false
```

### FSH Entity Types

#### Profile
```
Profile: MyPatientProfile
Parent: Patient
Id: my-patient-profile
Title: "My Patient Profile"
Description: "Constrained Patient for our IG"
* identifier 1..* MS
* name 1..* MS
* birthDate MS
* gender 1..1 MS
* gender from http://hl7.org/fhir/ValueSet/administrative-gender (required)
```

#### Extension (Simple)
```
Extension: PatientReligion
Id: patient-religion
Title: "Patient Religion"
Context: Patient
* value[x] only CodeableConcept
* value[x] from ReligionValueSet (extensible)
```

#### Extension (Complex — sub-extensions)
```
Extension: USCoreEthnicityExtension
Id: us-core-ethnicity
Context: Patient, RelatedPerson, Practitioner
* extension contains
    ombCategory 0..1 MS and
    detailed 0..* and
    text 1..1 MS
* extension[ombCategory].value[x] only Coding
* extension[ombCategory].value[x] from OmbEthnicityCategories (required)
* extension[text].value[x] only string
```

#### Instance
```
Instance: JaneDoe
InstanceOf: MyPatientProfile
Title: "Jane Doe"
Usage: #example            // #example | #definition | #inline
* name[0].family = "Doe"
* name[0].given[0] = "Jane"
* birthDate = 1970-01-01
* gender = #female
```

#### ValueSet
```
ValueSet: MyConditionStatusVS
Id: my-condition-status
Title: "Condition Status Codes"
* include codes from system $SCT where concept is-a #404684003
* $V3#active "Active"
* exclude $SCT#74964007 "Other"
```

#### CodeSystem
```
CodeSystem: MyCustomCodes
Id: my-custom-codes
* #pending "Pending" "Awaiting review"
* #approved "Approved" "Formally approved"
// Hierarchical:
* #body "Body"
  * #head "Head"
```

#### Invariant + Obeys
```
Invariant: my-inv-1
Description: "Value must be present for vital-signs category"
Severity: #error
Expression: "category.coding.code = 'vital-signs' implies value.exists()"

// In a Profile:
* obeys my-inv-1
```

#### RuleSet (Reusable / Parameterized)
```
RuleSet: PublicationMetadata
* ^status = #active
* ^experimental = false
* ^publisher = "My Org"

// Parameterized
RuleSet: SetContext(contextPath)
* ^context[+].type = #element
* ^context[=].expression = "{contextPath}"

// Usage:
* insert PublicationMetadata
* insert SetContext(Patient)
```

### FSH Rules Quick Reference

| Rule | Syntax | Example |
|------|--------|---------|
| Cardinality | `* elem min..max` | `* name 1..* MS` |
| Must Support | `* elem MS` | `* identifier MS` |
| Type constraint | `* elem only Type` | `* value[x] only Quantity` |
| Binding | `* elem from VS (strength)` | `* code from MyVS (required)` |
| Fixed value | `* elem = value` | `* status = #final` |
| Fixed coding | `* elem = $SYS#code "display"` | `* code = $LNC#29463-7` |
| Quantity | `* elem = n 'unit'` | `* value = 70 'kg'` |
| Slice (element) | `* arr contains name card` | `* component contains systolic 1..1` |
| Slice (extension) | `* extension contains Ext named n card` | `* extension contains $Race named race 0..1` |
| Obeys | `* obeys inv-id` | `* obeys us-core-6` |
| Caret (metadata) | `* ^property = val` | `* ^experimental = false` |
| Insert RuleSet | `* insert RSName` | `* insert PublicationMetadata` |

### Path Grammar Quick Reference

```
status                              // Simple element
name.family                         // Nested
valueQuantity                       // Choice [x] resolved
name[0]                             // Array index (zero-based)
name[+]                             // Soft index: next slot
name[=]                             // Soft index: same slot
component[respirationScore]         // Slice by name
extension[race]                     // Extension slice
performer[Practitioner]             // Reference target
^experimental                       // StructureDefinition metadata
code ^short                         // ElementDefinition property
```

### Coding / Quantity Syntax

```
// Alias declaration (top of file)
Alias: $LNC = http://loinc.org
Alias: $SCT = http://snomed.info/sct

// Code (no system)
#active

// Coding
http://loinc.org#29463-7 "Body Weight"
$LNC#29463-7 "Body Weight"

// Quantity (UCUM)
70.5 'kg' "kg"
120 'mm[Hg]' "mmHg"
```

### Naming Conventions

| Item | Convention | Example |
|------|-----------|---------|
| Profile/Extension/RuleSet names | `PascalCase` | `MyPatientProfile` |
| Item IDs | `kebab-case`, max 64 chars | `my-patient-profile` |
| Slice names | `lowerCamelCase` | `respirationScore` |
| Alias names | `$PrefixedName` | `$LNC`, `$SCT` |

### Key Rules
1. **Declare slices before constraining them** — `contains` rule must precede slice-specific rules
2. **Declare extensions before constraining sub-elements** — same ordering requirement
3. **`fsh-generated/` is owned by SUSHI** — never edit it; it is deleted and regenerated on each build
4. **Caret (`^`) rules are forbidden in Instances** — use them only in Profiles, Extensions, ValueSets, CodeSystems

### IG Authoring Workflow
```bash
# New IG from scratch
mkdir my-ig && cd my-ig
sushi init               # Interactive scaffold
# Edit sushi-config.yaml and write .fsh files
sushi build              # Compile
./_genonce.sh            # Run IG Publisher

# Migrate existing FHIR JSON to FSH
gofsh ./existing-resources -d hl7.fhir.us.core@6.1.0 \
  --style group-by-profile --indent --fshing-trip

# Debug a failing build
sushi build --log-level debug
sushi build --preprocessed   # Inspect resolved aliases/rulesets
```

### Reference Files
- Full spec: https://build.fhir.org/ig/HL7/fhir-shorthand/reference.html
- SUSHI docs: https://fshschool.org/docs/sushi/
- GoFSH docs: https://fshschool.org/docs/gofsh/
- See also: [references/fsh_ig_authoring.md](references/fsh_ig_authoring.md) for the complete reference
