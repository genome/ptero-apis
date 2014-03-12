# Petri Net API

## Critical API

### POST /v1/nets
Create a new net.

#### Request Body Format
Should be an augmented Petri net DSL, allowing specification of:

- core components
    - places
    - transitions
    - arcs
- transitions with complex actions
    - notification
    - split
    - barrier
    - join
- shortcuts
    - bridges (inserts opposite node, allows many to many connection)
        - between places
        - between transitions
    - compound transitions (acts like transition by connecting to places)
        - send notification, wait for one of N responses (N=2: success/failure)
        - send data request, wait for response, attach data to token

#### Responses
Success:

- HTTP 201 (Created)
    - sets the Location header to the newly created net
    - immediate
        - know net key/id for net
        - set net to inactive state, so no processing occurs
        - response body includes URL for creating tokens in each specially
          labeled place (e.g. `start_place`)
    - deferred
        - construct net in redis
        - enable the net

Errors:

- HTTP 400 (Bad Request)
    - petri net serialization is invalid

### PUT /v1/nets/(net-id)/places/(place-id)/tokens/(color)
Puts a token into a place.

#### Request Body
May be empty or contain specific data to be used by upcoming actions.  E.g. a
split action may require as input a `split_size` to know how many new colors of
tokens to create.

    PUT /v1/nets/7/places/23/tokens/0

    {
        "split_size": 23
    }

#### Responses
Success:

- HTTP 201 (Created)

Errors:

- HTTP 404 (Not Found)
    - no such net
    - no place for net

### DELETE /v1/nets/(id)
Delete persistent representation of a net.  May be deferred.

#### Responses
Success:

- HTTP 204 (No Content)

Errors:

- HTTP 404 (Not Found)


## Required Maintenance API

### PATCH /v1/nets/(id)
Used to pause execution of a net.

#### Request Body

To pause:

    {
      "active": false
    }

To resume:

    {
      "active": true
    }

#### Responses
Success:

- HTTP 204 (No Content)

Errors:

- HTTP 404 (Not Found)


## Optional Maintenance API

### GET /v1/nets
List nets.

#### Responses
Success:

- HTTP 200 (OK)

### GET /v1/nets/(id)
Get net topology.

#### Responses
Success:

- HTTP 200 (OK)

Errors:

- HTTP 404 (Not Found)

### GET /v1/nets/(id)/places
Get net marking.

#### Responses
Success:

- HTTP 200 (OK)

Errors:

- HTTP 404 (Not Found)
    - no such net


## Available HTTP Callbacks
Any response code other than HTTP 200 (OK) will be considered an error, and the
callback will be retried.

### Notification
Sent to notify the consumer application that a particular transition in the net
has been reached.

Specifically, the Workflow application will use this to know when to start
shell command jobs.  The `relative_token_color_stack` can be used to determine
specifically identify a particular operation inside a nested `parallel_by`.

#### Request

    PUT (user-specified-location)
    Content-Type: application/json
    Accepts: application/json

    {
        "token": {
            "color": 7,
            "color_group": 2,
            "parent_color": 0,
            "parent_color_group": 0
        },
        "response_places": {
            "success": "http://petri/v1/nets/7/places/12/tokens/7",
            "failure": "http://petri/v1/nets/7/places/13/tokens/7"
        }
    }

### Data Request
Sent to notify the consumer application that input is required to continue.

Maybe `requested_data` should specify validation for each piece of data,
e.g. int > 0.

#### Request

    PUT (user-specified-location)
    Content-Type: application/json
    Accepts: application/json

    {
        "token": {
            "color": 0,
            "color_group": 0,
            "parent_color": null,
            "parent_color_group": null
        },
        "requested_data": ["split_size"],
        "response_place": "http://petri/v1/nets/7/places/28/tokens/0"
    }
