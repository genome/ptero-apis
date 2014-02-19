# PTero APIs
<!--
Need to specify:
- media types
- endpoints
- callbacks
-->

## Services

### Petri
Required actions:

- create net [POST /nets]
- put token in place (accepts full token body or token id?)
    - POST /nets/(net-id)/places/(place-id)/tokens
- cancel net [PATCH /nets/(net-id) "status" field]

Optional actions:
- report net marking, topology, & details [GET /nets/(net-id)]


### Fork
Required actions:

- request job execution [POST /jobs]
- check status of job execution [GET /jobs/(job-id)]
- cancel job execution [PATCH /jobs/(job-id) "status" field]


### LSF
Required actions:

- request job execution [POST /jobs]
- check status of job execution [GET /jobs/(job-id)]
- cancel job execution [PATCH /jobs/(job-id) "status" field]


### Workflow
Required actions:

- create a workflow [POST /workflows]
    - interprets and stores workflow (and inputs) in database (immediate)
    - constructs & sends petri net serialization to petri service (deferred)
- get workflow status [GET /workflows/(workflow-id)]
    - calculated from status of operations
- cancel workflow [PATCH /workflows/(workflow-id) "status" field]
- get operation inputs [GET /operations/(operation-id) "inputs" field]
- save operation outputs [PATCH /operations/(operation-id) "outputs" field]
- set operation status [PATCH /operations/(operation-id) "status" field]
    - also allow metadata to be attached (like LSF job id)

#### Status Database
SQL

Operation statuses

- status
- timestamp
- execution type (shortcut, execute)
- execution system (fork, lsf, etc..)

#### IO Data
NoSQL


## Questions
- How do we help to ensure idempotency?
    - Should requests to create tokens in Core require a request id/tag, so that
      they can be guaranteed to not happen more than once?
        - since the generating app (Workflow) builds all the callbacks to petri
          service, it can generate a bunch of unique ids
        - maybe this could be optional instead of required (if it's there, use
          it)
