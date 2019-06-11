# iter8: Analytics-driven canary releases and A/B testing

## What is iter8 about?

Iter8's mission is to support automated canary releases and A/B testing, driven by analytics based on robust statistical techniques for canary releases and A/B testing. The project comprises two components:

* _iter8-analytics_: An analytics service that analyzes metrics from different versions of a microservice. The metric analysis takes into account success criteria provided by the user and can assess the health of a canary both absolutely and comparatively to a baseline version.

* _iter8-controller_: A Kubernetes controller that automates canary releases and A/B testing by gradually adjusting the traffic across different versions of a microservice. For instance, in the case of a canary release, the controller will gradually shift the traffic to the canary version if it is performing as expected, until the canary replaces the baseline (previous) version. If the canary is found not to be satisfactory, the controller rolls back by shifting all the traffic to the baseline version. Traffic decisions are made by the _iter8-analytics_ service and honored by the controller.

## Supported environments

The _iter8-controller_ supports the following Kubernetes-based environments, whose traffic-management capabilities are used:

* [Istio service mesh](https://istio.io)
* [Knative](https://knative.dev)

## Integrations

Iter8 is integrated with (Tekton Pipelines)[https://tekton.dev] for an end-to-end CI/CD experience, and with (KUI)[https://github.com/IBM/kui], for a richer Kubernetes command-line experience. Initial integrations with these two technologies already exist, but we are actively improving them. Stay tuned!
