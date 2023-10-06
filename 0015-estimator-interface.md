# Extended Estimator Interface

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0015                                         |
| **Authors**       | Samantha Barron (samantha.v.barron@ibm.com)  |
|                   | Lev Bishop (lsbishop@us.ibm.com)             |
|                   | Ikko Hamamura (ikko.hamamura1@ibm.com)       |
|                   | Ian Hincks (ian.hincks@ibm.com)              |
|                   | Takashi Imamichi (imamichi@jp.ibm.com)       |
|                   | Blake Johnson (blake.johnson@ibm.com)        |
|                   | Chris Wood (cjwood@us.ibm.com)               |
| **Deprecates**    | RFC that this RFC deprecates                 |
| **Submitted**     | 2023-10-03                                   |
| **Updated**       | YYYY-MM-DD                                   |


## Summary <a name="summary"></a>

The current `Estimator.run()` method requires that a user provide one circuit for every observable and set of parameter values that they wish to run.
(This was not always the case via circuit indices, but the history will not be discussed here.)
It is common, if not typical, for a user to want to estimate many observables corresponding to a single circuit. 
Likewise, it is common, if not typical, for a user to want to supply multiple parameter value sets for the same circuit.
This RFC proposes to fix these issues by changing the `Estimator.run()` signature in the following ways: 

 1. Take the transpose of the current signature; rather than accepting `circuits`, `parameter_values`, and `observables` as three different (iterable) arguments which are to be zipped together, instead localize the distinct tasks to be run via an iterable of triples `(circuit, parameter_values, observables)`.
 2. In combination with 1, extend `parameter_values` and `observables` to be array-valued, that is, add the ability to explicity and conveniently specify multiple parameter value sets and observables for a single circuit.

## Motivation <a name="motivation"></a>

Here is a summary of pain-points with the current `Estimator.run()` interface:

1. _Ergonomics._ It feels unnatural to invoke the estimator as `Estimator.run([circuit, circuit], observables=[obs1, obs2])` because of the redundancy of entering the circuit twice. 

1. _Trust._ Cases like `Estimator.run([circuit, circuit], observables=[obs1, obs2])` make a user question whether the primitive implementation is going to do any optimizations to check if the same circuit appears twice, and if so, whether it will be done by memory location, circuit equality, etc.; the interface itself creates gap in trust.

1. _Clarity._ Without reading the documentation in detail, it's not obvious that the invocation `Estimator.run([circuit], observables=[obs1, obs2])` wouldn't cause `circuit` to be run with both supplied observables. In other words, that zipping _is always what's done to the `run()` arguments_ is common source of user confusion. Conversely, it's not clear that the way to collect all combinations of two observables and two parameter sets is to invoke `Estimator.run([circuit] * 4, observables=[obs1, obs2] * 2), parameter_values=[params1] * 2 + [params2] * 2`.

1. _Performance._ Given that `QuantumCircuit` hashing should be avoided and circuit equality checks can be expensive, we should move away from an interface that necessitates performant primitive implementations to go down this path at all. As one example, qubit-wise commuting observables like `"IZZ"` and `"XZI"` can both be estimated using the same simulation/execution, but only if the estimator understands that they share a base circuit. As a second example, when the circuits need to be serialized before they are simulated/executed (e.g. runtime or pickling-for-multi-processing), it puts the onus on the primitive implementation to detect circuit duplications.
   

