# Payer-to-Payer API

## References
* [PDex STU 0.1.0](https://hl7.org/fhir/us/davinci-pdex/2019Jun/index.html)
* [PDex STU 2.0.0](https://hl7.org/fhir/us/davinci-pdex/STU2/payertopayerexchange.html)
* [PDex STU 2.1.0: P2P single member](https://hl7.org/fhir/us/davinci-pdex/STU2.1/payertopayerexchange.html)
* [PDex STU 2.1.0: P2P bulk](https://hl7.org/fhir/us/davinci-pdex/STU2.1/payertopayerbulkexchange.html)
* [HRex Member Match](https://build.fhir.org/ig/HL7/davinci-ehrx/en/OperationDefinition-member-match.html)

## Problem Statement

When a patient changes health plans, their data often does not move with them.
New payers must re‑collect information like: Medications, Conditions, Coverage history etc.
This causes delays, duplicate work, and poor member experience.

CMS 0057F is about making sure patient data can follow the patient when they move between payers.

## PDex

The **HL7 Da Vinci Payer Data Exchange (PDex)** Implementation Guide explains how health plans (payers) can share a member's health history with providers, other payers, or apps. **PDex** defines how payer-to-payer data sharing should work using standard FHIR APIs. PDex has evolved over time as real‑world implementations exposed limitations and operational challenges.

## PDex STU 0.1.0

```mermaid
sequenceDiagram
    participant Member
    participant NewPayerApp as New Plan P2P App
    participant OldPayerAuth as Old Plan Auth Server
    participant OldPayerFHIR as Old Plan FHIR Server

    Member->>NewPayerApp: Start data transfer
    NewPayerApp->>Member: Redirect to old plan login
    Member->>OldPayerAuth: Login with old plan credentials
    OldPayerAuth-->>NewPayerApp: Access token
    NewPayerApp->>OldPayerFHIR: GET Patient/{id}/$everything
    OldPayerFHIR-->>NewPayerApp: Member data (FHIR Bundle)
```

### Key Concepts
* **Member Initiated:** The member initiates the payer‑to‑payer data exchange.
* The member authenticates directly with the old health plan using existing credentials.
* **No Member Match:** identity is established through login.
* The old plan issues an access token to the new plan’s payer‑to‑payer application.
* Using the access token, the new plan retrieves data from the old plan using `GET Patient/{id}/$everything`.
* **Single Member:** Data is exchanged one member at a time.
* There is no bulk or asynchronous export in STU 0.1.0.

## PDex STU 2.0.0

STU 2.0.0 shifted the flow from member‑initiated login to payer‑mediated exchange with member matching, while still keeping the data exchange limited to a single member.

```mermaid
sequenceDiagram
    participant NewPayer as New Payer
    participant OldPayerAPI as Old Payer P2P API
    participant MemberSearch as Old Payer Member Search
    participant OldPayerFHIR as Old Payer FHIR Repo

    Note over NewPayer: Member enrolls at new plan<br/>Consent captured<br/>Old payer coverage

    NewPayer->>OldPayerAPI: POST $member-match<br/>Parameters:<br/>New Payer Patient,<br/>New Payer Coverage,<br/>Old Payer Coverage,<br/>HRex Consent

    OldPayerAPI->>MemberSearch: Perform single-member match<br/>(name, DOB, gender, etc.)<br/>or advanced matching service

    alt Unique member found
        MemberSearch-->>OldPayerAPI: Match successful<br/>Old Payer Patient identified
        OldPayerAPI->>OldPayerFHIR: Store HRex Consent
        OldPayerAPI-->>NewPayer: Parameters:<br/>Old Payer Patient<br/>with identifiers
        NewPayer->>OldPayerFHIR: GET Patient/{id}/$everything
        OldPayerFHIR-->>NewPayer: Member data<br/>(FHIR Bundle)
    else No match OR multiple matches<br/>OR consent cannot be processed
        OldPayerAPI-->>NewPayer: HTTP 422<br/>(Unprocessable Entity)
    end
```

### Key Concepts
* The exchange is still limited to a **single member**.
* **Payer‑mediated:** the member does not log in to the old plan.
* **Single Member Match:** HRex member match is used to identify the member at the old payer. The new payer sends patient, coverage, and consent information in a `Parameters` resource, and the old payer identifies the member before allowing data retrieval.
* Once the member is matched, data is retrieved using `GET Patient/{id}/$everything`.
* **Single Member:** Data is returned for that one member only.
* Single‑member bulk export is not supported.

In PDex STU 2.0.0, the new payer identifies the member at the old payer using the HRex member match process. This replaces the member login flow used in STU 0.1.0. Once a single matching patient is found, that patient’s FHIR ID is used for data retrieval.

## PDex STU 2.1.0

```mermaid
sequenceDiagram
  autonumber
  actor M as Member
  participant PB as Payer B (new plan)
  participant PA as Payer A (old plan)

  rect rgb(237, 233, 254)
    Note over M,PB: Enrollment & consent
    M->>PB: Enrolls in new plan
    M->>PB: Opts in to payer-to-payer data sharing
    PB->>PB: Records consent for each member
  end

  rect rgb(225, 245, 238)
    Note over PB,PA: SMART Backend Services auth
    PB->>PA: POST /auth/token (signed JWT assertion)
    PA-->>PB: Access token
  end

  rect rgb(237, 233, 254)
    Note over PB,PA: Bulk member match — POST /Group/$bulk-member-match
    PB->>PA: Parameters { parameter[ {name: MemberBundle, part[ MemberPatient, CoverageToMatch, Consent ]} x N ] }
    PA-->>PB: 202 Accepted + Content-Location: /match-status/job-123
    loop Poll for match results
      PB->>PA: GET /match-status/job-123
      PA-->>PB: 202 Accepted (in-progress)
    end
    PA-->>PB: 200 OK — Parameters { MatchedMembers Group (with Group.id), NonMatchedMembers Group, ConsentConstrainedMembers Group }
    PB->>PB: Store Group.id from MatchedMembers
    PB->>PB: Log NonMatchedMembers and ConsentConstrainedMembers
  end

  rect rgb(225, 245, 238)
    Note over PB,PA: Bulk data export — POST /Group/{MatchedMembers.Group.id}/$davinci-data-export
    PB->>PA: POST /Group/{Group.id}/$davinci-data-export (exportType: hl7.fhir.us.davinci-pdex)
    PA-->>PB: 202 Accepted + Content-Location: /export-status/job-456
    loop Poll for export completion
      PB->>PA: GET /export-status/job-456
      PA-->>PB: 202 Accepted (in-progress)
    end
    PA-->>PB: 200 OK — export manifest { output: [ { type, url } ... ] }
  end

  rect rgb(254, 243, 199)
    Note over PB,PA: Download and ingest
    PB->>PA: GET ndjson file URLs from export manifest
    PA-->>PB: NDJSON files (Condition, MedicationRequest, Procedure, ExplanationOfBenefit ...)
    PB->>PB: Incorporate data into member records
    PB->>PB: Tag each resource with Provenance (source: Payer A)
  end
```

### Key Concepts
* Single member and multi-member data exchange
* Single and bulk member match
* Single member data retrieval using $everything
* Multi-member data retrieval using Group export

The bulk Payer-to-Payer exchange is initiated by supplying a `Parameters` resource to the `$bulk-member-match` operation. A set of OAuth2.0/SMART-on-FHIR Client Credentials SHALL be required to access the secured bulk-member-match operation endpoint.

For each member submitted to the bulk-member-match operation the following parameters SHALL be supplied as a `parameter.part` element.

* `MemberPatient` - HRex Patient demographics
* `CoverageToMatch` - details of the prior health plan coverage, supplied by the member, typically from the health plan coverage card. Uses the HRex Coverage Profile
* `Consent` - Record of consent received by requesting payer from Member to retrieve their records from the prior payer. This is an **opt-in**. Uses the HRex Consent Profile
* `CoverageToLink` - Optional parameter. Details of new or prospective health plan coverage, provided by the health plan based upon the member’s enrolment. Uses the HRex Coverage Profile


