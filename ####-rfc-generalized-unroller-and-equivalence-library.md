# Generalized Unroller and Equivalence Library

| **Status**        | **Proposed**                                 |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Kevin Krsulich (kevin.krsulich@ibm.com),     |
|                   | Luciano Bello (Luciano.Bello@ibm.com),       |
|                   | Ali Javadi (Ali.Javadi@ibm.com),             |
| **Deprecates**    |                                              |
| **Submitted**     | YYYY-MM-DD                                   |
| **Updated**       | YYYY-MM-DD                                   |


# Summary

We propose a generalization of the Qiskit transpiler's unroller and gate definition infrastructure to enable circuit translation across a variety of bases.
We define an interface for an equivalence library which will be used to expose a repository of valid circuit translations, initially defined over the standard library and automatically integrating decompositions of user-defined gates.
Implementation of this library within the transpiler pipeline will support natively targeting a broader array of architectures without requiring re-synthesis, and enable basis-dependent logical optimizations.


# Motivation

Cross-architecture compatibility is a key architectural goal of the Qiskit transpiler.
To date, practical opportunities to pursue this goal have been few as available targets have been predominantly superconducting circuit devices and simulators.
As a result, the transpiler has, in several places, developed implicit assumptions on or preferences for superconducting backends.
As the number of deployed devices of varied architecture rapidly increases, ensuring that the Qiskit transpiler is able to achieve state-of-the-art optimizations while targeting diverse backends is a pressing practical goal.

Immediate objectives are to ensure that the majority of the transpiler pipeline is target-architecture agnostic and that the tooling provided is equally useful across architectures.
Logical optimization passes should be applicable regardless of target architecture, and device-level optimization passes should be applicable for any backend which reports the necessary quantities.
High level optimizations should be applied broadly, and architecture-specific optimization passes employing the strengths of a particular architecture should be easy to develop and utilize.
The limitation that `Instruction` s are allowed only a single decomposition, which must ultimately be reducible to `U3` and `CX` gates, will be lifted.
For devices which report an overcomplete basis, users will be able to control how instructions will be decomposed, or to provide specific decompositions of single instruction.

At a high level, current transpiler levels 0 and 1 are roughly summarized as follows:

    ┌───────────────┐                                                                                     
    │ Current Terra │░                                                                                    
    │  Transpiler   │░                                                                                    
    └───────────────┘░                                                                                    
     ░░░░░░░░░░░░░░░░░                                                                                    
               ┌──────────────┐                     ┌────────────────┐                ┌────────────────┐  
               │ User Circuit │                     │Embedded Circuit│                │ Device Circuit │  
               ├──────────────┤                     ├────────────────┤                ├────────────────┤  
               │custom        │                     │custom          │                │device gates*   │  
      Input    │built-in      │     Embedding       │built-in        │   Unrolling    │                │  
        ●──────│unitary gates │────────────────────▶│unitary gates   │───────────────▶│                │  
               │              │   LayoutSelection   │                │   UnrollU3CX   │                │  
               ├──────────────┤  AncillaAllocation  ├────────────────┤ Unitary Synth  ├────────────────┤  
               │virtual qubits│                     │ device qubits  │                │ device qubits  │  
               └──────────────┘                     └────────────────┘                └────────────────┘  
                                                                                               │          
                                                                                               │          
                                                                                               │          
                             ┌─────────────────┐                      ┌────────────────┐       │          
                             │Optimized Circuit│                      │Runnable Circuit│       │          
                             ├─────────────────┤ ┌ ─ ─ ─ ─ ─ ─ ─ ─ ┐  ├────────────────┤       │          
                   Device    │device gates     │     Optimization     │device gates    │       │   Mapping
      Output    Re-synthesis │                 │ │       Loop      │  │                │       │          
         ●───────────────────│                 │◀─────────────────────│                │◀──────┘          
                             │                 │ │ Optimize1QGates │  │                │                  
                             ├─────────────────┤   CXCancellation     ├────────────────┤                  
                             │  device qubits  │ │ConsolidateBlocks│  │ device qubits  │                  
                             └─────────────────┘         ...          └────────────────┘                  
                                                 └ ─ ─ ─ ─ ─ ─ ─ ─ ┘                                      

