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
- body contains basic info for all operations as an ordered tree

Errors:
- HTTP 404 (Not Found)

Sample response body:
{
  "name": "Some Exciting Workflow",
  "owner": "mburnett",
  "created": "2014-02-19 08:27:12-6",
  "begin": "2014-02-19 08:30:42-6",
  "status": "crashed",
  "operations": [
    {
      "name": "A unique name in this model",
      "type": "command",
      "class": "Genome::Model::Build::Command::RnaSeq::Something",
      "status": "done",
      "stderr-url": "file:///gscmnt/gc2013/info/model_data/build12345/logs/a_unique_name_in_this_model.2.err",
      "stdout-url": "file:///gscmnt/gc2013/info/model_data/build12345/logs/a_unique_name_in_this_model.2.out",
      "executions": [
        {
          "type": "shortcut",
          "method": "fork-shell",
          "begin": "2014-02-19 08:30:47-6",
          "end": "2014-02-19 08:32:00-6",
          "status": "done",
        }
      ]
    },
    {
      "name": "Another unique name",
      "type": "model",
      "status": "crashed",
      "children": [
        {
          "name": "A unique name in this model",
          "type": "command",
          "class": "Genome::Model::Build::Command::RnaSeq::SomethingElse",
          "status": "crashed",
          "stderr-url": "file:///gscmnt/gc2013/info/model_data/build12345/logs/another_unique_name_3/a_unique_name_in_this_model.5.err",
          "stdout-url": "file:///gscmnt/gc2013/info/model_data/build12345/logs/another_unique_name_3/a_unique_name_in_this_model.5.out",
          "executions": [
            {
              "type": "fork-shell",
              "method": "shortcut",
              "begin": "2014-02-19 08:30:47-6",
              "end": "2014-02-19 08:30:57-6",
              "status": "crashed",
            },
            {
              "type": "lsf-shell",
              "method": "execute",
              "begin": "2014-02-19 08:31:47-6",
              "end": "2014-02-19 08:34:00-6",
              "status": "crashed",
            }
          ]
        }
      ]
    },
    {
      "name": "A sweet, parallel-by operation",
      "type": "event",
      "class": "Genome::Model::Event::Build::Some::Thing",
      "parallel": {
        "statuses": {
          "crashed": 1,
          "done": 22,
          "running": 1
        },
        "crashed-data": [
          {
            "status": "done",
            "stderr-url": "file:///gscmnt/gc2013/info/model_data/build12345/logs/a_sweet_parallel_by_operation.4_14.err",
            "stdout-url": "file:///gscmnt/gc2013/info/model_data/build12345/logs/a_sweet_parallel_by_operation.4_14.out",
            "executions": [
              {
                "type": "fork-shell",
                "method": "shortcut",
                "status": "crashed",
                "begin": "2014-02-19 08:31:47-6",
                "end": "2014-02-19 08:31:47-6",
              },
              {
                "type": "lsf-shell",
                "method": "execute",
                "status": "crashed",
                "begin": "2014-02-19 08:33:47-6",
                "end": "2014-02-19 08:34:47-6",
              },
            ]
          }
        ]
      }
    }
  ]
}

<!-- Do we want to hide sub-model details if they are 'new' or 'done'? -->

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
