# Custom Metrics

## Note

This example is a replication of an Argo Workflow example in Hera.
The upstream example can be [found here](https://github.com/argoproj/argo-workflows/blob/main/examples/custom-metrics.yaml).




=== "Hera"

    ```python linenums="1"
    from hera.workflows import (
        Container,
        Counter,
        Gauge,
        Histogram,
        Label,
        Parameter,
        Steps,
        Workflow,
        models as m,
    )

    random_int = Container(
        name="random-int",
        image="alpine:latest",
        command=["sh", "-c"],
        args=["RAND_INT=$((1 + RANDOM % 10)); echo $RAND_INT; echo $RAND_INT > /tmp/rand_int.txt"],
        metrics=[
            Histogram(
                name="random_int_step_histogram",
                help="Value of the int emitted by random-int at step level",
                when="{{status}} == Succeeded",
                buckets=[2.01, 4.01, 6.01, 8.01, 10.01],
                value="{{outputs.parameters.rand-int-value}}",
            ),
            Gauge(
                name="duration_gauge",
                labels=Label(key="name", value="random-int"),
                help="Duration gauge by name",
                realtime=True,
                value="{{duration}}",
            ),
        ],
        outputs=Parameter(
            name="rand-int-value", global_name="rand-int-value", value_from=m.ValueFrom(path="/tmp/rand_int.txt")
        ),
    )
    flakey = Container(
        name="flakey",
        image="python:alpine3.6",
        command=["python", "-c"],
        args=["import random; import sys; exit_code = random.choice([0, 1, 1]); sys.exit(exit_code)"],
        metrics=Counter(
            name="result_counter",
            labels=[Label(key="name", value="flakey"), Label(key="status", value="{{status}}")],
            help="Count of step execution by result status",
            value="1",
        ),
    )

    with Workflow(
        generate_name="hello-world-",
        entrypoint="steps",
        metrics=Gauge(
            name="duration_gauge",
            labels=Label(key="name", value="workflow"),
            help="Duration gauge by name",
            realtime=True,
            value="{{workflow.duration}}",
        ),
    ) as w:
        with Steps(
            name="steps",
            metrics=Gauge(
                name="duration_gauge",
                labels=Label(key="name", value="steps"),
                help="Duration gauge by name",
                realtime=True,
                value="{{duration}}",
            ),
        ):
            random_int()
            flakey()
    ```

=== "YAML"

    ```yaml linenums="1"
    apiVersion: argoproj.io/v1alpha1
    kind: Workflow
    metadata:
      generateName: hello-world-
    spec:
      entrypoint: steps
      templates:
      - name: steps
        steps:
        - - name: random-int
            template: random-int
        - - name: flakey
            template: flakey
        metrics:
          prometheus:
          - name: duration_gauge
            help: Duration gauge by name
            labels:
            - key: name
              value: steps
            gauge:
              realtime: true
              value: '{{duration}}'
      - name: random-int
        container:
          image: alpine:latest
          args:
          - RAND_INT=$((1 + RANDOM % 10)); echo $RAND_INT; echo $RAND_INT > /tmp/rand_int.txt
          command:
          - sh
          - -c
        metrics:
          prometheus:
          - name: random_int_step_histogram
            help: Value of the int emitted by random-int at step level
            when: '{{status}} == Succeeded'
            histogram:
              value: '{{outputs.parameters.rand-int-value}}'
              buckets:
              - 2.01
              - 4.01
              - 6.01
              - 8.01
              - 10.01
          - name: duration_gauge
            help: Duration gauge by name
            labels:
            - key: name
              value: random-int
            gauge:
              realtime: true
              value: '{{duration}}'
        outputs:
          parameters:
          - name: rand-int-value
            globalName: rand-int-value
            valueFrom:
              path: /tmp/rand_int.txt
      - name: flakey
        container:
          image: python:alpine3.6
          args:
          - import random; import sys; exit_code = random.choice([0, 1, 1]); sys.exit(exit_code)
          command:
          - python
          - -c
        metrics:
          prometheus:
          - name: result_counter
            help: Count of step execution by result status
            labels:
            - key: name
              value: flakey
            - key: status
              value: '{{status}}'
            counter:
              value: '1'
      metrics:
        prometheus:
        - name: duration_gauge
          help: Duration gauge by name
          labels:
          - key: name
            value: workflow
          gauge:
            realtime: true
            value: '{{workflow.duration}}'
    ```

