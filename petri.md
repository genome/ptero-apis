# Petri Net API

## Critical API

### POST /v1/nets
<!-- submit -->

### PUT /v1/nets/(net-id)/places/(place-id)/tokens
<!-- should be idempotent -->
<!-- put token into net -->


## Required Maintenance API

### PATCH /v1/nets/(id)
<!-- disable/enable net -->


## Optional Maintenance API

### GET /v1/nets
<!-- list known nets (ephemeral) -->

### GET /v1/nets/(id)
<!-- topology -->

### GET /v1/nets/(id)/places
<!-- marking -->


## Available HTTP Callbacks

### Transition Notification
