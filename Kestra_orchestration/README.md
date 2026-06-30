# Note learning kestra orchestration

The first think learn about kestra it's a flow.
Flow it's a way you design a task you want to excute in kestra, you need to define a id ( you can't change after create), a namespace, description ( if you want to decribe) and finally the task, it's instruction you say to in your flow.

Above I explain a flow in this part I will more detail ( after lecture of that)

## Flow
Flows are defined in a declarative YAML syntax to keep the orchestration code portable and language-agnostic.

Each flow consists of three required components: id, namespace, and tasks.

- id is the unique identifier of the flow.
- namespace separates projects, teams, and environments.
- tasks is a list of tasks executed in order.

Here are those three components in a YAML file:

```
id: getting_started
namespace: company.team

tasks:
  - id: hello_world
    type: io.kestra.plugin.core.log.Log
    message: Hello World!

```

The id of a flow must be unique within its namespace. For example:

✅ You can have a flow named getting_started in company.team1 and another flow named getting_started in company.team2.
❌ You cannot have two flows named getting_started in company.team at the same time.
The combination of id and namespace is the unique identifier for a flow.

### Namespaces

Namespaces are used to group flows and provide structure. Keep in mind that a flow’s allocation to a namespace is immutable. Once a flow is created, you cannot change its namespace. If you need to change the namespace of a flow, create a new flow within the desired namespace and delete the old flow.

### Labels
To add another layer of organization, use labels to group flows with key-value pairs. In short, labels are customizable tags to simplify monitoring and filtering of flows and executions. For example, taking the flow above, we can add a label with the key tag to define the flow as Getting Started:

```
id: getting_started
namespace: company.team
labels:
  tag: Getting Started

tasks:
  - id: hello_world
    type: io.kestra.plugin.core.log.Log
    message: Hello World!
```

### Descriptions
You can optionally add a description property to document your flow’s purpose or other useful information. The description is a string that supports markdown syntax. This markdown description is rendered and displayed in the UI.

You can also add a description property to tasks and triggers to document all the components of your workflow.

Here is the same flow as before, but with labels and descriptions:

```
id: getting_started
namespace: company.team

description: |
  # Getting Started
  Let's `write` some **markdown** - [first flow](https://t.ly/Vemr0) 🚀

labels:
  tag: Getting Started

tasks:
  - id: hello_world
    type: io.kestra.plugin.core.log.Log
    message: Hello World!
    description: |
      ## About this task
      This task prints "Hello World!" to the logs.
```

## Tasks
Now that you know how to document and organize your flows, it’s time to get to the core of orchestration: tasks.

Tasks are atomic actions in your flows. You can design your tasks to be small and granular, such as fetching data from a REST API or running a self-contained Python script. However, tasks can also represent large and complex processes, like triggering containerized processes or long-running batch jobs (e.g., using dbt, Spark, AWS Batch, Azure Batch, etc.) and waiting for their completion.

### Task execution order
Tasks are defined as a list. By default, all tasks in the list will be executed sequentially — the second task will start as soon as the first one finishes successfully.

Kestra provides additional customization to run tasks in parallel, iterate (sequentially or in parallel) over a list of items, or allow specific tasks to fail without stopping the flow. These kinds of actions are called Flowable tasks because they define the flow logic. We’ll cover Flowable tasks in more detail later in the tutorial, but for now it is good to know they exist.

A task in Kestra must have an id and a type. This is similar to how a flow must have an id and a namespace. Other task properties depend on the task type. You can think of a task as a step in a flow that executes a specific action, such as running a Python or Node.js script in a Docker container or loading data from a database.

We’ve shown a Log task in some example flows before, and below is the same flow with an additional Python script task added. The Log task runs first and then the Python task (copy and run for yourself to see the results):

```
id: getting_started
namespace: company.team

description: |
  # Getting Started
  Let's `write` some **markdown** - [first flow](https://t.ly/Vemr0) 🚀

labels:
  tag: Getting Started

tasks:
  - id: hello_world
    type: io.kestra.plugin.core.log.Log
    message: Hello World!
    description: |
      ## About this task
      This task prints "Hello World!" to the logs.

  - id: python
    type: io.kestra.plugin.scripts.python.Script
    containerImage: python:slim
    script: |
      print("Hello World!")

```

### Iterate quickly with Playground
When you want to tweak a flow step by step without rerunning everything, use the Playground in the editor. It lets you play tasks one at a time, keep prior outputs, and iterate like a notebook. See the short guide in UI → Playground and try it with the getting_started example above before moving on.

