## Authors
- Donny Greenberg - @dongreenberg
- Ali Javadi - @ajavadia
- Kevin Krsulich - @kdk
- Erick Winston - @ewinston

## Introduction

The `QuantumCircuit` class in Qiskit is inherently tied to `QuantumCircuit`s and `ClassicalRegister`s. 
This is due to the OpenQASM 2.0 spec, which stipulates circuits being defined over registers. However, arguably
in a majority of cases the user needs to define circuits over an integer number of qubits only. 
In recent versions, Qiskit has started to move towards register-less circuits. However, sometimes being
able to use registers is still useful, because it helps the programmer reason about the circuit functionality.

So now both of these options work:

```python
qr_a = QuantumRegister(3, 'a')
qr_b = QuantumRegister(3, 'b')
circuit = QuantumCircuit(qr_a, qr_b)
circuit.h(qr_A)
```
which yields:
```
        ┌───┐
a_0: |0>┤ H ├
        ├───┤
a_1: |0>┤ H ├
        ├───┤
a_2: |0>┤ H ├
        └───┘
b_0: |0>─────
             
b_1: |0>─────
             
b_2: |0>─────
             
```
or
```python
circuit = QuantumCircuit(6)
circuit.h(range(0,3))
```
which yields
```
        ┌───┐
q_0: |0>┤ H ├
        ├───┤
q_1: |0>┤ H ├
        ├───┤
q_2: |0>┤ H ├
        └───┘
q_3: |0>─────
             
q_4: |0>─────
             
q_5: |0>─────
             
```

Note that in the 2nd case, a register is still defined under the hood (register `q`). This was done for backwards
compatibility, since much of Qiskit's internal code uses `circuit.qregs`.

This RFC is for fully migrating to circuits being defined over `Qubit`s and `Clbit`s, and making `QuantumRegister` and
`ClassicalRegister` defined as pointers to circuit bits. The goal is to simplify internal usage of circuits
by making Qubits/Clbits into fundamental building blocks of circuits, and only defining registers when the need arises.
This would also help circuits maintain a canonical ordering of their bits, which is more natural for the datastructures
that define circuits.

## Requirement 1: Still support circuits over registers

Some circuits naturally benefit from a definition of registers. Examples include:
1. Error correction circuits, which are defined over ancilla and data registers
2. Multi-controlled circuits, which might have a control register and a target register
3. Arithmetic circuits, like a full adder, which is defined over to registers representing an integer each.

## Requirement 2: Circuits aware of registers if they exist

One design option is to make circuits only aware of their `Qubit`s/`Clbit`s, and make registers 3rd party pointers into
those. We explicitly rule out this option, since we sometimes want the circuit to be aware of registers too, if they
exist. Usage examples include:
1. Drawing a circuit, and visualizing its registers
2. Exposing a library of circuits, with registers exposed as well for ease of use

## Design
We propose some changes to Terra that will facilitate the following programs, annotated for clarity:

```python
# you can assign circuit qubits to registers upon creation of the circuit.
# These registers are now nothing but lists containing the circuit's qubits
qr_a = QuantumRegister(3, 'a')
qr_b = QuantumRegister(3, 'b')
circuit = QuantumCircuit(qr_a, qr_b)

# this can alternatively (and more explicitly) be written as
qr_a = QuantumRegister(name='a', bits=3*[Qubit])
qr_b = QuantumRegister(name='b', bits=3*[Qubit])
circuit = QuantumCircuit(qr_a, qr_b)

# the same outcome can be achieved by assigning registers after circuit creation
circuit = QuantumCircuit(6)   # a register-less circuit over 6 qubits
print(circuit.qubits)
>> [Qubit(), Qubit(), Qubit(), Qubit(), Qubit(), Qubit()]
print(circuit.qregs)
>> {}

q = circuit.qubits
circuit.mark_register([q[0], q[1], q[2]], 'a')  # now tag the bits as an 'a' and 'b' register
circuit.mark_register([q[2], q[3], q[4]], 'b')  # (this info will be kept in circuit)
print(circuit.qregs)
>> {'a': QuantumRegister(3, 'a'),
    'b': QuantumRegister(3, 'b')}

circuit.mark_register([q[0]], 'c')  # marking a bit as belonging to more than one register is disallowed
>> raise CircuitError
```

