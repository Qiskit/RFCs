# Executor primitive

| **Status**        | **Proposedd**                                |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0023                                         |
| **Authors**       | Salva de la Puente (salva@ibm.com)           |
|                   | Ian Hincks (ian.hincks@ibm.com)              |
|                   | Antonio Corcoles (adcorcol@us.ibm.com)       |
| **Deprecates**    | TBD                                          |
| **Submitted**     | 2025-07-30                                   |
| **Updated**       | 2025-08-30                                   |

## Summary
This RFC proposes the introduction of a new **Executor primitive** into Qiskit
as a vendor-agnostic substrate for execution. The Executor provides
infrastructure through three core components—**Executor**, **ExecutorProgram**,
and **ExecutorResult**—which capture the relationship between an execution
engine, its semantics, and its results.

The immediate motivation is to make **randomization semantics for error
mitigation** first-class citizens in Qiskit. Today these workflows exist in an
implicit and opaque form; this RFC proposes to standardize how users express
their *intent* through annotations and portable representations. This enables
fine-grained control, reproducibility, and composability of error-mitigation
experiments while keeping provider implementations free to optimize expansion
strategies.

It is important to note that this RFC does **not** propose adopting a specific
engine such as IBM’s Samplomatic into Qiskit. Instead, the scope is limited to
defining the substrate (Executor) and the representation of user intent. Vendor
implementations can build their own engines on top of these abstractions.
Follow-up RFCs may introduce additional semantics or reference implementations
(e.g. a noisy-statevector executor), but this proposal focuses on laying the
foundation.

## Motivation
The purpose of this proposal is to address the growing demand in the quantum
information science community for advanced error-mitigation techniques. Users
and researchers seek finer control over these methods to improve the reliability
of computations on noisy quantum hardware, with interest in capabilities such as
noise learning, twirling, and expectation value calculations.

Qiskit already supports large-scale mitigation experiments, but today the
process is **implicit**—difficult to explain, compose, or control. By shifting
circuit-variation generation to the backend through a portable representation,
researchers can run these workflows more efficiently and without the high
network cost of transmitting thousands of circuit variants. Making this process
explicit also enables reproducibility through seeds and access to structured
metadata for post-processing.

To support this, we propose a new **Executor primitive**. The Executor provides
a vendor-agnostic substrate for execution where semantics are captured in an
**ExecutorProgram**. This allows providers to expose arbitrary programming
models and gives Qiskit a single, extensible foundation for advanced semantics—
with error mitigation as a leading use case, but equally applicable to future
methods such as estimation, hybrid execution, noise learning protocols, or shot
scheduling.

## User Benefit
This proposal will primarily benefit researchers and practitioners working with
noisy quantum hardware who require precise control over advanced execution
semantics. Error mitigation is a leading example: by making variation generation
explicit and user-definable, users can explore and tune mitigation techniques
more directly.

Backend and platform developers will also benefit, as the proposed
**representation** for variations and execution semantics provides a standard
interface that can be implemented consistently across vendors. This improves
portability, reduces vendor lock-in, and enables backends to apply optimizations
without sacrificing user intent.

Finally, the broader Qiskit community, including educators and tool developers,
will gain a clearer and more composable model for execution workflows, making it
easier to experiment with new techniques, reproduce results, and share methods.

## Design Proposal
The intent of the following listing is to demonstrate how a user could leverage
the new interface to implement a client-side basic estimator.

