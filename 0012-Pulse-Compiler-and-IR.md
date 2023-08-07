# Pulse Compiler & IR

| **Status**        | **Proposed** |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0012                                         |
| **Authors**       | Tsafrir Armon (tsafrir.armon@ibm.com),  Naoki Kanazawa (knzwnao@jp.ibm.com)    |
| **Deprecates**    | -                 |
| **Submitted**     | 2023-07-31                                   |
| **Updated**       | 2023-08-07                                   |

## Summary
This RFC summarizes the proposal for new Pulse Compiler & IR. The introduction of the new compiler paves the way to the transition to frame aware model, which is also discussed. 
The proposal is based on a series of discussions
within Qiskit Pulse's development team, and is brought here for the community to weigh in. The main changes proposed include:

- Introduction of a new pass based Pulse Compiler and the supporting Pulse IR (intermediate representation).
- Introduction of a new model to supplement the existing `Channel` model, that will allow writing backend-agnostic template pulse programs.

Comments are welcome.

## Motivation
Qiskit Pulse currently has no unified compilation pathway - Different tasks are handled by the different code segments. This not only creates code clutter which is hard to maintain,
but also limits the ability of vendors to adapt Qiskit Pulse, or adjust Qiskit Pulse to new HW developments. The circuit module of Qiskit already uses a pass based compilation process,
which gives the flexibility to add\change\adapt passes according to the changing needs.

A prime example for the need for a better compilation process, is given by the second part of this proposal - rework of the `Channel` model.

The legacy `Channel`s correspond to what the backend calls a mixed frame - a combination of specific HW port, and the frame (frequency and phase) needed to play pulses.
However, to specify a Channel one must be aware of the backend mapping. For example, the following pulse schedule is an ECR pulse for qubits 3 and 4:

```python
with builder.build() as ecr_schedule_q3q4:
    with align_sequential():
        with align_left():
            Play(GaussianSquare(...), ControlChannel(6))
            Play(GaussianSquare(...), DriveChannel(4))
        Play(Gaussian(...), DriveChannel(3))
        with align_left():
            Play(GaussianSquare(...), ControlChannel(6))
            Play(GaussianSquare(...), DriveChannel(4))
```

Note how one can't make sense of the code, without knowing the mapping of the control channels.
Similarly, frame synchronization also has to be carried out manually and depends on the backend mapping.
This dependency on the backend prohibits the ability to write a template code, which in turn complicates modules like Qiskit Experiments.

Under the new proposal, the dependency on backend mapping will be removed, and Qiskit Pulse users (and particularly Qiskit Experiments users) will be able to write
backend-agnostic template programs, which could be used across devices. Additionally, custom frames will make it easier to experiment with qudit control.
However, supporting this change adds significant compilation needs, which are hard to implement in the current workflow.

## User Benefit
- Qiskit contributors as well as vendors will have a unified compilation code which will be simpler to adjust for future needs and changes.
- Qiskit Pulse users will have a clearer and more efficient way to write their Pulse programs.
- Qiskit Experiments users will be able to write more streamlined template experiments.
- Qiskit Pulse users will have easier access to qudit control.

## Design Proposal
The new pulse compiler will be a pass based compiler. Dedicated passes will perform every analysis,
transformation and validation operations needed to convert a pulse program into a desired output format. A detailed workflow will be described below, but the main tasks include
concretely scheduling `ScheduleBlock` programs, mapping virtual qubits to physical ones, applying backend constraints, and converting to desired output format. To support this,
the new IR will consist of nested instruction blocks, which will be transformed by the compiler as it operates.

Regarding the `Channel` model rework, one can schematically split Qiskit Pulse's user interface into three layers -

- Pulse level (`SymbolicPulse`,`WaveForm`)
- Instruction level (`Play`,`Delay`...)
- Program level (`Schedule`,`ScheduleBlock`)
This rework focuses on the instruction level. The main goal is to allow for clear and simple backend agnostic (but not architecture agnostic) pulse templates.
To do this, the legacy `Channel`s will be supplemented by `Frame`s and `LogicalElement`s.

No API breaking changes are needed. New and legacy options can coexist.

## Detailed Design
### Channel rework
To replace the legacy `Channel`s we propose to specify instructions in terms of `LogicalElement`s and `Frame`s:

- `LogicalElement` - every type of element in the HW which the user can control ("play pulses on"). For example, a qubit is a `LogicalElement`, and not the specific port used to drive it. The most notable example of a logical element is of course the `Qubit`, but in the future one can imagine dedicated couplers, flux controls and so on.
- `Frame` - a combination of frequency and phase. Subclasses used to identify the frame with a backend default. Notable examples are `QubitFrame` and `MeasurementFrame` associated with the default driving and measurement frequencies (respectively) of a qubit.

Using these objects, the above code takes the form:

```python
with builder.build() as ecr_schedule_q3q4:
    with align_sequential():
        with align_left():
            Play(GaussianSquare(...), Qubit(3),QubitFrame(4))
            Play(GaussianSquare(...), Qubit(3),QubitFrame(3))
        Play(Gaussian(...), Qubit(3),QubitFrame(3))
        with align_left():
            Play(GaussianSquare(...), Qubit(3),QubitFrame(4))
            Play(GaussianSquare(...), Qubit(3),QubitFrame(3))
```

The new code not only allows to work with virtual backend-agnostic qubits, but is also much clearer in conveying the actual actions in play (acting on qubit 3 with the frame of qubit 4 vs with the frame of qubit 3).

On top of the `LogicalElement` and the `Frame`, another layer of `MixedFrame` will be introduced.
The `MixedFrame` is simply a combination of a `LogicalElement` and a `Frame`, and in many ways is similar to the legacy `Channel`.
The difference being that a `MixedFrame` does not depend on the backend mapping and has clear relations with other `MixedFrames` as opposed to a `Channel` which hasn't.
Play\Delay instructions will be allowed to take one of `LogicalElement`+`Frame`, `MixedFrame` or `Channel` (for backwards compatibility).
Set\Shift Frequency\Phase will be allowed to take one of `Frame` (to be broadcast to every associated `MixedFrame`), `MixedFrame` (not to be broadcasted) or `Channel`.

It should be noted that custom `Frame` and `MixedFrame` will make it easier to control qudit experiments.

Classes hierarchy:

```python
class LogicalElement(ABC):
    @property
    def index(self) -> Tuple[int, ...]
        # index

    @property
    @abstractmethod
    def name(self) -> str:
        # opcode/string identifier

class Qubit(LogicalElement):
    
    def __init__(self, index: int):
        ....

    @property
    def index(self) -> int
        return self._index

    @property
    def name(self) -> str:
        return f"Q{index}"

class Coupler(LogicalElement):
    
    def __init__(self, index: Tuple[int, int]):
        self._index = index

    @property
    def index(self) -> Tuple[int, int]
        return self._index

    @property
    def name(self) -> str:
        qubits = ",".join(self._index)
        return f"Coupler({qubits})"

class Frame(ABC):

    @property
    def name(self) -> str:
        # opcode/string identifier

class GenericFrame(Frame):
    def __init__(self, name, frequency, phase):
        self._name = name
        self._frequency = frequency
        self._phase = phase
    
    @property
    def name(self) -> str:
        return self._name

    @property
    def frequency(self) -> float: # The initial frequency of the frame

    @property
    def phase(self) -> float: # The initial phase of the frame


class QubitFrame(Frame):
    def __init__(self, index):
        self._index = index
     
    @property
    def index(self) -> int
        # qubit index

    @property
    def name(self) -> str:
        return f"QFrame{self.index}"

class MeasurementFrame(Frame):
    def __init__(self, index):
        self._index = index
     
    @property
    def index(self) -> int
        # qubit index

    @property
    def name(self) -> str:
        return f"MFrame{self.index}"

class MixedFrame():
    def __init__(self, logical_element: LogicalElement, frame: Frame):
        self._logical_element = logical_element
         self._frame = frame

    @property
    def logical_element(self) -> LogicalElement
        return self._logical_element

    @property
    def frame(self) -> Frame
        return self._frame
```

### Pulse IR
The Pulse IR will provide the framework for the compiler to work on and perform the necessary compilation steps. The main components of the IR will be:

- IR Block - A block of instructions (or nested blocks) with an alignment context. This structure is similar to the structure of `ScheduleBlock` and is needed to support one of the main tasks of the compiler - scheduling.
- IR Instruction - A pulse program instruction. Typically, it will be initialized without concrete timing, and will be scheduled during the compiler operation.

Two instructions types are used:
- `GenericInstruction` - for play, delay, set\shift frequency\phase. Assignment of `LogicalElement` will be optional for set\shift frequency\phase and will determine whether or not the instruction will be
broadcaseted to all mixed frames associated with the frame or not.
- `AcquireInstruction` - for acquire instructions (due to the different elements involved).

