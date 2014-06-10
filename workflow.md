# Workflow Application API

## Critical User Facing API

### POST /v1/workflows or /v1/executions/(execution-id)/workflows
Submits a new workflow.  If posted to the operations URL, creates the workflow
as a sub-workflow of the operation's workflow.

#### Request Body
Sample body for an N-shaped workflow:

    {
        "workflow": {
            "operations": {
                "A": {
                    "methods": [
                        {
                            "name": "shortcut",
                            "submitUrl": "http://ptero-fork/v1/jobs",
                            "parameters" {
                                "commandLine": ["genome-ptero-wrapper",
                                    "command", "shortcut", "NullCommand"]
                            },
                        },
                        {
                            "name": "execute",
                            "submitUrl": "http://ptero-lsf/v1/jobs",
                            "parameters" {
                                "commandLine": ["genome-ptero-wrapper",
                                    "command", "execute", "NullCommand"],
                                "limit": {
                                    "virtual_memory": 204800
                                },
                                "request": {
                                    "min_cores": 4,
                                    "memory": 200,
                                    "temp_space": 5
                                },
                                "reserve": {
                                    "min_cores": 4,
                                    "memory": 200,
                                    "temp_space": 5
                                }
                            },
                        }
                    ],
                },

                "B": {
                    ...
                },
                "C": {
                    ...
                },
                "D": {
                    ...
                }
            },

            "links": [
                {
                    "source": "input connector",
                    "destination": "A",
                    "source_property": "in_a",
                    "destination_property": "param",
                },
                {
                    "source": "input connector",
                    "destination": "B",
                    "source_property": "in_b",
                    "destination_property": "param",
                },
                {
                    "source": "input connector",
                    "destination": "C",
                    "source_property": "in_c",
                    "destination_property": "param",
                },
                {
                    "source": "input connector",
                    "destination": "D",
                    "source_property": "in_d",
                    "destination_property": "param",
                },

                {
                    "source": "A",
                    "destination": "output connector",
                    "source_property": "result",
                    "destination_property": "out_a",
                },
                {
                    "source": "B",
                    "destination": "output connector",
                    "source_property": "result",
                    "destination_property": "out_b",
                },
                {
                    "source": "C",
                    "destination": "output connector",
                    "source_property": "result",
                    "destination_property": "out_c",
                },
                {
                    "source": "D",
                    "destination": "output connector",
                    "source_property": "result",
                    "destination_property": "out_d",
                },

                {
                    "source": "A",
                    "destination": "C",
                    "source_property": "result",
                    "destination_property": "res1",
                },
                {
                    "source": "A",
                    "destination": "D",
                    "source_property": "result",
                    "destination_property": "res1",
                },
                {
                    "source": "B",
                    "destination": "D",
                    "source_property": "result",
                    "destination_property": "res2",
                }

            ]
        },

        "inputs": {
            "in_a": "one",
            "in_b": "two",
            "in_c": "three",
            "in_d": "four"
        },

        "environment": {
            "PERL5LIB": "some:user:path"
        }
    }

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
    - response body contains the submitted workflow with some metadata
        - returned urls
            - for polling workflow status
            - for "build-view" like report

Errors:

- HTTP 400 (Bad Request)
    - The workflow cannot be validated.
        - Needed inputs not specified.
        - Invalid link.
        - Invalid operation names: 'input connector' and 'output connector' are
          reserved
    - The environment variables are not complete enough (e.g. no user or PWD)

Errors discovered after initial submission should show up in later polling of
workflow status as "error".

### GET /v1/workflows/(id)
Fetches the data for a given workflow.  This should include the original data
from the POST, plus additional fields like `status`, timestamps, and links to
related data like `executions` and `reports`.

#### Responses
Success:

- HTTP 200 (OK)
    - body may have abbreviated content depending on options (see above)

Errors:

- HTTP 404 (Not Found)


### GET /v1/reports/(report-type)
This will be used for diverse, complex actions:
- polling for workflow status
- "build view" report generation
- complex maintenance queries

Request:

    GET /v1/reports/workflow-view?workflow-id=1234

