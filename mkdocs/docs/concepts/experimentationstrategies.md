---
template: overrides/main.html
---


!!! abstract ""
    Iter8's model of experimentation is powerful and expressive. Flexibly combine a variety of testing & deployment patterns, and traffic shaping & version promotion features in experiments.

## Testing pattern

Testing pattern defines the number of versions evaluated in the experiment and the logic used to determine the `winner`. Iter8 supports `Canary` and `Conformance` testing patterns.

=== "Canary"
    `Canary` testing involves two versions, a `baseline` and a `candidate`. In a `Canary` experiment, Iter8 assesses if versions satisfy the objectives[^1] specified in the experiment. If `candidate` satisfies objectives, then `candidate` is the `winner`; else, if `baseline` satisfies the objectives, then `baseline` is the `winner`; else, there is no `winner`.

    ![Canary](/assets/images/canary-progressive-kubectl.png)

    !!! tip ""
        Try a [`Canary` experiment](/getting-started/quick-start/with-knative/).

=== "Conformance"
    `Conformance` testing involves a single version, a `baseline`. Iter8 assesses if the `baseline` satisfies the objectives specified in the experiment. If it does, then `baseline` is the `winner`; else, there is no `winner`.

    ![Conformance](/assets/images/conformance.png)

    !!! tip ""
        Try a [`Conformance` experiment](/code-samples/iter8-knative/conformance/).

## Deployment pattern

Deployment pattern determines how traffic is split between versions. Iter8 supports `Progressive` and `FixedSplit` deployment patterns.

=== "Progressive"
    `Progressive` deployment incrementally shift traffic towards the `winner` over multiple iterations.

    ![Canary](/assets/images/canary-progressive-helm.png)

    !!! tip ""
        Try a [`Progressive` experiment](/code-samples/iter8-knative/canary-progressive/).

=== "FixedSplit"
    `FixedSplit` deployment does not shift traffic between versions during iterations.

    ![Canary](/assets/images/canary-fixedsplit-kustomize.png)

    !!! tip ""
        Try a [`FixedSplit` experiment](/code-samples/iter8-knative/canary-fixedsplit/).

## Traffic shaping

Traffic shaping refers to features such as `Traffic mirroring / shadowing` and `Request routing` that provide advanced controls over how traffic is routed to and from app versions. Iter8 enables you to take total advantage of all the traffic shaping features available in the service mesh, ingress technology or networking layer present in your Kubernetes or OpenShift stack.

=== "Traffic mirroring / shadowing"
    Traffic mirroring or shadowing enables experimenting with a *dark* launched version with zero-impact on end-users. Mirrored traffic is a replica of the real user requests[^2] that is routed to the dark version. Metrics are collected and evaluated for the dark version, but responses from the dark version are ignored.

    ![Canary](/assets/images/mirroring.png)

    !!! tip ""
        Try a [traffic mirroring experiment](/code-samples/iter8-knative/mirroring/).

=== "Request routing"
    Request routing is the ability to route requests dynamically to different versions of the app based on attributes such as user identity, URI, or request origin. Use request routing in experiments to specify the segment of the traffic that will participate in the experiment. For example, in a `Canary` experiment, requests within the specified segment may be routed to `baseline` or `candidate`; requests not in this segment will be routed only to the `baseline`.

    ![Canary](/assets/images/request-routing.png)

    !!! tip ""
        Try a [request routing experiment](/code-samples/iter8-knative/request-routing/).


## Version promotion

Iter8 can optionally `promote` a version at the end of an experiment. The version recommended for promotion is the `winner` if a `winner` has been found. If not, the version recommended for promotion is the `baseline`. As part of version promotion, Iter8 can configure Kubernetes resources by installing or upgrading `Helm` charts, building and applying `Kustomize` resources, or using the `kubectl` CLI to apply YAML/JSON resource manifests and perform other cleanup actions such as resource deletion.

=== "Helm"
    An experiment that uses `helm upgrade` for version promotion is illustrated below.

    ![Canary](/assets/images/canary-progressive-helm.png)

    !!! tip ""
        Try an [experiment that uses `Helm`](/code-samples/iter8-knative/canary-progressive/).