Levels 2 and 3 are similar, with expanded passes in the optimization loop and the order of embedding and unrolling reversed.

A transpiler architecture of the following form is envisioned:

    ┌───────────────┐                                                                                                                   
    │  Envisioned   │░                                                                                                                  
    │     Terra     │░                                                                      High-Level Logical                          
    └───────────────┘░                                                                       Optimization Loop                          
     ░░░░░░░░░░░░░░░░░                                                                                                                  
                                                                                             ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─                
               ┌──────────────┐               ┌────────────────┐            ┌────────────────┐  Unroll to               │               
               │ User Circuit │               │                │            │                │ optimization                             
               ├──────────────┤               ├────────────────┤            ├────────────────┤    basis                 │               
               │custom        │ Unroll Custom │built-in        │ Synthesize │built-in gates  ├─────────────■                            
      Input    │built-in      │     Gates     │unitary gates   │ Unitaries  │                │             │    Apply   │               
        ●──────│unitary gates │──────────────▶│                │───────────▶│                │             │optimization                
               │              │  UnrollUser   │                │            │                ◀─────────────■            │               
               ├──────────────┤               ├────────────────┤            ├────────────────┤  LLL Opt.                                
               │virtual qubits│               │ virtual qubits │            │ virtual qubits │    Loop                  │               
               └──────────────┘               └────────────────┘            └────────────────┤                                          
                                                                                     │        ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ Template matching     
                                                                            Embedding└───────────┐              Peephole optimization   
                                                                                                 ▼              Clifford resynthesis    
                        ┌────────────────┐                  ┌────────────────┐           ┌────────────────┐      Block consolidation    
                        │ Device Circuit │                  │ Device Circuit │           │ Device Circuit │  FinalToffoliToRelativePhase
                        ├────────────────┤                  ├────────────────┤           ├────────────────┤              ...            
                        │device gates    │       Device     │device gates    │           │device gates*   │                             
      Output            │                │   Optimizations  │                │ Mapping   │                │                             
         ●──────────────│                │◀─────────────────│                │◀──────────│                │                             
                        │                │  Device Unroller │                │           │                │                             
                        ├────────────────┤  Approx Synthesis├────────────────┤           ├────────────────┤                             
                        │ device qubits  │                  │ device qubits  │           │ device qubits  │                             
                        ├───┬──────▲─────┘─ ─ ─ ─ ─ ─ ─     └────────────────┘           └────────────────┘                             
                            │      │  Optimize1QGates  │                                                                                
      Low-Level Logical │   │      │  CXCancellation                                                                                    
      Optimization Loop     │      │        ...        │                                                                                
                        │   ■──────■                                                                                                    
                                                       │                                                                                
                        └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─                                                                                 

Low-level logical optimizations will in general be more restricted than high-level optimizations.
For example, they must be executable without a basis change on any circuit defined over any complete basis from the standard gates.

This approach allows the high-level logical optimizations which are expected to bring the most substantial performance improvements to be utilized broadly and made distinct from the architecture-specific optimization passes intended to optimize to a given architecture's strengths.


# Guide-level explanation

Before being executed on a device, a `QuantumCircuit` will be transformed through several representations, including the gate set over which the circuit is expressed.
Each backend supports a fixed gate set, so instructions will need to be translated into an equivalent form in the device's native gate set.
Similarly, some transpiler optimizations may only operate on certain gate sets.
Terra provides the necessary tooling to automatically apply these translations, as well as ways to inspect and update the set of valid translations.

