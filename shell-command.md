# Shell Command Service (Fork & LSF) API

## Critical API

### POST /v1/jobs
<!-- submit -->
- logging
- locking


## Required Maintenance API

### PATCH /v1/jobs/(id)
<!-- cancel job -->
<!-- ?modify a job -->

### GET /v1/jobs/(id)
<!-- job status -->


## Optional Maintenance API

### GET /v1/jobs
<!-- list known jobs (ephemeral) -->


## Available HTTP Callbacks

### Job Scheduled
### Job Begin
### Job Succeeded
### Job Failed
### Job Cancelled
