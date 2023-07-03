# JSON:API bulk create extension

> WARNING: This is a first release candidate pending for review. There might be breaking changes until releasing version 1.0.

The JSON:API bulk create extension allows creating multiple resources in a single request over JSON:APIs.

> Note: [Extensions](https://jsonapi.org/extensions/) allow extending [JSON:API specification](https://jsonapi.org) to add additional features.

> Note: Updating and deleting existing resources is not supported by this extension. Only the relationship of existing resources may be changed implicitly by creating a new resource related to it.

## URI

This extension has the URI `https://github.com/jelhan/json-api-bulk-create-extension`.

## Namespace

This extension uses the namespace `bulk`.

## Document Structure

A document that supports this extension **MAY** include any of the top-level members allowed by the base specification with the exception of `data` and `included`, which **MUST NOT** be included.

In addition, such a document **MUST** include the `bulk:data` member. The value of the `bulk:data` member **MUST** be an array of one or more [resource objects](https://jsonapi.org/format/#document-resource-objects). Those resource objects are called _primary data_.

The document **MAY** include the `bulk:included` member. The value of the `bulk:included` member **MUST** be an array of resource objects. Those resource objects are called _included resources_.

### Linkage between related resources

The JSON:API bulk extension allows creation of related resources in a single request.

Resource objects included as primary data **MAY** reference existing resources, but **MUST NOT** reference any other resources included in the document.

> Note: A resource provided as primary data may be related to another resource in the request. But this relationship must be defined by the included resource only.

An included resource **MUST** only reference

- existing resources,
- resources provided as primary data, and
- included resources listed before itself in the `bulk:included` array.

> Note: This ensures that a server can process the document top-down. Especially the server does not need to lookup resources defined later in the document to validate relationships.

Every included resource **MUST** reference resources included as primary data either directly or through a chain of other included resources.

> Note: This is similar to the "full linkage" requirement as defined for [compound documents](https://jsonapi.org/format/#document-compound-documents) by the JSON:API specification itself. But the direction of the graph is reversed. In a compound document every included resource must be referenced by a primary resource. This extension requires that every included resource references a primary resource.

All resources, which are referenced by another resource, **MUST** either have [client-generated IDs](https://jsonapi.org/format/#crud-creating-client-ids) or a `lid` to locally [identify the resource](https://jsonapi.org/format/#document-resource-object-identification) within the document.

### Example request document

This example shows a request document to create a post and a related tag in a single request. Additionally, the post is associated with an existing tag. See an [example response document](#example-response-document) as well.

```json
{
  "bulk:data": [
    {
      "type": "posts",
      "lid": "1"
      "attributes": {
        "title": "Awesome JSON:API"
      },
      "relationships": {
        "tags": {
          "data": [
            {
              "type": "tags",
              "id": "7c237585-983e-4767-a425-5f2277ba7351"
            }
          ]
        }
      }
    }
  ],
  "bulk:included": [
    {
      "type": "tags",
      "attributes": {
        "name": "api-design"
      },
      "relationships": {
        "posts": {
          "data": [
            {
              "type": "posts",
              "lid": "1"
            }
          ]
        }
      }
    }
  ]
}
```

## Processing 

All HTTP requests send with this extension **MUST** be issued with `POST`.

A server **MAY** support the bulk create extension only for some resource types. If supporting bulk creation for a specific resource type, a server **SHOULD** support it on all URLs that allow creation of a single resource of that type.

A server **MUST** perform the request atomically. If creating one resource fails, no resource **MUST** be created.

A server **MUST** create the resources in the order they appear in `bulk:data` and `bulk:included`. A server **MUST** create the resources in `bulk:data` before the resources in `bulk:included`.

> Note: If two resources, which should be created, both have a to-one relationship to the same resource, the one listed in the document last wins. A server may validate the document against such scenarios and reject the request.

## Response

### 201 Created

If the requested resources have been created successfully and the server changes the resources in any way (for example, by assigning an `id`), the server **MUST** return a `201 Created` response and a JSON:API document. The document **MUST** contain all JSON:API documents, which have been changed, as primary data. Additionally, the document **MAY** contain any other resource the client requested to create as primary data. The document **MUST** only contain resources the client requested to create explicitly as primary data.

> Note: A server may include other resources, which have been created or modified as a side-effect of the resource creation as included resources in a compound document  unless conflicting with any other processing rule like the `include` query parameter.

###  202 Accepted

If a request to create resources has been accepted for processing, but the processing has not been completed by the time the server responds, the server **MUST** return a `202 Accepted` status code.

### 204 No Content

If the requested resources have been created successfully and the server does not change the resources in any way (for example, by assigning an `id` or `createdAt` attribute to any of them), the server **MUST** return either a `201 Created` status code and a response document (as described above) or a `204 No Content` status code with no response document.

### 403 Forbidden

A server **MAY** return `403 Forbidden` if creating one or more resources listed in the request is not supported.

A server **SHOULD** include error details and provide enough information to recognize, which resources can not be created.

### 404 Not Found

A server **MUST** return `404 Not Found` when processing a request that references a related resource that does not exist and is not included in the request.

A server **SHOULD** include error details and provide enough information to recognize, which reference points to an inexistent resource.

### 409 Conflict

A server **MUST** return `409 Conflict` when processing a request to create a resource with a client-generated ID that already exists.

A server **SHOULD** include error details and provide enough information to recognize the source of the conflict.

### Other Responses

A server **MAY** respond with other HTTP status codes.

A server **MAY** include error details with error responses.

A server **MUST** prepare responses, and a client **MUST** interpret responses, in accordance with HTTP semantics.

### Example response document

This example shows a response document from the [example request document](#example-request-document). Both the post and the tag which was requested to be created are included as primary data. The additional already existing tag shouldn't be placed in the included data since the response only should return created resources, but it could be included if requested via the include query parameter.

```json
{
  "data": [
    {
      "type": "posts",
      "id": "42f5bc89-616a-42a1-b374-80507e71bb9c"
      "attributes": {
        "title": "Awesome JSON:API"
      },
      "relationships": {
        "tags": {
          "data": [
            {
              "type": "tags",
              "id": "7c237585-983e-4767-a425-5f2277ba7351"
            },
            {
              "type": "tags",
              "id": "97277267-920b-4f65-87dd-a699e4ae5c8b"
            }
          ]
        }
      }
    },
    {
      "type": "tags",
      "id": "97277267-920b-4f65-87dd-a699e4ae5c8b",
      "attributes": {
        "name": "api-design"
      }
    }
  ]
}
```