```python
from qiskit.transpiler import generate_preset_pass_manager
from qiskit.transpiler.passes import NoiseLearningLayering
from qiskit.circuit import QuantumCircuit, Parameter, QuantumRegister, ClassicalRegister

# This proposal assumes the randomizaton semantics implemented in Samplomatic will become
# part of Qiskit semantics.
from qiskit.circuit.annotations import BasisTransform, InjectNoise, Twirl

# IBM-proprietary Samplex becomes the vendor-agnostic RandomizationPlan.
from qiskit.circuit.annotations.rand import RandomizationPlan

from qiskit_ibm_runtime import QiskitRuntimeService, Session

# Vendor-specific (such as IBM) classes derive from `Base` classes defined
# in Qiskit. In particular, `QuantumProgram` and `NoiseLearningProgram` are both
# `ExecutorProgram` instances.  
from qiskit_ibm_runtime import Executor, QuantumProgram
from qiskit_ibm_runtime import NoiseLearningProgram

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

# Convert into ISA circuit.
isa_pm = generate_preset_pass_manager(backend=backend, optimization_level=0)
isa_circuit = isa_pm.run(circuit)

# Extract layers of the circuit
circuit_layers = NoiseLearningProgram.find_unique_layers(isa_circuit)

# Prepare circuit for executing
template, plan = RandomizationPlan.build(isa_circuit)

with Session(backend=backend) as session:
  # Instantiate the programming model primitive
  executor = Executor(backend)
  
  # Learning noise
  noise_learner = NoiseLearningProgram(circuit_layers)
	job = executor.run(noise_learner)
  result = job.result()
  
  # Extract noise map
  noise_map = { layer: model for zip(circuit_layers, result) }

  # Preparing a quantum program for noise-aware sampling
  program = QuantumProgram(shots=1024)
  program.define_symbol("my_noise", noise_map)
  program.define_symbol("my_basis", "XX")
  program.append(template, randomization_plan=plan)
  job = executor.run(program)

# And this section should estimate based on the results from sampling the
# template and executing the samplex.
result = job.result()
signs = result["signs"]
counts = result["meas"]
expectation = mitigated_estimation(signs, counts)
```

Both `QuantumProgram` and `NoiseLearningProgram` are implemented deriving
from the base class in Qiskit `BaseExecutorProgram` and `Executor` is derived
from `BaseExecutor`, with the following definitions:

```python
# qiskit/primitives/containers/quantum_program.py

from abc import abstractmethod, ABC

class BaseExecutorProgram(ABC):
  ...

# qiskit/primitives/base/base_executor.py

from abc import abstractmethod, ABC
from ..containers import ExecutorProgram, ExecutorResult

class BaseExecutor(ABC):
  ...
```

## Detailed Design

### Executor, ExecutorProgram, and ExecutorResult
The `Executor` primitive introduces a general substrate for execution in
Qiskit. A quantum circuit alone represents only part of the operation of a
quantum computer. Just as a classical computer requires instructions beyond ALU
operations, a quantum computer requires additional directives to govern
*aspects* such as observable measurements, randomization, batching, or shot
scheduling. The `ExecutorProgram` interface captures these semantics in a
portable form.

Execution produces not only raw measurement data but also metadata about the
run, including reproducibility details and non-sensitive machine state. The
`ExecutorResult` interface provides this data in a normalized detailed here:

TBD: insert Python class definition.

Providers may include additional metadata (backend id, layout, noise
diagnostics, etc.) but may redact or aggregate sensitive calibration data. Each
provider is responsible for documenting what fields it returns.

### Randomization annotations and plans
In addition to circuit operations, users need a way to express semantics for
error mitigation. **Annotations** (e.g., `Twirl`, `InjectNoise`) act as
domain-specific instructions bound to circuit regions. These form a
domain-specific language (DSL) that augments circuit semantics to express
*intent*, such as running a randomization protocol.

