# Callback Structure

- listening services will specify the URI for any callbacks a server might make
- callbacks must be PUT requests
- no particular headers are required in callback requests
- callback request bodies must be signed JWTs
- services providing callbacks must register their public keys with the auth
  server

## Use Cases
- Workflow service creates tokens in Petri nets
    - data embedded in url
        - net key (C)
        - place id (C)
        - token color (C)
    - request body
        - (sometimes) size of input
- Shell command services notify Workflow status
    - request body
        - job id (S)
        - new status (S)
        - available action urls
            - cancel job
- Petri notifies Workflow that a transition has been reached
    - request body
        - net key
        - transition id
        - token color
        - next action urls (that put tokens into the specific places)
            - success/failure subnet
                - success
                - failure
            - parallelize subnet
                - give size

## Form
These will be standard webhooks.  Anyone wishing to receive a callback, must
have an HTTP(s) server listening for the callbacks.