Sample abbreviated content:

    {
        "owner": "mburnett",
        "urls": {
            "workflow": "http://ptero-workflow/v1/workflows/1234"
        },
        "created": "2014-02-19 08:27:12-6",
        "updated": "2014-02-19 08:34:00-6",
        "status": "failing",
        "statusHistory": [
            {
                "status": "running",
                "timestamp": "2014-02-19 08:30:42-6"
            },
            {
                "status": "failing",
                "timestamp": "2014-02-19 08:34:00-6"
            }
        ],

        "operations": [
            {
                "name": "A unique name in this model",
                "status": "succeeded",
                "statusHistory": [
                    {
                        "status": "running",
                        "method": "shortcut",
                        "timestamp": "2014-02-19 08:30:47-6"
                    },
                    {
                        "status": "succeeded",
                        "method": "shortcut",
                        "timestamp": "2014-02-19 08:32:00-6"
                    }
                ],
                "constants": {
                    "stderrLog": "file:///gscmnt/gc2013/info/model_data/build12345/logs/a_unique_name_in_this_model.err",
                    "stdoutLog": "file:///gscmnt/gc2013/info/model_data/build12345/logs/a_unique_name_in_this_model.out"
                }
            },

            {
                "name": "Another unique name",
                "status": "failed",
                "statusHistory": [
                    {
                        "status": "running",
                        "timestamp": "2014-02-19 08:30:47-6"
                    },
                    {
                        "status": "failed",
                        "timestamp": "2014-02-19 08:34:00-6"
                    }
                ],
                "operations": [
                    {
                        "name": "A unique name in this model",
                        "constants": {
                            "stderrLog": "file:///gscmnt/gc2013/info/model_data/build12345/logs/another_unique_name_3/a_unique_name_in_this_model.err",
                            "stdoutLog": "file:///gscmnt/gc2013/info/model_data/build12345/logs/another_unique_name_3/a_unique_name_in_this_model.out"
                        },
                        "status": "failed",
                        "statusHistory": [
                            {
                                "method": "shortcut",
                                "status": "running",
                                "timestamp": "2014-02-19 08:30:47-6"
                            },
                            {
                                "method": "shortcut",
                                "status": "failed",
                                "timestamp": "2014-02-19 08:30:57-6"
                            },
                            {
                                "method": "execute",
                                "status": "running",
                                "timestamp": "2014-02-19 08:31:47-6"
                            },
                            {
                                "method": "execute",
                                "status": "failed",
                                "timestamp": "2014-02-19 08:34:00-6"
                            }
                        ]
                    }
                ]
            },

            {
                "name": "A sweet, parallel-by operation",
                "status": "failing",
                "statusHistory": [
                    {
                        "status": "running",
                        "timestamp": "TIMESTAMP OF FIRST OP START"
                    },
                    {
                        "status": "failing",
                        "timestamp": "2014-02-19 08:34:47"
                    },
                ],
                "constants": {
                    "stderrLog": {
                        "template": "file://{{allocation_path}}/logs/a_sweet_parallel_by_operation/{{parallelIndex}}.err",
                        "allocation_id": 6789
                    },
                    "stdoutLog": {
                        "template": "file://{{allocation_path}}/logs/a_sweet_parallel_by_operation/{{parallelIndex}}.out",
                        "allocation_id": 6789
                    },
                },
                "parallelStatusCounts": {
                    "failed": 1,
                    "succeeded": 22,
                    "running": 1
                },
                "failedParallelSteps": [
                    {
                        "metadata": {
                            "parallelIndex": 7
                        },
                        "status": "failed",
                        "statusHistory": [
                            {
                                "method": "shortcut",
                                "status": "running",
                                "timestamp": "2014-02-19 08:31:47-6"
                            },
                            {
                                "method": "shortcut",
                                "status": "failed",
                                "timestamp": "2014-02-19 08:31:50-6"
                            },
                            {
                                "method": "execute",
                                "status": "running",
                                "timestamp": "2014-02-19 08:33:47-6"
                            },
                            {
                                "method": "execute",
                                "status": "failed",
                                "timestamp": "2014-02-19 08:34:47-6"
                            }
                        ]
                    }
                ]
            }

        ]
    }