Here is why the [Detailed Design](#detailed-design) section suggests "transposing" the signature and using array-based arguments with broadcasting:

 1. Transposing the signature will make it obvious what the primitive intends to do with each circuit.
 2. Transposing the signature will let us introduce and reason about the notion  "primitive unit of work", for which this RFC proposes the name "task".
 3. Array-based arguments will let users assign operational meaning to each axis (this axis is for twirling, that axis is for basis changes, etc.).
 4. Broadcasting rules will let users choose how to combine parameter value sets with observables in different ways, such as
    1. use one observable for all of N parameter value sets
    2. use one parameter value set for all of N observables
    3. zip N parameter value sets against N observables
    4. take all NxM combinations of N parameter value sets with M observables
    5. etc.

## User Benefit <a name="user-benefit"></a>

All users of the primitives stand to benefit from this proposal.
Immediately, it will enable sophisticated and convenient workflows for power users though arrays and broadcasting.
However, standard broadcasting rules are such that the 0D and 1D cases will feel natural to existing users---0D is essentially what we already have, and 1D will be perceived as a simple-yet-welcome bonus.

Moreover, the interface changes in this proposal will enable specific primitive implementations to enhance performance through reduced bandwidth on the runtime, compatibility with fast parameter binding, and multiplexing qubit-wise commuting observables.

## Design Proposal <a name="design-proposal"></a>

We will lead the design proposal with an example:

```python
# construct two circuits, the first with 2039 Parameters, the second with no Parameters
circuit1 = QuantumCircuit(9)
...
circuit2 = QuantumCircuit(2)
...

# specify 128 different parameter value sets for circuit1, in a 4x32 shape
parameter_values1 = np.random.uniform(size=(4, 32, 2039))
# specify 4 observables to measure for circuit 1
observables1 = ["ZZIIIIIII", "IZXIIIIII", "IIIIIIYZI", "IZXIIYIII"]

# specify 18 observables to measure for circuit2
observables2 = [Pauili("XYZIIII"), ..., SparsePauliOp({"YYIIIII": 0.2})]

# invoke an estimator
estimator = Estimator()
job = estimator.run([
  (circuit1, observables1, parameter_values1),
  (circuit2, observables2)
])

# wait for the result
job.result()

# get a bundle of results for every task
>> EstimatorResult(
>>     TaskResult<{evs: ndarray<4, 32>, stds: ndarray<4, 32>}, local_metadata>, 
>>     TaskResult<{evs: ndarray<18>, stds: ndarray<18>}, local_metadata>, 
>>     global_metadata
>> )
```

The above example shows various features of the design, including combining large numbers of observables with large numbers of parameter value sets for a single circuit.

The detailed targetted signature, following the [deprecation period](#migration-path),
is as follows:

```python
T = TypeVar("T", bound=Job)

BindingsArrayLike = Union[
    BindingsArray, 
    NDArray, 
    Dict[Parameter | Tuple[Parameter], NDArray]
]

ObservablesArrayLike = Union[
    ObservablesArray,
    Iterable[str | Pauli | SparsePauliOp]
]

EstimatorTaskLike = Union[
    EstimatorTask,
    Tuple[QuantumCircuit, ObservablesArrayLike, BindingsArrayLike]
]

class EstimatorBase(ABC, Generic[T]):
    ...
    def run(self, tasks: EstimatorTaskLike | Iterable[EstimatorTaskLike], **options) -> T:
        pass
```

## Detailed Design <a name="detailed-design"></a>

### Tasks <a name="tasks"></a>

We propose the concept of a _task_, which we define as _a single circuit along with auxiliary data required to execute the circuit relative to the primitive in question_. This concept is general enough that it can be used for all primitive types, current and future, where we stress that what the “auxiliary data” is can vary between primitive types. 

For example, a circuit with unbound parameters (or in OpenQASM3 terms, a circuit with `input`s) alone could never qualify as a task for any primitive because there is not enough information to execute it, namely, numeric parameter binding values. On the other hand, conceptually, a circuit with no unbound parameters (i.e. an circuit with no `input`s) alone could form a task for a hypothetical primitive that just runs circuits and returns counts. This suggests a natural base for all tasks:

```python
@dataclass
class BaseTask:
    circuit: QuantumCircuit
```

This base class can be added to Qiskit if and when needed as a non-breaking change, and has been included here mainly to make the concepts clear, rather than to indicate that we need such a base class immediately.
Most relevant, for the `Estimator` primitive, in order to satisfy the definition as stated above, we propose the task structure

```python
class EstimatorTask(BaseTask):
    observables: ObservablesArray = None
    parameter_values: BindingsArray = None, 
```

We expect the formal primitive API and primitive implementations to have a strong sense of tasks, but we will not demand that users construct them manually in Python as they are little more than simple data containers (incidentically implemented as dataclasses above), and we do not wish to overburden them with types. This is discussed further in [Type Coercion Strategy](#type-coercion-strategy).

### BindingsArray <a name="bindingsarray"></a>

It is common for a user to want to do a sweep over parameter values, that is, to execute the same parametric circuit with many different parameter binding sets. 
We propose a new class `BindingsArray` that stores multiple sets of parameters, which can be bound to a circuit. 
For example, if a circuit has 200 parameters, and a user wishes to execute the circuit for 50 different sets of values, then a single instance of `BindingsArray` could represent 50 sets of 200 parameter values. 
Moreover, we propose that it be array-like, so that, in this example, the `BindingsArray` instance would be one-dimensional and have shape equal `(50,)`. 

The proposed `BindingsArray` object would support, at a minimum, the following constructor examples for a circuit with three input parameters `a`, `b`, and `c`, where we will use `<>` to denote some `array_like` of the specified shape:

```python
# a single value for each a, b, and c; 0-dimensional, and compatible with current interface
BindingsArray([0.0, 1.0, 2.0])

# a single value for each a, b, and c; 0-dimensional, and compatible with current interface
# satisfies user wish to be able to specify values by name
BindingsArray(kwvals={Parameter("a"): 0.0, Parameter("b"): 1.0, Parameter("c"): 2.0})

# specify all 50 binding parameter sets in one big array
BindingsArray(<50, 3>) 

# specify bindings separately for each parameter, required if they have different types
BindingsArray([<50>, <50>, <50>]) 

# include parameter names with the arrays, where parameters can be grouped together in tuples, or supplied separately
BindingsArray(kwargs={(a, c): <50, 2>, b: <50>}) 

# “args” and “kwargs” can be mixed
BindingsArray(<50, 2>, {c: <50>}) 
```

We expect the formal primitive API and primitive implementations to have a strong sense of `BindingsArray`, but we will not demand that users construct them manually because we do not wish to overburden them with types, and we need to remain backwards compatible. This is discussed further in the [Type Coercion Strategy](#type-coercion-strategy) and [Migration Path](#migration-path) sections.

### ObservablesArray <a name="observablesarray"></a>

With the `Estimator`, it is common for a user to want to estimate many observables of a single circuit. 
For example, all weight-1 and weight-2 Paulis that are adjacent on the connectivity graph. 
For a one-hundred qubit device, this corresponds to hundreds of unique estimates to be made for a single circuit, noting that for this particular example, on the heavy-hex graph, in the absence of mitigation, only 9 circuits need to be physically run.

We propose a new `ObservablesArray` object, where each element corresponds to an observable the user wants an estimated expectation value of.
We propose that the internal data model of this object have an element format

```python
List[Tuple[str, List[int], float]]
```

where, for example, the observable `0.5 * IIZY + 0.2 * XIII` would be stored as

```python
[("ZY", [2, 3], 0.5), ("X", [0], 0.2)]
```

This is proposed instead of `{"IIZY": 0.5, "XIII": 0.2}` anticipating the overhead of storing and working with so many `"I"`'s for large devices.

Note that the above is simply the data model; the `ObservablesArray` object, either in the constructor or special static coercion method, will accept the `quantum_info` objects that are convenient to users.
However, only types such as `Pauli` and `SparsePauliOp` that are known to have sparse Pauli representations will be allowed; we will drop general support for `BaseOperator`.
Further, through [type coersion](#type-coercion-strategy) in `Estimator.run()` itself, a user won't need to manually construct an `ObservablesArray`, as shown in the example in the [Design Proposal](#design-proposal) section.

It is up to each `Estimator` implementation to, if it sees fit, to solve a graph coloring problem that decides how to produce a sufficient set of physical circuits that are capable of producing data to make each of these estimates.

### Arrays and Broadcasting <a name="arrays-and-broadcasting"></a>

An `nd-array` is an object whose elements are indexed by a tuple of integers, where each integer must be bounded by the dimension of that axis. 
The tuple of dimensions, one for each axis, is called the shape of the `nd-array`. 
For example, 1D list of 10 objects (typically numbers) is a vector and has shape `(10,)`; a matrix with 5 rows and 2 columns is 2D and has shape `(5, 2)`; a single number is 0D and has an empty shape tuple `()`; an `nd-array` with shape (20, 10, 5) can be interpreted as a length-20 list of 10×5 matrices.

We propose the constraint that a `BindingsArray` instance and an `ObservablesArray` instance that live in the same task must have shapes that are broadcastable, in which case, the `Estimator` promises to return one expectation value estimate for each element of the broadcasted shape.

Here are some examples of common patterns expressed in terms of array broadcasting, and their accompanying visual representation in the figure that follows:

```python
# Broadcast single observable
parameter_values = np.array(1.0)
parameter_values.shape == ()
observables = ObservablesArray(["III", "XXX", Pauli("YYY"), "ZZZ", Pauli("XYZ")])
observables.shape == (5,)
>> task_result.shape == (5,)

# Inner/Zip
parameter_values = BindingsArray(np.random.uniform(size=(5,)))
parameter_values.shape == (5,)
observables = ObservablesArray([Pauli("III"), Pauli("XXX"), Pauli("YYY"), Pauli("ZZZ"), Pauli("XYZ")])
observables.shape == (5,)
>> task_result.shape == (5,)

# Outer/Product
parameter_values = BindingsArray(np.random.uniform(size=(1, 6)))
parameter_values.shape == (1, 6)
observables = ObservablesArray([[Pauli("III")], [Pauli("XXX")], [Pauli("YYY")], [Pauli("ZZZ")]])
observables.shape == (4, 1)
>> task_result.shape == (4,6)

# Standard nd generalization
parameter_values = BindingsArray(np.random.uniform(size=(3, 6)))
parameter_values.shape == (3, 6)
observables = ObservablesArray([
    [[Pauli(...), Pauli(...)]],
    [[Pauli(...), Pauli(...)]],
    [[Pauli(...), Pauli(...)]]
])
observables.shape == (3, 1, 2)
>> task_result.shape == (3, 2, 6)
```

<img src="./0015-estimator-interface/broadcasting.svg">

### Type Coercion Strategy <a name="type-coercion-strategy"></a>

To minimize the number of container types that an every-day user will need to interact with, and to make the transition more seamless, we propose that several container types be associated with a `TypeLike` pattern and a static method

```python
def coerce(argument: TypeLike) -> Type
```

that is invoked inside of the `run()` method.

For example,

```python
BindingsArrayLike = Union[BindingsArray, ArrayLike, Mapping[Parameter, float]]
```

and

```python
@staticmethod
def coerce(bindings_array: BindingsArrayLike) -> BindingsArray:
    if isinstance(bindings_array, (list, ArrayLike)):
        bindings_array = BindingsArray(bindings_array)
    elif isinstance(bindings_array, Mapping):
        bindings_array = BindingsArray(kwargs=bindings_array)

    return bindings_array
```

In particular, we propose this kind of Coercion for the types:
* `EstimatorTask`
* `BindingsArray`
* `ObservablesArray`

### TaskResults <a name="TaskResults"></a>

The results from each `EstimatorTask` will be array valued, and each `EstimatorTask` in the same job may have a different shape.
Consider an `EstimatorTask` with shape `<20, 30>`, where the shape has come from the broadcasting rules discussed elsewhere. 
This will result in a 20×30 array of real estimates.
Moreover, we will want to return an array of standard deviations of the same shape.
This would result in a bundle of broadcastable arrays:

```python
TaskResult({“evs”: <20, 30>, “stds”: <20, 30>}, metadata)
```

The reason we are proposing a generic container for the return type instead of, e.g., an `Estimator`-specific container, is because it

1. Provides unified experience across primitives for users. 
2. code-reuse
3. Provides a certain certain amount of flexibility for what can be returned without modifying the container object. Here are some examples:
  1. Suppose that we want to give users the option of additionally returning the covariances between estimates that arise because of the circuit multiplexing, then we could update with the field `{"cov": <20,30,20,30>}`.
  2. Suppose we want to return some indication of which estimates came from the same physical circuit.

## Migration Path <a name="migration-path"></a>

We need to remain backwards compatible with the existing interface to adhere to the [Qiskit Deprecation Policy](https://qiskit.org/documentation/deprecation_policy.html).

In summary, we propose the following strategy: If the user has provided no `TaskLike`s, proceed with the old API, the old API output, and emit a deprecation warning, or an error if something mandatory like `observables` has been omitted. Otherwise, proceed with the new API, raising if they have tried to use the old arguments in addition to providing tasks.

We now describe the strategy in detail, beginning with a restatement of the current interface, where `Estimator.run` has `circuits` as the only positional argument and accepts `observables` and `parameter_values` as keyword arguments (in addition to the var keyword `**run_options` arguments which isn't affected):

```python
class Estimator(BasePrimitive):

    # current signature
    def run(
            self,
            circuits: Sequence[QuantumCircuit] | QuantumCircuit,
            observables: Sequence[BaseOperator | PauliSumOp | str] | BaseOperator | PauliSumOp | str,
            parameter_values: Sequence[Sequence[float]] | Sequence[float] | float | None = None,
            **run_options,
        ):
        ...
```

We propose the migration to using tasks in two phases.

### Deprecation Phase 1

In the first phase:
* Introduce `tasks: Sequence[EstimatorTask] | EstimatorTask` as the only positional argument, and make `circuits` a keyword argument.
* Coerce the arguments to account for the fact that the only existing positional argument is `circuit: Sequence[QuantumCircuit] | QuantumCircuit`.
* Check that the user does not attempt to mix the old/new APIs.
* If necessary, coerce the old-API arguments (now all keyword arguments) into `EstimatorTask`s, **raise deprecation warning**.
* Always eventually run with `Estimator._run(tasks: Sequence[EstimatorTask], **run_options)`.

Here is a mock implementation, with missing type annotations as above:
```python
class Estimator(BasePrimitive):

    def run(
        self, 
        tasks: Sequence[EstimatorTask] | EstimatorTask, 
        circuits = None,
        observables = None, 
        parameter_values = None, 
        **run_options
    ):
        # Coerce into standard form to later allow for `tasks` to be the only non-kwarg.
        if isinstance(tasks, QuantumCircuit) or all(isinstance(task, QuantumCircuit) for task in tasks):
            if circuits is not None:
                raise ValueError("Cannot mix old and new APIs")
            circuits = tasks
            tasks = None

        # Do not mix new/old API
        if tasks and (circuits or observables or parameter_values):
            raise ValueError("Cannot mix old and new APIs")

        # Deprecated path
        if tasks is None:
            # Verify values in old API
            if not (circuits and observables):
                raise ValueError(f"Need both circuits and observables for old API, got circuits={circuits}, observables={observables}")
            
            # Coerce into old form
            if parameter_values is None:
                parameter_values = itertools.repeat(None)
            if isinstance(circuits, QuantumCircuit):
                circuits = [circuits]
            if isinstance(observables, (BaseOperator, str)):
                observables = [observables]

            # Coerce old form into `EstimatorTask`s
            tasks = [EstimatorTask.coerce(task) for task in zip(circuits, observables, parameter_values)]
            warnings.warn("Deprecated API use")
        
        # Coerce into sequence form
        if isinstance(tasks, EstimatorTasks):
            tasks = [tasks]
        
        # Run using only tasks
        return self._run(tasks, **run_options)
```

### Deprecation Phase 2

After the deprecation period, remove extraneous arguments and checks:

```python
class Estimator(BasePrimitive):

    def run(
        self, 
        tasks: Sequence[EstimatorTask] | EstimatorTask, 
        **run_options
    ):
        # Coerce into sequence form
        if isinstance(tasks, EstimatorTasks):
            tasks = [tasks]
        
        # Run using only tasks
        return self._run(tasks, **run_options)
```

## Alternative Approaches <a name="alternative-approaches"></a>

An alternative is to consider letting the `run()` method accept, effectively, only a single `EstimatorTask`:

```python
Estimator.run(cirucuit, parameter_values_array, observables_array)
```

This has the advantage of a simpler interface, where multiple tasks could be run 
by invoking the estimator multiple times. The disadvantages, which we feel are significant enough to forego this simplification, are that:

 1. For real backends, the user would lose the ability to cause multiple types of circuits to be loaded into the control hardware at one time. For example, if using an estimator to perform randomized benchmarking, each circuit depth would need to be a separate job.
 2. It would be difficult for implementations that include mitigation to share resources between tasks. For example, if different tasks represent different trotter step counts, there would need to be a complicated mechanism to share learning resources---that are specific to the application circuits---between multiple jobs.

## Questions <a name="questions"></a>
Open questions for discussion and an opening for feedback.

## Future Extensions <a name="future-extensions"></a>

### Circuit Formats

In this proposal we have typed circuits as `QuantumCircuit`. It would be possible to extend this to a `CircuitLike` class which could be as simple as `Union[QuantumCircuit, str]` to explicitly allow OpenQASM3 circuits as first-class inputs.

### Tasks in the base class (and other primitives)

A consequence of switching to the concept of Tasks (mentioned in [Tasks](#tasks)) is that this will allow us to introduce the `.run()` method into `BasePrimitive`

```python
class BasePrimitive(ABC, Generic[T]):
    ...
    def run(self, T | Iterable[T], **options) -> List[TaskResult]:
        ...
```

### `BindingsArray` generalizations

`BindingsArray` is somewhat constrained by how `Parameters` currently work in Qiskit, namely, there is no support for array-valued inputs in the same way that there is in OpenQASM 3; `BindingsArray` assumes that every parameter represents a single number like a `float` or an `int`.
One solution could be to extend the class to allow different sub-arrays to have extra dimensions.
For example, for an input angle-array `input array[angle, 15] foo;`, and for a bindings array with shape `(30, 20)`, the corresponding array could have shape `(30, 20, 1, 15)`, where the singleton dimension exists to express that it is targetting a single `input`.

Another generalization is to allow broadcastable shapes inside of a single `BindingsArray`.
For example, one could have one array with shape `(1, 4, 5)` for five parameters, and another with shape `(3, 1, 2)` for two parameters, resulting in a bindings array with shape `(3, 4)`.
The advantage of this is allowing sparser representations of bindings arrays, and also specifying "fast" and "slow" parameters.