# Workflow Application API

## Critical User Facing API

### POST /v1/workflows or /v1/operations/(id)/workflows
Submits a new workflow.  If posted to the operations URL, creates the workflow
as a sub-workflow of the operation's workflow.

#### Request Body
The request body should be in multiple parts:

1. workflow.xml (Content-Type: application/xml) should be valid Workflow XML.
2. data.json (Content-Type: application/json)
    - inputs for the workflow
    - environment variables

#### Responses
Success:

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
    - The environment variables are not complete enough (e.g. no user or PWD)

Errors discovered after initial submission should show up in later polling of
workflow status as "error".

### GET /v1/workflows/(id)/report
Fetches the data for a given workflow like `genome model build view` or
`workflow show`.

#### Query String
- `expand-parallel-by`
    - enum
        - none: only summarize parallel-by statuses -- no details
        - crashed-only: show details for crashed steps (default)
        - all: show details for all steps
- `exceution-history`
    - boolean
    - whether to show details of shortcut/execute history
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


## Critical System Facing API

### GET /v1/workflows/(id)/details
Fetches the top-level status for a given workflow.

Used by client to poll for workflow completion.  Errors in deferred portions of
workflow submission must show up in this query.

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
          "root_operation": "http://workflow.ptero.gsc.wustl.edu/v1/operations/42",
          "parent_workflow": {
              "details": "http://workflow.ptero.gsc.wustl.edu/v1/workflows/3/details"
          },
          "errors": []
    }

### GET /v1/operations/(op-id)/inputs
Used by wrappers running individual operations to fetch their inputs.

#### Query String Parameters
- `parallel_identifier`
    - used to identify the entire parallel stack

#### Responses
Success:

- respond with HTTP 200 (OK)
- body is simple hash of inputs

Errors:

- HTTP 404 (Not Found)
    - unknown operation
    - invalid `parallel_identifier`
    - inputs for this operation are not yet available (other status code?)

### PUT /v1/operations/(op-id)/outputs
Used by wrappers running individual operations to save their outputs.

#### Query String Parameters
- `parallel_identifier`
    - used to identify the entire parallel stack

#### Request Body
Simple hash of outputs.

#### Responses
Success:

- HTTP 204 (No Content)

Errors:

- HTTP 404 (Not Found)
    - unknown operation
    - invalid `parallel_identifier`


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

### GET /v1/workflows/(id)
Fetches the workflow.xml, and data.json provided in the POST.

#### Responses
Success:

- HTTP 200 (OK)

Errors:

- HTTP 404 (Not Found)


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
      "end": "2014-02-19 08:32:00-6",
      "errors": [
        {
          "operation": "http://workflow.ptero.gsc.wustl.edu/v1/operations/73",
          "message": "Some kind of error happened!"
        }
      ]
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
      "end": "2014-02-19 08:32:00-6",
      "user": "nnutter",
      "reason": "Hung due to disk failure, see ticket #54321."
    }


## Callback Listeners
Note that this section is contingent on understanding the APIs of the shell
command and petri services.

### PUT /v1/callbacks/petri/notifications/operations/(id)/(method)
This notification tells the service to launch a shell command job, and where to
respond when it's done.

### PUT /v1/callbacks/petri/data-requests/operations/(id)
This callback is used to request the size of a parallel-by operation.

### PUT /v1/callbacks/(shell-command-type)/(notification-type)?execution_identifier=(exec_id)
- update operation status
- create appropriate token in the petri net
