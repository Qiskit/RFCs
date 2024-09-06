# Introducing `Block`s in qiskit

| **Status**        | **Proposed** |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0022                                         |
| **Authors**       | [Sam Ferracin](sam.ferracin@ibm.com), [Ian Hincks](ian.hincks@ibm.com), [Chris Wood](cjwood@ibm.com), [Jake Lishman](jake.lishman@ibm.com), [Joshua Skanes-Norman](joshua.sn@ibm.com)    |
| **Submitted**     | YYYY-MM-DD                                   |

## Summary
Objective: Introduce a new `Block` instruction to enhance the user experience with Qiskit's Runtime primitives, with the goal of providing more flexibility and transparency with respect to twirling and mitigation.

The concept of a block of gates is fundamental to many quantum computing routines. For example, a block is a natural concept when reasoning about quantum errors because the error profile often depends on factors such as which gates are applied simultaneously. However, Qiskit does not provide a dedicated object to represent a block. Traditionally, users have worked around this by defining blocks *indirectly* using barriers. While this approach has been adequate for tasks like scheduling and transpilation, barriers can be hard to parse when pre-processing circuits in preparation for twirling or mitigation. Suboptimal handling of barriers in these post-processing steps can introduce significant slowdowns in some mitigation experiments (for example, it can unnecessarily increase the time required by the noise learning steps) and produce outcomes that may seem unintuitive to typical users.

To overcome these issues, we propose to establish the concept of a block of gates through the introduction of a new `Block` class. The primary goal of this class is to encapsulate an isolated block of `CircuitInstruction`s, treating it as a single unit for tasks like twirling and mitigation. We believe that `Block`s will offer:
* Full transparency: 
  * Users will be able to learn how their circuits are broken into blocks, twirled, and mitigated *before* running a job
  (today, they can only find this out once the job is done).
* Full flexibility: 
  * Users will be able to draw blocks around every combination of instructions that they want to be grouped together for the purpose of twirling or mitigation (today, all blocks are effectively layers, i.e., depth-one blocks).
* Ability to specify a twirling context:
  * Users will be able to define a twirling group such as "Pauli" and a twirling strategy such as "active-accum" that can potentially be different for each block (today, this can only be done at the circuit level).
* Ability to assign a noise handle to each block that uniquely specifies its noise model (today, this requires hashing circuits and is inherently unreliable, since circuits are mutable objects).

## `Block`: Definition and use cases

A `Block` represents an isolated block of `CircuitInstruction`s that is treated as a single unit in the context of twirling and mitigation. In more detail, when a circuit contains blocks:
- Twirling targets the blocks and ignores all the circuit instructions that live outside of the blocks.
- Noise learning targets the blocks and ignores all the circuit instructions that live outside of the blocks.
- Gate mitigation (e.g. PEC and PEA) targets the blocks and ignores all the circuit instructions that live outside of the blocks.

We envision two main workflows that users will follow once `Block`s become available, one for regular users and another one for power users:
1. The workflow for regular users:
    - Users initialize a `QuantumCircuit` without blocks, as they do today.
    - They apply all of the desired transiler passes, for example to map the circuit to an ISA circuit for the backend that they wish to use.
    - They use (a convenience method built around) a new transpiler pass that collects circuit's gates into blocks, with the ability of specifying different collection strategies.
    - They submit their job.
2. The workflow for power users:
    - Users initialize a `QuantumCircuit` adding blocks manually as they wish.
    - They submit their job.

Setting up these two workflows presents notable advantages, such as:
- All of the existing transpiler passes can be asked (at least initially) to simply ignore the blocks, since transpilation is meant to happen before the blocks are generated. This can save quite a lot of time that would otherwise be spent adding logic to every single transpiler pass -- and yet, we can choose to do this in the future.
- Users of the primitives can inspect the blocks before submitting their jobs. This is a big improvement: today, the block generation happens in the server and can be unintuitive (see the example in the background section), and users have to wait for the job to be over before they can find out how the circuit was broken into blocks.

In addition to supporting the two workflow above, we believe that the following workflow should *not* be supported:

3. The workflow for "lazy" users:
    - Users initialize a `QuantumCircuit` without blocks, as they do today.
    - They apply all of the desired transiler passes, for example to map the circuit to an ISA circuit for the backend that they wish to use.
    - They submit their job.
    - The server creates blocks for the users "behind the scenes."

This workflow essentially takes the same steps as are taken today and therefore presents the same disadvantages discussed above, and also violates the principle "don't do hidden transpilation server-side".

