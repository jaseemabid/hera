# Workflow Template  Dag

## Note

This example is a replication of an Argo Workflow example in Hera.
The upstream example can be [found here](https://github.com/argoproj/argo-workflows/blob/main/examples/workflow-template/dag.yaml).




=== "Hera"

    ```python linenums="1"
    from hera.workflows import DAG, Task, Workflow
    from hera.workflows.models import TemplateRef

    with Workflow(
        generate_name="workflow-template-dag-diamond-",
        entrypoint="diamond",
    ) as w:
        whalesay_template_ref = TemplateRef(name="workflow-template-print-message", template="print-message")
        inner_template_ref = TemplateRef(name="workflow-template-inner-dag", template="inner-diamond")
        with DAG(name="diamond"):
            A = Task(name="A", template_ref=whalesay_template_ref, arguments={"message": "A"})
            B = Task(name="B", template_ref=whalesay_template_ref, arguments={"message": "B"})
            C = Task(name="C", template_ref=inner_template_ref)
            D = Task(name="D", template_ref=whalesay_template_ref, arguments={"message": "D"})

            A >> [B, C] >> D
    ```

=== "YAML"

    ```yaml linenums="1"
    apiVersion: argoproj.io/v1alpha1
    kind: Workflow
    metadata:
      generateName: workflow-template-dag-diamond-
    spec:
      entrypoint: diamond
      templates:
      - name: diamond
        dag:
          tasks:
          - name: A
            arguments:
              parameters:
              - name: message
                value: A
            templateRef:
              name: workflow-template-print-message
              template: print-message
          - name: B
            depends: A
            arguments:
              parameters:
              - name: message
                value: B
            templateRef:
              name: workflow-template-print-message
              template: print-message
          - name: C
            depends: A
            templateRef:
              name: workflow-template-inner-dag
              template: inner-diamond
          - name: D
            depends: B && C
            arguments:
              parameters:
              - name: message
                value: D
            templateRef:
              name: workflow-template-print-message
              template: print-message
    ```

