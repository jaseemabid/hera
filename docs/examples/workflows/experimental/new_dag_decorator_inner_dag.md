# New Dag Decorator Inner Dag






=== "Hera"

    ```python linenums="1"
    from hera.shared import global_config
    from hera.workflows import Input, Output, Workflow

    global_config.experimental_features["decorator_syntax"] = True


    w = Workflow(generate_name="my-workflow-")


    class SetupOutput(Output):
        environment_parameter: str


    @w.script()
    def setup() -> SetupOutput:
        return SetupOutput(environment_parameter="linux", result="Setting things up")


    class ConcatInput(Input):
        word_a: str
        word_b: str


    @w.script()
    def concat(concat_input: ConcatInput) -> Output:
        return Output(result=f"{concat_input.word_a} {concat_input.word_b}")


    class WorkerInput(Input):
        value_a: str
        value_b: str


    class WorkerOutput(Output):
        value: str


    @w.dag()
    def worker(worker_input: WorkerInput) -> WorkerOutput:
        setup_task = setup()
        task_a = concat(ConcatInput(word_a=worker_input.value_a, word_b=setup_task.environment_parameter))
        task_b = concat(ConcatInput(word_a=worker_input.value_b, word_b=setup_task.result))
        final_task = concat(ConcatInput(word_a=task_a.result, word_b=task_b.result))

        return WorkerOutput(value=final_task.result)


    @w.set_entrypoint
    @w.dag()
    def outer_dag(worker_input: WorkerInput) -> WorkerOutput:
        sub_dag_a = worker(WorkerInput(value_a="dag_a", value_b=worker_input.value_a))
        sub_dag_b = worker(WorkerInput(value_a="dag_b", value_b=worker_input.value_b))

        sub_dag_c = worker(WorkerInput(value_a=sub_dag_a.value, value_b=sub_dag_b.value))

        return WorkerOutput(value=sub_dag_c.value)
    ```

=== "YAML"

    ```yaml linenums="1"
    apiVersion: argoproj.io/v1alpha1
    kind: Workflow
    metadata:
      generateName: my-workflow-
    spec:
      entrypoint: outer-dag
      templates:
      - name: setup
        outputs:
          parameters:
          - name: environment_parameter
            valueFrom:
              path: /tmp/hera-outputs/parameters/environment_parameter
        script:
          image: python:3.9
          source: '{{inputs.parameters}}'
          args:
          - -m
          - hera.workflows.runner
          - -e
          - examples.workflows.experimental.new_dag_decorator_inner_dag:setup
          command:
          - python
          env:
          - name: hera__outputs_directory
            value: /tmp/hera-outputs
          - name: hera__script_pydantic_io
            value: ''
      - name: concat
        inputs:
          parameters:
          - name: word_a
          - name: word_b
        script:
          image: python:3.9
          source: '{{inputs.parameters}}'
          args:
          - -m
          - hera.workflows.runner
          - -e
          - examples.workflows.experimental.new_dag_decorator_inner_dag:concat
          command:
          - python
          env:
          - name: hera__outputs_directory
            value: /tmp/hera-outputs
          - name: hera__script_pydantic_io
            value: ''
      - name: worker
        dag:
          tasks:
          - name: setup-task
            template: setup
          - name: task-a
            depends: setup-task
            template: concat
            arguments:
              parameters:
              - name: word_a
                value: '{{inputs.parameters.value_a}}'
              - name: word_b
                value: '{{tasks.setup-task.outputs.parameters.environment_parameter}}'
          - name: task-b
            depends: setup-task
            template: concat
            arguments:
              parameters:
              - name: word_a
                value: '{{inputs.parameters.value_b}}'
              - name: word_b
                value: '{{tasks.setup-task.outputs.result}}'
          - name: final-task
            depends: task-a && task-b
            template: concat
            arguments:
              parameters:
              - name: word_a
                value: '{{tasks.task-a.outputs.result}}'
              - name: word_b
                value: '{{tasks.task-b.outputs.result}}'
        inputs:
          parameters:
          - name: value_a
          - name: value_b
        outputs:
          parameters:
          - name: value
            valueFrom:
              parameter: '{{tasks.final-task.outputs.result}}'
      - name: outer-dag
        dag:
          tasks:
          - name: sub-dag-a
            template: worker
            arguments:
              parameters:
              - name: value_a
                value: dag_a
              - name: value_b
                value: '{{inputs.parameters.value_a}}'
          - name: sub-dag-b
            template: worker
            arguments:
              parameters:
              - name: value_a
                value: dag_b
              - name: value_b
                value: '{{inputs.parameters.value_b}}'
          - name: sub-dag-c
            depends: sub-dag-a && sub-dag-b
            template: worker
            arguments:
              parameters:
              - name: value_a
                value: '{{tasks.sub-dag-a.outputs.parameters.value}}'
              - name: value_b
                value: '{{tasks.sub-dag-b.outputs.parameters.value}}'
        inputs:
          parameters:
          - name: value_a
          - name: value_b
        outputs:
          parameters:
          - name: value
            valueFrom:
              parameter: '{{tasks.sub-dag-c.outputs.parameters.value}}'
    ```

