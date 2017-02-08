# Pipelines
## Goals
- **Define constructs which provide a more powerful and flexible execution engine for RM/Build/Deployment**: Allow pipeline execution with minimal intervention points required from consumers
- **Provide a simple yet powerful config as code model**: Easily scale from very simple processes to more complex processes without requiring cumbersome hierarchies and concepts
- **Provide data flow constructs for simple variables and complex resources**: Provide semantic constructs for describing how data flows through the system

## Non-Goals
- **Provide a full replacement for all existing application-level constructs**: This is not meant to encompass all application semantics in the Build and RM systems

## Terms
- **Pipeline**: A construct which defines the inputs and outputs necessary to complete a set of work, including how the data flows through the system and in what order the steps are executed
- **Job**: A container for task execution which supports different execution targets such as server, queue, or deploymentGroup
- **Condition**: An [expression language](conditions.md) supporting rich evaluation of context for conditional execution
- **Policy**: A generic construct for defining wait points in the system which indicates pass or fail
- **Task**: A smallest unit of work in the system 
- **Variable**: A name and value pair, similar to environment variables, for passing simple data values
- **Resource**: An which defines complex data and semantics for upload and download using a pluggable provider model. See [resources](resources.md) for a more in-depth look at the resource extensibility model.

## Simple pipeline
The pipeline process may be defined completely in the repository using YAML as the definition format. A very simple definition may look like the following:
```yaml
pipeline:
  resources:
    - name: vso
      type: self
      
  jobs:
    - name: simple build
      target:
        type: queue
        name: default
      tasks:
        - task: msbuild@1.*
          name: Build solution 
          inputs:
            project: vso/src/project.sln
            arguments: /m /v:minimal
        - export: 
            name: drop
            type: artifact
            inputs:
              include:
                - /bin/**/*.dll
              exclude:
                - /bin/**/*Test*.dll
```
This defines a pipeline with a single job which acts on the current source repository. Since all file paths are relative to a resource within the working directory, there is a resource defined with the type `self` which indicates the current repository. This allows the pipeline author to alias the current repository like other repositories, and allows separation of process and source if that model is desired as there is no implicit mapping of the current repository. After selecting an available agent from a queue named `default`, the agent runs the msbuild task from the server locked to the latest version within the 1.0 major milestone. Once the project has been built successfully the system will run an automatically injected  task for the `artifact` resource provider to publish the specified data to the server at the name `drop`.

## Job dependencies
For a slightly more complex model, here is the definition of two jobs which depend on each other, propagating the outputs of the first job including environment and artifacts into the second job.
```yaml
pipeline:
  resources:
    - name: vso
      type: self

  jobs:
    - name: job1
      target: 
        type: queue
        name: default
      tasks:
        - task: msbuild@1.*
          name: Build solution 
          inputs:
            project: vso/src/project.sln
            arguments: /m /v:minimal
        - export: drop 
          type: artifact
          inputs:
            include:
              - /bin/**/*.dll
            exclude:
              - /bin/**/*Test*.dll
        - export: outputs
          type: environment
          inputs:
            var1: myvalue1
            var2: myvalue2
              
    - name: job2
      target: 
        type: queue
        name: default
      tasks:
        - import: jobs('job1').exports.outputs
        - import: jobs('job1').exports.drop
        - task: powershell@1.*
          name: Run dostuff script
          inputs:
            script: drop/scripts/dostuff.ps1
            arguments: /a:$(job1.var1) $(job1.var2)
```
This is significant in a few of ways. First, we have defined an implicit ordering dependency between the first and second job which informs the system of execution order without explicit definition. Second, we have declared a flow of data through our system using the `export` and `import` verbs to constitute state within the actively running job. In addition we have illustrated that the behavior for the propagation of the `environment` (TODO: the concept of an environment may or may not make sense as a `resource`; further discussion is needed) resource type across jobs which will be well-understood by the system; the importing of an external environment will automatically create a namespace for the variable names based on the source which generated them. In this example, the source of the environment was named `job1` so the variables are prefixed accordingly as `job1.var1` and `job1.var2`.

## Conditional job execution
By default a job dependency requires successful execution of all previous dependent jobs. This default behavior may be modified by specifying a job execution [condition](conditions.md) and specifying requirements. For instance, we can modify the second job from above as follows to provide different execution behaviors:

### Always run
```yaml
- name: job2
  target: 
    type: queue
    name: default
  condition: "in(jobs('job1').result, 'succeeded', 'failed', 'canceled', 'skipped')"
  ....
```
### Run based on outputs
```yaml
- name: job2
  target: 
    type: queue
    name: default
  condition: "and(eq(jobs('job1').result, 'succeeded'), eq(jobs('job1').exports.outputs.var1, 'myvalue'))"
  ....
```

### Open Questions
- Should the job dependency success/fail decision be an explicit list which is different than condition execution? For instance, instead of combining dependent jobs into the condition expression, have an explicit section in the job. A potential issue with lumping them together is we need to define the behavior if you have a condition but do not explicitly check the result of one or more of your dependencies. We could inject a default `eq(jobs('job2').result, 'succeeded')` into an and condition for you if you depend on a job but don't explicitly list the results you would like to run, but that could become problematic with more complex conditionals.
```yaml
- name: job2
  condition: "eq(jobs('job1').exports.outputs.var1, 'myvalue')"
  dependsOn: 
    - name: job1
      result: 'Succeeded'
```