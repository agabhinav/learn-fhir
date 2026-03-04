# CRD

## References
* [CDS Hooks](https://cds-hooks.hl7.org/)
* [Trusting CDS Services](https://cds-hooks.hl7.org/#trusting-cds-services)
* [CRD Request Logical Model](https://build.fhir.org/ig/HL7/davinci-crd/en/StructureDefinition-CRDHooksRequest.html)
* [CRD Request order-sign context](https://build.fhir.org/ig/FHIR/fhir-tools-ig/StructureDefinition-CDSHookOrderSignContext.html)
* [CRD Request draftOrders Bundle](https://build.fhir.org/ig/HL7/davinci-crd/en/StructureDefinition-profile-bundle-request.html)
* [CRD Response Logical Model](https://build.fhir.org/ig/HL7/davinci-crd//en/StructureDefinition-CRDHooksResponse.html)

## Discovery

1. A CDS Service provider exposes its discovery endpoint at `{baseURL}/cds-services`.
This endpoints provide the JSON descriptions of the CDS Hooks services.
2. CDS Client can retrieve the list of CDS Services by invoking `GET https://{baseURL}/cds-services`.
3. Response is an array of CDS Services. Example:

```json
{
  "services": [
    {
      "hook": "order-select",
      "title": "Order Echo CDS Service",
      "description": "An example of a CDS Service that simply echoes the order(s) being placed",
      "id": "order-echo",
      "prefetch": {
        "patient": "Patient/{{context.patientId}}",
        "medications": "MedicationRequest?patient={{context.patientId}}"
      }
    },
    {
      "hook": "order-sign",
      "title": "Pharmacogenomics CDS Service",
      "description": "An example of a more advanced, precision medicine CDS Service",
      "id": "pgx-on-order-sign",
      "usageRequirements": "Note: functionality of this CDS Service is degraded without access to a FHIR Restful API as part of CDS recommendation generation."
    }
  ]
}
```

## Calling a CDS Service

**CRD Request:** CDS Client (typically the EHR) calls a CDS service by `POST`ing a JSON document to `{baseUrl}/cds-services/{service.id}`. The request payload includes:
* `hook` (e.g., order-sign, order-select, appointment-book)
* `context` tells the CDS Service what is happening in the workflow - for example, which order is being placed and which patient it is for - so the payer can evaluate coverage requirements. A CRD request always applies to a single patient, but within that request the EHR may include multiple orders for that patient.
* Optional `prefetch` results - data the EHR retrieves before the request is sent
* `fhirServer` - EHR FHIR Server base URL, used when the CDS Service must fetch missing resources. This is required when the CDS Client cannot provide all needed prefetch data.
* `fhirAuthorization` - bearer access token that CDS Server can use when querying EHR FHIR Server.

In most EHRs, a clinician does not sign one order at a time.  
They build a “shopping cart” of orders during an encounter: E.g.
* MRI knee
* Physical therapy referral
* Pain medication
* DME knee brace

And then they press Sign All.  
At that moment, the EHR fires a single order-sign CDS Hooks request, and all pending draft orders go into the `draftOrders` Bundle.  

draftOrders Bundle  
 ├── ServiceRequest #1 (MRI Knee)  
 ├── ServiceRequest #2 (Physical Therapy)  
 ├── MedicationRequest #3 (Pain Medication)  
 └── DeviceRequest #4 (Knee Brace)  


**CRD Response:** CDS Service returns a response containing a `cards` array and optionally a `systemActions` array.

* `cards` are intended for display to an end user. Cards can provide a combination of information (for reading), suggested actions (to be applied if a user selects them), and links (to launch an app if the user selects them).

* `systemActions` allows the CDS Client to auto-apply the actions proposed by the CDS Service.

Each item in `systemActions` array represents one action on one order from the request. This order is extended with Coverage Information Extension. This extension tells the EHR the key things the payer knows at the moment:
* Is it covered?
* Is prior authorization needed?
* Is more documentation needed?, and if so, optionally which questionnaire(s) can collect it.

Below high-level diagram shows important parts of the CRD Request and Response:

![crd-request-response](../images/diagrams-crd-request-response.png)