## Circuit Libraries
Circuits are exposed in libraries to the user. These circuits can be register-less, or be defined over
registers where it makes sense. Below is an example of each:

1. Quantum Volume Circuit
```python
from qiskit.circuit.library import QuantumVolume
qv = QuantumVolume(width=3, depth=3)
print(qv)
```
This circuit does not have any registers, and is drawn as such. One can continue to modify the circuit by referencing
the bits themselves.
```
            ┌──────────┐             ░ ┌─┐
────────────┤0         ├─────────────░─┤M├──────
┌──────────┐│          │┌──────────┐ ░ └╥┘┌─┐
┤1         ├┤  Unitary ├┤1         ├─░──╫─┤M├───
│  Unitary ││          ││  Unitary │ ░  ║ └╥┘┌─┐
┤0         ├┤1         ├┤0         ├─░──╫──╫─┤M├
└──────────┘└──────────┘└──────────┘ ░  ║  ║ └╥┘
════════════════════════════════════════╩══╬══╬═
                                           ║  ║
═══════════════════════════════════════════╩══╬═
                                              ║
══════════════════════════════════════════════╩═
```

2. Full Adder Circuit
```python
from qiskit.circuit.library.arithmetic import FullAdder
one_bit_adder = FullAdder(width=1)
print(one_bit_adder)
```
This circuit has been built using registers, and it exists in the library with those registers.
```
                                 
a_0:  ──■────■──────────────■──
        │  ┌─┴─┐          ┌─┴─┐
b_0:  ──■──┤ X ├──■────■──┤ X ├
        │  └───┘  │  ┌─┴─┐└───┘
c_in: ──┼─────────■──┤ X ├─────
      ┌─┴─┐     ┌─┴─┐└───┘     
out:  ┤ X ├─────┤ X ├──────────
      └───┘     └───┘          
```
One can continue to modify this circuit by working with those registers.
```
regs = one_bit_adder.qregs   # a dictionary of {register_name: register} existing on the circuit
qr_out = regs['out']
cr_out = ClassicalRegister(1, 'out')
one_bit_adder.add_register(cr_out)
one_bit_adder.measure(qr_out, cr_out)
```

## Composing Circuits
For consistency with other classes (namely `BaseOperator` and `DAGCircuit`), the `QuantumCircuit` class shall have
a `compose()` method. This will take a keyword argument `front=True/False` which indicates where to append.
```python
init = Initialize(Statevector.from_label('1001'))  # here assuming that initialize is a circuit (currently instruction)
adder = FullAdder(1)
one_plus_one = init.compose(adder)  # shorthand can be init @ adder
# alternatively adder.compose(init, front=True)
print(one_plus_one)
```
Note that the resulting circuit does not have registers anymore, because the LHS registers take precedence, and RHS registers
are forgotten.
```
┌───┐                          
┤ X ├──■────■──────────────■───
└───┘  │  ┌─┴─┐          ┌─┴─┐ 
───────■──┤ X ├──■────■──┤ X ├─
       │  └───┘  │  ┌─┴─┐└───┘ 
───────┼─────────■──┤ X ├──────
┌───┐┌─┴─┐     ┌─┴─┐└───┘      
┤ X ├┤ X ├─────┤ X ├───────────
└───┘└───┘     └───┘           
```

## Open Questions
1. Is there a need for a qubit/clbit belonging to multiple registers (currently disallowed)?
2. Is there a need to mark the output of a circuit with different registers? In the full adder example, the input
registers are `a`/`b`/`c_in`, and the output registers are `a`/`b`/`sum`/`c_out`.
