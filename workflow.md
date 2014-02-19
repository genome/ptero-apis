# Workflow Application API

## User Facing API

### POST /v1/workflows
Submits a new workflow.

#### Request Body
The request body should be in multiple parts:

1. workflow.xml (Content-Type: application/xml) should be valid Workflow XML.
2. data.json (Content-Type: application/json)
    - inputs for the workflow
    - environment variables

#### Responses
Successful posts should:

- HTTP 201 (Created)
    - set the Location header set to the newly created workflow
    - immediate
        - stores the primary workflow database entry
    - deferred
        - builds the entire workflow tree in the status database
        - stores top level inputs in the IO backend
        - compiles and submits a net to petri service
            - including initial start token

Errors:

- HTTP 400 (Bad Request)
    - The workflow XML cannot be validated against the schema.
    - The inputs JSON is not a usable hash.

Errors discovered after initial submission should show up in later polling of
workflow status as "error".

### GET /v1/workflows/(id)
Fetches the top-level status for a given workflow.

#### Responses
Success:

- HTTP 200 (OK)

Errors:

- HTTP 404 (Not Found)

Sample content:

    {
          "name": "Some Exciting Workflow",
          "owner": "mburnett",
          "created": "2014-02-19 08:27:12-6",
          "begin": "2014-02-19 08:30:42-6",
          "status": "crashed",
    }

### GET /v1/workflows/(id)/operations
Fetches the operations for a given workflow.

Polling for completion of a workflow can be done by requesting depth=0.

#### Query String
- `expand-parallel-by`
    - boolean or enum
- `exceution-history`
    - boolean or enum
- `depth`
    - integer
    - maximum nesting depth for workflow models/sub-workflows
    - if unspecified, no limit

#### Responses
Success:

- HTTP 200 (OK)
    - body may have abbreviated content depending on options (see above)

Errors:

- HTTP 404 (Not Found)

Sample abbreviated content:

    [
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

<!-- Do we want to hide sub-model details if they are 'new' or 'done'? -->


## System Facing API

### GET /v1/workflows/(wf-id)/inputs/(op-id)?parallel_index=(index)
Used by wrappers running individual operations to fetch their inputs.

#### Responses
Success:

- respond with HTTP 200 (OK)
- body is simple hash of inputs

Errors:

- HTTP 404 (Not Found)
    - unknown workflow
    - no such operation id associated with workflow
    - no such parallel index associated with operation
    - inputs for this operation are not yet available?

### PUT /v1/workflows/(wf-id)/outputs/(op-id)?parallel_index=(index)
Used by wrappers running individual operations to save their outputs.

#### Request Body
Simple hash of outputs.

#### Responses
Success:

- HTTP 204 (No Content)

Errors:

- HTTP 404 (Not Found)
    - unknown workflow id
    - no such operation id associated with workflow


## Required Maintenance API

### PATCH /v1/workflows/(id)
This is only used to cancel the workflow.

#### Request Body

    {
      "status": "cancelled"
    }

#### Responses
Success:

- HTTP 204 (No Content)

Errors:

- HTTP 404 (Not Found)
    - unknown workflow id


## Optional Maintenance API

### GET /v1/workflows
List known workflows.

#### Query String
- `limit`
    - max results per page
- `offset`
    - result to start looking at

#### Responses
- HTTP 200 (OK)


## Available HTTP Callbacks
These are HTTP requests that the workflow service can make to registered
participants if requested.

### Workflow Succeeded
Request:

    PUT (user-specified-url)
    Content-Type: application/json
    Accepts: application/json

    {
      "id": 42,
      "status": "done",
      "owner": "mburnett",
      "begin": "2014-02-19 08:30:42-6",
      "end": "2014-02-19 08:32:00-6"
    }

### Workflow Failed
Request:

    PUT (user-specified-url)
    Content-Type: application/json
    Accepts: application/json

    {
      "id": 42,
      "status": "crashed",
      "owner": "mburnett",
      "begin": "2014-02-19 08:30:42-6",
      "end": "2014-02-19 08:32:00-6"
    }

### Workflow Cancelled
Request:

    PUT (user-specified-url)
    Content-Type: application/json
    Accepts: application/json

    {
      "id": 42,
      "status": "cancelled",
      "owner": "mburnett",
      "begin": "2014-02-19 08:30:42-6",
      "end": "2014-02-19 08:32:00-6"
    }


## Callback Listeners
Note that this section is contingent on understanding the APIs of the shell
command and petri services.

### PUT /v1/callbacks/petri-notifications/
Types of notifications

- requesting parallel size
- requesting job execution

### PUT /v1/callbacks/(shell-command-type)-notifications/
- update operation status
- create appropriate token in the petri net
