# Qiskit RFCs

The purpose of a Qiskit Request for Comments (RFC) is to communicate and engage
with the wider community in the development and direction of Qiskit. RFCs enable
engineering and research stakeholders to communicate design changes to any
component of the larger Qiskit project.

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put through a
bit of a design process and produce a consensus among the Qiskit community and
the sub-teams.

This repository is for any project in the Qiskit project to make these design
proposals and review and discuss the changes. An RFC can be contained to a
single project or it can span multiple projects, if a design decision requires
some planning and discussion before implementation an RFC can be used to
facilitate that and collect feedback prior to implementation.

| RFC | Status | References/Discussion |
| --- | ------ | --------------------- |
| `0020` [Qiskit release cycle and versioning](0020-release_cycle.md) | Implementation in progress | [RFC PR](https://github.com/Qiskit/RFCs/pull/34/) \| [Implementation](https://github.com/Qiskit/RFCs/issues/64) |
| `0019` [Classical storage and load instructions](0019-classical-stores.md) | Implementation in progress | [RFC PR](https://github.com/Qiskit/RFCs/pull/50/) \| [Implementation](https://github.com/Qiskit/qiskit/issues/10922) |
| `0018` [Primitive options type change](0018-primitive-options-type.md) | Implementation in progress | [RFC PR](https://github.com/Qiskit/RFCs/pull/52/) \| Implementation: [1](https://github.com/Qiskit/qiskit/pull/11542), [2](https://github.com/Qiskit/qiskit/pull/11594) |
| `0017` [Base Primitive and Units of Primitive Work](0017-base-primitive-unification.md) | Implementation in progress | [RFC PR](https://github.com/Qiskit/RFCs/pull/53/) \| [Implementation](https://github.com/Qiskit/qiskit/pull/11524) |
| `0016` [SamplerV2](0016-sampler-interface.md) | Implemented | [RFC PR](https://github.com/Qiskit/RFCs/pull/56/) \| [Implementation](https://github.com/Qiskit/RFCs/issues/71) |
| `0015` [Extended Estimator Interface](0015-estimator-interface.md) | Implementation in progress | [RFC PR](https://github.com/Qiskit/RFCs/pull/51) \| [Implementation](https://github.com/Qiskit/qiskit/pull/11227) |
| `0014` [Overhaul Qiskit Experiments](0014-overhaul-qiskit-experiments.md) | Implementation in progress | [RFC PR](https://github.com/Qiskit/RFCs/pull/47) \| [Implementation](https://github.com/Qiskit-Extensions/qiskit-experiments/issues/1268) |
| `0013` [Plan to move `Qiskit/qiskit-terra/algorithms` module to `qiskit-community/qiskit-algorithms` independent repo](0013-algorithms-migration.md) | Implemented | [RFC PR](https://github.com/Qiskit/RFCs/pull/44/) \| [Implementation](https://github.com/Qiskit/RFCs/issues/48)|
| `0012` [Pulse Compiler & IR](0012-Pulse-Compiler-and-IR.md) | Implementation in progress | [RFC PR](https://github.com/Qiskit/RFCs/pull/45) \| [Implementation](https://github.com/Qiskit/qiskit/issues/10759)|
| `0011` [Plan to rename `Qiskit/qiskit-terra` repo to `Qiskit/qiskit`](0011-repo-rename.md) | Implemented | [RFC PR](https://github.com/Qiskit/RFCs/pull/31) \| [Implementation](https://github.com/Qiskit/RFCs/issues/41) |
| `0010` [Preliminary representation of rvalue classical expression in Qiskit](0010-simple-classical-representations.md) | Implemented | [RFC PR](https://github.com/Qiskit/RFCs/pull/30) \| [Implementation](https://github.com/Qiskit/qiskit-terra/issues/10239) |
| `0009` [`Operation`: the interface for valid `QuantumCircuit` operations](0009-interface-for-circuit-operations.md) | Implemented | [RFC PR](https://github.com/Qiskit/RFCs/pull/25) \| [Implementation](https://github.com/Qiskit/qiskit-terra/pull/7087)|
| `0008` [Unify Pulse Commands and Instructions](0008-unify-pulse-commands-and-instructions.md) | Implemented | [RFC PR](https://github.com/Qiskit/RFCs/pull/12) \| [Implementation](https://github.com/Qiskit/qiskit-terra/issues/3750)
| `0007` [Dataframe for Qiskit Experiments](0007-experiment-dataframe.md) | Implementation in progress | [RFC PR](https://github.com/Qiskit/rfcs/pull/28) \| [Implementation](https://github.com/Qiskit/RFCs/issues/62) |
| `0006` [Generalized Unroller and Equivalence Library](0006-rfc-generalized-unroller-and-equivalence-library.md) | Implemented | [RFC PR](https://github.com/Qiskit/rfcs/pull/6) \| [Implementation](https://github.com/Qiskit/qiskit-terra/pull/3946)|
| `0005` [Aqua Circuit Interoperability Update](0005-Aqua_circuit_interoperability.md) | Deprecated | [RFC PR](https://github.com/Qiskit/RFCs/pull/17) |
| `0004` [Richardson error mitigation](0004-error-mitigation.md) | Implemented | [RFC PR](https://github.com/Qiskit/rfcs/pull/3) \| [Original PR](https://github.com/Qiskit/qiskit-metapackage/pull/768) \| [Implementation](https://github.com/qiskit-community/prototype-zne) | 
| `0003` [Aqua 0.7 Operator Redesign](0003-Aqua_0.7_operator_redesign.md) | Deprecated | [RFC PR](https://github.com/Qiskit/rfcs/pull/8) \| [Original PR](https://github.com/Qiskit/qiskit-aqua/pull/742)|
| `0002` [Aqua 0.7 Release Priorities and Plan](0002-Aqua_0.7_release_priorities_and_plan.md) | Deprecated |[RFC PR](https://github.com/Qiskit/rfcs/pull/7) \| [Original PR](https://github.com/Qiskit/qiskit-aqua/pull/785)|
| `0001` [Aqua 0.7: Ansatz Design Doc](0001-ansatz-rfc.md) | Deprecated |[RFC PR](https://github.com/Qiskit/rfcs/pull/5) \| [Original PR](https://github.com/Qiskit/qiskit-aqua/pull/747) |


# How to RFC?
Use the [Qiskit RFC template](0000-template.md) to prepare your RFC. Make sure you read [CONTRIBUTING.md](CONTRIBUTING.md) first.

## License
[License]: #license

This repository is licensed under

 Apache License, Version 2.0, ([LICENSE](LICENSE) or <http://www.apache.org/licenses/LICENSE-2.0>)
