# Low level sampler

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0023                                         |
| **Authors**       | Salva de la Puente (salva@ibm.com),  Antonio Corcole, Ian Hincks    |
| **Deprecates**    | TBD                 |
| **Submitted**     | 2025-07-30                                   |
| **Updated**       | YYYY-MM-DD                                   |

## Summary
In Qiskit, the quantum computer interface is built around primitives, which are
the fundamental operations carried on by a quantum computer on a quantum circuit.
Currently, Qiskit offers two such primitives: the Sampler and the Estimator.
The Estimator leverages advanced error mitigation techniques to enhance the quality
of results from noisy quantum computers. On a noiseless quantum computer, the
Estimator could instead be constructed atop the Sampler, simplifying its role.
However, given current noise levels, robust error mitigation is essential,
requiring executing thousands or even millions of circuit variations to implement
certain mitigation protocols. Transmitting these variations across cloud environments
is inefficient, but this can be optimized using a "samplex" — a domain-specific
language (DSL) program describing the way of producing these variations. The
samplex enables efficient server-side generation of variations, with results
transmitted back as samples accompanied by metadata, facilitating client-side
implementation of error-mitigated estimations.

## Motivation
The purpose of this development is addressing the growing demand within the
quantum information science community for advanced error mitigation techniques.
Users and researchers seek finer control over these techniques to improve the
reliability of quantum computations, especially on noisy quantum hardware.
Vendors like IBM have identified that users desire capabilities for precise
noise learning, twirling, and expectation value calculations.

This proposal will enable Qiskit users to perform large-scale, backend-optimized
error mitigation experiments without incurring the high network cost of
transmitting thousands or millions of circuit variations. By shifting variation
generation to the backend through a portable DSL, researchers can run advanced
techniques more efficiently.

Today, Qiskit users can already run large-scale error mitigation experiments,
but the process is **implicit** — it is difficult to explain and difficult to
compose, and users have no direct control over how circuit variations are
generated or how mitigation is applied. This proposal introduces the **samplex**
DSL, a portable description language that lets users explicitly define the
variation strategies applied by the backend.

With this, researchers can tailor error mitigation to their needs. The approach
preserves backend optimizations while giving users fine-grained
control, reproducibility through deterministic seeds, and access to structured
metadata for postprocessing.

## User Benefit
This proposal will primarily benefit researchers and practitioners working with
noisy quantum hardware who require precise control over error mitigation
strategies. By making the variation generation process explicit and
user-definable, it empowers users to explore and tune novel error mitigation
techniques.

Backend and platform developers will also benefit, as the **samplex** DSL
creates a standard interface for describing variation strategies that can be
implemented consistently across vendors. This improves portability, reduces
vendor lock-in, and enables backends to apply optimizations without sacrificing
user intent.  

Finally, the broader Qiskit community, including educators and tool developers,
will gain a clearer and more composable model for error mitigation workflows,
making it easier to experiment, reproduce results, and share techniques.

## Design Proposal

The intent of the following listing is to demonstrate how a user could leverage
the new low-level interface to implement a client-side basic estimator.

```python
from qiskit_ibm_runtime import QiskitRuntimeService, Session, NoiseLearner
from qiskit.transpiler import PassManager
from qiskit.transpiler.passes import NoiseLearningLayering
from qiskit.circuit import QuantumCircuit, Parameter, QuantumRegister, ClassicalRegister

# This proposal assumes the semantics implemented in samplomatic will become
# part of qiskit semantics at some moment.
from qiskit import samplex
from qiskit.samplex.annotations import InjectNoise, Twirl

# The proposal wants to acknowledge the transtionary reality in which the
# primitives continue being the entry point for operating the quantum computer
# and this is a low level interface for advanced usage.
from qiskit.primitives.implementation import QuantumProgram, Executor

import numpy as np
import matplotlib.pyplot as plt

service = QiskitRuntimeService(name="staging-cloud")
backend = service.backend("test_heron")

# Build a circuit for which we need to learn the noise.
circuit = QuantumCircuit(2)

with circuit.box([Twirl(), InjectNoise(ref="my_noise")]):
  circuit.cx(0, 1)

with circuit.box([Twirl(), BasisTransform(ref="my_basis")]):
  circuit.measure_all()

# Extract layers of the circuit
layers = find_unique_layers(circuit)

# Prepare circuit for executing
template, samplex_ = samplex.build()

with Session(backend=backend) as session:
  # Learning noise
  noise_learner = NoiseLearner(backend)
  noise_learner_result = noise_learner.run(layers).result()
  noise_map = noise_learner_result["noise_map"]

  # Preparing a quantum program for noise-aware sampling
  program = QuantumProgram(shots=1024)
  program.declare_noise_map("my_noise", noise_map)
  program.append(
    template,
    samplex=samplex_,
    basis_transforms={"my_basis": "XX"},
  )

  # Execute (sample) the circuit
  executor = Executor(backend)
  job = executor.run(program)

# And this section should estimate based on the results from sampling the
# template and executing the samplex.
results = job.results()
signs = results["signs"]
counts = results["meas"]
expectation = mitigated_estimation(signs, counts)
```

## Detailed Design
TBD

## Alternative Approaches
An alternative to introducing the **samplex** DSL is to generate all circuit
variations entirely on the client side and transmit them to the backend for
execution. This approach offers maximum flexibility and complete control over
how variations are produced, since the backend would simply execute the
provided circuits without influencing their structure. Users could implement
any custom error mitigation workflow they wish, with no restrictions imposed by
backend capabilities or vendor‑specific optimizations.

However, this approach has significant drawbacks. Large‑scale error mitigation
often requires thousands or even millions of circuit variations, making
client‑side generation impractical for real‑world workloads. Transmitting such a
large volume of circuits over the network introduces high latency, increases
execution time, and places substantial demands on both client and backend
infrastructure. It also complicates reproducibility across vendors, since each
vendor may handle large‑batch execution differently.

By contrast, the **samplex** DSL allows users to describe their variation
generation strategies in a portable, serialized form that can be transmitted
efficiently. The backend can then expand these strategies into actual circuit
variations locally, reducing network load while preserving user control over
error mitigation techniques.

## Questions
Open questions for discussion and an opening for feedback.
TBD

## Future Extensions

The introduction of a low‑level sampler is only the first step. The long‑term
goal is to evolve Qiskit’s compute model toward accepting a single, unified
operation—this primitive—potentially deprecating the current definition of
primitives altogether. In this model, noise learning, error mitigation, and any
other computation could be implemented directly on top of the unified primitive
abstraction.

Future RFCs may explore implementing advanced workflows such as noise learning
natively within this computation model, as well as replacing Qiskit’s existing
compute interface with the unified approach. This shift would simplify the
execution pipeline, make workflows more composable, and provide a consistent
foundation for new quantum algorithms and techniques.

Additional future work could explore defining domain‑specific languages beyond
**samplex**, including DSLs dedicated to calculating expectation values or
other specialized tasks. These DSLs could be layered on top of the unified
primitive to further extend Qiskit’s flexibility and expressiveness while
retaining backend portability.