```python
class PulseIR:
    def __init__(self):
        self.instructions # List of instructions or nested PulseIR objects
        self.alignment  

class GenericInstruction:
    def __init__(
        self,
        instruction_type: str, 
        duration: int, 
        logical_element: Optional[LogicalElement] = None, 
        frame: Optional[Frame] = None, 
        **operands,
    ):  
        self.t0 = None
        self.instruction = instruction_type  # opcode
        self.duration = duration
        self.logical_element = logical_element
        self.frame = frame
        self.operands = operands
```

### Compiler
The compiler will be built with a shared design of Qiskit's pass manager (based on PR #10474), and will thus be easily extendable to other tasks. The pass based approach allows for simple modification of the lowering process for vendors or Qiskit team. 
A typical compiler call will look like:
```python
payload = compile(my_pulse_prog, backend, target="pulse_qobj")
```
where the backend is provided to accomodate the compiled program to the backend constraints and mappings. An optional argument will be a mapping between virtual and physical qubits, as under the new model all indices will be assumed virtual. If no mapping is provided, the trivial one will be used. The mapping will be done before all other operations, during the initialization of the IR.

The first step of every compiler run will be to initialize the IR of the pulse program. The IR will be initialized with no concrete timing, and the scheduling will be carried out next.

Next we highlight some of the tasks handled by the compiler.

#### Scheduling
- Canonicalization (transform pass) - If legacy `Channel`s were used, they will be mapped to `MixedFrame`s using the backend mapping. This is the only scenario where backend mapping will be needed for this stage.
- Collect frames and mixed frames (analysis pass) - Shift\set frequency\phase instructions on a `Frame` will need to be broadcasted to every `MixedFrame` associated with that `Frame`. Therefore, we need collect every `MixedFrame`, `Frame` and the relations between them.
- Schedule each block -
  - Starting from the lowest blocks, each block will be scheduled on its own.
  - Set\shift frequency\phase instructions will be broadcasted when necessary, and the additional mixed frames will be added to the block.
  - According to the alignment, the instructions will be scheduled within the block.
  - Nested blocks will be shifted according to the alignment, but their internal timing will not be altered by parent block alignment.
- Apply backend scheduling constraints.
- Validate timing and constraints.

#### Mapping to backend aware "channels"
For a target format consumed by a backend, the `Frame`s and `MixedFrame`s will have to be converted to backend aware "channels" (for a lack of a better generic term).
It remains to be seen if this conversion will be done on the frontend or the backend, but a scheme for frontend mapping could look like this.

- Consume the backend accepted channels (For example, "D0" is the driving mixed frame of qubit 0 while "CR12" is the cross resonance mixed frame of qubit 1 in the frame of qubit 2.).
- First, map `MixedFrame`s which natively map to the backend. For example, a `MixedFrame` associated with `Qubit(1)` and `QubitFrame(1)` is natively mapped to the backend's "D1" "channel" in the example above.
  Similarly MixedFrame associate with `Qubit(1)` and `QubitFrame(2)` is natively mapped to the backend's "CR12" "channel".
- Next, map remaining `MixedFrames` into unused "channels".
- Lastly, provide the backend with `MixedFrame`s which couldn't have been mapped by the frontend, in hope that the backend could support them. (This will obviously require dedicated backend support).

#### Conversion to output format
There is still ongoing discussion about desired output formats, but every choice of output format could be supported at the cost of creating a conversion mechanism from IR to that format.
One can imagine various tasks associated with this step:
- Conversion of `SymbolicPulse` to `Waveform` for non-backend-supported pulses.
- Creation of pulse dictionary to reduce payload memory foot print.

#### Optimizations
While pulse level programs are low level, one might consider optimization passes, like dynamical decoupling addition. The pass based compiler will allow such optimization passes to be created and applied.

## Alternative Approaches
This proposal combines two seemingly separate issues - the channel model rework and the introduction of Pulse IR and Compiler. It would be possible to do one without the other, but it seems like this is
a good opportunity to do both.

## Questions
- Naming - Names were not finalized. While not in the same namespace, some names here clash with other Qiskit modules (most notably `Qubit`), and it might be better to modify the names.
- IR instructions - The existing `Instruction` class provides similar functionality. Should we use it instead of introducing new IR Instruction classes?

## Future Extensions
- Frontend & backend coordination to support custom frames and mixed frames. This has the potential to better utilize HW control options (as discussed above).
- Introuction of new optimizations at the pulse level programing.
