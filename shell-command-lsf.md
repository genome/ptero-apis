# LSF Shell Command Service API

## Critical API

### POST /v1/jobs
Schedule a new job.

The job status should be polled in case the pre/post-exec callbacks don't occur
as expected.

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
        - whether prepend lines with data (timestamps, etc.)
        - whether to transform data (e.g. to JSON)
- `queue`
    - string
- `resources`
    - hash with limit, request, & reserve sub-hashes
    - sub hashes are all string -> (string, int)
- `umask`
- `user`
    - string
    - the user to run the job as (may be different from the authenticated user)

Optional parameters:

- `project`
    - string
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
            "scheduled": "http://workflow/v1/callbacks/shell-command-lsf/scheduled?execution_identifier=42",
            "begun": "http://workflow/v1/callbacks/shell-command-lsf/begun?execution_identifier=42",
            "succeeded": "http://workflow/v1/callbacks/shell-command-lsf/succeeded?execution_identifier=42",
            "failed": "http://workflow/v1/callbacks/shell-command-lsf/failed?execution_identifier=42",
            "cancelled": "http://workflow/v1/callbacks/shell-command-lsf/cancelled?execution_identifier=42"
            "error": "http://workflow/v1/callbacks/shell-command-lsf/error?execution_identifier=42"
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
        },

        "resources": {
            "limit": {
                ...
            },
            "request": {
                ...
            },
            "reserve": {
                ...
            },
        },
        "queue": "long"
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

To modify e.g. the LSF queue:

    {
        "queue": "apipe"
    }

### GET /v1/jobs/(id)
Return job details.

#### Query String Parameters

- `fields`
    - what fields to include in the response
        - `command_line`
        - `environment`
        - `resources`
        - `status`
        - `user`
        - status history
        - additional lsf-details (parts of bjobs -l)

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
occur.  If a particular callback fails and is being retries, then future
callbacks should wait until all previous callbacks succeed.

Callbacks are available for each status update:

- begun
- succeeded
- failed
- cancelled
- scheduled: the job has been successfully scheduled with LSF
- error: the service has failed to fulfill its promise to run the job
    - the job cannot be run for some reason
        - it could not be successfully submitted to LSF
        - there are no possible hosts that can fulfil the job (e.g. due to large
          resource requests)
    - internal inconsistency about job state was detected
        - e.g. a successful job was found to be failed after querying LSF
        - e.g. celery result storage backend failure


Request bodies will be essentially the same, though some fields are not
available with all statuses (e.g. the end time is not available for scheduled
status).

Sample LSF Begun callback:

    PUT (user-specified-url)
    Content-Type: application/json
    Accepts: application/json

    {
        "callback_type": "begun",
        "job_id": 1234,
        "host": "some-execution-host",
        "begin": "2014-02-20 11:23:47-6"
    }
