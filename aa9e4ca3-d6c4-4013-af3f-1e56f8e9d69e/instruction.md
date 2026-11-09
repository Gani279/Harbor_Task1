## CloudFormation change-set workflow

Teams need to review proposed infrastructure changes and, in many cases, obtain approval before applying them. The plugin currently applies a CloudFormation update immediately, so it cannot support that workflow.

Add a two-stage CloudFormation deployment flow. A Pipeline must be able to create a named change set for a named stack without applying it, then execute that change set later. The public Pipeline steps are `cfnCreateChangeSet` and `cfnExecuteChangeSet`. Their Java step implementations are `CFNCreateChangeSetStep` and `CFNExecuteChangeSetStep`; the existing public `CloudFormationStack` abstraction performs the corresponding `createChangeSet` and `executeChangeSet` operations.

The creation step should offer the same deployment inputs as the existing CloudFormation update flow: a workspace template or template URL, explicit and retained parameters, a JSON or YAML parameter file, tags, polling configuration, stack-creation control, and an optional role ARN. Supplied values must be preserved. Invalid parameter-file formats and invalid required inputs must fail before an AWS request is made.

When the stack exists, create an update change set; a template may be omitted in that case so CloudFormation retains the existing template. Creation is enabled by default. When the stack is absent, create a create-type change set only when creation is enabled, and require a template for that operation. If creation is disabled, do not make a change-set request.

The execution step should run the requested change set, wait using its configured polling interval, and return the final stack outputs. Polling defaults to 1000 ms; a value of zero suppresses event output while waiting. Empty change sets are successful no-ops: remove the unusable change set instead of reporting a deployment failure.

Document both Pipeline steps, including their typical create-then-execute usage and supported options.