Even when comparing between complete bases, not all gate sets are equivalent in terms of expressiveness.
Applying a given translation will in general yield a circuit with a different gate count and depth.
Practically, within the context of the transpiler, these translations will be followed by optimization passes which will reduce any incurred overhead.


## `qiskit.circuit.EquivalenceLibrary`

The `EquivalenceLibrary` interface allows for recording and retrieval of interchangeable sets of instructions.
It is used by transpiler developers to store and retrieve identities, or equivalent replacements.
Equivalent here means unitarily equivalent, *not* equivalence up to a global phase or up to a permutation (which could be introduced later).

Users can easily access the decompositions for a given gate via the `.decompositions` property.
Previously, gates stored only a single definition in their `.definition` property, which has been deprecated.


## Examples


### Setting and fetching gate decompositions via properties

    from qiskit.extensions.standard import CnotGate
    
    cx = CnotGate()
    print(cx.definition)

    None

    for entry in cx.decompositions:
      print(entry)

                         
    q_0: |0>──────■──────
            ┌───┐ │ ┌───┐
    q_1: |0>┤ H ├─■─┤ H ├
            └───┘   └───┘

    from qiskit.extensions.standard import CzGate
    cz = CzGate()
    for gate, qargs, cargs in cz.definition:
      print(gate.name, qargs, cargs)

    h [Qubit(QuantumRegister(2, 'q'), 1)] []
    cx [Qubit(QuantumRegister(2, 'q'), 0), Qubit(QuantumRegister(2, 'q'), 1)] []
    h [Qubit(QuantumRegister(2, 'q'), 1)] []

    for entry in cz.decompositions:
      print(entry)

                           
    q_0: |0>───────■───────
            ┌───┐┌─┴─┐┌───┐
    q_1: |0>┤ H ├┤ X ├┤ H ├
            └───┘└───┘└───┘


### Adding user-definition to standard gate

    from qiskit.extensions.standard import HGate
    
    for entry in HGate().decompositions:
      print(entry)

            ┌──────────┐
    q_0: |0>┤ U2(0,pi) ├
            └──────────┘

    from qiskit.circuit import Gate, QuantumCircuit
    from qiskit.extensions.standard import HGate
    
    my_gate = Gate('special_h', 1, [])
    qc = QuantumCircuit(1)
    qc.append(my_gate, [0])
    
    HGate().decompositions += [qc]
    
    for entry in HGate().decompositions:
      print(entry)

            ┌──────────┐
    q_0: |0>┤ U2(0,pi) ├
            └──────────┘
            ┌───────────┐
    q_0: |0>┤ Special_h ├
            └───────────┘


### Decomposing Toffoli/GHZ cross-architecture

### Expanding device basis sets with cover h,rx,p

### Tree traversal, CX->CZ->iSWAP


# Detailed Design


This proposal introduces two new concepts, that of the `EquivalenceLibrary` and the `BasisTranslator`.

## `qiskit.circuit.EquivalenceLibrary`

    print(pydoc.render_doc(qk.circuit.EquivalenceLibrary, renderer=pydoc.plaintext))

    Python Library Documentation: class EquivalenceLibrary in module qiskit.circuit.equivalence
    
    class EquivalenceLibrary(builtins.object)
     |  A library storing equivalence translation rules.
     |  
     |  Methods defined here:
     |  
     |  __init__(self, *, base=None)
     |      Create a new equivalence library.
     |      
     |      Args:
     |          base - Optional[CircuitEquivalenceLibrary]: Base equivalence library
     |              which will be referenced if an entry is not found in this library.
     |  
     |  add_entry(self, gate, equivalent_circuit)
     |      Add one new equivalence definition to the library.
     |      
     |      Will be added to all existing equalilities (including base).
     |      
     |      Args:
     |          gate - Gate: \ldots
     |          equivalent_circuit - QuantumCircuit: \ldots
     |  
     |  draw_basis_graph(self)
     |  
     |  get_entry(self, gate)
     |      Get
     |      
     |      Args:
     |          gate - Gate: \ldots
     |      
     |      Returns: List[qc], if empty list, library contains no known decompositions
     |  
     |  set_entry(self, gate, entry)
     |      Set 
     |      
     |      Will override existing definitions.
     |      
     |      Args:
     |          gate - Gate: \ldots
     |          entry - List[QuantumCircuit]: \ldots
     |  
     |  ----------------------------------------------------------------------
     |  Data descriptors defined here:
     |  
     |  __dict__
     |      dictionary for instance variables (if defined)
     |  
     |  __weakref__
     |      list of weak references to the object (if defined)

