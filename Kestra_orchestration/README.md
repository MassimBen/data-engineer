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


