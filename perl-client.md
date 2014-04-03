# Perl Client Spec

## User Stories

Critical stories:

1. Submit a legacy Workflow XML plus inputs as a perl hash to run.
2. Query the report for a particular workflow (with options).

Additional stories:

1. Inspect the DAG of a particular workflow.
2. Programmatically construct a workflow.
3. Submit a programmatically constructed workflow.


## Usage
### Construct Workflow

This is inspired by the `Genome::WorkflowBuilder` module from
[GMS](https://github.com/genome/gms-core/tree/master/lib/perl/Genome/WorkflowBuilder).

```
my $workflow = PTero::Workflow->new(name => "Lovely Named DAG");
my $op1 = $workflow->add_operation(PTero::Command->new(...));
my $subflow = $workflow->add_operation(
    PTero::Workflow->new(name => "Nested DAG"));
$workflow->add_link(...);
...
```

### Submit Workflow

- Validate the DAG
- Convert the DAG to json
- Construct the submit message body
- POST the request
- (optional) poll the workflow server until execution is complete

```
$workflow->set_inputs(foo => "bar");
$workflow->submit(block => 1);
```

### Query Workflow Status

```
# Instantiate a lazy workflow object.
my $workflow = PTero::Workflow->new(url => "http://workflow/v1/workflows/42");

my @errors = $workflow->errors;
my @running_executions = $workflow->executions(status => 'running');
```

### Query Workflow Report
Gather data similar to the `workflow show` command from the legacy TGI Workflow
system.

```
my $workflow = PTero::Workflow->new(url => "http://workflow/v1/workflows/42");

my $report = $workflow->report;
```

### Convert Workflow XML

```
my $workflow = PTero::Deprecated::GMSWorkflowConverter::convert($workflow_xml);
```
