# Workflow Application API

## Critical API

### POST /v1/workflows
Submits a new workflow.

#### Request Body
The request body should be in two parts:
1. workflow.xml (Content-Type: application/xml) should be valid Workflow XML.
2. inputs.json (Content-Type: application/json) should be a hash of top level
   inputs for the workflow.

#### Responses
Successful posts should:
- respond with HTTP 201
- respond with the Location header set to the newly created workflow
- build the entire workflow tree in the status database
- top level inputs stored in the IO backend
- compile and submit a net to petri service (deferred)
    - including initial start token

Errors:
- HTTP 400 (Bad Request)
    - The workflow XML cannot be validated against the schema.
    - The inputs JSON is not a usable hash.

### GET /v1/workflows/(id)
Fetches "build view" style data.

#### Responses
Success:
- respond with HTTP 200 (OK)
- body contains basic info for all operations as a tree

Errors:
- HTTP 404 (Not Found)

### GET /v1/operations/(id)
Used by wrappers running individual operations to fetch their inputs.

#### Query String
- `fields` specifies what fields to return in the response body
    - specifically include/exclude inputs

#### Responses
Success:
- respond with HTTP 200 (OK)
- body
    - basic info for operation
    - inputs
    - outputs

Errors:
- HTTP 404 (Not Found)

### PATCH /v1/operations/(id)
Used by wrappers running individual operations to save their outputs.

#### Request Body
- `outputs`
#### Responses

## Required Maintenance API

### PATCH /v1/workflows/(id)
<!-- cancel workflow (multiple modes) -->
#### Query String
#### Request Body
#### Responses


## Optional Maintenance API

### GET /v1/workflows
<!-- list known workflows -->
#### Query String
#### Responses

### GET /v1/operations
<!-- list known operations -->
#### Query String
#### Responses


## Available HTTP Callbacks

### Workflow Succeeded
### Workflow Failed
### Workflow Cancelled


## Callback Listeners

### PUT /v1/callbacks/petri-notifications/
Types of notifications
- requesting parallel size
- requesting job execution

### PUT /v1/callbacks/(shell-command-type)-notifications/
- update operation status
- create appropriate token in the petri net