### Autocompletion
Kestra supports hundreds of tasks integrating with various external systems. It’s neither necessary nor possible to memorize all potential tasks or properties, maybe one day. Use the shortcut CTRL + SPACE on Windows/Linux or fn + control + SPACE on macOS to trigger autocompletion and list available tasks or properties for a given task. Kestra also has built-in documentation accessible through the UI for Flow, Task, and Trigger properties, so you don’t have to context switch between building a flow and learning the ins and outs of a component.

If you want to comment or uncomment out part of your code, use CTRL + / on Windows/Linux or ⌘ + / on macOS. All available keyboard shortcuts are listed in the code editor context menu.

### Create and run a flow
To this point, we have shown some flows to run and get familiar with. Now, let’s create a flow to use throughout the rest of the tutorial. Open the Flows view and click + Create: For other step follow the tutoriel fundamental

## Inputs

It's allow to modify the flow to put parameter and use is everywhere in the flow.

```
id: getting_started
namespace: company.team

inputs:
  - id: api_url
    type: STRING
    defaults: https://dummyjson.com/products

tasks:
  - id: api
    type: io.kestra.plugin.core.http.Request
    uri: "{{ inputs.api_url }}"
```

## Outputs

Use it to see the output of the flow you use execute, he can be a message in log or result for script or query.

Below there two example flow with output message and flow with output table 

message 

```
id: getting_started
namespace: company.team

inputs:
  - id: api_url
    type: STRING
    defaults: https://dummyjson.com/products

tasks:
  - id: api
    type: io.kestra.plugin.core.http.Request
    uri: "{{ inputs.api_url }}"

  - id: log
    type: io.kestra.plugin.core.log.Log
    message: "API Status Code: {{ outputs.api.code }}"
```

output with result

```
id: getting_started
namespace: company.team

inputs:
  - id: api_url
    type: STRING
    defaults: https://dummyjson.com/products

tasks:
  - id: api
    type: io.kestra.plugin.core.http.Request
    uri: "{{ inputs.api_url }}"

  - id: python
    type: io.kestra.plugin.scripts.python.Script
    containerImage: python:slim
    beforeCommands:
      - pip install polars
    outputFiles:
      - "products.csv"
    script: |
      import polars as pl
      data = {{ outputs.api.body | jq('.products') | first }}
      df = pl.from_dicts(data)
      df.glimpse()
      df.select(["brand", "price"]).write_csv("products.csv")

  - id: sqlQuery
    type: io.kestra.plugin.jdbc.duckdb.Query
    inputFiles:
      in.csv: "{{ outputs.python.outputFiles['products.csv'] }}"
    sql: |
      SELECT brand, round(avg(price), 2) as avg_price
      FROM read_csv_auto('{{ workingDir }}/in.csv', header=True)
      GROUP BY brand
      ORDER BY avg_price DESC;
    store: true
```

## Triggers

Triggers help us to configure a repetive task in the flow, you can define triggers like task .

```
triggers:
  - id: every_monday_at_10_am
    type: io.kestra.plugin.core.trigger.Schedule
    cron: 0 10 * * 1
```

this trgiggers define to run a flow define above all monday at 10 AM

the instruction cron can explain : 

```
┌───────── minute (0)
│ ┌─────── hour (10 = 10:00 AM)
│ │ ┌───── day of month (* = every day)
│ │ │ ┌─── month (* = every month)
│ │ │ │ ┌─ day of week (1 = Monday)
│ │ │ │ │
0 10 * * 1
```
## Flowable task

It's allow to custom flow to put some iteration and condition for executing task like language of programmation.

Tasks from the Core Flow plugin control flow logic. Use them to run tasks in parallel or sequentially, branch conditionally, iterate over items, pause, or allow specific tasks to fail without stopping the execution.

### if task
For example, you can use the If task to specify your conditions and define what action to take based on whether those conditions are met.
You excute a task if condition it's verify and it's not verify execute the else

```
id: getting_started
namespace: company.team

inputs:
  - id: category
    type: SELECT
    displayName: Select a category
    values: ['beauty', 'notebooks']
    defaults: 'beauty'

tasks:
  - id: api
    type: io.kestra.plugin.core.http.Request
    uri: "https://dummyjson.com/products/category/{{ inputs.category }}"
    method: GET

  - id: check_products
    type: io.kestra.plugin.core.flow.If
    condition: "{{ json(outputs.api.body).products | length > 0 }}"
    then:
      - id: log_status
        type: io.kestra.plugin.core.log.Log
        message: "Found {{ json(outputs.api.body).products | length }} products for category {{ inputs.category }}"
      - id: python
        type: io.kestra.plugin.scripts.python.Script
        containerImage: python:slim
        dependencies:
          - polars
        outputFiles:
          - "products.csv"
        script: |
          import polars as pl
          data = {{ outputs.api.body | jq('.products') | first }}
          df = pl.from_dicts(data)
          df.glimpse()
          # Keep a simple view for this category
          df.select(["title", "brand", "price"]).write_csv("products.csv")
      - id: sqlQuery
        type: io.kestra.plugin.jdbc.duckdb.Query
        inputFiles:
          in.csv: "{{ outputs.python.outputFiles['products.csv'] }}"
        sql: |
          SELECT brand, round(avg(price), 2) AS avg_price, count(*) AS cnt
          FROM read_csv_auto('{{ workingDir }}/in.csv', header=True)
          GROUP BY brand
          ORDER BY avg_price DESC;
        store: true
    else:
      - id: when_false
        type: io.kestra.plugin.core.log.Log
        message: "No products found for category {{ inputs.category }}."

triggers:
  - id: every_monday_at_10_am
    type: io.kestra.plugin.core.trigger.Schedule
    cron: 0 10 * * 1
```