=== "Kustomize"
    An experiment that uses `kustomize build` for version promotion is illusted below.

    ![Canary](/assets/images/canary-fixedsplit-kustomize.png)

    !!! tip ""
        Try an [experiment that uses `Kustomize`](/code-samples/iter8-knative/canary-fixedsplit/).

=== "kubectl with YAML/JSON manifests"
    An experiment that uses `kubectl apply` for version promotion is illustated below.

    ![Canary](/assets/images/canary-progressive-kubectl.png)

    !!! tip ""
        Try an [experiment that uses `kubectl`](/getting-started/quick-start/with-knative/).

<!-- 
??? example "Sample experiment"
    ```yaml linenums="1"
    apiVersion: iter8.tools/v2alpha1
    kind: Experiment
    metadata:
      name: quickstart-exp
    spec:
      # `sample-app` Knative service in `default` namespace is the target of this experiment
      target: default/sample-app
      # information about versions participating in this experiment
      versionInfo:         
        # every experiment has a baseline version
        # we will name it `current`
        baseline: 
          name: current
          variables:
          # `revision` variable is used for fetching metrics from Prometheus
          - name: revision 
            value: sample-app-v1 
          # `promote` variable is used by the finish task
          - name: promote
            value: baseline
        # candidate version(s) of the app
        # there is a single candidate in this experiment 
        # we will name it `candidate`
        candidates: 
        - name: candidate
          variables:
          - name: revision
            value: sample-app-v2
          - name: promote
            value: candidate 
      criteria:
        objectives: 
        # mean latency should be under 50 milliseconds
        - metric: mean-latency
          upperLimit: 50
        # 95th percentile latency should be under 100 milliseconds
        - metric: 95th-percentile-tail-latency
          upperLimit: 100
        # error rate should be under 1%
        - metric: error-rate
          upperLimit: "0.01"
      strategy:
        # canary testing => candidate `wins` if it satisfies objectives
        testingPattern: Canary
        # progressively shift traffic to candidate, assuming it satisfies objectives
        deploymentPattern: Progressive
        weights: # fine-tune traffic increments to candidate
          # candidate weight will not exceed 75 in any iteration
          maxCandidateWeight: 75
          # candidate weight will not increase by more than 20 in a single iteration
          maxCandidateWeightIncrement: 20
        actions:
          # run tasks under the `start` action at the start of an experiment   
          start:
          # the following task verifies that the `sample-app` Knative service in the `default` namespace is available and ready
          # it then updates the experiment resource with information needed to shift traffic between versions
          - library: knative
            task: init-experiment
          # run tasks under the `finish` action at the end of an experiment   
          finish:
          # promote an app version
          # `https://raw.githubusercontent.com/iter8-tools/iter8/master/samples/knative/quickstart/candidate.yaml` will be applied if candidate satisfies objectives
          # `https://raw.githubusercontent.com/iter8-tools/iter8/master/samples/knative/quickstart/baseline.yaml` will be applied if candidate fails to satisfy objectives
          - library: common
            task: exec # promote the winning version
            with:
              cmd: kubectl
              args:
              - "apply"
              - "-f"
              - "https://raw.githubusercontent.com/iter8-tools/iter8/master/samples/knative/quickstart/{{ .promote }}.yaml"
      duration: # 12 iterations, 20 seconds each
        intervalSeconds: 20
        iterationsPerLoop: 12
    ```

## How Iter8 runs an experiment
1. Iter8 determines if it is safe to start an experiment using its [concurrency policy](http://localhost:8000/usage/experiment/target/#concurrent-experiments).

2. When the experiment starts, Iter8 runs the tasks specified under `spec.actions.start` such as setting up or updating resources needed for the experiment.

3. During each iteration, Iter8 evaluates app versions based on `spec.criteria`, determines the winner, and optionally shifts traffic towards the winner.

4. When the experiment finishes, Iter8 runs tasks specified under `spec.actions.finish` such as version promotion.

## Experiment spec in-brief
A brief explanation of the key fields in an experiment spec is given below.

### spec.target

`spec.target` is a string that identifies the app under experimentation and determines which experiments can run concurrently.

### spec.versionInfo

`spec.versionInfo` is an object that describes the app versions involved in the experiment. Every experiment involves a `baseline` version, and may involve zero or more `candidates`.

### spec.criteria

`spec.criteria` is an object that specifies the metrics used for evaluating versions along with acceptable limits for their values.

### spec.strategy.testingPattern

`spec.strategy.testingPattern` is a string enum that determines the logic used to evaluate the app versions and determine the winner of the experiment. Iter8 supports two testing patterns, namely, `Canary` and `Conformance`.

### spec.strategy.deploymentPattern

`spec.strategy.deploymentPattern` is a string enum that determines if and how traffic is shifted during an experiment[^1]. Iter8 supports two deployment patterns, namely, `Progressive` and `FixedSplit`.

### spec.strategy.weights

`spec.strategy.weights` is an object with  two integer fields, namely, `maxCandidateWeight` and `maxCandidateWeightIncrement`, that can be used to fine-tune traffic increments to the candidate. This field is applicable only for `Progressive` experiments. `maxCandidateWeight` specifies the maximum candidate weight that can be set by Iter8 during an iteration. `maxCandidateWeightIncrement` specifies the maximum increase in candidate weight during a single iteration.

### spec.strategy.actions

An action is a sequence of tasks executed during an experiment. `spec.strategy.actions` is an object that can be used to specify `start` and `finish` actions that will be executed at the start and end of an experiment respectively.

### spec.duration

`spec.duration` is an object with two integer fields, namely, `iterationsPerLoop` and `intervalSeconds`. The former specifies the number of iterations in the experiment. The latter specifies the time interval in seconds between successive iterations.

[^1]: Traffic shifting is relevant only when an experiment involves two or more versions. `Conformance` testing experiments involve a single version. Hence, `spec.strategy.deploymentPattern` is ignored in these experiments.


## Realtime Observability

!!! abstract ""
    The  **iter8ctl** CLI enables you to observe an experiment in realtime. Use iter8ctl to observe metric values for each version, whether or not versions satisfy objectives, and the winner.


??? example "Sample output from iter8ctl"
    ```shell
    ****** Overview ******
    Experiment name: quickstart-exp
    Experiment namespace: default
    Target: default/sample-app
    Testing pattern: Canary
    Deployment pattern: Progressive

    ****** Progress Summary ******
    Experiment stage: Running
    Number of completed iterations: 3

    ****** Winner Assessment ******
    App versions in this experiment: [current candidate]
    Winning version: candidate
    Recommended baseline: candidate

    ****** Objective Assessment ******
    +--------------------------------+---------+-----------+
    |           OBJECTIVE            | CURRENT | CANDIDATE |
    +--------------------------------+---------+-----------+
    | mean-latency <= 50.000         | true    | true      |
    +--------------------------------+---------+-----------+
    | 95th-percentile-tail-latency   | true    | true      |
    | <= 100.000                     |         |           |
    +--------------------------------+---------+-----------+
    | error-rate <= 0.010            | true    | true      |
    +--------------------------------+---------+-----------+

    ****** Metrics Assessment ******
    +--------------------------------+---------+-----------+
    |             METRIC             | CURRENT | CANDIDATE |
    +--------------------------------+---------+-----------+
    | request-count                  | 429.334 |    16.841 |
    +--------------------------------+---------+-----------+
    | mean-latency (milliseconds)    |   0.522 |     0.712 |
    +--------------------------------+---------+-----------+
    | 95th-percentile-tail-latency   |   4.835 |     4.750 |
    | (milliseconds)                 |         |           |
    +--------------------------------+---------+-----------+
    | error-rate                     |   0.000 |     0.000 |
    +--------------------------------+---------+-----------+
    ```    

See [here](/getting-started/quick-start/with-knative/#7-observe-experiment) for an example of using iter8ctl to observe an experiment in realtime. -->

[^1]: Objectives correspond to service level objectives or SLOs. In Iter8 experiments, objectives are specified in the form of metrics along with acceptable limits on their values.

[^2]: It is possible to mirror only a certain percentage of the requests instead of all requests.