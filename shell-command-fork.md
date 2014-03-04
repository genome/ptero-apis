# Fork Shell Command Service API

<!-- Should we support multiple worker queues, so that we could for example
separate jobs needing network disk from those that do not?

If we do, we need to provide endpoints for querying what's in each queue and
what queues are available.
-->

## Critical API

### POST /v1/jobs
Schedule a new job.

#### Request Body
Required parameters:

- `command_line`
    - list of strings
- `environment`
    - hash of string -> string
- `logging`
    - hash
    - define logging parameters
        - where to go: files, syslog, etc.
        - whether/how to transform data
            - to JSON
            - prepend lines with data (timestamps, etc.)
- `umask`
- `user`
    - string
    - the user to run the job as (may be different from the authenticated user)

Optional parameters:

- `webhooks`
    - hash string -> URL

Sample:

    {
        "command_line": [
            "joinx",
            "-h"
        ],
        "environment": {
            ...
        },
        "umask": "0002",
        "user": "mburnett",
        "webhooks": {
            "begun": "http://workflow/v1/callbacks/shell-command-fork/begun?execution_identifier=42",
            "succeeded": "http://workflow/v1/callbacks/shell-command-fork/succeeded?execution_identifier=42",
            "failed": "http://workflow/v1/callbacks/shell-command-fork/failed?execution_identifier=42",
            "cancelled": "http://workflow/v1/callbacks/shell-command-fork/cancelled?execution_identifier=42"
            "error": "http://workflow/v1/callbacks/shell-command-fork/error?execution_identifier=42"
        },
        "logging": {
            "stderr": {
                "type": "file",
                "path": "/gscmnt/gc2013/info/model_data/build12345/logs/some_job.err"
            },
            "stdout": {
                "type": "file",
                "path": "/gscmnt/gc2013/info/model_data/build12345/logs/some_job.out"
            }
        }
    }

#### Responses
Success:
- HTTP 201 (Created)

Errors:
- HTTP 400 (Bad Request)
    - missing key environment, etc.


## Required Maintenance API

### PATCH /v1/jobs/(id)
Used to cancel a job, or modify request details like resources, queue or
environment.

In the LSF service, this is also used by the wrapper to update the status of
the job based on the exit code of the shell command.

#### Request Body
To cancel a job:

    {
        "status": "cancelled"
    }

### GET /v1/jobs/(id)
Return job details.

#### Query String Parameters

- `fields`
    - what fields to include in the response
        - `command_line`
        - `environment`
        - `status`
        - `user`
        - status history

#### Responses
Success:

- HTTP 200 (OK)

Errors:

- HTTP 404 (Not Found)


## Optional Maintenance API

### GET /v1/jobs
Return list of jobs.

#### Query String Parameters
Useful for filtering & pagination.

#### Responses
Success:

- HTTP 200 (OK)


## Available HTTP Callbacks

Callbacks should be guaranteed to be delivered in the order in which they
occur.  If a particular callback fails and is being retried, then future
callbacks should wait until all previous callbacks succeed.

Callbacks are available for each status update.  Common callbacks for all
services:

- begun
- succeeded
- failed
- cancelled
- error: the service has failed to fulfill its promise to run the job
    - the result backend crashed, and now the status of a job is unknown

Request bodies will be essentially the same, though some fields are not
available with all statuses (e.g. the end time is not available for scheduled
status).

Sample Begun callback:

    PUT (user-specified-url)
    Content-Type: application/json
    Accepts: application/json

    {
        "callback_type": "begun",
        "job_id": 1234,
        "host": "some-execution-host",
        "begin": "2014-02-20 11:23:47-6"
    }