The `EquivalenceLibrary` defines an interface for declaring equivalence between an `Instruction` and one or more `QuantumCircuit`s.
No specific equivalence measure is defined nor enforced by the library, though the transpiler will expect that any defined equivalence can be used as a substitution without impacting the correctness of the circuit.

Two `EquivalenceLibrary` instances will be automatically instantiated during each Qiskit session:

-   the `StandardEquivalenceLibrary` will be pre-constructed for the gates in the standard library,
-   and a `SessionEquivalenceLibrary` will be constructed and will be the source and target of definitions for user-defined gates.

`EquivalenceLibrary`s can be singly nested, so derived libraries layer modifications on top of an existing library without affecting the original.

The set of circuits returned in an equivalence entry by the library does not have a defined order, the library consumer should inspect the set to find a desired circuit,.

If an input gate is parameterized by a set of `qiskit.circuit.Parameters`, the returned circuit will be parameterized by the same `Parameter` set.

A consequence of this approach is that setting custom definitions on individual `Instruction` instances will no longer be supported.
In the future, this can be replaced by incorporating `EquivalenceLibrary` entries to be tied to an `Instruction` label or instance.


## Example: Fetching standard definition

    from qiskit.extensions.standard import CzGate, CnotGate, ToffoliGate
    from qiskit.extensions.standard import StandardEquivalenceLibrary as sel
    
    cz_entry = sel.get_entry(CzGate())
    cz_entry[0].draw()
    
    sel.get_entry(ToffoliGate())[0].draw(output='mpl')

## `qiskit.transpiler.passes.BasisTranslator`

As a replacement for the current `Unroller` pass, we introduce a `BasisTranslator` pass which utilizes a provided `EquivalenceLibrary` to find a set of translations to map from a source basis to a target basis via a bramble-search algorithm, as compared to a tree-descent search in the existing `Unroller`.

# Alternative Approaches

# Unresolved questions

Gates which are defined over a variable number of qubits, such as the Mølmer–Sørensen gate, may require additional handling in the `EquivalenceLibrary` interface.
Entries could be defined as callables rather than circuits, either generally or specifically for variadic gates, though this may hinder the ability of transpiler passes to inspect available decompositions and intelligently choose among them.
Storing one circuit entry per gate width in the library is another option, growing the library size proportional to the number of variadic gates and their maximum widths.

The `Decompose` transpile pass and corresponding `QuantumCircuit.decompose` method depend on a single unique path through `Instruction.definition`.
These methods could be deprecated and removed, maintained via a decomposition search targeting `U3` and `CX`, or maintained via additional library entry labeling to support identification of one entry as the legacy `definition`.

An efficient means of serializing and reconstructing an `EquivalenceLibrary` will be necessary, both for initially distributing and constructing the `StandardEquivalenceLibrary`, and for users to store, transmit and re-use custom definitions.


# Future posibilities

Following the implementation of this proposal, possible future steps include expanding user control over the scope of decomposition application to a specific `Instruction` label or instance.
Additionally, the `EquivalenceLibrary`'s matching procedures can be enhanced to support fixed parameter simplifications (e.g. `U3(pi/2, 0, pi)` to `H`) and circuit to circuit equivalence.
Envisioned future transpiler passes include noise-aware unrolling, and ancilla-aware unrolling with active ancilla management
