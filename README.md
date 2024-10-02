## Design
The SDS is built upon a HAPI FHIR server (version 6.10.1) with a PostgreSQL database serving as its backend. Below is a breakdown of the key design choices:

- **Partitioning**: The SDS manages the creation of local and foreign paritions as well as `Linkage` resources. 
  - **Local**: The local partition stores `Patient`, `Linkage`, `RelatedPerson`, and other resources, such as `Goal` and `QuestionnaireResponse` resources, associated with Patient-Reported Outcomes (PROs).
  - **Foreign**: The SDS creates a foreign parition for each third-party FHIR endpoint.
  - Resources from third-party FHIR endpoints that are ***not*** contained within the `Patient` compartment, such as `Medication` or `Practitioner` resources, are ***not*** stored.
- **Access**: 
  - **Authentication**: Access tokens are introspected to verify each user's identity.
  - **Authorization**: The SDS manages the creation of `Linkage` and `RelatedPerson` resources in the local partition. These resources determine authorization for specific resources.
  - Providers are granted read-only access to all partitions, with provider status confirmed via token introspection.

## Resources

### Extending the Linkage Resource
`Linkage` resources use extensions to identify the partition for each reference. From the `Linkage` resource, the `item` with a type of "source" is used for reading from and writing to the local partition. In the  example below, the `item` with `type` of "source" and `valueUrl` of "SDS-LOCAL" refers to the local partition, while the `item` with `type` of "alternate" and `vauleUrl` of "https://example.org/fhir" refers to the foreign partition.

##### Example
```json
{
  "resourceType": "Linkage",
  "id": "31477a20-2749-4d01-a232-a12000493600",
  "meta": {
    "versionId": "1",
    "lastUpdated": "2024-09-18T15:58:39.104+00:00"
  },
  "item": [
    {
      "type": "source",
      "resource": {
        "extension": [
          {
            "url": "urn:sds:partition-name",
            "valueUrl": "SDS-LOCAL"
          }
        ],
        "reference": "Patient/a210436a-f32b-4a6d-b924-efb01920b40e"
      }
    },
    {
      "type": "alternate",
      "resource": {
        "extension": [
          {
            "url": "urn:sds:partition-name",
            "valueUrl": "https://example.org/fhir"
          }
        ],
        "reference": "Patient/123"
      }
    }
  ]
}
```

## API Calls

### Query Linkage Resources

##### Process Flow

![alt text](./diagram.png)

1. Determine if the patient already exists within the SDS by querying for a `Linkage` resource. Use the `Patient` reference (e.g., Patient/123) as the value for the `item` parameter in the GET request.
###### Example:
GET http://localhost:8080/fhir/Linkage?item=Patient/123
|Header|Value|Notes|
|---|---|---|
|Accept|application/fhir+json||
|Authorization|Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJhY3RpdmUiOnRydWUsInNjb3BlIjoicGF0aWVudC8qIG9wZW5pZCBmaGlyVXNlciIsInN1YiI6Imh0dHBzOi8vZXhhbXBsZS5vcmcvZmhpci9QYXRpZW50LzEyMyJ9.gPkk0A4N2I4mb9RmzXmyutCsyzVsEWdYjrC4MBWjaBRrVV2nQucbTEqwgaxUMMThlbe8s3V7VDVsHhzzjCq-2Z0nB9BGXmMaVe7e4zFVA41DKfuM50BSsC1CcEjXVZ_r6T7Y1vAij-y3N_LZIV-cYzs8HA4ue6m3FuaJLzQwo6KysnjdolM2NDInpHZPoBXZ_1e-Z2L34KDT4G9Sx2Uu7p3kpJmKRygSQgiJEnGHi1slRDTyMtJF7SzlIIZ6Oh4-y65oL3lhSIc7nzYek1teAR7LNAQq5867bFbb5g1vDhA4LuUDFLBA6zxY5gsD45fe8R31Dim7ILOEUIMtFEqGjA|The `sub` claim within the token matches the patient ID used in the `item` parameter|

2. If the Linkage query returns a `Bundle` without any entries, the patient doesn't exist in the SDS and needs to be created. Use a POST request to create the `Patient` resource. Include the a minimal `Patient` resource that does not include the `id` (e.g., {"resourceType":"Patient"}) in the body of the request. The request will create an empty `Patient` resource, or stub, in the foreign partition. After creating the `Patient` resource, re-query for the `Linkage` resource as described above.
##### Example:
POST http://localhost:8080/fhir/Patient
|Header|Value|Notes|
|---|---|---|
|Content-Type|application/fhir+json||
|Authorization|Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJhY3RpdmUiOnRydWUsInNjb3BlIjoicGF0aWVudC8qIG9wZW5pZCBmaGlyVXNlciIsInN1YiI6Imh0dHBzOi8vZXhhbXBsZS5vcmcvZmhpci9QYXRpZW50LzEyMyJ9.gPkk0A4N2I4mb9RmzXmyutCsyzVsEWdYjrC4MBWjaBRrVV2nQucbTEqwgaxUMMThlbe8s3V7VDVsHhzzjCq-2Z0nB9BGXmMaVe7e4zFVA41DKfuM50BSsC1CcEjXVZ_r6T7Y1vAij-y3N_LZIV-cYzs8HA4ue6m3FuaJLzQwo6KysnjdolM2NDInpHZPoBXZ_1e-Z2L34KDT4G9Sx2Uu7p3kpJmKRygSQgiJEnGHi1slRDTyMtJF7SzlIIZ6Oh4-y65oL3lhSIc7nzYek1teAR7LNAQq5867bFbb5g1vDhA4LuUDFLBA6zxY5gsD45fe8R31Dim7ILOEUIMtFEqGjA|The `sub` claim within the token matches the patient ID used in the `item` parameter|

