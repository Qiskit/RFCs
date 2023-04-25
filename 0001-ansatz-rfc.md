# Aqua 0.7: Ansatz Design Doc

| **Status**        | **Accepted** |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0001                                         |
| **Authors**       | Julien Gacon (jul@zurich.ibm.com)            |
| **Submitted**     | 2020-01-06                                   |
| **Updated**       | 2020-01-24                                   |

# Overview

1. [Motivation](#motivation)
1. [User benefit](#benefit)
1. [Proposed structure](#structure)
1. [Implementation](#implementation)
    1. [The `Ansatz` object](#ansatz)
    1. [Variational forms](#varforms)
    1. [Feature maps](#featmaps)
    1. Others

# Motivation<a id="motivation"></a>

## Creating a new variational form

Currently, creating a new variational form class requires a lot of copy+paste. 

Remember: `RYRZ` has all of that, too!


```python
class RY(VariationalForm):
    """Layers of Y rotations followed by entangling gates."""

    def __init__(self, num_qubits, depth=3, entangler_map=None, entanglement='full', initial_state=None,
                 entanglement_gate='cz', skip_unentangled_qubits=False, skip_final_ry=False):
        self.validate(locals())
        super().__init__()
        """ 30 more lines ... """
        self._bounds = [(-np.pi, np.pi)] * self._num_parameters
        self._support_parameterized_circuit = True

    def construct_circuit(self, parameters, q=None):
        if len(parameters) != self._num_parameters:
            raise ValueError('The number of parameters has to be {}'.format(self._num_parameters))
        """ 39 more lines ... """
        return circuit
```

Setting up a new variational form should be quick and easy, more like:


```python
class RY(TwoQubitAnsatz):
    def __init__(self, ...):
        super.__init__(single_qubit_gate='ry', ...)
```

If a user wants to quickly try a new format, it would convenient to set up the variational form in just a single line. Not all ideas must become a complete, new class.


```python
rxrycx = MultiQubitAnsatz(num_qubits, single_qubit_gate=['rx', 'ry'], two_qubit_gate='cx', depth=4)
```

## Adding and inserting layers

The circuits that current variational forms build are not extensible. For algorithms, such as adaptive VQE, adding new layers to the variational forms is necessary. 


```python
varform.add(additional_gate, qubit_indices)  # add `additional_gate` at `qubit_indices`
```

Other applications require the insertion of a new gate or layer not at the end of the variational form but somewhere in the middle. The position could e.g. be handled by the appearance of a certain `Parameter` instance.

This can be used for the calculation of analytic gradients. By inserting a set of Paulis after a parameter and controlling the operation on an ancilla, we can compute the gradient with respect to that parameter (see [this paper](https://arxiv.org/pdf/1701.01450.pdf)).


```python
target_param = Parameter('θ')  # this variable is also used in the circuit 
varform.insert(target_param, instr_to_insert, qubit_indices)
```

The insertion might be difficult to implement, what would the `position` be? This feature might be too specific and implementation of special cases may be left up to the users.

## Combining variational forms

A variational form is essentially a gate preparing a parameterized state. For chemistry, this could be the preparation of certain excitation. If we want multiple excitations, we should be able to just combine the respective variational forms.


```python
S0, S1 = single_excitation('θ0'), single_excitation('θ1')
D = double_excitation('θ2')
UCCSD = S0 + S1 + D
```

If some variational forms prepare subsystems, they should be able to be combined at certain qubit indices.


```python
varform = sub_varform.append(other_varform, indices)
```

Possibly allow only `combine` and not `__add__`, since this is dangerously close to `add(new_layer)` and might result in confusion of what types can be added.

## DIY: special variational forms

Variational forms can be very diverse and for special problems, special structures may perform better than others. Therefore, users should be able to their own Ansaetze as variable as possible.
For this we can support automatic repetition (with correct setting of the parameters).

![Automatic repetition](0001-ansatz-rfc/ansatz_rep.jpg)

The repetitions can either be specified by a single integer or layer indices.


```python

```


```python
varform = Ansatz([gate_1, gate_2], [indices_1, indices_2], reps=3)
varform = Ansatz([gate_1, gate_2], [indices_1, indices_2], reps=[0, 1, 0, 1, 0, 1])
```

Instead of gates we can also allow fancier options, e.g. provide a Callable that accepts a repetition number or similar.

What happens to the parameters in the gates if they get repeated? Do they just get repeated? 
If we stored them in a list, the user could later on overwrite them in `construct_circuit`. 

We could also add some default parameters, since the user might not expect there to be the same parameters? If these default parameters haven't been substituted we could raise a warning.

Repeating the parameters is probably the more intuitive solution. If the user wants different parameters, they must be set afterwards -- as it is also the case with default parameters.

## Operator Ansatz 

Several algorithms, such as QAOA, require an Operator-evolution type Ansatz, such as 

$$
V(\theta) = e^{i\theta_1 \hat O_1} e^{i\theta_2 \hat O_2} e^{i \theta_3 \hat O_1} e^{i\theta_4 \hat O_2}\cdots ~.
$$

Currently, this is cumbersome to implement. Similar to the ''DIY'' Ansaetze, we could support an `OperatorAnsatz` based on `Ansatz`:


```python
varform = OperatorAnsatz([Op1, Op2], reps=4)
```

## Unification of circuit-producing concepts

The concept of a variational form is broad. In support of a clear structure, we should group similar concepts. Then,
* users are less confused and see what concepts have the same base,
* we can reuse the same base functionalities,
* we ensure that similar objects in Aqua behave the same way.

Therefore, a common base for feature maps and variational forms would be sensible. Aqua hosts more facilities to construct circuits, as e.g. the `InitialState` or the `UncertaintyModel`, but as discussed towards the end of this doc these are best not derived from Ansatz.

However, it should be our goal that all objects producing circuits in Aqua behave the same way. Ideally, they also feel like objects from Terra. This means finding a similar base for 

* variational forms
* feature maps
* initial states
* uncertainty models
* grover operators
* amplitude estimation $\mathcal A$ operators


```python
class MyFeatureMap(Ansatz):
    # implement
    
class MyMultiQubitVarform(Ansatz):
    # implement
```

## Pre-transpilation & default parameters

Terra provides the `Parameter` class for parameterized circuits. This can speedup the repeated execution of the same circuits with different parameters, as we can pre-transpile the circuit and substitute the parameters afterwards.
This significantly improves our performance since transpiling is costly!

By transpiling the circuit the first time `construct_circuit` is called, the user benefits from pre-transpiling without specifying it manually.


```python
def construct_circuit(self, transpiled=True,  # set to False for untranspiled circuit
                      parameter_prefix=default_prefix)
```

Also, we could simplify the setting of parameters, if the user just quickly wants a circuit to look at.


```python
circuit = varform.construct_circuit()  # uses some default naming 
circuit = varform.construct_circuit(parameter_prefix='θ')
```

## Optional barriers

Barriers can be very useful for visualization, but if a circuit is executed only and not looked at, we don't need them and they can even slow down transpilation. Why insert them by default?


```python
circuit = varform.construct_circuit(insert_barriers=True)  # False by default
```

For this feature we would have to store a `has_barriers` variable and if the user changes back and forth between barriers and no barriers we would have to rebuild the circuit. (Workaround possible by storing both, but back-and-forth changing is not likely anyways.)

# User benefit<a id="benefit"></a>

Let's summarise the benefits of having an `Ansatz` class satisfying the requirements from the motivation.

1. Beginner user
    1. flexibility: can get creative and easily assemble new variational forms
    2. performance: can `construct_circuit` without specifying parameters, and thereby benefits from potential speedup without even touching Terra's `Parameter` class
2. Advanced user
    1. needs less copy+paste to implement standard variational forms
    2. can also benefit from `Parameter` related speedup
    3. has lots of lots of flexibility
        * by adding and combining varforms
        * inserting new layers
        * wrapping parameterized gates
    4. gets less confused by the multitude of Ansatz-like objects in Aqua

# Proposed Structure<a id="structure"></a>

The Ansatz class reconciles all classes that prepare an ansatz or wavefunction. Objects of the Ansatz type share the Ansatz interface and can be used as building blocks in larger circuits and algorithms. This has the key advantage, that the same concepts behave similarly and we avoid code duplication. 

More specialized types, that can also provide an easy setup or special behaviour, are derived from the Ansatz. 

See the next Figure.

![Ansatz design](0001-ansatz-rfc/ansatz_scheme.jpg)

## Unify base classes

Currently, Aqua (6.x) distinguishes between a `VariationalForm` and a `FeatureMap`, which are both abstract classes that specify interfaces. These interfaces, however, almost identical and could be merged. Further, to new users the terms "variational form" and "feature map" might be confusing, whereas the word "Ansatz" wraps both concepts.

Thus, by introducing the `Ansatz` as joint class we can directly derive `PauliExpansion` and `TwoLocalGateAnsatz` without introducing new levels of inheritance, which might be sources of confusion.

## Composition vs. Inheritance

The objects `RY`/`RYRZ` are displayed as objects derived from not `Ansatz` but an intermediate type. This is discussed in the [implementation section](#standard_varforms).

# Implementation <a id='implementation'></a>

To be discussed in more detail.

1. Ansatz
1. Variational forms
    1. TwoLocalGateAnsatz
    1. Standard Forms
1. Feature maps
    1. Pauli expansions
    1. First/Second/... order
1. Initial state objects

Note: We know use the term Ansatz for the concept of a parameterized circuit preparing a parameterized state, not variational form.

## Ansatz <a id='ansatz'></a>

### Requirements recap

1. Take a list of gates and a number of repetitions and construct an Ansatz of desired depth
2. Provide `construct_circuit(params=None, transpiled=True, insert_barriers=False)`. Note, that `params` must support `list[float | Parameter]`.
3. Combine two Ansatz objects at given indices
4. Insert new gates at a specified place
5. Backward compatibility with all previous `VariationalForm` objects

Code checklist:


```python
def __init__(self, layers, indices, reps)

def construct_circuit(params=None, transpiled=True, parameter_prefix=default_prefix, insert_barriers=False)

def __add__(other)  # easy appending of Ansaetze, and maybe for various kinds of RHS/LHS types?

def append(other, connection_indices=None)  # for Ansatz + Ansatz at special indices

def add_layer(gate, indices=None, position=end)  # to add a new gate/QuantumCircuit

def insert_at_param(target_param, gate, indices=None)   # to insert after/before a certain parameter
```

### Implementation proposal #1: `QuantumCircuit`

Make an Ansatz essentially a `QuantumCircuit` object. 

#### Advantages 

* already provides: `__add__(QuantumCircuit/Ansatz)` and `append(Gate)`
* gives the user a Terra-like feel for the Ansatz

#### Technical Issues

* parameter setting by `list[float | Parameter]` not inherently supported
* a circuit doesn't have `construct_circuit` -- but we can work without that
* how to insert barriers or new gates at arbitrary positions? (use `data` member?)

We lose flexibility. What was the initial gate object the user gave us? Missing performance benefit of pre-transpilation.

#### Design Issues

* inserts another level of inheritance (or maybe just use `QuantumCircuit` as component?)
* `construct_circuit` is required for backward compatibility and the method name is very misleading 
* an Ansatz fits more the concept of an Gate in the sense that it has no registers attached. Therefore it should probably not be a `QuantumCircuit`.

### Implementation proposal #2: `Gate`

Make an Ansatz essentially a `Gate` object. This could achieved by either deriving from a gate or wrapping it.

#### Advantages

* intuitively that's what an Ansatz is: a (parameterized) gate
* provides some required functionalities: allows setting of parameters as list, can be controlled, repeated, etc. 
* no registers bound to it
* stores parameters as `list`, setting parameters by `list[float | Parameter]` is straightforward
* a `construct_circuit` method makes sense and is maybe missing anyways (or `to_circuit`)
* gives the user a Terra-like feel for the Ansatz

#### Technical Issues

* how to insert barriers or new gates at arbitrary positions?
* need to implement the appending of new gates/instructions (use `definition`?)
* exposes us to future changes of Terra's hierarchy

#### Design Issues

* inserts another level of inheritance (or maybe just use `Gate` as component?)

### Implementation proposal #3: `list[Gate]`

Base an Ansatz on a list of `Gate`s or `Instruction`s (each item with a set of qubit indices).
The parameters would be kept in a list.

#### Advantages

* natural implementation of repeating layers
* not dependent on changes in Terra
* internal data structure reflects input parameters of the user
* appending new instructions is trivial
* adding two Ansaetze (or even other objects, e.g. `QuantumCircuit` or `Gate`) is straightforward: just add the respective lists of gates, the indices, reps, and the parameters
* barriers are easily inserted in between the layers

#### Technical Issues

* same issue as with `Gate`s to insert a new gate somewhere inside a layer

#### Design Issues

* this implements a parallel structure of functionalities that might go into Terra anyways

## Conclusion: Suggested design

Ansatz as `QuantumCircuit` is not suitable: not the same performance benefit as the other designs, loose information of user input, `construct_circuit` is miselading. 
Using `Gate`s instead seems very convenient, conceptually and from an implementation point of view. If the Ansatz interally maintains a `Gate` (vs. being derived from it) gives us better control of the user information and naturally gives a sense of layers. 

**Suggestion** Use design #2, and implement it as the Ansatz internally maintaining a `Gate`, but feeling like one. 

## (Previously called) Variational forms <a id="varforms"></a>

### Backward compatibility

Does Ansatz provide all functionalities of the current `VariationalForm`? 

Almost.

Missing:
* `num_parameters`
* `setting(self)`
* `parameter_bounds(self)`
* `depth`

#### `setting`

Prints a formatted version of `self.__dict__`.  This can be replaced by implementing `__rep__` and thus, `setting` can be removed.

#### `support_parameterized_circuit`

This is necessary since Terra doesn't allow to bind complex values to a `Parameter` object. Once this is allowed, the `support_parameterized_circuit` attribute can be removed.

#### `parameter_bounds`

The `parameter_bounds` are supposed to be the valid ranges for the parameters of an Ansatz. They can be forwarded to an optimizer to limit the search range and, thus, improve performance.

An issue with allowing this attribute is that it suggests the Ansatz will enforce the parameter bounds to be respected. However, this is not correct.

Possible options:

1. Keep the attribute and give a warning, that the bounds are not enforced. Here we have two choices:
    1. the bounds are an attribute of the Ansatz itself
    1. the bound on a parameter is an attribute of the parameter itself. The Ansatz forwards parameter bounds it gets to its parameters, and if asked for bounds, the Ansatz asks the parameters.
1. Generate the parameter bounds on demand, based on the gates the Ansatz contains. If the standard gates knew their parameter range for optimization, this could be solved elegantly.
    1. Issue: the standard gates providing bounds is dangerous, as periodicity might be a feature we want to use, or the gate angle is subject to a coefficient invalidating the ranges.

#### `depth`

This equals the `reps` keyword of `Ansatz`. Both `VariationalForm` and `FeatureMap` use the word `depth` instead. Depth however, is ambiguous and could also refer to the circuit depth instead of the number of layers. 
Using `reps` instead of `depth` in the future is therefore desirable. With a deprectation warning the attribute `depth` can still be supported for a while. 

Another possible use of `depth` could be the following. One restriction on current hardware is the depth of the circuit to be simulated. To easily create Ansaetze of feasible circuit depth, the `depth` attribute could specify the maximum depth of the Ansatz. Instead of repeating the input gates _x_ times, we could repeat them until the maximum depth is reached.

## TwoLocalGateAnsatz

### Requirement recap

Input: number of qubits, one qubit gate(s), two qubit gate(s), depth/reps, entanglement, skip_unentangled_qubits, skip_final_rotation, (initial state?)

Output: the specified Ansatz

### Implementation proposal

In the initializer, build the rotation and entanglement gates from the single- and two-qubit gates and the entanglement information. From the depth/reps set up the correct repetition indices. In `construct_circuit`, build the circuit. Theoretically, the `construct_circuit` of the `Ansatz` class can be used.

Hence derivation is sensible.


```python
class TwoLocalGateAnsatz(Ansatz)
```

#### Points to discuss

* Should the keywords `entanglement` and `entangler_map` be merged to `entanglement`?
* Deprecate the `initial_state` keyword?

Should the keywords `entanglement` and `entangler_map` be merged to `entanglement`?

#### Pro

* Having multiple keywords forces the code to decide which keyword overrules which. This can be opaque to the user. If we don't enforce a "keyword hierarchy", we could throw errors if multiple are set. (And who likes runtime errors?) 
* Assume we add a new possible input type, e.g. `Callable`. Then we would require yet another keyword and choose which keyword is actually active if multiple are set. With one keyword only, we just have to add an `isinstance` if-statement (which we anyways need).
* The user doesn't need to think about which keyword he needs to use for the entanglement input, whether it is specified via string or pairs of qubits.

#### Contra

* The signature doesn't clearly show what input types of entanglement exist. (Then again, there is such a thing as docstrings.)

**Conclusion** Yes, merge `entanglement` and `entangler_map` in `entanglement`.

## Standard Ansaetze<a id='standard_varforms'></a>

Such forms as `RY` or `RYRZ`.

### Implementation proposal #1

As subclass of `TwoLocalGateAnsatz`.

#### Advantages

* super easy to implement
* natural design: The standard Ansaetze are specifications of the two-local gate ansatz, obtained by fixing the single qubit gate and providing a default two qubit gate.

#### Issues

* adds another level of inheritance
* `RY(Ansatz)` is elegant to read, since `RY` is just an Ansatz

### Implementation proposal #2

As subclass of `Ansatz`, using `TwoLocalGateAnsatz` as component.

#### Advantages

* `RY(Ansatz)` is intuitive to read
* no new level of inheritance

#### Issues

* probably need to wrap all functions of `TwoLocalGateAnsatz`

### Conclusion

Inheritance is easier to implement and reading `RY(TwoLocalGateAnsatz)` is still comprehensible.
Therefore, choose inheritance over composition.


```python
class RY(TwoLocalGateAnsatz):
    def __init__(self, ...):  # repeat all keywords here to enable IDE detection
        super().__init__(single_qubit_gate='ry', ...)
```


```python
class RYRZ(TwoLocalGateAnsatz):
    def __init__(self, ...):
        super().__init__(single_qubit_gate=['ry', 'rz'], ...)
```

## Feature maps <a id="featmaps"></a>

### Backward compatibility

Does Ansatz provide all functionalities of the current `FeatureMap`? 

Yes -- except `self.feature_dimension`.

#### `feature_dimension`

Possible solutions include

* New class `FeatureMap(Ansatz)`: An additional class for just one new method, that exists under another name, is not really justified.

* Add the method `feature_dimension` to `Ansatz`: If Ansatz is not used as replacement of current feature maps this method remains unused.

* Deprecate `feature_dimension`, use `num_parameters` instead: Deprecating `feature_dimension` is probably not a good idea, since it is a legitimate attribute of a feature map.

**Conclusion** The best solution seems to be adding, but deprecating, the `feature_dimension` method.

### Implementation proposal

The hierarchy within the "feature map" branch should be the same as for the "variational form" branch. Meaning, we implement a `PauliExpression` and the specialized classes as derived objects. This design is also the current form.

However -- is the additional layer `PauliZExpansion` truly necessary?


```python
def __init__(self, feature_dimension, depth=2, entangler_map=None, entanglement='full', z_order=2, 
            data_map_func=self_product):
    pauli_string = []
    for i in range(1, z_order + 1):
        pauli_string.append('Z' * i)
    super().__init__(..., paulis=pauli_string)
```

Later on used in the definition of the first- and second-order expansions.


```python
class FirstOrderExpansion(PauliZExpansion):  # resp. SecondOrderExpansion
    def __init__(self, feature_dimension, depth=2, data_map_func=self_product):
        super().__init__(..., z_order=1)  # resp. z_order=2
```

#### Pro

* simplifies the definition of Pauli Z expansion feature maps (such as `First/SecondOrderExpansion`)

#### Contra

* introduces an additional layer of inheritance
* if `First/SecondOrderExpansion` is the main motivation for this class, then the two minor changes they introduce is probably not worth introducing the `PauliZExpansion`

## Initial states

The `InitialState` class allows to obtain circuits that prepare certain (fixed) states. Currently, these include a the zero-state, a uniform superposition, the preparation of a specified statevector, or the an Ansatz with certain parameters.

A fixed circuit is merely a special case of a Ansatz with zero parameters, thus, the initial states can be seen as Ansaetze. 

However, all the parameter business and repeating is not needed for an initial state object and we could probably simplify the object. Also the initial state includes some functionalities (as statevector preparation) that an Ansatz doesn't. Thus, it might be less confusing for the user to keep the `InitialState` object. 

## Uncertainty models

Uncertainty models (UM) encode their underlying probability distribution in the amplitude of the qubits, i.e. they perform the mapping

$$
|0\rangle_n \mapsto \mathcal P_X |0\rangle_n = \sum_{x} \sqrt{p_x} |x\rangle_n.
$$

In this sense they could also fall in the category of an Ansatz with empty parameter set, i.e. an initial state.

However, to set up the UM for an arbitrary probability distribution they must be trained (e.g. via qGANs). To this end, they accept an Ansatz  $V(\theta)$ and tune the parameters until the operator $\mathcal P_X$ is approximated well enough. 

Hence, they should not be an Ansatz, but rather use on as component. 

# Notes from the design review

Notetaker/Steering: Donny

### High-level Impressions
* Kevin (@kdk)
    * Terra can help in some areas, e.g. Barriers for print but not transpilation
    * Should clarify what the Terra objects under the hood will be, or hide them, so as not to confuse users
    * Should think harder about eager transpilation timing, so transpile only happens when we know about the backend
* Stefan (@stefan-woerner )
    * What is the difference between the three types of terra circuits?
    * Transpilation - there is a difference between optimizing the gate and transpiling for hardware
    * Initialstate, UncertaintyModel - All overlapping, should be made the same
* Panos (@pbark )
    * Transpilation - Not doable to do pre-transpilation, because it depends on the parameters. Would be hard if doable.
    * Agrees with Stefan that more thought needs to go into organizing Featuremaps, InitialState, etc.
    * Inserting parameters will be difficult and maybe overly complicated
* Christa (@Zoufalc )   
    * We need the gate insertion for gradients
    * Initial state and uncertainty problem isn’t really an ansatz, it’s just a fixed circuit
* Ali (@ajavadia )
    * Implementation - Should this just be derived from QuantumCircuit, or gate or something else, because it has a lot of overlapping functionality (append, etc.)?
    * Difference between circuit, gate, instr - On our radar. Transpiling only one section of a gate should work by transpiling a circuit, and then calling to_gate(), and using that as is.
    * Steve (@woodsp-ibm )
    * Originally, Aqua came from a place where components were each pieces of circuits. We didn’t care what was under the hood, but want a consistent interface. Whatever we do here should be the same in the all the components.
    * It’s good to have common classes, but not necessary to mush everything, e.g. InitialState, together into one class just to say we did.
    * Parameters, transpile - if it’s not necessary to pass hardware information we shouldn’t. It would be better to just pass the user a circuit and have them transpile.
    * Mutability - If I have an Ansatz and do an append, will the algorithm understand? If you want to build an ansatz, can you just stack some together, instead of building complicated functionality to modify?
* Donny (@dongreenberg )
    * Like that the purpose of the Anastz as a thing that provides a service of repeating gates, and like Ali’s point that maybe this is just a sparse representation of a circuit.
Context on transpile - when we got rid of caching, we started trying to do this eager transpile.

### Quick walkthrough of the main ideas + open questions
* Julien (@Cryoris)  
    * Philosophy: Python is a language with lots of syntactic sugar and just works, and wanted to see that in the ansatz.
    * Wanted things to be easy to understand across Aqua, where if you understand the Ansatz you realize that you understand the  Featuremap and other too
    * Backward compatibility must be maintained
    * Combining things - InitialState, FeatureMap, etc. motivation is just to simplify these to make them familiar.

    
* Questions
    * Bounds - The Ansatz has bounds, but doesn’t enforce them. Should a gate have bounds?
        * **Unresolved, table for later**
    * Can we support a “parameterized_circuit” property in the gate?
        * Kevin: yes
    * Inherit or hold Circuit / Gate / Instruction?
        * **Table for later**

### Ratings
* All - Conditionally Approve, pending resolving Circuit inheritance vs. encapsulation discussion.

After follow up discussion, it was decided for the Ansatz to hold a gate or circuit, rather than inherit from Gate or QuantumCircuit, to allow for some flexibility as this design settles (inheritance introduces strong coupling which is hard to extricate later), and to allow for diverse Ansatze, such as linear combinations of circuits, not-gate.

**Final rating: Approved**

# How would the Ansatz turn out?

Here's a code skeleton of the `Ansatz` class wrapping a gate, along with how this design would fit in the workflow of Aqua. 
This is just an interface.


```python
from qiskit import QuantumCircuit
from qiskit.circuit import Gate

class Ansatz:
    def __init__(self, gates, qargs=None):
        pass  # include the setup functionalities as discussed in this document
    
    @property
    def num_qubits(self):
        return self._gate.num_qubits
    
    @property
    def params(self):
        return self._gate.params
    
    @params.setter
    def params(self, params):
        self._gate.params = params
            
    def control(self, num_ctrl_qubits=1, label=None):
        return self._gate.control(num_ctrl_qubits, label)
    
    def repeat(self, n):
        return self._gate.repeat(n)
    
    def inverse(self):
        return self._gate.inverse()
    
    def copy(self):
        """Get a copy of self. Can be used to append a copy of self to a QuantumCircuit."""
        return copy.deepcopy(self)
    
    def to_instruction(self):
        """To support QuantumCircuit.append(this)"""
        return self._gate  
    
    # potentially
    @property
    def definition(self):
        return self._gate.definition
```

### Amplitude Estimation

Using the `Gate` interface also for Amplitude estimation's $\mathcal A$ operator, we would write the algorithm as follows.


```python
A, Q = AOperator(...), QOperator(...) # A and Q support all Gate functionality

m = 3
i_objective = m
ae = QuantumCircuit(m + 1)

# initial A operator
ae.append(A, qargs=[i_objective])  

# Q operators
for mi in range(m):
    ae.barrier()
    ae.append(Q.repeat(2**mi).control(), qargs=[mi, i_objective])
    
# final fourier transform, this should also behave like a Gate
ae.append(Standard(m), qargs=list(range(m)))
```

Replacing the $\mathcal A$ operator with a `Gate`-interface leads to the following changes and replacements summarized in this table.


| before | now|
| :- | :- |
| inherit from `CircuitFactory` | inherit from `Gate` |
| overwrite `build(self, qc, q)` | overwrite `_define(self)` |
| `build_inverse()` | `inverse()` |
| `build_power(n)` | `repeat(n)` |
| `build_controlled()` | `control()` |
| `build_controlled_power(n)` | `repeat(n).control()` <br> specialized doesn't exist yet: add `controlled_power` |
| `a_factory.build(qc, q)` | `qc.append(a_operator, indices, [])` <br> or `qc += a_operator.to_circuit()` |

The following cell shows an example code skeleton for the $\mathcal A$ operator and concrete implementations of such operators used in AE. Since the $\mathcal A$ operator has barely any functionality on top of the `Gate` methods, it is sensible to directly derive it from the `Gate` class.


```python
class AOperator(Gate):
    def __init__(self):
        pass  
    
    def value_to_estimation(self, value):
        return value  # not doing anything by default
        
        
class BernoulliAOperator(AOperator):
    def __init__(self, probability):
        self._probability = probability
        
    def _define(self):  # this was previously in the build method
        circuit = QuantumCircuit(1)
        circuit.ry(probability, 0)
        self._definition = circuit.data
        
        
class LinearYRotation(Gate):
    def __init__(self, slope, offset, num_state_qubits, i_state=None, i_target=None):
        pass  # define self.i_state, self.i_target
    
    def _define(self):
        circuit = QuantumCircuit(max([self.i_target] + self.i_state[:]))
        circuit.ry(self.offset, i_target)
        for power, i in enumerate(i_state):
            circuit.cry(self.slope * 2**power, i, i_target)
        self._definition = circuit.data
        
        
class MyFunction(AOperator):
    def __init__(self, c_approx, num_state_qubits):
        pass  # define self.slope, self.offset, self.num_state_qubits
        
    def _define(self):
        circuit = QuantumCircuit(self.num_state_qubits + 1)
        linear_y = LinearYRotation(self.slope, self.offset, self.num_state_qubits)
        circuit.append(linear_y, list(range(self.num_state_qubits + 1)), [])
        circuit.z(num_state_qubits)
        circuit.append(linear_y.inverse(), list(range(self.num_state_qubits + 1)), [])
        self._definition = circuit.data
```

### Uncertainty Model

Uncertainty models in Aqua allow us to load a discretized probability distribution in the amplitudes of qubits. Currently, they are implemented as `CircuitFactory`, but should follow the same flow as the Ansatz and amplitude estimation's $\mathcal A$ operator.


```python
class UnivariateDistribution(Gate): 
    def __init__(self, num_uncertainty_qubits):
        self._definition = Custom(...).definition
        
    @property
    def low(self): # also implement: high, num_values, values, probabilities
        pass
    
    @staticmethod
    def pdf_to_probabilities(self):
        pass

class UnivariateVariationalDistribution(UnivariateDistribution, Ansatz):  # use case? UVD as Ansatz
    def __init__(self, ansatz):
        self._definition = ansatz.definition  # or ansatz.to_instruction().definition or such

    def set_probabilities(self):
        pass
```

Application of uncertainty models on QGANs:


```python
qgan = QGAN(real_data, bounds, num_qubits)
distribution = UnivariateVariationalDistribution(some_ansatz)
qgan.set_generator(distribution)
```

Application to a circuit:


```python
distribution = NormalDistribution(num_uncertainty_qubits=4, mu=0, sigma=1)
circuit.append(distribution, list(range(num_uncertainty_qubits)))
```

### Advanced amplitude estimation example: Ansatz + uncertainty model + function mapping


This is an example combining an ansatz, an uncertainty model plus the mapping of a linear function in an amplitude estimation experiment. Here, the ansatz can encode optimization variable and the uncertainty model could represent a probability distribution.


```python
n = 3  # qubits for the uncertainty model
k = 3  # qubits for the Ansatz

prob = NormalDistribution(n, mu, sigma)  
varform = RY(k, entanglement='linear', reps=2)

# either encapsulate the logic in a fixed class derived from AOperator
a_operator = MyAOperator(prob, varform)

# or do directly what would be done in MyAOperator
f_mapping = MyFunction(n, k)
a_circuit = QuantumCircuit(n + k + 1)
a_circuit.append(prob, prob_indices)
a_circuit.append(varform, varform_indices)
a_circuit.append(f_mapping, f_indices)
a_operator = AOperator(a_circuit)

# or do the application of the cost function more explicitely
a_circuit = QuantumCircuit(n + k + 1)
a_circuit.append(prob, prob_indices)
a_circuit.append(varform, varform_indices)
# apply the mapping F directly
for i in range(n):
    a_circuit.ry(angle, i) 
# etc.
a_operator = AOperator(a_circuit)

# or maybe even use the Ansatz constructor facilities?
f_mapping = MyFunction(n, k)
prob_indices, varform_indices, f_indices = list(range(n)), list(range(n, n + k)), list(range(n + k + 1))
a_as_ansatz = Ansatz([prob, varform, a_operator], [prob_indices, varform_indices, f_indices])
a_operator = AOperator(a_as_ansatz.to_circuit()) 

# then go about calling AE
ae = AmplitudeEstimation(m, a_operator)
```

### VQE

Under the hood, not much would change in the VQE algorithm. We are more flexible in how to use the Ansatz though. Depending on if we implement a `to_circuit` method (which seems very useful at the moment) the changes boil down to using `ansatz.to_circuit(params)` instead of `var_form.construct_circuit(params)`.


```python
def VQAlgorithm(QuantumAlgorithm):
    # no changes at all
    pass
```


```python
def VQE(VQAlgorithm):
    def __init__(self, var_form: Ansatz):
        # only change: either add a preferred_init_points method to Ansatz, or accept such a kwarg in VQE
    
    def construct_circuit(self):
        # we could do
        wave_function = self._var_form.to_circuit(parameter)
        
        # or
        self._var_form.params = parameter
        wave_function = self._var_form.to_circuit() 
        
        # or
        self._var_form.params = parameter
        wave_function = QuantumCircuit(self._var_form.num_qubits)
        wave_function.append(self._varform, list(range(self._var_form.num_qubits)))
        
        # rest of the implementation...
        pass
    
    def _eval_aux_ops(self):
        if params is None:
            params = self.optimal_params
        self._var_form.params = parameter
        wavefn_circuit = QuantumCircuit(self._var_form.num_qubits)
        wavefn_circuit.append(self._varform, list(range(self._var_form.num_qubits)))   
        
    def _energy_evaluation(self, parameters):
        # parameterized circuits should be supported completely (Terra will implement support for complex values)
        # otherwise this is supported exactly the same as before
        
    def get_optimal_circuit(self):
        self._var_form.params = self._ret['opt_params']
        circuit = QuantumCircuit(self._var_form.num_qubits)
        circuit.append(self._varform, list(range(self._var_form.num_qubits)))
        return circuit
```
