---
title: "Formally Verifying Kubernetes Controllers"
date: 2025-08-10T15:00:00+09:00
draft: false
katex: true
---

A [Kubernetes Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) is an extension pattern that enables users to build custom [controllers](https://kubernetes.io/docs/concepts/architecture/controller/) for managing applications and clusters.
Because operators often need to manipulate [custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) asynchronously and coordinate with external systems, implementing them requires a careful and deliberate approach.

To build complex applications while avoiding subtle bugs, developers sometimes employ [formal methods](https://en.wikipedia.org/wiki/Formal_methods)--mathematically rigorous techniques for specifying and verifying software behavior.
**Anvil** is a framework introduced by Sun et al. at USENIX OSDI '24 that applies formal methods to the design and verification of Kubernetes operators.
In this article, we will take a closer look at Anvil based on its conference [paper](https://www.usenix.org/conference/osdi24/presentation/sun-xudong) and public [GitHub repository](https://github.com/anvil-verifier/anvil).

> Source: Xudong Sun, Wenjie Ma, Jiawei Tyler Gu, Zicheng Ma, Tej Chajed, Jon Howell, Andrea Lattuada, Oded Padon, Lalith Suresh, Adriana Szekeres, and Tianyin Xu. 2024. Anvil: Verifying Liveness of Cluster Management Controllers. In Proceedings of the 18th USENIX Conference on Operating Systems Design and Implementation (OSDI'24). USENIX Association, USA, Article 35, 649–666. [(usenix.org)](https://www.usenix.org/conference/osdi24/presentation/sun-xudong)

## Anvil

[Anvil](https://github.com/anvil-verifier/anvil) is a Rust framework that supports both the implementation and formal verification of Kubernetes controllers.
It enables developers to formally verify that their controllers satisfy certain properties, such as:

- **Liveness** — for example, ensuring that for any given Kubernetes custom resource object, the corresponding ConfigMap is _eventually_ created.
- **Safety** — for example, ensuring that the number of replicas _always_ remains at or above its current value.

For specification verification, Anvil uses [Verus](https://verus-lang.github.io/verus/guide/overview.html), a formal verification framework built on top of Rust.
Developers can write specifications and proofs directly in Rust code using the [`verus!` macro](https://verus-lang.github.io/verus/guide/verus_macro_intro.html), then run the `verus` command to verify that the executable code satisfies those specifications.
Code written exclusively for specifications and proofs is called *ghost code*, and it is stripped out when compiling the final executable.

## Implementing a Controller

In Anvil, a Kubernetes controller is a state machine that implements the [`Reconciler`](https://github.com/anvil-verifier/anvil/blob/main/src/reconciler/exec/reconciler.rs) trait.
The controller's initial state and state transitions are defined by `reconcile_init_state` and `reconcile_core`, respectively[^1].

```rust
pub trait Reconciler {
    // S: type of the reconciler state of the reconciler.
    type S;
    // K: type of the custom resource.
    type K;
    // EReq: type of request the controller sends to the external systems (if any).
    type EReq;
    // EResp: type of response the controller receives from the external systems (if any).
    type EResp;

    // reconcile_init_state returns the initial local state that the reconciler starts
    // its reconcile function with.
    // It conforms to the model's reconcile_init_state.
    fn reconcile_init_state() -> Self::S;

    // reconcile_core describes the logic of reconcile function and is the key logic we want to verify.
    // Each reconcile_core should take the local state and a response of the previous request (if any) as input
    // and outputs the next local state and the request to send to API server (if any).
    // It conforms to the model's reconcile_core.
    fn reconcile_core(cr: &Self::K, resp_o: Option<Response<Self::EResp>>, state: Self::S) -> (Self::S, Option<Request<Self::EReq>>);

    // reconcile_done is used to tell the controller_runtime whether this reconcile round is done.
    // If it is true, controller_runtime will requeue the reconcile.
    // It conforms to the model's reconcile_done.
    fn reconcile_done(state: &Self::S) -> bool;

    // reconcile_error is used to tell the controller_runtime whether this reconcile round returns with error.
    // If it is true, controller_runtime will requeue the reconcile with a typically shorter waiting time.
    // It conforms to the model's reconciler_error.
    fn reconcile_error(state: &Self::S) -> bool;
}
```

`reconcile_core` accepts three inputs: the current Kubernetes custom resource object (`cr`), the response (`resp_o`) to the most recent request sent to an external system, and the current controller state `state`.
It then computes the next state and the requests to be sent to external systems—such as the Kubernetes API server or other managed systems.
These state transitions and request handling are repeatedly executed by the `Reconciler` runner, [`run_controller`](https://github.com/anvil-verifier/anvil/blob/main/src/shim_layer/controller_runtime.rs#L36).

For instance, in the [`zookeeper_controller`](https://github.com/anvil-verifier/anvil/tree/main/src/controllers/zookeeper_controller), `reconcile_core` transitions the controller state from the [initial `Init` state to `AfterKRequestStep`](https://github.com/anvil-verifier/anvil/blob/main/src/controllers/zookeeper_controller/exec/reconciler.rs#L89-L95), and returns the next request to execute (retrieving the `HeadlessService`).

Implementing controllers as state machines enables the verification of specifications expressed in [TLA (Temporal Logic of Actions)](https://en.wikipedia.org/wiki/Temporal_logic_of_actions).
Since Verus itself does not natively support TLA, Anvil implements TLA models and predicates internally as a _TLA embedding_.

## Verifying a Controller

A typical Kubernetes controller updates the managed objects based on the `spec` described in the Kubernetes resources, aiming to bring those objects to a desired state.
The primary goal of formal verification for Kubernetes controllers is to ensure that the controller will **eventually update the managed objects to this desired state**.

However, expressing such a specification in a practical and sound way is challenging.
For instance, if the `spec` of a Kubernetes object is frequently updated, the managed object might never reach a stable desired state because the target keeps changing.
Any specification must account for such scenarios.
Furthermore, verifying liveness properties requires accurately modeling possible failures in the controller or environment, as well as making appropriate assumptions about fairness in asynchronous systems.

As a general formal verification framework for Kubernetes controllers, Anvil generalizes this specification as **ESR** (eventually stable reconciliation) and provides APIs and lemmas to support its verification.

### ESR

The ESR specification is formally expressed using the following TLA formula:

$$
\forall d.\Box(\Box\mathrm{desire}(d) \implies \Diamond\Box\mathrm{match}(d)).
$$

Here, $d$ represents a state description of the object. $\mathrm{desire}(d)$ is true when $d$ matches the desired state currently specified in the Kubernetes object, and $\mathrm{match}(d)$ is true when $d$ matches the actual state of the managed object.
The temporal operators $\Diamond$ (eventually) and $\Box$ (always) are used to express timing properties.

In essence, ESR states that once the desired state $d$ is fixed indefinitely in the Kubernetes object ($\Box\mathrm{desire}(d)$), the managed object will eventually reach and remain in that desired state from some point onward ($\Diamond\Box\mathrm{match}(d)$).
The outer $\Box$ operator captures scenarios where the Kubernetes object's `spec` may change multiple times, causing repeated reconciliations.

### Failure Model

Anvil models two types of failures: controller crashes and request failures. However, simplistic models where failures occur either constantly or only once do not adequately capture realistic system behavior.
Therefore, Anvil assumes that failures can happen arbitrarily many times but **eventually cease after some point**, as [formalized here](https://github.com/anvil-verifier/anvil/blob/050ed26698c677c9fd8f6c27ca1a224b949c0066/src/kubernetes_cluster/proof/failures_liveness.rs#L13).

### Verification Process

Developers verify that their Kubernetes controller implementation satisfies the specification through the following steps:

1. Create a model of the controller based on its implementation.
2. Verify that the controller model accurately reflects the behavior of the implementation.
3. Verify that the controller model satisfies the specification.

In the first step, developers create the controller model by implementing the `reconciler::spec::Reconciler` trait from the controller implementation (the `reconciler::exec::Reconciler` trait implementation).
Within this model, data structures such as `Vec` and `HashMap` are replaced with proof-friendly counterparts provided by Verus, such as [`Seq`](https://verus-lang.github.io/verus/verusdoc/vstd/seq/struct.Seq.html) and [`Map`](https://verus-lang.github.io/verus/verusdoc/vstd/map/struct.Map.html)[^2].

The second step ensures that the model faithfully mimics the implementation.
This is achieved by asserting the equivalence of return values from corresponding methods in both the implementation and the model, expressed as postconditions using [`ensure` clauses](https://verus-lang.github.io/verus/guide/requires_ensures.html#postconditions-ensures-clauses).
For an example, see the [`zookeeper_controller`](https://github.com/anvil-verifier/anvil/blob/050ed26698c677c9fd8f6c27ca1a224b949c0066/src/controllers/zookeeper_controller/exec/reconciler.rs#L84).

Finally, the controller model is verified against specifications such as ESR.
This verification incorporates Anvil’s environment models—including the Kubernetes API server and asynchronous network.
To assist developers in writing proofs, Anvil provides a library of approximately 60 lemmas covering temporal logic and environmental properties.

## Case Study

The authors reimplemented several widely used Kubernetes operators using Anvil and verified that they meet specifications such as ESR.
During verification, they discovered bugs in the controller implementations, some of which also exist in the original operators.

- The [pravega/zookeeper-operator](https://github.com/pravega/zookeeper-operator) is a Kubernetes operator for ZooKeeper provided by [Pravega](https://www.cncf.io/projects/pravega/).
While reimplementing and verifying this operator with Anvil, they found a bug where the StatefulSet would stop updating if the controller crashed just before creating a znode.
This liveness bug was confirmed to also exist in the original operator ([pravega/zookeeper-operator#569](https://github.com/pravega/zookeeper-operator/issues/569)).
- The [rabbitmq/cluster-operator](https://github.com/rabbitmq/cluster-operator) is the official Kubernetes operator managing RabbitMQ clusters.
Since downsizing a RabbitMQ cluster may cause data loss, validation rules were set on the CRD to prevent the `replicas` count from decreasing.
However, they discovered a bug where the number of replicas could decrease if the deletion of StatefulSets by [Kubernetes garbage collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/) was delayed.

## Impressions

For stateful systems like Kubernetes controllers, it is common practice to separate the model from the implementation and verify only the model to validate the design, especially when checking temporal properties such as liveness and safety.
Model checking with [TLA+](https://lamport.azurewebsites.net/tla/tla.html?from=https://research.microsoft.com/en-us/um/people/lamport/tla/tla.html&type=path) is a prominent example, employed in verifying distributed transactions in [TiDB](https://github.com/pingcap/tla-plus) and designs of [AWS DynamoDB and S3](https://cacm.acm.org/research/how-amazon-web-services-uses-formal-methods/).

In contrast, when temporal properties are not required, tools and languages supporting formal verification of the implementation itself—such as [Dafny](https://dafny.org/) and [F*](https://fstar-lang.org/)—are often used.
Notable examples include the formal verification of the [IAM SDK authorization engine inside AWS](https://www.amazon.science/publications/formally-verified-cloud-scale-authorization) and memory safety verification of cryptographic libraries like [HACL*](https://github.com/hacl-star/hacl-star).

Anvil is a pioneering project in that it enables verification that the **implementation itself of a practical system like a Kubernetes controller satisfies temporal properties**.
It is also notable for adopting a proof-based approach with Verus, rather than relying on exhaustive model checking of all execution paths.
However, the adoption barrier remains high—for instance, due to the introduction of a TLA embedding and the requirement for developers to write both the implementation and the model[^3].
For these challenges, breakthroughs in verification frameworks are highly anticipated.

[^1]: The paper uses the names `initial_state` and `step`, but here we follow the naming conventions used in the implementation published on GitHub.

[^2]: The [Verus tutorial on verifying binary search trees](https://verus-lang.github.io/verus/guide/container_bst_first_draft.html) serves as a helpful reference.

[^3]: In Dafny, instead of separating implementation and model, you write only the implementation code for verification and compile it to languages like Java.
However, since Dafny’s collection types such as [`seq`](https://dafny.org/latest/DafnyRef/DafnyRef#5551-sequences-and-arrays) and [`map`](https://dafny.org/latest/DafnyRef/DafnyRef#5553-maps) are specialized for proofs, they can become performance bottlenecks after compilation, as in the [IAM SDK authorization engine verification paper](https://www.amazon.science/publications/formally-verified-cloud-scale-authorization).

