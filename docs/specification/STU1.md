<img style="float: left;padding-right: 5px;" src="/img/hl7-logo-header.png" width=90px" />
# FHIRcast

> "Standard for Trial Use" (STU1) This is the 1.0 release of the FHIRcast specification. We are currently working towards a 1.1 release and would love your feedback and proposed changes. Look at our [current issue list](https://github.com/fhircast/docs/issues) and get involved!

## Overview
The FHIRcast specification describes the APIs used to synchronize disparate healthcare applications' user interfaces in real time,  allowing them to show the same clinical content to a user (or group of users). 

Once the subscribing app [knows about the session](#session-discovery), the app may [subscribe](#subscribing-and-unsubscribing) to specific workflow-related events for the given session. The subscription is [verified](#intent-verification-request) and the app is [notified](event-notification) when those workflow-related events occur; for example, by the clinician opening a patient's chart. The subscribing app may [initiate context changes](#request-context-change) by accessing APIs exposed by the Hub; for example, closing the patient's chart. The app [deletes its subscription](#unsubscribe) to no longer receive notifications. The notification message describing the workflow event is a simple json wrapper around one or more FHIR resources. 

FHIRcast is modeled on the webhook design pattern and specifically the [W3C WebSub RFC](https://www.w3.org/TR/websub/), such as its use of GET vs POST interactions. FHIRcast recommends the [HL7 SMART on FHIR launch protocol](http://www.hl7.org/fhir/smart-app-launch) for both session discovery and API authentication. The below [flow diagram](https://drive.google.com/file/d/1wyOXdp0U6tyYc5zx0SrVNfnNlMvt3KjX/view?usp=sharing) illustrates the series of interactions. 

![FHIRcast flow diagram overview](/img/FHIRcast%20overview%20for%20abstract.png)

All data exchanged through the HTTP APIs SHALL be sent and received as [JSON](https://tools.ietf.org/html/rfc8259) structures, and SHALL be transmitted over channels secured using the Hypertext Transfer Protocol (HTTP) over Transport Layer Security (TLS), also known as HTTPS and defined in [RFC2818](https://tools.ietf.org/html/rfc2818). 

## Session Discovery

A session is an abstract concept representing a shared workspace, such as user's login session over multiple applications or a shared view of one application distributed to multiple users. FHIRcast requires a session to have a unique, unguessable and opaque identifier. This identifier is exchanged as the value of the `hub.topic` parameter. Before establishing a subscription, an app must not only know the `hub.topic`, but also the the `hub.url` which contains the base url of the hub. 

Systems SHOULD use SMART on FHIR to authorize, authenticate and exchange the `hub.url` and `hub.topic` as SMART on FHIR launch context parameters. If using SMART, the app SHALL either be launched from the driving application following the [SMART on FHIR EHR launch](http://www.hl7.org/fhir/smart-app-launch#ehr-launch-sequence) flow or the app may initiate the launch following the [SMART on FHIR standalone launch](http://www.hl7.org/fhir/smart-app-launch/#standalone-launch-sequence). In either case, the app SHALL request and, if authorized, SHALL be granted the `fhircast` OAuth2.0 scope. Accompanying this scope grant, the authorization server SHALL supply the `hub.url` and `hub.topic` SMART launch parameters alongside the access token. Per SMART, when scopes of `openid` and `fhirUser` are granted, the authorization server SHALL additionally send the current user's identity in an `id_token`.

If not using SMART on FHIR, the mechanism enabling the app to discover the `hub.url` and `hub.topic` is not defined in FHIRcast.

### SMART Launch Example
Note that the SMART launch parameters include the Hub's base url and and the session identifier in the `hub.url` and `hub.topic` fields.

```
{
  "access_token": "i8hweunweunweofiwweoijewiwe",
  "token_type": "bearer",
  "expires_in": 3600,
  "patient":  "123",
  "encounter": "456",
  "imagingstudy": "789",
  "hub.url" : "https://hub.example.com",
  "hub.topic": "fdb2f928-5546-4f52-87a0-0648e9ded065",
}
```
Although FHIRcast works best with the SMART on FHIR launch and authorization process, implementation-specific launch, authentication, and authorization protocols may be possible. See [other launch scenarios](/launch-scenarios) for guidance.

## Subscribing and Unsubscribing

Subscribing consists of two exchanges:

* Subscriber requests a subscription at the `hub.url` url.
* Hub confirms the subscription was actually requested by the subscriber by contacting the `hub.callback` url. 

Unsubscribing works in the same way, except with a single parameter changed to indicate the desire to unsubscribe.

### Subscription Request
To create a subscription, the subscribing app SHALL perform an HTTP POST ([RFC7231](https://www.w3.org/TR/websub/#bib-RFC7231)) to the Hub's base url (as specified in `hub.url`) with the parameters in the table below.

This request SHALL have a `Content-Type` header of `application/x-www-form-urlencoded` and SHALL use the following parameters in its body, formatted accordingly:

Field | Optionality | Type | Description
---------- | ----- | -------- | --------------
`hub.callback` | Required | *string* | The Subscriber's callback URL where notifications should be delivered. The callback URL SHOULD be an unguessable URL that is unique per subscription.
`hub.mode` | Required | *string* | The literal string "subscribe" or "unsubscribe", depending on the goal of the request.
`hub.topic` | Required | *string* | The identifier of the user's session that the subscriber wishes to subscribe to or unsubscribe from. MAY be a guid.
`hub.secret` | Required | *string* | A subscriber-provided cryptographically random unique secret string that SHALL be used to compute an [HMAC digest](https://www.w3.org/TR/websub/#bib-RFC6151) delivered in each notification. This parameter SHALL be less than 200 bytes in length.
`hub.events` | Required | *string* | Comma-separated list of event types from the Event Catalog for which the Subscriber wants notifications.
`hub.lease_seconds` | Optional | *number* | Number of seconds for which the subscriber would like to have the subscription active, given as a positive decimal integer. Hubs MAY choose to respect this value or not, depending on their own policies, and MAY set a default value if the subscriber omits the parameter. If using OAuth, the hub SHALL limit the subscription lease seconds to be less than or equal to the access token's expiration.

If OAuth2 authentication is used, this POST request SHALL contain the Bearer access token in the HTTP Authorization header.

Hubs SHALL allow subscribers to re-request subscriptions that are already activated. Each subsequent and verified request to a Hub to subscribe or unsubscribe SHALL override the previous subscription state for a specific topic / callback URL combination.

The callback URL MAY contain arbitrary query string parameters (e.g., `?foo=bar&red=fish`). Hubs SHALL preserve the query string during subscription verification by appending new, Hub-defined, parameters to the end of the list using the `&` (ampersand) character to join. When sending the event notifications, the Hub SHALL make a POST request to the callback URL including any query string parameters in the URL portion of the request, not as POST body parameters.

The client that creates the subscription may not be the same system as the server hosting the callback url. (For example, some type of federated authorization model could possibly exist between these two systems.) However, in FHIRcast, the Hub assumes that the same authorization and access rights apply to both the subscribing client and the callback url.

#### Subscription Request Example
In this example, the app asks to be notified of the `open-patient-chart` and `close-patient-chart` events.
```
POST https://hub.example.com
Host: hub.example.com
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/x-www-form-urlencoded

hub.callback=https%3A%2F%2Fapp.example.com%2Fsession%2Fcallback%2Fv7tfwuk17a&hub.mode=subscribe&hub.topic=fdb2f928-5546-4f52-87a0-0648e9ded065&hub.secret=shhh-this-is-a-secret&hub.events=patient-open-chart,patient-close-chart
```

### Subscription Response
If the Hub URL supports FHIRcast and is able to handle the subscription or unsubscription request, the Hub SHALL respond to a subscription request with an HTTP 202 "Accepted" response to indicate that the request was received and will now be verified by the Hub. The Hub SHOULD perform the verification of intent as soon as possible.

If a Hub finds any errors in the subscription request, an appropriate HTTP error response code (4xx or 5xx) MUST be returned. In the event of an error, the Hub SHOULD return a description of the error in the response body as plain text, used to assist the client developer in understanding the error. This is not meant to be shown to the end user. Hubs MAY decide to reject some callback URLs or topic based on their own policies.

#### Subscription Response Example
```
HTTP/1.1 202 Accepted
```

### Subscription Denial

If (and when) the subscription is denied, the Hub SHALL inform the subscriber by sending an HTTP GET request to the subscriber's callback URL as given in the subscription request. This can occur when the subscription is requested for a variety of reasons, or it can occur after the subscription had already been accepted because the Hub no longer supports that subscription (e.g. it has expired). This request has the following query string arguments appended, to which the subscriber SHALL respond with an HTTP success (2xx) code.

Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.mode` | Required | *string* | The literal string "denied".
`hub.topic` | Required | *string* | The topic given in the corresponding subscription request. MAY be a guid.
`hub.events` | Required | *string* | A comma-separated list of events from the Event Catalog corresponding to the events string given in the corresponding subscription request. 
`hub.reason` | Optional | *string* | The Hub may include a reason for which the subscription has been denied. The subscription MAY be denied by the Hub at any point (even if it was previously accepted). The Subscriber SHOULD then consider that the subscription is not possible anymore.

The below [flow diagram](https://drive.google.com/file/d/1Z7Z7mw0f_gm8lqdBJcwqQV8MD9PnVhQs/view?usp=sharing) and example illustrate the subscription denial sequence and message details.

###### Subscription Denial Sequence
![Subscription denial flow diagram](../img/Denied%20Subscription%20Sequence.png)

###### Subscription Denial Example
```
GET https://app.example.com/session/callback/v7tfwuk17a?hub.mode=denied&hub.topic=fdb2f928-5546-4f52-87a0-0648e9ded065hub.events=open-patient-chart,close-patient-chart&hub.reason=session+unexpectedly+stopped HTTP 1.1
Host: subscriber
```

### Intent Verification
If the subscription is not denied, the Hub SHALL perform the verification of intent of the subscriber, this applies to apps unsubscribing as well. The `hub.callback` url verification process ensures that the subscriber actually controls the callback url.

#### Intent Verification Request
In order to prevent an attacker from creating unwanted subscriptions on behalf of a subscriber (or unsubscribing desired ones), a hub must ensure that the subscriber did indeed send the subscription request. The hub SHALL verify a subscription request by sending an HTTPS GET request to the subscriber's callback URL as given in the subscription request. This request SHALL have the following query string arguments appended

Field | Optionality | Type | Description
---  | --- | --- | --- 
`hub.mode` | Required | *string* | The literal string "subscribe" or "unsubscribe", which matches the original request to the hub from the subscriber.
`hub.topic` | Required | *string* | The session topic given in the corresponding subscription request. MAY be a guid.
`hub.events` | Required | *string* | A comma-separated list of events from the Event Catalog corresponding to the events string given in the corresponding subscription request. 
`hub.challenge` | Required | *string* | A hub-generated, random string that SHALL be echoed by the subscriber to verify the subscription.
`hub.lease_seconds` | Required | *number* | The hub-determined number of seconds that the subscription will stay active before expiring, measured from the time the verification request was made from the hub to the subscriber. If provided to the client, the hub SHALL unsubscribe the client once `lease_seconds` has expired. If the subscriber wishes to continue the subscription it MAY resubscribe.

##### Intent Verification Request Example
```
GET https://app.example.com/session/callback/v7tfwuk17a?hub.mode=subscribe&hub.topic=fdb2f928-5546-4f52-87a0-0648e9ded065&hub.events=open-patient-chart,close-patient-chart&hub.challenge=meu3we944ix80ox&hub.lease_seconds=7200 HTTP 1.1
Host: subscriber
```

#### Intent Verification Response
If the `hub.topic` of the Intent Verification Request corresponds to a pending subscription or unsubscription that the subscriber wishes to carry out it SHALL respond with an HTTP success (2xx) code, a header of `Content-Type: text/html`, and a response body equal to the `hub.challenge` parameter. If the subscriber does not agree with the action, the subscriber SHALL respond with a 404 "Not Found" response.

The Hub SHALL consider other server response codes (3xx, 4xx, 5xx) to mean that the verification request has failed. If the subscriber returns an HTTP success (2xx) but the content body does not match the `hub.challenge` parameter, the Hub SHALL also consider verification to have failed.

The below [flow diagram](https://drive.google.com/file/d/1VcgI3dn6mAXPXkNaxRJzaBfl2HqQZUKW/view?usp=sharing) and example illustrate the successful subscription sequence and message details.

###### Successful Subscription Sequence
![Successful subscription flow diagram](../img/Successful%20Subscription%20Sequence.png)

###### Intent Verification Response Example
```
HTTP/1.1 200 OK
Content-Type: text/html

meu3we944ix80ox
```

> NOTE
> The spec uses GET vs POST to differentiate between the confirmation/denial of the subscription request and delivering the content. While this is not considered "best practice" from a web architecture perspective, it does make implementation of the callback URL simpler. Since the POST body of the content distribution request may be any arbitrary content type and only includes the actual content of the document, using the GET vs POST distinction to switch between handling these two modes makes implementations simpler.

### Unsubscribe

Once a subscribing app no longer wants to receive event notifications, it SHALL unsubscribe from the session. The unsubscribe request message mirrors the subscribe request message with only a single difference: the `hub.mode` MUST be equal to the lowercase string _unsubscribe_.

#### Unsubscribe Request Example

```
POST https://hub.example.com
Host: hub
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/x-www-form-urlencoded

hub.callback=https%3A%2F%2Fapp.example.com%2Fsession%2Fcallback%2Fv7tfwuk17a&hub.mode=unsubscribe&hub.topic=fdb2f928-5546-4f52-87a0-0648e9ded065&hub.secret=shhh-this-is-a-secret&hub.events=open-patient-chart,close-patient-chart

```


## Event Notification

The Hub SHALL notify subscribed apps of workflow-related events to which the app is subscribed, as the event occurs. The notification is an HTTPS POST containing a JSON object in the request body.

### Event Notification Request

Using the `hub.secret` from the subscription request, the hub SHALL generate an HMAC signature of the payload and include that signature in the request headers of the notification. The `X-Hub-Signature` header's value SHALL be in the form _method=signature_ where method is one of the recognized algorithm names and signature is the hexadecimal representation of the signature. The signature SHALL be computed using the HMAC algorithm ([RFC6151](https://www.w3.org/TR/websub/#bib-RFC6151)) with the request body as the data and the `hub.secret` as the key.

The notification to the subscriber SHALL include a description of the subscribed event that just occurred, an ISO 8601-2 formatted timestamp in UTC and an event identifier that is universally unique for the Hub. The timestamp MAY be used by subscribers to establish message affinity (message ordering) through the use of a message queue. The event identifier MAY be used to differentiate retried messages from user actions.

#### Event Notification Request Details

The notification's `hub.event` and `context` fields inform the subscriber of the current state of the user's session. The `hub.event` is a user workflow event, from the Event Catalog (or an organization-specific event in reverse-domain name notation). The `context` is an array of named FHIR resources (similar to [CDS Hooks's context](https://cds-hooks.hl7.org/1.0/#http-request_1) field) that describe the current content of the user's session. Each event in the [Event Catalog](#event-catalog) defines what context is expected in the notification. Hubs MAY use the [FHIR _elements parameter](https://www.hl7.org/fhir/search.html#elements) to limit the size of the data being passed while also including additional, local identifiers that are likely already in use in production implementations. Subscribers SHALL accept a full FHIR resource or the [_elements](https://www.hl7.org/fhir/search.html#elements)-limited resource as defined in the Event Catalog.

Field | Optionality | Type | Description
--- | --- | --- | ---
`timestamp` | Required | *string* | ISO 8601-2 timestamp in UTC describing the time at which the event occurred with subsecond accuracy. 
`id` | Required | *string* | Event identifier used to recognize retried notifications. This id SHALL be unique for the Hub, for example a GUID.
`event` | Required | *object* | A json object describing the event. See below.


Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | The session topic given in the subscription request. MAY be a guid.
`hub.event`| Required | string | The event that triggered this notification, taken from the list of events from the subscription request.
`context` | Required | array | An array of named FHIR objects corresponding to the user's context after the given event has occurred. Common FHIR resources are: Patient, Encounter, ImagingStudy and List. The Hub SHALL only return FHIR resources that are authorized to be accessed with the existing OAuth2 access_token.

#### Event Notification Request Example

```
POST https://app.example.com/session/callback/v7tfwuk17a HTTP/1.1
Host: subscriber
X-Hub-Signature: sha256=dce85dc8dfde2426079063ad413268ac72dcf845f9f923193285e693be6ff3ae

{
  "timestamp": "2018-01-08T01:37:05.14",
  "id": "q9v3jubddqt63n1",
  "event": {
    "hub.topic": "fdb2f928-5546-4f52-87a0-0648e9ded065",
    "hub.event": "open-patient-chart",
    "context": [
      {
        "key": "patient",
        "resource": {
          "resourceType": "Patient",
          "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
          "identifier": [
             {
               "type": {
                    "coding": [
                        {
                            "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                            "value": "MR",
                            "display": "Medication Record Number"
                         }
                        "text": "MRN"
                      ]
                  }
              }
          ]
        }
      }
    ]
  }
}
```

### Event Notification Response

The subscriber SHALL respond to the notification with an appropriate HTTP status code. In the case of a successful notification, the subscriber SHALL respond with an HTTP [RFC7231] 2xx response code to indicate a success; otherwise, the subscriber SHALL respond with an HTTP error status code. The Hub MAY use these statuses to track synchronization state.

#### Event Notification Response Example

```
HTTP/1.1 200 OK
```

## Request Context Change

Similar to the Hub's notifications to the subscriber, the subscriber MAY request context changes with an HTTP POST to the `hub.url`. The Hub SHALL either accept this context change by responding with any successful HTTP status or reject it by responding with any 4xx or 5xx HTTP status. The subscriber SHALL be capable of gracefully handling a rejected context request. 

Once a requested context change is accepted, the Hub SHALL broadcast the context notification to all subscribers, including the original requestor. 

### Request Context Change Request
###### Request Context Change Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`timestamp` | Required | *string* | ISO 8601-2 timestamp in UTC describing the time at which the event occurred with subsecond accuracy. 
`id` | Required | *string* | Event identifier, which MAY be used to recognize retried notifications. This id SHALL be uniquely generated by the subscriber and could be a GUID. Following an accepted context change request, the hub MAY re-use this value in the broadcasted event notifications.
`event` | Required | *object* | A json object describing the event. See [below](#request-context-change-event-object-parameters).

###### Request Context Change Event Object Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | The session topic given in the subscription request. MAY be a guid.
`hub.event`| Required | string | The event that triggered this request for the subscriber, taken from the list of events from the subscription request.
`context` | Required | array | An array of named FHIR objects corresponding to the user's context after the given event has occurred. Common FHIR resources are: Patient, Encounter, ImagingStudy and List. The subscriber SHALL only include FHIR resources that are authorized to be accessed with the existing OAuth2 `access_token`.

```
POST https://hub.example.com/7jaa86kgdudewiaq0wtu HTTP/1.1
Host: hub
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/json

{
  "timestamp": "2018-01-08T01:40:05.14",
  "id": "wYXStHqxFQyHFELh",
  "event": {
    "hub.topic": "fdb2f928-5546-4f52-87a0-0648e9ded065",
    "hub.event": "close-patient-chart",
    "context": [
      {
        "key": "patient",
        "resource": {
          "resourceType": "Patient",
          "id": "798E4MyMcpCWHab9",
          "identifier": [
             {
               "type": {
                    "coding": [
                        {
                            "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                            "value": "MR",
                            "display": "Medication Record Number"
                         }
                        "text": "MRN"
                      ]
                  }
              }
          ]
        }
      }
    ]
  }
}
```

## Event Notification Errors

If the subscriber cannot follow the context of the event, for instance due to an error or a deliberate choice to not follow a context, the subscriber MAY respond with a 'sync-error' event. The Hub MAY use these events to track synchronization state and MAY also forward these events to other subscribers of the same topic.

### Event Notification Error Example

```
POST https://hub.example.com/7jaa86kgdudewiaq0wtu HTTP/1.1
Host: hub
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/json

{
  "timestamp": "2018-01-08T01:37:05.14",
  "id": "q9v3jubddqt63n1",
  "event": {
    "hub.topic": "fdb2f928-5546-4f52-87a0-0648e9ded065",
    "hub.event": "sync-error",
    "context": [
      {
        "key": "operationoutcome",
        "resource": {
          "resourceType": "OperationOutcome",
          "issue": [
            {
              "severity": "warning",
              "code": "processing",
              "diagnostics": "AppId3456 failed to follow context"
            }
          ]
        }
      }
    ]
  }
}
```

## Event Catalog
Each event definition in the catalog, below, specifies a single event name, a description of the event, and the  required or optional contextual information associated with the event. Alongside the event name, the contextual information is used by the subscriber.

FHIR is the interoperable data model used by FHIRcast. The fields within `context` are subsets of FHIR resources. Hubs MUST send the results of FHIR reads in the context, as specified below. For example, when the `open-image-study` event occurs, the notification sent to a subscriber MUST include the ImagingStudy FHIR resource. Hubs SHOULD send the results of an ImagingStudy FHIR read using the *_elements* query parameter, like so:  `ImagingStudy/{id}?_elements=identifier,accession` and in accordance with the [FHIR specification](https://www.hl7.org/fhir/search.html#elements). 

A FHIR server may not support the *_elements* query parameter; a subscriber MUST gracefully handle receiving a full FHIR resource in the context of a notification.

The name of the event SHOULD succinctly and clearly describe the activity or event. Event names are unique so event creators SHOULD take care to ensure newly proposed events do not conflict with an existing event name. Event creators SHALL name their event with reverse domain notation (e.g. `org.example.patient-transmogrify`) if the event is specific to an organization. Reverse domain notation SHALL not be used by a standard event catalog.

### open-patient-chart
#### Description: 
User opened patient's medical record. 
#### Example: 
```
{
  "context": [
    {
      "key": "patient",
      "resource": {
        "resourceType": "Patient",
        "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
        "identifier": [
           {
             "type": {
                  "coding": [
                      {
                          "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                          "value": "MR",
                          "display": "Medication Record Number"
                       }
                      "text": "MRN"
                    ]
                }
            }
        ]
      }
    }
  ]
}
```

Context | Optionality | FHIR operation to generate context|  Description
--- | --- | --- | ---
`patient` | Required| `Patient/{id}?_elements=identifier` | FHIR Patient resource describing the patient whose chart is currently in context.
`encounter` | Optional | `Encounter/{id}?_elements=identifier` | FHIR Encounter resource in context in the newly opened patient's chart.


### switch-patient-chart

#### Description: 
User changed from one open patient's medical record to another previously opened patient's medical record. The context documents the patient whose record is currently open.
#### Example: 
```
{
  "context": [
    {
      "key": "patient",
      "resource": {
        "resourceType": "Patient",
        "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
        "identifier": [
           {
             "type": {
                  "coding": [
                      {
                          "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                          "value": "MR",
                          "display": "Medication Record Number"
                       }
                      "text": "MRN"
                    ]
                }
            }
        ]
      }
    }
  ]
}
```

Context | Optionality | FHIR operation to generate context|  Description
--- | --- | --- | ---
`patient` | Required |  `Patient/{id}?_elements=identifier` | FHIR Patient resource describing the patient whose chart is currently in context..
`encounter` | Optional | `Encounter/{id}?_elements=identifier` | FHIR Encounter resource in context in the newly opened patient's chart.


### close-patient-chart

#### Description: User closed patient's medical record. 

#### Example: 
```
{
  "context": [
    {
      "key": "patient",
      "resource": {
        "resourceType": "Patient",
        "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
        "identifier": [
           {
             "type": {
                  "coding": [
                      {
                          "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                          "value": "MR",
                          "display": "Medication Record Number"
                       }
                      "text": "MRN"
                    ]
                }
            }
        ]
      }
    }
  ]
}
```

Context | Optionality | FHIR operation to generate context|  Description
--- | --- | --- | ---
`patient` | Required |  `Patient/{id}?_elements=identifier` | FHIR Patient resource describing the patient whose chart is currently in context..
`encounter` | Optional | `Encounter/{id}?_elements=identifier` | FHIR Encounter resource in context in the newly opened patient's chart.

### open-imaging-study
#### Description: User opened record of imaging study.
#### Example: 
```
{
  "context": [
    {
      "key": "patient",
      "resource": {
        "resourceType": "Patient",
        "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
        "identifier": [
          {
            "system": "urn:oid:1.2.840.114350",
            "value": "185444"
          },
          {
            "system": "urn:oid:1.2.840.114350.1.13.861.1.7.5.737384.27000",
            "value": "2667"
          }
        ]
      }
    },
    {
      "key": "study",
      "resource": {
        "resourceType": "ImagingStudy",
        "id": "8i7tbu6fby5ftfbku6fniuf",
        "uid": "urn:oid:2.16.124.113543.6003.1154777499.30246.19789.3503430045",
        "identifier": [
          {
            "system": "7678",
            "value": "185444"
          }
        ],
        "patient": {
          "reference": "Patient/ewUbXT9RWEbSj5wPEdgRaBw3"
        }
      }
    }
  ]
}
```

Context | Optionality | FHIR operation to generate context|  Description
--- | --- | --- | ---
`patient` | Optional| `Patient/{id}?_elements=identifier`| FHIR Patient resource describing the patient whose chart is currently in context.
`study` | Required | `ImagingStudy/{id}?_elements=identifier,accession` | FHIR ImagingStudy resource in context. Note that in addition to the request identifier and accession elements, the DICOM uid and FHIR patient reference are included because they're required by the FHIR specification. 

### switch-imaging-study
#### Description: User changed from one open imaging study to another previously opened imaging study. The context documents the study, and optionally patient, for the currently open record.
#### Example: 
```
{
  "context": [
    {
      "key": "patient",
      "resource": {
        "resourceType": "Patient",
        "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
        "identifier": [
          {
            "system": "urn:oid:1.2.840.114350",
            "value": "185444"
          },
          {
            "system": "urn:oid:1.2.840.114350.1.13.861.1.7.5.737384.27000",
            "value": "2667"
          }
        ]
      }
    },
    {
      "key": "study",
      "resource": {
        "resourceType": "ImagingStudy",
        "id": "8i7tbu6fby5ftfbku6fniuf",
        "uid": "urn:oid:2.16.124.113543.6003.1154777499.30246.19789.3503430045",
        "identifier": [
          {
            "system": "7678",
            "value": "185444"
          }
        ],
        "patient": {
          "reference": "Patient/ewUbXT9RWEbSj5wPEdgRaBw3"
        }
      }
    }
  ]
}
```

Context | Optionality | FHIR operation to generate context|  Description
--- | --- | --- | ---
`patient` | Optional| `Patient/{id}?_elements=identifier` | FHIR Patient resource describing the patient whose chart is currently in context.
`study` | Required | `ImagingStudy/{id}?_elements=identifier,accession` | FHIR ImagingStudy resource in context. Note that in addition to the request identifier and accession elements, the DICOM uid and FHIR patient reference are included because they're required by the FHIR specification. 

### close-imaging-study

#### Description: User closed imaging study.

#### Example: 
```
{
  "context": [
    {
      "key": "patient",
      "resource": {
        "resourceType": "Patient",
        "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
        "identifier": [
          {
            "system": "urn:oid:1.2.840.114350",
            "value": "185444"
          },
          {
            "system": "urn:oid:1.2.840.114350.1.13.861.1.7.5.737384.27000",
            "value": "2667"
          }
        ]
      }
    },
    {
      "key": "study",
      "resource": {
        "resourceType": "ImagingStudy",
        "id": "8i7tbu6fby5ftfbku6fniuf",
        "uid": "urn:oid:2.16.124.113543.6003.1154777499.30246.19789.3503430045",
        "identifier": [
          {
            "system": "7678",
            "value": "185444"
          }
        ],
        "patient": {
          "reference": "Patient/ewUbXT9RWEbSj5wPEdgRaBw3"
        }
      }
    }
  ]
}
```

Context | Optionality | FHIR operation to generate context|  Description
--- | --- | --- | ---
`patient` | Optional| `Patient/{id}?_elements=identifier` | FHIR Patient resource describing the patient whose chart is currently in context.
`study` | Required | `ImagingStudy/{id}?_elements=identifier,accession` | FHIR ImagingStudy resource in context. Note that in addition to the request identifier and accession elements, the DICOM uid and FHIR patient reference are included because they're required by the FHIR specification. 

### user-logout
#### Description: User gracefully exited the application.
#### Example: 
{
}


No Context 

### user-hibernate
#### Description: User temporarily suspended her session. The user's session will eventually resume.
#### Example: 
{
}

No Context

### sync-error
#### Description: A syncronization error has been detected. Inform subscribed clients.
#### Example: 
```
{
  "context": [
    {
      "key": "operationoutcome",
      "resource": {
        "resourceType": "OperationOutcome",
        "issue": [
          {
            "severity": "warning",
            "code": "processing",
            "diagnostics": "AppId3456 failed to follow context"
          }
        ]
      }
    }
  ]
}
```

Context | Optionality | FHIR operation to generate context|  Description
--- | --- | --- | ---
`operationoutcome` | Optional |  `OperationOutcome` | FHIR resource describing an outcome of an unsuccessful system action..