Body:
```
{
  "resourceType": "Patient"
}
```
3. To get the `Linkage` resources for all of a patient's partitions, first query for the `Linkage` resource for the authorized patient in the foreign partition. From the `Linkage` resource returned, identify the `Patient` reference for the local partition (for the `item` with `type` of "source" and `valueUrl` of "SDS-LOCAL"). Then, using the `Patient` reference in the local partition, query for the `Linkage` resources in the local partition.

### Data Access (Read/Write):

Reading and writing data to the SDS requires separate calls to each partition as appropriate. Writing should be performed synchonously to prevent 409 errors that indicates a conflict between a client's request and the current state of a resource on the server.

Local Partition:

To read or write to the local partition, identify the `Patient` reference for the local partition from the `Linkage` resource as described above. Include the `Patient` reference for the local partition as a query parameter for a GET request (e.g., GET http://localhost:8080/fhir/Observation?patient=Patient/a210436a-f32b-4a6d-b924-efb01920b40e) or in the request body for a POST or PUT request. Always include the `Authorization` header in the request. And, for GET requests, include the `Accept` header. For POST requests, include the `Content-Type` header and the JSON resource body. These requests should never include the `X-Partition-Name` header.
###### Example:
GET http://localhost:8080/fhir/Linkage?item=Patient/a210436a-f32b-4a6d-b924-efb01920b40e
|Header|Value|Notes|
|---|---|---|
|Accept|application/fhir+json||
|Authorization|Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJhY3RpdmUiOnRydWUsInNjb3BlIjoicGF0aWVudC8qIG9wZW5pZCBmaGlyVXNlciIsInN1YiI6Imh0dHBzOi8vZXhhbXBsZS5vcmcvZmhpci9QYXRpZW50LzEyMyJ9.gPkk0A4N2I4mb9RmzXmyutCsyzVsEWdYjrC4MBWjaBRrVV2nQucbTEqwgaxUMMThlbe8s3V7VDVsHhzzjCq-2Z0nB9BGXmMaVe7e4zFVA41DKfuM50BSsC1CcEjXVZ_r6T7Y1vAij-y3N_LZIV-cYzs8HA4ue6m3FuaJLzQwo6KysnjdolM2NDInpHZPoBXZ_1e-Z2L34KDT4G9Sx2Uu7p3kpJmKRygSQgiJEnGHi1slRDTyMtJF7SzlIIZ6Oh4-y65oL3lhSIc7nzYek1teAR7LNAQq5867bFbb5g1vDhA4LuUDFLBA6zxY5gsD45fe8R31Dim7ILOEUIMtFEqGjA|The `sub` claim within the token matches the patient ID used in the `item` parameter|

Foreign Partition:

To read or write to the foreign partitions, use the appropriate `Patient` reference and X-Partition-Header to specify the partition in the request (e.g., X-Partition-Name: https://example.org/fhir). To create or update resources in the foreign partitions, always use a PUT request.
###### Example:
PUT http://localhost:8080/fhir/Patient/456
|Header|Value|Notes|
|---|---|---|
|Content-Type|application/fhir+json||
|Authorization|Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJhY3RpdmUiOnRydWUsInNjb3BlIjoicGF0aWVudC8qIG9wZW5pZCBmaGlyVXNlciIsInN1YiI6Imh0dHBzOi8vZXhhbXBsZS5vcmcvZmhpci9QYXRpZW50LzEyMyJ9.gPkk0A4N2I4mb9RmzXmyutCsyzVsEWdYjrC4MBWjaBRrVV2nQucbTEqwgaxUMMThlbe8s3V7VDVsHhzzjCq-2Z0nB9BGXmMaVe7e4zFVA41DKfuM50BSsC1CcEjXVZ_r6T7Y1vAij-y3N_LZIV-cYzs8HA4ue6m3FuaJLzQwo6KysnjdolM2NDInpHZPoBXZ_1e-Z2L34KDT4G9Sx2Uu7p3kpJmKRygSQgiJEnGHi1slRDTyMtJF7SzlIIZ6Oh4-y65oL3lhSIc7nzYek1teAR7LNAQq5867bFbb5g1vDhA4LuUDFLBA6zxY5gsD45fe8R31Dim7ILOEUIMtFEqGjA|The `sub` claim within the token matches the patient ID used in the `item` parameter|
|X-Parition-Name|https://another-example.org/fhir||

Body: 
```
{
    "resourceType": "Patient",
    "id": "456",
    "meta": {
        "versionId": "2",
        "lastUpdated": "2024-09-18T15:59:53.147+00:00",
        "source": "#qH0GIOTJUlS0WiRa"
    },
    "identifier": [
        {
            "type": {
                "coding": [
                    {
                        "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                        "code": "MR",
                        "display": "Medical Record Number"
                    }
                ],
                "text": "Medical Record Number"
            },
            "system": "https://another-example.org",
            "value": "4815162342"
        }
    ],
    "name": [
        {
            "family": "Kent",
            "given": [
                "Clark"
            ]
        }
    ],
    "gender": "male",
    "birthDate": "1938-04-18"
}
```

### Delete:

The SDS supports both logical and physical delete operations. The logical delete operations are performed synchronously while physical delete operations are performed asynchronously. Depending on application requirements, it may be useful to first perform a logical and then a physical delete.

#### Logical Delete

Requests should include the `_cascade` parameter with a value of `delete` (e.g., DELETE http://localhost:8080/fhir/Patient/a210436a-f32b-4a6d-b924-efb01920b40e?_cascade=delete). For deleting from a foreign partition, the request should include the `X-Partition-Name` header.
##### Example:
DELETE http://localhost:8080/fhir/Patient/123?_cascade=delete
|Header|Value|Notes|
|---|---|---|
|Content-Type|application/fhir+json||
|Authorization|Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJhY3RpdmUiOnRydWUsInNjb3BlIjoicGF0aWVudC8qIG9wZW5pZCBmaGlyVXNlciIsInN1YiI6Imh0dHBzOi8vZXhhbXBsZS5vcmcvZmhpci9QYXRpZW50LzEyMyJ9.gPkk0A4N2I4mb9RmzXmyutCsyzVsEWdYjrC4MBWjaBRrVV2nQucbTEqwgaxUMMThlbe8s3V7VDVsHhzzjCq-2Z0nB9BGXmMaVe7e4zFVA41DKfuM50BSsC1CcEjXVZ_r6T7Y1vAij-y3N_LZIV-cYzs8HA4ue6m3FuaJLzQwo6KysnjdolM2NDInpHZPoBXZ_1e-Z2L34KDT4G9Sx2Uu7p3kpJmKRygSQgiJEnGHi1slRDTyMtJF7SzlIIZ6Oh4-y65oL3lhSIc7nzYek1teAR7LNAQq5867bFbb5g1vDhA4LuUDFLBA6zxY5gsD45fe8R31Dim7ILOEUIMtFEqGjA|The `sub` claim within the token matches the patient ID used in the `item` parameter|
|X-Parition-Name|https://example.org/fhir||

#### Physical Delete:

Physical deletes are completed using a POST request with the `$expunge` operation (e.g., POST http://localhost:8080/fhir/Patient/a210436a-f32b-4a6d-b924-efb01920b40e/$expunge). The request parameters should be included in the body of the request. The `X-Partition-Header` should only be used for the authorized foreign patient. The requests to the various partitions should be made synchronously, first to all the foreign partitions (except the foreign partition for the authorized patient) then the local partition and then the foreign partition for the autorized patient. Although these calls need to be made synchronously, the `$expunge` operation occurs asynchronously and does not return a response. 

##### Example:
POST http://localhost:8080/fhir/Patient/123/$expunge
|Header|Value|Notes|
|---|---|---|
|Content-Type|application/fhir+json||
|Authorization|Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJhY3RpdmUiOnRydWUsInNjb3BlIjoicGF0aWVudC8qIG9wZW5pZCBmaGlyVXNlciIsInN1YiI6Imh0dHBzOi8vZXhhbXBsZS5vcmcvZmhpci9QYXRpZW50LzEyMyJ9.gPkk0A4N2I4mb9RmzXmyutCsyzVsEWdYjrC4MBWjaBRrVV2nQucbTEqwgaxUMMThlbe8s3V7VDVsHhzzjCq-2Z0nB9BGXmMaVe7e4zFVA41DKfuM50BSsC1CcEjXVZ_r6T7Y1vAij-y3N_LZIV-cYzs8HA4ue6m3FuaJLzQwo6KysnjdolM2NDInpHZPoBXZ_1e-Z2L34KDT4G9Sx2Uu7p3kpJmKRygSQgiJEnGHi1slRDTyMtJF7SzlIIZ6Oh4-y65oL3lhSIc7nzYek1teAR7LNAQq5867bFbb5g1vDhA4LuUDFLBA6zxY5gsD45fe8R31Dim7ILOEUIMtFEqGjA|The `sub` claim within the token matches the patient ID used in the `item` parameter|
|X-Parition-Name|https://example.org/fhir||

Body:
```
{
    "resourceType": "Parameters",
    "parameter": [
        {
            "name": "expungeDeletedResources",
            "valueBoolean": true
        },
        {
            "name": "_cascade",
            "valueString": "delete"
        }
    ]
}
```