### forEach 

Like for loop you use it to excute a task for value in list 

The ForEach flowable task executes a group of tasks for each value in the list. There are many ways to implement ForEach for complex looping operations, possibly incorporating conditional flowable tasks or subtasks. See more examples in the [ForEach documentation](https://kestra.io/plugins/core/flow/io.kestra.plugin.core.flow.foreach).
```
id: for_loop_example
namespace: tutorial

tasks:
  - id: for_each
    type: io.kestra.plugin.core.flow.ForEach
    values: ["pynchon", "dostoyevsky", "hedayat"]
    tasks:
      - id: api
        type: io.kestra.plugin.core.http.Request
        uri: "https://openlibrary.org/search.json?author={{ taskrun.value }}&sort=new"
```

### LoopUntil
You can also loop until an external system reports a healthy status. The LoopUntil task reruns its child tasks until a condition becomes true, which is helpful for polling APIs or long-running jobs.

Key options:

- condition — evaluated after each run and can reference the latest child outputs (for example {{ outputs.healthCheck.code }}).
- tasks — the steps executed on every loop iteration.
- checkFrequency — optional guardrails controlling the poll interval plus maximum iterations or duration.

```
id: loop_until_health_check
namespace: tutorial

tasks:
  - id: loop
    type: io.kestra.plugin.core.flow.LoopUntil
    condition: "{{ outputs.healthCheck.code == 200 }}"
    checkFrequency:
      interval: PT30S
      maxIterations: 50
    tasks:
      - id: healthCheck
        type: io.kestra.plugin.core.http.Request
        method: GET
        uri: https://kestra.io
```

This flow checks an HTTP endpoint every 30 seconds and stops either when it returns 200 or after 50 attempts, whichever comes first. You can reference the child task outputs (here outputs.healthCheck.code) inside the condition expression. See the [LoopUntil task documentation](https://kestra.io/plugins/core/flow/io.kestra.plugin.core.flow.loopuntil) for additional options.

## Add parallelism using Flowable tasks
A common orchestration requirement is executing independent processes in parallel. For example, you can process data for each partition in parallel. This can significantly speed up the processing time.

The flow below uses the ForEach flowable task to execute a list of tasks in parallel.

- The concurrencyLimit property with value 0 makes the list of tasks to execute in parallel.
- The values property defines the list of items to iterate over.
- The tasks property defines the list of tasks to execute for each item in the list. You can access the iteration value using the {{ taskrun.value }} variable.

  ```
  id: python_partitions
namespace: company.team

description: Process partitions in parallel

tasks:
  - id: getPartitions
    type: io.kestra.plugin.scripts.python.Script
    taskRunner:
      type: io.kestra.plugin.scripts.runner.docker.Docker
    containerImage: ghcr.io/kestra-io/pydata:latest
    script: |
      from kestra import Kestra
      partitions = [f"file_{nr}.parquet" for nr in range(1, 10)]
      Kestra.outputs({'partitions': partitions})

  - id: processPartitions
    type: io.kestra.plugin.core.flow.ForEach
    concurrencyLimit: 0
    values: '{{ outputs.getPartitions.vars.partitions }}'
    tasks:
      - id: partition
        type: io.kestra.plugin.scripts.python.Script
        taskRunner:
          type: io.kestra.plugin.scripts.runner.docker.Docker
        dependencies:
          - kestra
        script: |
          import random
          import time
          from kestra import Kestra

          filename = '{{ taskrun.value }}'
          print(f"Reading and processing partition {filename}")
          nr_rows = random.randint(1, 1000)
          processing_time = random.randint(1, 20)
          time.sleep(processing_time)
          Kestra.counter('nr_rows', nr_rows, {'partition': filename})
          Kestra.timer('processing_time', processing_time, {'partition': filename})
  ```