<!-- Do we want to hide sub-model details if they are 'new' or 'done'? -->

Here's an example used by the client to poll for workflow completion. Errors in
deferred portions of workflow submission must show up in this query. This set
of fields can be produced with the \_details shortcut.

Request:

    GET /v1/reports/workflow-status?workflow-id=1234

Content:

    {
        "name": "Some Exciting Workflow",
        "url": "http://ptero-workflow/v1/reports/workflow-status?workflow-id=1234",
        "owner": "mburnett",
        "created": "2014-02-19 08:27:12-6",
        "updated": "2014-02-19 08:35:42-6",
        "status": "failed",
        "errors": []
    }

## Critical System Facing API

### GET /v1/executions/(execution-id)
Used by wrappers running individual operations to fetch their inputs. The
execution id uniquely specifies the entire parallel stack and its
operation ID.

#### Responses
Success:

- HTTP 200 (OK)

Errors:

- HTTP 404 (Not Found)
    - invalid `execution-id`
    - inputs for this operation are not yet available (other status code?)


Sample response body:

    {
        "status": "running",
        "inputs": {
            "foo": "bar"
        },
        "operation": {
            "type": "perl-ur-command",
            "commandClass": "Some::Command::Class"
        }
        "attempts": [
            {
                "method": "shortcut",
                "service": "fork",
                "status": "failed",
                "resources": {
                    ...
                },
                "stats": {
                    "cpu_time": 34.1,
                    "wall_time": 14.2,
                    "max_rss": 1234,
                    ...
                }
                "host": "host123.some.place.edu",
                ...
            },
            {
                "method": "execute",
                "service": "lsf",
                "status": "running",
                "resources": {
                    ...
                },
                "host": "host321.some.place.edu",
                ...
            }
        ],
    }

### PATCH /v1/executions/(execution-id)
Used by wrappers running individual operations to save their outputs. The
execution id uniquely specifies the entire parallel stack and its
operation ID.

#### Request Body

    {
      "output": { ... dictionary of outputs ... }
    }

#### Responses
Success:

- HTTP 204 (No Content)

Errors:

- HTTP 404 (Not Found)
    - invalid `execution-id`


## Required Maintenance API

### PATCH /v1/workflows/(id)
This is only used to cancel (or possibly pause) the workflow.  Whether this
kills running jobs should be configurable.

Caution:  This needs to be sure to cancel all sub-workflows as well.  That may
mean pausing/expiring multiple petri nets.

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

### GET /v1/executions and /v1/workflows/(workflow-id)/executions
This could be useful to collect stats about particular kinds of executions
(like `wall_time` for a particular command class).

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
      "url": "http://workflow/v1/workflows/42",
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
      "url": "http://workflow/v1/workflows/42",
      "status": "crashed",
      "owner": "mburnett",
      "begin": "2014-02-19 08:30:42-6",
      "end": "2014-02-19 08:32:00-6",
      "errors": [
        {
          "execution": "http://workflow/v1/executions/73",
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
      "url": "http://workflow/v1/workflows/42",
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

### Petri Listeners

#### PUT /v1/callbacks/workflows/(workflow-id)/create-scope-set
The end point for callbacks from the `create-color-group` Petri action.

Defines a new layer of scopes in a workflow based on the color group sent in
the body.  This is needed to associated token colors with nested parallel-by
scopes.

#### PUT /v1/callbacks/workflows/(workflow-id)/create-scope
An end point for the `notify` Petri action.

Defines the top level scope for a workflow based on the color group information
of the token that was sent in the body.

#### PUT /v1/callbacks/operations/(operation-id)/begin/(method)
An end point for the `notify` Petri action.

This callback is used to notify us to begin either `shortcut` or `execute` for
a particular operation + scope (token color is specified in the body, which can
be used to figure out the scope).

#### PUT /v1/callbacks/operations/(operation-id)/data-request?input=foo
An end point for the `notify` Petri action.

This callback is used to request the size of a parallel-by operation.

### Shell Command Listeners

#### PUT /v1/callbacks/executions/(execution-id)/(shell-command-type)/(notification-type)
End points for the various shell command interactions.  These should be
described individually.

- update operation status
- create appropriate token in the petri net
