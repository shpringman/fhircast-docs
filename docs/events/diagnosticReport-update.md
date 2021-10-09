# DiagnosticReport-update
eventMaturity | [1 - Submitted](../../specification/STU3/#event-maturity-model)

## Workflow
The `DiagnosticReport-update` event is used by clients to support content sharing in communication with a Hub which also supports content sharing.  A `DiagnosticReport-update` request will be posted to the Hub when an application desires a change be made to the current state of exchanged information or to add or remove a reference to a contained FHIR resource in the content of the current anchor context. One or more updates MAY occur while the anchor context is open.

The updates could include (but are not limited to) any of the following:
* adding, updating, or removing observations contained in the DiagnosticReport anchor context
* adding, updating, or removing other FHIR resources contained in the anchor context
* updating attributes of the anchor context resource's attributes

The context MUST contain a `Bundle` resource in an `updates` key which contains one or more resources which are to be updated as entries in the Bundle.

Exchange of information is made using a transactional approach using change sets in the `DiagnosticReport-update` event (i.e., the complete current state of the content is not provided in the `Bundle` resource in the `updates` key); therefore it is essential that applications interested in the current state of exchanged information process all events and process the events in the order in which they were successfully received by the Hub.  Each `DiagnosticReport-update` event posted to the Hub SHALL be processed atomically by the Hub (i.e., all entries in the request's `Bundle` should be processed prior to the Hub accepting another request).

The Hub plays a critical role in helping applications stay synchronized with the current state of exchanged information.  On receiving a `[FHIR resource]-update` request the Hub SHALL examine the `versionId` of the anchor context, which is in the `version` key of the event.   The Hub SHALL compare the `versionId` of the incoming request with the `versionId` the Hub previously assigned to the anchor context (i.e, the `versionId` assigned by the Hub when the previous `DiagnosticReport-open` or `DiagnosticReport-update` request was processed). If the incoming `versionId` and last assigned `versionId` do not match, the request SHALL be rejected and the Hub SHALL return a 4xx/5xx HTTP Status Code.
 
If the `versionId` values match, the Hub proceeds with processing each of the FHIR resources in the Bundle and SHALL process all Bundle entries in an atomic manner.  After updating its copy of the current state of exchanged information, the Hub SHALL assign a new `versionId` to the anchor context and use this new `versionId` in the `DiagnosticReport-update` event it forwards to subscribed applications.  The distributed update event SHALL contain a Bundle resource with the same Bundle `id` which was contained in the request. 

When a  `DiagnosticReport-update` event is received by an application, the application should respond as is appropriate for its clinical use.  For example, an image reading application may choose to ignore an observation describing a patient's blood pressure.  Since transactional change sets are used during information exchange, no problems are caused by applications deciding to ignore exchanged information not relevant to their function.  However, they should read and retain the `versionId` of the anchor context provided in the event for later use.

## Context

### Context
Key | Optionality | FHIR operation to generate context | Description
--- | --- | --- | ---
`report`| REQUIRED | `DiagnosticReport/{id}?_elements=identifier` | Anchor context
`updates` | REQUIRED | not applicable | Changes to be made to the current content of the anchor context 
`version`| REQUIRED | not applicable | Current content version

## Supported Update Request Methods
Each entry in the `updates` Bundle resource must contain one of the below `method` values in the entry's `request` attribute.

Request Method | Operation
--- | ---
`POST` | Add a new resource
`PUT` | Replace/update an existing resource
`PATCH` | Update attributes of an existing resource
`DELETE` | Remove an existing resource

## Examples

### DiagnosticReport-update Request Example
The following example shows adding an imaging study to the existing diagnostic report context.  The `context` holds the `id` and `versionId` of the diagnostic report as required in all  `DiagnosticReport-update` events.  The Bundle holds the addition (POST) of an imaging study and adds (POST) an observation derived from this study. 

```
{
  "timestamp": "2019-09-10T14:58:45.988Z",
  "id": "0d4c7776",
  "event": {
    "hub.topic": "DrXRay",
    "hub.event": "DiagnosticReport-update",
    "context": [
      {
        "key": "report",
        "resource": {
          "resourceType": "DiagnosticReport",
          "id": "40012366"
        }
      },
      {
        "key": "updates",
        "resource": {
          "resourceType": "Bundle",
          "id": "8i7tbu6fby5fuuey7133eh",
          "type": "transaction",
          "entry": [
            {
              "request": {
                "method": "POST"
              },
              "resource": {
                "resourceType": "ImagingStudy",
                "description": "CHEST XRAY",
                "started": "2010-02-14T01:10:00.000Z",
                "id": "3478116342",
                "identifier": [
                  {
                    "type": {
                      "coding": [
                        {
                          "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                          "code": "ACSN"
                        }
                      ]
                    },
                    "value": "3478116342"
                  },
                  {
                    "system": "urn:dicom:uid",
                    "value": "urn:oid:2.16.124.113543.6003.1154777499.30276.83661.3632298176"
                  }
                ]
              }
            },
            {
              "request": {
                "method": "POST"
              },
              "resource": {
                "resourceType": "Observation",
                "id": "435098234",
                "partOf": {
                  "reference": "ImagingStudy/3478116342"
                },
                "status": "preliminary",
                "category": {
                  "system": "http://terminology.hl7.org/CodeSystem/observation-category",
                  "code": "imaging",
                  "display": "Imaging"
                },
                "code": {
                  "coding": [
                    {
                      "system": "http://www.radlex.org",
                      "code": "RID49690",
                      "display": "simple cyst"
                    }
                  ] 
                },
                "issued": "2020-09-07T15:02:03.651Z"
              }
            }
          ]
        }
      },
      {
        "key": "version",
        "versionId": "1"
      }
    ]
  }
}
```

#### DiagnosticReport-update Event Example
The HUB SHALL distribute a corresponding event to all applications currently subscribed to the topic. The Hub SHALL replace the `versionId` in the request with a new `versionId` generated and retained by the Hub.  The prior `versionId` of the content is also provided in the `version` key to ensure that a client is currently in sync with the content prior to applying the new changes.

```
{
  "timestamp": "2019-09-10T14:58:45.988Z",
  "id": "0d4c7776",
  "event": {
    "hub.topic": "DrXRay",
    "hub.event": "DiagnosticReport-update",
    "context": [
      {
        "key": "report",
        "resource": {
          "resourceType": "DiagnosticReport",
          "id": "40012366"
        }
      },
      {
        "key": "updates",
        "resource": {
          "resourceType": "Bundle",
          "id": "8i7tbu6fby5fuuey7133eh",
          "type": "transaction",
          "entry": [
            {
              "request": {
                "method": "POST"
              },
              "resource": {
                "resourceType": "ImagingStudy",
                "description": "CHEST XRAY",
                "started": "2010-02-14T01:10:00.000Z",
                "id": "3478116342",
                "identifier": [
                  {
                    "type": {
                      "coding": [
                        {
                          "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                          "code": "ACSN"
                        }
                      ]
                    },
                    "value": "3478116342"
                  },
                  {
                    "system": "urn:dicom:uid",
                    "value": "urn:oid:2.16.124.113543.6003.1154777499.30276.83661.3632298176"
                  }
                ]
              }
            },
            {
              "request": {
                "method": "POST"
              },
              "resource": {
                "resourceType": "Observation",
                "id": "435098234",
                "partOf": {
                  "reference": "ImagingStudy/3478116342"
                },
                "status": "preliminary",
                "category": {
                  "system": "http://terminology.hl7.org/CodeSystem/observation-category",
                  "code": "imaging",
                  "display": "Imaging"
                },
                "code": {
                  "coding": [
                    {
                      "system": "http://www.radlex.org",
                      "code": "RID49690",
                      "display": "simple cyst"
                    }
                  ] 
                },
                "issued": "2020-09-07T15:02:03.651Z"
              }
            }
          ]
        }
      },
      {
        "key": "version",
        "versionId": "2",
        "priorVersionId": "1"
      }
    ]
  }
}
```

## Change Log
Version | Description
---- | ----
0.1 | Initial draft