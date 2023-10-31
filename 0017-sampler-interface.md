# SamplerV2

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0017                                         |
| **Authors**       | Lev Bishop (lsbishop@us.ibm.com)             |
|                   | Ian Hincks (ian.hincks@ibm.com)              |
|                   | Blake Johnson (blake.johnson@ibm.com)        |
|                   | Chris Wood (cjwood@us.ibm.com)               |
| **Submitted**     | 2023-10-03                                   |
| **Updated**       | YYYY-MM-DD                                   |


## Summary <a name="summary"></a>

This RFC proposes an interface for the next version of the `Sampler`.
The changes proposed here mirror those accepted in [(the paired Estimator inteface RFC)[https://github.com/Qiskit/RFCs/pull/51]](https://github.com/Qiskit/RFCs/pull/51), where the arguments of the `Estimator.run()` signature were transposed to accept, instead of three iterable arguments, an iterable of "tasks", with each task comprising a circuit and auxiliary execution information.
In the case of the `Sampler`, a task consists only of a circuit and an array of parameter value sets if it is a parametric circuit.
We aim to move the `Sampler` to be a viable replacement for `backend.run`, and eliminate the notion of sample weights (i.e. quasi-probabilities).

## Motivation <a name="motivation"></a>

We will propose a `Sampler` that

 * _Obeys program outputs._ Every circuit defines a set of classical registers whose output names and types are known before execution. The prototypical type is a bit-array, but the OpenQASM 3 specification allows many other types, too, which Qiskit aims to follow. The current return structure of the sampler does not handle this gracefully, squishing bitstrings from separate registers together. It also does not attempt make a careful distinction between sampling from a program's output registers, and sampling from the output state that it prepares---it did not need to as these two views are equivalent for circuits that measure all qubits at the end of the circuit. However, this proposal takes the former stance, and separates returned data according to its output names. The motivation for this choice is that it would make the `Sampler` fully compliant with dynamic circuits.
 * _Naturally allows multiple parameter value sets._ A common use case for many users is running the same parametric program with many parameter value sets in the same `run()` invocation. These changes will make it easier for implementations to do this efficiently with late-stage binding. Efficient invocation of the same parametric circuit between multiple `run()` invocations is out of scope of this RFC.
 * _Is a "`memory=True`" exection model._ We will break from the tradition of using a mapping structure as the default container type for circuit exection. Instead, results will always be returned in a format where one output axis iterates over shots, and maintains execution order. The motivation is that, in a program whose total output size is more than 100 qubits, due to gate and measurement noise, the chances that any two output values are equal to each other is too small to make special accomodation for via a hash map. When the output size is smaller than this loose threshhold, so that output value collisions are likely, the data will take up less space anyways. Container objects will contain methods returning count-like structures to ease migration.
 * _Has no sample weights._ The current sampler attaches real weights summing to one to all samples. In effect, it returns an estimate of a (quasi)-distribution instead of samples from the distribution. This proposal eliminates this feature and returns only samples, and without weights. The rationale is to make the `Sampler` into more of a streamlined circuit executor, paring down all frills.


## User Benefit <a name="user-benefit"></a>

Users will be able to transition from `backend.run` to `sampler.run` without any compromises in execution efficiency. They will be able to run dynamic circuits without having to specify special flags, or invoke a special program. They will get their outputs in a format where they don't need to manually separate bitstrings from different registers.

As with the [(paired Estimator inteface RFC)[https://github.com/Qiskit/RFCs/pull/51]](https://github.com/Qiskit/RFCs/pull/51), implementations will more easily be able to enable features like late stage parameter value binding.

## Design Proposal <a name="design-proposal"></a>

We will lead the design proposal with an example.

```python
# Our first circuit has many Parameters, but elects to do all measurement at 
# the end. Note that measure_all() chooses the register name "meas" for you.
circuit1 = QuantumCircuit(9)
...
<use 2039 Parameters in this circuit>
...
circuit1.measure_all()

# Specify 128 different parameter value sets for circuit1, in a 32x4 shape.
parameter_values1 = np.random.uniform(size=(32, 4, 2039))

# Our second circuit happens to have no parameters, but three output registers 
# named alpha, beta, and gamma of various sizes
alpha = ClassicalRegister(127, "alpha")
beta = ClassicalRegister(10, "beta")
gamma = ClassicalRegister(15, "gamma")
circuit2 = QuantumCircuit(QuantumRegister(127), alpha, beta, gamma)
...
<insert gates, measure into the registers, apply dynamic circuit features, etc.>
...

# invoke a sampler
sampler = Sampler(..., shots=1024)
job = sampler.run([
    (circuit1, parameter_values1),
    circuit2
])

# wait for the result
job.result()

# get a bundle of results for every task
>> PrimitiveResult(
>>     SamplerTaskResult(
>>         DataBundle(meas=BitArray(<num_samples=1024, num_bits=127, shape=(32, 4)>)), 
>>         <metadata>
>>     ), 
>>     SamplerTaskResult(
>>         DataBundle(
>>             alpha=BitArray(<num_samples=1024, num_bits=127, shape=()>)
>>             beta=BitArray(<num_samples=1024, num_bits=10, shape=()>)
>>             gamma=BitArray(<num_samples=1024, num_bits=15, shape=()>)
>>         ), 
>>         <metadata>
>>     ), 
>>     metadata=<global_metadata>
>> )
```

The above example shows various features of the design.
Notice the output results are placed in a `DataBundle`, a namespace container described [here](https://github.com/Qiskit/RFCs/pull/53), with one entry for each output of the corresponding task's circuits.
These circuits only have bit-array outcomes, as allowed by Qiskit, and therefore all results have type `BitArray`, described later.
For now, consider the following usage examples.

```python

# get result from the second task above
result = job.result()[1]

# get the entire quasi-probability distribution
result.get_quasiprobabilities()
>> QuasiProbabilityDistribution({1351235663: 0.0009765625, 325: 0.0009765625, 45262346723465234523453: 0.0009765625, 35435345235234372457453: 0.0009765625, ...})

# marginalize a couple of indexes in the beta output
result.data.beta.get_counts(marginalize_idxs=[0, 1])
>> Counts({"00": 189, "10": 11, "01": 801, "11": 123})

# flip all of the bits in the gamma according to some other compatibly-shaped bit-array
flips = BitArray(my_flips, num_bits=15)
flipped_gamma = result.data.gamma ^ flips
```

If a circuit had a non bit-array outcome, it would use a different type in place of `BitArray`.
We suggest just using `numpy.ndarrays` with the closest appropriate `dtype` for non-bit-array outputs. 
The shape of such an array would be `(*task_shape, num_samples, *output_register_shape)`.
For particular types, like `angle`, if it is for some reason decided that no NumPy type is satisfactory,
we can add more containers analagous to `BitArray`.

The detailed signature is as follows:

```python
T = TypeVar("T", bound=Job)

BindingsArrayLike = Union[
    BindingsArray, 
    NDArray, 
    Dict[Parameter | Tuple[Parameter], NDArray]
]

SamplerTaskLike = Union[
    SamplerTask,
    QuantumCircuit,
    Tuple[QuantumCircuit, BindingsArrayLike]
]

class SamplerBaseV2(PrimitiveBase[SamplerTask, SamplerTaskResult]):
    ...
    def run(
        self, tasks: SamplerTaskLike | Iterable[SamplerTaskLike]
    ) -> Job[PrimitiveResult[SamplerTask, SamplerTaskResult]]:
        pass
```


## Detailed Design <a name="detailed-design"></a>

Most new types and conventions have been described in either [the paired Estimator RFC](https://github.com/Qiskit/RFCs/pull/51) or the [`BasePrimitive.run()` RFC](https://github.com/Qiskit/RFCs/pull/53).

### Tasks and Task Results <a name="tasks"></a>

A task consists of a `QuantumCircuit` and, if it has parameters, a `BindingArray` that specifies an array of parameter value sets each of which to bind it against during execution.

A `SamplerTaskResult` is a subclass of `TaskResult` with the addition of methods for convenience, and to ease migration. For example,

```python
class SamplerTaskResult(TaskResult):
    def get_quasiprobabilities(self, loc: Tuple[int, ...] | None = None) -> QuasiProbabilityDistribution:
        """Join all bit-array output registers together into a new quasi-probability distribution.
        
        Args:
            loc: Which task coordinate to fetch.

        Raises:
            ValueError: If no ``loc`` is given and this task result is more than 0-D.
        """

    def get_counts(self, loc: Tuple[int, ...] | None = None) -> Counts:
        """Join all bit-array output registers together into a new counts structure.
        
        Args:
            loc: Which task coordinate to fetch.

        Raises:
            ValueError: If no ``loc`` is given and this task result is more than 0-D.
        """
```

### BitArray

To store lots of bit-array outcomes, to enable fast iteration over bitstrings, and to enable bitwise operations, we propose a container that has a contiguous memory model over the entire shot-loop, where 8-bits are packed to the byte.
For this purpose, we choose to use a NumPy `uint8` array because, unlike `bytes` or `bytearray`, it has convenient methods for slicing and indexing multiple dimensions.
Its buffer can be directly accessed for those not wishing to use NumPy, and, as seen both in the previous section and also below, we can include convenience methods for converting to, for example, a `Counts` object.
We choose a convention where the second last axis is over samples (shots), and the last dimension has a big-endian byte order which describes a single value of the output bit-array.
For example, ``np.array([[1, 1], [2, 0]], dtype=np.uint8)`` would describe 2 shots with outcomes `257 = (1 << 8) + 1` and `512 = (2 << 8) + 0` in decimal, respectively.

However, a `uint8` array alone is insufficient because, at the very least, it is incapabable of specifying whether it describes a bit-array register of size 14 or 16, for example, both of which require two bytes.
For this reason, and to handle convenience methods, we introduce the `BitArray`.
A bare bones sketch is below.
Methods like `__or__` or static constructors from other formats are in-scope of the vision, but excluded for brevity.

```python
class BitArray(ShapedMixin):
    """Stores bit-array outcomes."""

    def __init__(self, array: NDArray, num_bits: int):
        """
        Args:
            array: The data, where the last axis is over packed bits whose byte order is 
                big-endian, the second last axis is over shots, and the preceding axes 
                correspond to the shape of the experiment.
            num_bits: How many bit are in each outcome; the size of the bit-array register.
        """
        self._array = np.array(array, copy=False, dtype=np.uint8)
        if self._array.ndim < 2:
            raise ValueError("The data arary needs to be at least 2-D")
        self._num_bits = num_bits
        if self._array.shape[-1] * 8 < self._num_Bits:
            raise ValueError("The last axis of the data array is not big enough.")
        # second last dimension is shots, last dimension is packed bits
        self._shape = self._array.shape[:-2]

    def __repr__(self):
        desc = f"<num_samples={self.num_shots}, num_bits={self.num_bits}, shape={self.shape}>"
        return f"BitArray({desc})"

    @property
    def array(self) -> NDArray[np.uint8]:
        """The raw array of data."""
        return self._array

    @property
    def num_bits(self) -> int:
        """The number bits in each outcome."""
        return self._num_bits

    @property
    def num_samples(self) -> int:
        """The number of samples."""
        return self._array.shape[-2]

    def get_counts(self, loc: Tuple[int,...] | None = None, marginal_idxs: Tuple[int,...] | None = None) -> Counts:
            def get_counts(self, idx: Tuple[int, ...] | None = None) -> Counts:
        """Join all bit-array output registers together into a new counts structure.
        
        Args:
            idx: Which task coordinate to fetch.

        Raises:
            ValueError: If no ``idx`` is given and this task result is more than 0-D.
        """
```

## Migration Path <a name="migration-path"></a>

An analagous path as [the paired Estimator RFC](https://github.com/Qiskit/RFCs/pull/51). Convenience methods can be placed on result containers to transform into familiar return types.