Having described motivation and use cases for circuit blocks, we can now provide details regarding their implementation.

## `Block` as an `Instruction`
In this implementation, `Block` are `Instruction` objects that can be initialized from a circuit.
```python
class Block(Instruction):
    """A container for an isolated block of operations in a larger circuit.
    
    Args:
        circuit: A quantum circuit containing all the operations in the block.
        twirling_strategy: Some specification of a twirling group (such as "pauli")
            and of a twirling mode (such as "active-accum") to apply to this block.
        noise_handle: A mapping object that maps a unique identifier of this block to a noise model.
        label: A label.
    """

    def __init__(self, circuit: QuantumCircuit, twirling_strategy, noise_handle, label: str = ""):
        self._circuit = circuit
        self._twirling_strategy = twirling_strategy
        self._noise_handle = noise_handle

        super().__init__(
            "block",
            self.circuit.num_qubits,
            self.circuit.num_clbits,
            params=self.circuit.parameters,
            label=label,
        )
```

Being `Instruction`s, they can be appended to `QuantumCircuit`s simply by specifying the lists of `qargs` and `clargs`, for example via a dedicated `block` function wrapping the `append` method of `QuantumCircuit`:
```python
class QuantumCircuit:
    ...

    def block(
        self, 
        circuit: QuantumCircuit,
        ...,  # the other inputs required by `Block`s
        qargs: Optional[list[QubitSpecifier]] = None,
        cargs: Sequence[ClbitSpecifier] | None = None,
    ) -> InstructionSet:
        from .block import Block

        # These use a `dict` not a `set` to guarantee a deterministic order to the arguments.
        qargs = tuple(
            {q: None for qarg in qargs for q in self._qbit_argument_conversion(qarg)}
        )
        cargs = tuple(
            {b: None for carg in cargs for b in self._clbit_argument_conversion(carg)}
        )

        return self.append(Block(circuit, ...), qargs, cargs)
```

The logic in the `block` function above returns an `InstructionSet` containing a single `CircuitInstruction`, with the block as `operation` and the correct register's slices as `qubits` and `clbits`:
```python
block = QuantumCircuit(2)
block.cx(1, 0)

qreg1 = QuantumRegister(3, "qreg1")
qreg2 = QuantumRegister(3, "qreg2")

circuit = QuantumCircuit(qreg1, qreg2)
circuit.s(qreg1[1])
circuit.h(qreg2[1])
my_block = circuit.block(block, [1, 5])
# equivalent to: my_block = circuit.block(block, [qreg1[1], qreg2[2]])

print(my_block[0])
>>> "CircuitInstruction(operation=Instruction(name='block', num_qubits=2, num_clbits=0, params=[]), qubits=(Qubit(QuantumRegister(3, 'qreg1'), 1), Qubit(QuantumRegister(3, 'qreg2'), 2)), clbits=())"

print(circuit.draw())
>>> qreg1_0: ────────────────
>>>          ┌───┐┌────────┐
>>> qreg1_1: ┤ S ├┤0       ├─
>>>          └───┘│        │
>>> qreg1_2: ─────┤        ├─
>>>               │        │
>>> qreg2_0: ─────┤  Block ├─
>>>          ┌───┐│        │
>>> qreg2_1: ┤ H ├┤        ├─
>>>          └───┘│        │
>>> qreg2_2: ─────┤1       ├─
>>>               └────────┘
```

## Outstanding questions:
1. In the existing implementation of blocks that exists in `pec-runtime`, blocks are hashable, so that they can be used as keys in dictionaries (specifically in the dictionary porduced by the `NoiseLearner` that maps blocks to noise models). This is clearly unsafe since circuits are mutable. How can we design the noise handle so that it is safer than that, but also more flexible (e.g., if users want to mutate the block but still assign the same noise to it during mitigation, they should be allowed to do so)?
2. Should `Block` be added to qiskit core, alongside the transpiler pass groups gates in blocks workflow 1? If so, should they be written in Rust or Python?
3. What packages and subpackages need to be modified once `Block` is introduced? For example, should we add logic to qiskit-aer to be able to simulate circuits with blocks, or to Qiskit's optimizer, ..?
4. How can we (should we?) facilitate re-use of CircuitInstruction or Block instances? For example, suppose we want the same block in a circuit 100 times:
```python
for _ in range(100):
    with circuit.block(context=my_context, name="foo") as block:
        block.h(0) 
        block.cx(0, 1)
```
