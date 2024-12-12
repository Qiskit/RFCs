# Introducing `Box`es in Qiskit

| **Status**        | **Proposed** |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0022                                         |
| **Authors**       | [Sam Ferracin](sam.ferracin@ibm.com), [Ian Hincks](ian.hincks@ibm.com), [Jake Lishman](jake.lishman@ibm.com), [Joshua Skanes-Norman](joshua.sn@ibm.com), [Erick Winston](ewinston@us.ibm.com)    |
| **Submitted**     | YYYY-MM-DD                                   |

## Summary

Introduce a way of grouping instructions in Qiskit SDK that can have downstream data attached, pass through transpilation, and up and down an execution stack.
The case-study use of this is for Qiskit Runtime primitives to provide more flexibility and transparency in Pauli twirling and error mitigation.

Qiskit does not currently have the concept of a "grouping" of instructions, other than as a user-defined custom gate.
However, the ability to group certain instructions to appear as a "block" in a larger context is important for many compilation tasks:

* a block of several gates might have a known unitary action, and can be treated as atomic for commutation purposes, even if a candidate gate doesn't commute with each block element individually.
* noise-learning techniques might be applied to simultaneous logical blocks of operations, rather than individual 2q hardware gates.
* a user might wish to Pauli twirl around a block of instructions, including idling wires, rather than a single one.
* a block of instructions might be assigned some duration as part of a larger circuit, and internally it is scheduled with stretchable durations to implement optimal dynamic decoupling spacing without increasing the complexity of the outer scheduling problem.

Several downstream projects of Qiskit have made ad-hoc blocks by using labelled `Barrier` instructions, but this optimisation barrier has further effects than simple grouping, and the single-sided nature of it makes it very hard to work with as an actual "grouping" construct.
`Barrier`s also are hard for users to interpret when inspecting circuits after compilation.