Annotations are compiled into a **Randomization Plan** (vendor-agnostic version
of IBM Samplomatic's “samplex” data structure), a portable data structure describing
how to generate circuit variations. The plan is constructed **after
transpilation/layout**, ensuring that annotations refer to stable circuit regions. It
is serializable and can be transmitted efficiently to the backend. Providers then
expand the plan into potentially millions of concrete circuit variants server-side,
reducing network load and preserving reproducibility.  

The plan, together with user-defined symbols (e.g., noise maps, measurement
bases), is packaged inside the `ExecutorProgram`. This allows fast template
hydration within a provider’s runtime without requiring clients to generate or
transmit large batches of circuits.

### Noise learning
Noise learning is a provider-specific capability where the system characterizes
its own error profile. Learned artifacts such as a **noise map** can be returned
in metadata or bound into an `ExecutorProgram` as symbols (e.g.,
`program.define_symbol("my_noise", noise_map)`). This RFC does not standardize a
specific noise-learning engine; it only ensures the `ExecutorProgram` can carry
such symbols in a portable way.

TBD: how an ExecutorProgram is serialized?

### System drift between learning and mitigation
Separating noise learning from execution raises the risk of **system drift**:
the characterization may become outdated between the time a noise map is built
and when it is applied in mitigation. To reduce this risk, providers may offer
*session-like constructs* that keep system state stable across a sequence of
runs. In the IBM Runtime, for example, sessions can provide exclusive access so
that noise learning and subsequent mitigation are performed back-to-back.  

This RFC does not mandate sessions or exclusivity; these remain provider-owned
features. The important point is that the `ExecutorProgram` substrate can
represent both noise-learning and noise-mitigated execution, and users can
combine them in a controlled workflow when supported.

### Relation to existing primitives
While this RFC does not replace existing primitives, it introduces a substrate
that could host them. In particular, `Sampler` and `Estimator` can be
re-implemented as thin profiles on top of the `Executor` abstraction. This would
allow users to keep their familiar APIs while providers optimize around a single
execution foundation. Such a reimplementation is left to future RFCs.

### Security and privacy considerations

The introduction of randomization semantics and metadata raises potential
security and privacy issues:

- **Noise maps and calibration data.** These may reveal proprietary details
  about a provider’s hardware or calibration process. Providers may need to
  redact, aggregate, or abstract sensitive information before returning it to
  users.
- **User-generated metadata.** Seeds, labels, and symbolic parameters may
  contain experiment identifiers or other sensitive data. These must be stored
  and transmitted securely, especially when persisted for reproducibility.
- **Cross-provider reproducibility.** Metadata required for reproducibility
  (e.g., seeds, plan identifiers) should be retained, but providers remain free
  to limit or sanitize additional fields that could expose internal details.
- **Documentation requirements.** Each provider should document what metadata is
  returned, what may be redacted, and under what circumstances. This ensures
  transparency for users while respecting provider confidentiality.

These considerations do not affect the core `Executor` interfaces, but they are
important for implementers to address in provider-specific documentation.

## Alternative Approaches
An alternative to introducing the randomization plan DAG is to generate all circuit
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

By contrast, the randomization plan allows users to describe their variation
generation strategies in a portable, serialized form that can be transmitted
efficiently. The backend can then expand these strategies into actual circuit
variations locally, reducing network load while preserving user control over
error mitigation techniques.

Yet another alternative would have been sending a circuit template and a set
of parameters on the wire for server parametrization although server expansion
of the plan enables lazy generation of parameters where new parameters can be
generated "on the fly" while other already-parametrized circuits are running. 

An intermediate solution would have been to save the user from generating the
plan in the client, sending the annotated circuit only. However, local
inspectability and debuggability would require local generation of the reandomization
plan anyhow. More importantly, the compute model becomes simpler and honors its main
responsibillity: execution.

It is worth noting that earlier drafts of this RFC explored introducing the
`Executor` as a sampler-like primitive that would directly capture additional
semantics beyond the current `Sampler` and `Estimator`. While this approach
could have delivered short-term functionality, we concluded that it risked
creating yet another primitive with overlapping scope. Framing the design instead
around the **`Executor`** as an abstract programming model is more future-proof
and flexible: it allows different providers to expose their own semantics within a single
substrate, and it mitigates fragmenting Qiskit’s compute interface with multiple
too specialized primitives.

## Questions
Open questions for discussion and an opening for feedback.

- Should Qiskit `quantum_info` include noisy statevector evolution?
- Should Qiskit include a reference `Executor` implementation, executing Samplomatic
  locally based on a noisy `StateVector`?

## Future Extensions
The introduction of the Executor is a first step toward evolving Qiskit's compute
model with a single, general substrate for execution. This primitive can live
side by side with existing primitives, offering an alternative path that
gradually demonstrates its value without requiring immediate changes to the way
Qiskit works. In this model, error mitigation, noise learning, and other forms
of computation can be expressed directly on top of the Executor abstraction.

A natural follow-up will be to reimplement the existing `Sampler` and `Estimator`
interfaces on top of `Executor`. Doing so would show how today’s most common
workflows can be captured as lightweight profiles over the new substrate. This
would give users a migration path with familiar APIs, while enabling providers
to optimize around a single execution foundation.

Beyond this, future RFCs may explore implementing more advanced workflows such
as noise learning natively within the Executor model. This would simplify the
execution pipeline, make workflows more composable, and provide a consistent
foundation for new quantum algorithms and techniques.

Additional extensions could include domain-specific representations beyond
**randomization** annotations, such as those dedicated to calculating expectation
values or shot scheduling. These could be layered on top of the Executor to
further extend Qiskit’s flexibility and expressiveness while retaining backend
portability.
