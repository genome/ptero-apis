# PTero APIs

These are the approximate design documents for the entire PTero system, living
together in harmony during their initial development.

<!--
Need to specify:
- media types
- endpoints
- callbacks
-->

## Services

### Petri
This service executes Petri nets with a color & color-group extension.

### Shell Command
This service runs commands in a bash shell.  There are two implementations of
this service:  one that runs jobs locally using fork/exec, and one that runs
jobs using the job scheduler LSF.

### Workflow
This is the most directly user-facing service.  It supports running workflows
specified using the [legacy workflow system from
TGI](https://github.com/genome/tgi-workflow) on top of the other services.