The RFC proposes to add a new `Box` instruction to Qiskit, whose semantics will be very similar to the `box`
[concept from OpenQASM 3](https://openqasm.com/language/scope.html#block-scope).
This `Box` should be able to pass through the Qiskit compiler (under certain limitations), be QPY serialisable, and be able to be transmitted up and down execution stacks.
We will examine how this `Box` might also permit downstream "annotations", outside of the control of the Qiskit SDK compiler, and how the compiler will still be able to reason about such annotations.

The nearest-term use cases for this are:

* have a grouping construction to simplify stretch-based scheduling in Qiskit SDK
* allow advanced users to specify the noise-learning and twirling blocks IBM Primitives error mitigation should use, and the per-block strategy
* allow regular users to see what blocks the IBM Primitives divided their circuit into for noise learning and twirling


## `Box`: a one-off grouping of instructions

As in OpenQASM 3, a `Box` will be:

* a group of non-overlapping (but potentially nested) circuit instructions,
* which introduces a new scope for declared variables,
* which can have internal scheduling within itself, but is scheduled atomically within the circuit (i.e. all wires touched within the box are delayed to match the critical path length),
* which forbids optimisations from crossing from outside to inside the box or vice-versa, but permits optimisations across the entire box,
* and can have arbitrary backend- and SDK-agnostic annotations attached to it.

Aside from variable scoping concerns and the resolution of delay lengths, the execution of a circuit containing a `box` should be the same as the execution of the same circuit with the boxed instructions inlined into the circuit.

### How to implement in Qiskit

A `box` can be seen as the trivial case of a control-flow operation, equivalent to an `if (true)` block in a control-flow graph.
Box, however, additionally has the semantics that is not a valid optimisation to simply remove the box.
A `box` is a non-reusable grouping of instructions; it is not a function call that can be called multiple times (this is still a useful concept, it's just separate to `box` and not addressed here).

With this in mind, the most straightforward path to implementation in Qiskit SDK is to make `Box` a `ControlFlowOp`.
Almost all of the desired semantics will automatically be inherited, and this fits in with existing objects, rather than requiring new special casing paths.

Some additional implications of this:

* Aer will need to be taught to "unwrap" `box` instructions as part of its control-flow handling.
  This should be straightforwards.
* Hardware vendors may need to tell Qiskit they understand `box`, similarly to how they put `IfElseOp` in their `Target` instances
* Qiskit's Sabre layout and routing may want to relax its requirement that a control-flow scope has the same layout at the start and end; for single scopes that dominate all following basic blocks, this is unnecessary and can cause performance problems.
* Qiskit's `qiskit.circuit.CONTROL_FLOW_OP_NAMES` will need updating.

The actual implementation of the object looks something like

```python
import typing
from qiskit.circuit.controlflow import ControlFlowOp

@typing.Final
class Box(ControlFlowOp):
    def __init__(self, circuit, annotations):
        super().__init__("box", circuit.num_qubits, circuit.num_clbits, [circuit])
        self.annotations = annotations

    @property
    def blocks(self):
        return (self.params[0],)

    def replace_blocks(self, blocks):
        assert len(blocks) == 1
        return Box(blocks[0], self.annotations.copy())
```

See below for more considerations about the ``annotations`` parameter.

This then implies that Qiskit should implement the control-flow builder interfaces for `box`.
Again, using some dummy "annotations" (which are not specified by this RFC, and are given for illustrative purposes only):
```python
from qiskit.circuit import QuantumCircuit

# Hypothetical core-Qiskit box-level annotation.
from qiskit.transpiler.contexts import ScheduleStrategy
# Hypothetical annotations for IBM-specific primitives.
from qiskit_ibm_runtime.contexts import TwirlingStrategy

qc = QuantumCircuit(5, 5)
with qc.box([ScheduleStrategy.ALAP, TwirlingStrategy(...)]):
    qc.h(0)
    qc.cx(0, 1)
    qc.x(2)
with qc.box([ScheduleStrategy.ASAP, TwirlingStrategy(...)]):
    qc.h(2)
    qc.cx(2, 3)
    qc.x(4)
qc.measure(qc.qubits, qc.clbits)
```

For `box` in particular (but also for other control-flow operations), it can be useful to mark certain qubits and other circuit resources as being explicitly "used" by a block, even if the actual operation is a no-op.
For example, twirling and noise learning may want to consider certain qubits along with a complete block, even if that qubit is undergoing the logical identity for the span.
This can be worked around by inserting explicit `id` gates into the box, or some other form of no-op operation, but these instructions can sometimes be treated by hardware as something other than a no-op, which might cause performance problems if the compiler fails to optimise them out or understand their intention.
For example, IBM hardware has, in the past, treated the `id` instruction as a single-cycle delay on a single qubit, which is not precisely equal to a no-op for the purposes of scheduling, especially if the rest of the block has a natural duration of zero cycles (such as if it comprises only virtual phase update `rz` gates).
We also do not currently have the concept of a no-op on classical data.
To make the user intent clearer here, we propose adding a `QuantumCircuit.noop` to make the intent explicit:

```python
from qiskit.circuit import QuantumCircuit
qc = QuantumCircuit(3, 3)
with qc.box():
    # This `box` will be full-width on the circuit, despite every qubit and clbit undergoing the identity.
    qc.noop(*qc.qubits, *qc.clbits)
```

The `noop` instruction will do the standard Qiskit circuit error checking to validate its inputs are valid resources for the circuit, but will not cause a new entry in the instruction list.
The control-flow builders will still pick this up as a "used" resource.


### Transpiler semantics of `box`

The transpiler must fail if the backend reports that it cannot handle the `box` instruction, as for other control-flow operations.
In the absence of any annotations, the transpiler must retain any boxes in the input circuit, and the content of the box must be transpiled up to the same equivalent and hardware-compatibility guarantees that the rest of the tranpsiler guarantees, as if the boxed content was a stand-alone circuit (e.g. a unitary operation must have the same action before and after compilation, up to layout and routing effects, and all gates must be ISA supported).

Transpiler passes are permitted to insert new boxes around instruction groups.
A `box` may only be removed (and replaced by some representation of its body) by a transpiler pass if the annotations on the `box` permit it (see below).
The default should be to assume that the `box` cannot be removed.


## Annotations on `box`

In the above, we have talked about annotations on the `box`.
This is by analogy to the OpenQASM 3 concept.
In more generic terms, an annotation will be

> a compiler directive applied in a limited scope

For this RFC, we only consider annotations on `box`es, but in the future, this may be expanded to other general instructions.

Annotations may affect how the scope of the body is compiled.
For example, an annotation may mark that one `box` should be scheduled using an ALAP schedule internally, regardless of how the outer scope of the circuit is being scheduled.
Such an annotation may live inside Qiskit SDK.

Other use cases for `box` may have downstream transpiler passes introduce new `box`es with annotations that may be consumed by some later point in their compilation stack.
For example, IBM's primitives implementations may want to create blocks around blocks of instructions that will undergo noise learning as a single entity, and annotated these boxes with identifiers and strategies around the noise learning, for consumption by a lower part of its own stack.
Qiskit SDK must provide a framework such that transpiler passes can function in the presence of annotations that they don't necessarily understand entirely.

> [!NOTE]
> This section is definitely not complete yet; the text of it just represents a starting point for the thinking.

### Semantics of custom

Qiskit SDK may (in the future, not in this RFC) supply some core directives, such as the hypothetical `ScheduleStrategy` one used above, which affect Qiskit's built-in compiler passes.
More generally, these may be defined by any compiler pass author, and the API must be fixed so that other passes know what they may/may not do with an annotated box.

The extremes are:

1. any compiler pass is allowed to entirely ignore any annotation it doesn't understand, provided it leaves the annotation in place for a later pass.
2. a compiler pass is forbidden to act on a box unless it declares that it understands all annotations attached to it.
3. the annotation somehow declares what compiler passes are allowed to run on the box.

Taking point 2 would be unsatisfactory for Qiskit SDK.
Our core transpiler passes can't be aware of all the possible annotations, so custom-annotation authors would either have to do all the compilation work Qiskit usually does for them, or require their users to do that instead.
That's obviously not suitable from an API perspective.

Point 1 is limiting because it means a backend-specific author can't have an annotation that lets the box use backend-specific information without risking another compiler pass mess it up.
For example, consider a future where backends are all error corrected and defined in terms of logical qubits, but a backend wants to use some sort of "verbatim" box to allow temporary and very low-level temporary access to the physical qubits (something akin to inline `__asm__` blocks in gcc/clang-flavoured C).
The core of the idea is that a compiler directive might be either an additive "apply this additional behaviour" marker, or it might be a subtractive "require these temporary restrictions" marker; taking point 1 or point 2 as the default is akin to choose either the former or the latter as the only type of directive permitted, respectively.

Point 3 alone is not great for the same reasons as point 2; an annotation can't know all possible passes.
That said, a limited subset of this might be possible: an annotation might mark itself as "allow me to be ignored" or "require me to be understood".

> [!IMPORTANT]
> This needs some more thought, and potentially we'd want to have a set of "predicates" that must be upheld by transpiler passes acting on the box, for example "allow only _these_ passes to inspect it" or "no special action required".
> We'd probably need helper methods in the pass-manager and pass infrastructure to make it easier to do the right thing in these cases.

We do also have a potential option in that core Qiskit can specify a small number of "special case" annotations, which all passes are required to interpret and respect.
This gets tricky from a backwards compatibility perspective, though - if we add a new one of these special annotations in Qiskit 1.3, say, it might invalidate code that worked happily in Qiskit 1.2.


### Serialisation and deserialisation of annotations

If we allow arbitrary annotations, we reasonably need some way of constructing / deconstructing these, without sending arbitrary code to be executed over the wire - that would be a huge security hole.
This is the problem that QPY solves for circuits.
The general principle is that we can rely on reconstruction on the far side of a network boundary, if the receiver only runs code that it chose to install ahead of time.
`pickle` is not suitable for this.

In this model, QPY would need to be extended in a way that allows the serialisation and deserialisation of these attributes.
Two ways of making this extension would be:

* have a plugin interface to QPY, where installed packages can register additional serialisers / deserialisers for annotations.
* add additional keyword arguments to the `dump`/`load` functions that allow a user to pass in the additional serialisers / deserialisers manually (like how is done for JSON).

Major risks for this in QPY are that both sides of the deserialiser / serialiser would need to have compatible versions of the custom packages available, and that these custom serialisers would need to uphold the same highly stringent backwards compatibility guarantees that QPY supports.
Because of these risks, the keyword-argument form might be more appropriate, so it is very clear at the point of call that QPY is being extended, and that the caller must be certain that they trust the custom serialisation/deserialisation.

The same sort of logic can be used for import/export from OpenQASM 3, but since that's lossy and we certainly can never make the same guarantees about backwards compatibility with that, it's rather less of a security and compatibility concern.
For clarity, OpenQASM 3 already includes the concept of backend-specific annotations, like
```openqasm3
OPENQASM 3.0;
include "stdgates.inc";

@ibm twirl
box {
    cx $0, $1;
}
```

Certain hardware providers may also have direct-access APIs that do not require submission of the job via Qiskit.
Custom serialisers/deserialisers of annotations should be aware of this as well; it is effectively the same problem as serialisation to and from OpenQASM 3.


### Communication of "allowed" annotations from a backend

For hardware instructions, we use the `Target` to represent the ISA of a given QPU.
A `box` annotation remaining on a circuit after Qiskit SDK transpilation implies that the backend will do further processing on the circuit to prepare this for execution.
This can be associated with further processing by the "quantum computer" before final compilation to the QPU.

In IBM's specified terminology, a "quantum computer" is the full execution environment that a user interacts with to submit quantum jobs to, and might perform further circuit processing, error mitigation, etc, as part of passing the circuit down to the control hardware.
A "QPU" is more of a representation of the control hardware itself.

In Qiskit SDK, a `Target` represents a QPU, while we might (but have not necessarily yet done so) associate the `Backend` with what we call the entire "quantum computer".
As an annotation implies further processing by the quantum computer before execution by the QPU, the information on what annotations may be left in the circuit by the transpiler should be drawn from new information specified by the quantum computer, not the particular QPU.

The transpiler should fail if a user or a pass in its pass manager inserts annotations on boxes that the backend does not support, and are not removed by the end of the transpilation.
We need a way to communicate this information from backends to Qiskit SDK.

For example, an advanced user might attempt to compile an `Estimator` pub with explicit IBM-specific markings of how the noise learning should be performed.
The compilation of a pub represents partial compilation for a particular quantum computer, which will in turn perform further processing to turn this into the set of circuits that will actually be run on the QPU, and so Qiskit SDK's transpiler perhaps ought to have a similar specification of what constitutes "valid" output of such a compilation for quantum-computer processing, much like `Target` represents what constitues valid output for QPU processing.

#### Very very rough sketch of how this might motivate a `BackendV3`

> [!NOTE]
> This is a _very vague_ sketch, I'm just trying to make sure we have a clear definition of what a `Target` is, and we don't start having it become so abstract that it is no longer a good model for a QPU.
> This is absolutely not necessary to finalise for this RFC, but I feel like it's worth thinking about this as a joined-up story, since previously we have treated the output of `transpile` as "ready for QPU execution", but annotations stymie that.

If we go this route, this might motivate a `BackendV3` definition which includes information on what annotations should be permitted to remain.
This has some benefits:

* the transpiler can reject the output from malformed pass-manager pipelines, or input circuits where the user put on a custom annotation that the backend does not know how to resolve.  This allows client-side validation of the circuits.
* the QPY API could be extended to retrieve the serialisers/deserialisers from a `backend` object, rather than needing a user to construct it themselves.

If we _truly_ associated the Qiskit object `Backend` with a "quantum computer" (as opposed to `Target`'s QPU), we might even consider an API where the primitives entry points are moved onto the `BackendV3` object, as in:

```python
class BackendV3:
    @property
    @abstractmethod
    def target(self) -> Target: ...
    @abstractmethod
    def allowed_annotations(self) -> AnnotationChecker: ...
    @abstractmethod
    def sample(self, sampler_pub_likes) -> SamplerJob: ...
    @abstractmethod
    def estimate(self, estimator_pub_likes) -> EstimatorJob: ...
```

This might naturally extend to such a `BackendV3` providing the additional information that is needed for Qiskit SDK to safely transpile entire sampler and estimator pubs; the compilation of a pub _also_ implies further processing will be done by a quantum computer, just like an annotation remaining in a single circuit.


## Case study: twirling, noise learning, and mitigation

Twirling is the process of taking a single base circuit and randomly inserting gates from some distribution, composing them with existing gates, such that the unitary implemented by the circuit and its structure are unchanged. When data is averaged over the result of executing many such independent randomizations, this has the effect of modifiying the noise profile of the circuit into a more tractable one, such as removing the coherent parts of the noise model.

There are many choices for how one might choose to perform these randomizations, where the random gates should be inserted, and with what existing gates they should be composed with.
If this process is completely automated by the execution stack, then the user needs to be able to clearly specify how these choices should be made.
This is especially important in the era of utility because as clients start running larger and larger circuits, it becomes apparent that some twirling choices are better than others.
Moreover, when twirling is used in conjunction with mitigation techniques that rely on noise learning, there needs to be a transparent correspondance between how portions of an application circuit are twirled with how those portions are independently characterized by noise learning techniques.

Therefore, one application of boxes and annotations is to let users declare how they would like to perform twirling, for example with respect to how the gates are randomly drawn, where they are inserted, and what they are composed with.
Once a circuit has been decorated with these boxes and annotations, it is a self-contained description of how twirling should take place.

> **_NOTE:_**  In what follows, we attempt to give a flavor of what twirling annotations might look like in `qiskit_ibm_runtime`. They will not be defined by the `qiskit` library itself. The particular syntax and conventions used here are preliminary and do necessarily represent what will come.

```python
from qiskit.circuit import Parameter, QuantumCircuit
from qiskit_ibm_runtime.annotations import PauliTwirl

circuit = QuantumCircuit(6)

# define a twirled layer of CZs
with circuit.box([PauliTwirl(<specifiers>)]):
    # both of these single-qubit gates will be composed with the random 
    # Pauli selections
    circuit.sx(0)
    circuit.rz(Parameter("p"), 0)
    
    circuit.cz(0, 1)
    circuit.cz(2, 3)
    circuit.noop(5)

# define a twirled layer of measurements
with circuit.box([PauliTwirl(<specifiers>)]):
    circuit.measure_all()
```

We envision two main workflows that users will follow once `box`es and annotations become available to specify twirling, one for regular users and another for power users:

1. The workflow for regular users:
    - Users initialize a `QuantumCircuit` without boxes, as they do today.
    - They construct a pass manager that contains a late-stage twirling-decorator pass.
    - The early passes of the pass manager do steps like routing and layout as usual.
    - The twirling-decorator pass can be configured according to various strategies to collect regions of the circuit and surround them by annotated twirling `box`es
    - Any DD passes can be applied after twirling boxes have been added to ensure that they are compatible with the scheduling constraints implicated by the `box`es. Boxes are treated as hard boundaries with regard to the evaluation of idle periods; idle periods are are not additive across `box` boundaries nor can sequences be correlated outside the width of the box. The DD pass can recurse into the boxes, if directed and possible, to perform layer-level DD.
    - They submit their job. 
2. The workflow for power users:
    - Users initialize a `QuantumCircuit` adding boxes manually as they wish.
    - They submit their job.
    - Validation routines can be tied to annotations for client-side checking of compatibility with box contents.

Setting up these two workflows presents notable advantages, such as:
- All of the existing transpiler passes can be asked (at least initially) to simply ignore the boxes, since transpilation is meant to happen before the boxes are generated. 
- Users of the primitives can inspect the boxes before submitting their jobs. This is a big improvement: today, the box generation happens in the server and can be unintuitive.

In addition to supporting the two workflow above, we believe that the following workflow should *not* be supported:

3. The workflow for "lazy" users:
    - Users initialize a `QuantumCircuit` without boxes, as they do today.
    - They apply all of the desired transiler passes, for example to map the circuit to an ISA circuit for the backend that they wish to use.
    - They submit their job.
    - The server creates boxes for the users "behind the scenes."
  
This workflow essentially takes the same steps as are taken today and therefore presents the same disadvantages discussed above, and also violates the principle "don't do hidden transpilation server-side".

Another application of boxes is to let users declare how they would like to perform noise learning and mitigation. To do so, they can assign a unique identifier to a box via a separate UID annotation, and subsequently provide the primitives' options with a map from those identifiers to noise models they wish to attach to the identifier.

```python
circuit = QuantumCircuit(6)

with circuit.box([PauliTwirl(<specifiers>), uuid0 := uuid.create()]):
    circuit.cz(0, 1)
    circuit.cz(2, 3)
    circuit.noop(4)

with circuit.box([PauliTwirl(<specifiers>), uuid1 := uuid.create()]):
    circuit.cz(1, 2)
    circuit.cz(3, 4)
    
with circuit.box([PauliTwirl(<specifiers>)]):
    circuit.measure_all()

estimator = Estimator(backend)

# specify mapping between uuid and noise models that were previously learned
estimator.options.resilience.layer_noise_model = {
    uuid0: noise_model0,
    uuid1: noise_model1,
}
