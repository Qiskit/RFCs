# [WIP] [RFC] Scheduled Circuit

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Chris Wood (cjwood@us.ibm.com), Toshinari Itoko (itoko@jp.ibm.com), Ali Javadi (Ali.Javadi@ibm.com)    |
| **Deprecates**    | NA                 |
| **Submitted**     | 2020-04-DD                                   |
| **Updated**       | YYYY-MM-DD                                   |

## Summary
Add "scheduled circuit" in-between “gate circuit”(QuantumCircuit) and “pulse schedule”(pulse.Schedule)
to make time-aware circuit-based noise simulation possible.

## Motivation
Currently there is no way to instruct `delay` (or `wait`/`idle`) for a specific duration in circuit level.
This prevents from easy noise simulation of experiments with `delay`s, e.g. T1/T2 measurement, on Aer.
A new circuit class for incorporating with instructions having `duration` should be added,
as well as scheduling methods to construct it from a QuantumCircuit.

### Main use cases
1) accurate simulation of decoherence in Aer (T1&T2, Cross-talk)
2) insert gates in idle timeslots

## User Benefit
- Time-aware circuit-based noise simulation on Aer is enabled.
- Scheduling in circuit level will enable more optimization than that in pulse level in the future.

## Design Proposal
### High-level API:
1. Noise simulation flow (new):
```
timed_circuit = circuit.schedule(transpiled_circuit, ...)  # transpiled_circuit: QuantumCircuit
qobj = assemble(timed_circuit)  # timed_circuit: ScheduledCircuit
```

2. Circuit-to-Schedule flow (update):
```
schedule = scheduler.circuit_schedule(transpiled_circuit, ...)  # transpiled_circuit: QuantumCircuit
```
->
```
timed_circuit = circuit.schedule(transpiled_circuit, ...)  # transpiled_circuit: QuantumCircuit
schedule = pulse.build_schedule(timed_circuit, ...)  # timed_circuit: ScheduledCircuit
```

### Requirements:
- It needs to mimic the scheduling that would be run on a real device as closely as possible
- It needs serialize to standard qasm qobj, but with a `duration` (or `gate_length`) field to each instruction
   all qubits should have an instruction assigned to them at all times so:
- It needs to insert `delay`/wait/idle instructions on any qubit that is idle
   (with the duration saying how long it is idle for)
- It needs to support some interfaces for inserting gates to idle timeslots.
- [For future (cross-talk simulation on Aer)] It may need to support optional insertion of all-qubit time-step instructions at each layer,
   which would also serialize to a qasm qobj instruction with a duration value given by the total time of the time-step layer.


## Detailed Design
Technical reference level design. Elaborate on details such as:
- Implementation procedure
  - If spans multiple projects cover these parts individually
- Interaction with other features
- Dissecting corner cases
- Reference definition, eg., formal definitions.

### Example code
```
def schedule(qc: QuantumCircuit,
             backend_props: BackendProperties,
             method: str = "alap",
             dependency: Optional[DependencyGraph] = None) -> ScheduledCircuit:
    """
    Args:
        qc: a transpiled circuit to be scheduled.
        backend_props: to get gate_length
        method: same as in pulse.scheduler.schedule_circuit
        dependency: (for future) asap and alap can work without this
    Returns:
        The scheduled circuit.
    """
    if dependency is None:
        # schedule based on the standard dependency that is the same as used in DAGCircuit
        # algorithm is almost the same as in pulse.scheduler.methods.algorithm
    else:
        # scheudle based on the extended dependency for the future


Qubit = int # assume only physical qubits
ArgumentedInstruction = Tuple[Instruction, List[Qubit], List[Clbit]]
Duration = float
LengthedInstruction = Tuple[ArgumentedInstruction, Duration]
LengthedInstructions = Union[LengthedInstruction, Iterable[LengthedInstruction]]
StartTime = float
TimedInstruction = Tuple[StartTime, LengthedInstruction]
Index = int
Position = Tuple[Qubit, Index]
Positions = Union[Position, Iterable[Position]]


class ScheduledCircuit:
    """Circuit with timings for each instruction."""

    def __init__(self, name=None):
        # TBD

    def append(self,
               target: LengthedInstruction,
               buffer_duration: Duration = 0):
        """Mutably append the target instruction after the latest instruction on its acting qubits.
        For all the acting qubits, time-slots are filled with delay/idle instructions until the end time of the latest one.
        If `buffer_duration` > 0, additional delay/idle instruction is also appended before the target instruction.
        """

    def instructions(self) -> List[LengthedInstruction]:
        """Instructions sorted by their start time."""

    def makespan(self, *qubits: int):
        """The lastest end time of instructions on the qubits."""

    def filter(self,
               instruction_type: Instruction,
               *qubits: Qubit,
               unique: bool = True) -> List[LengthedInstruction?Position?]:
        """Return instructions with only the instructions from this ``ScheduledCircuit`` which pass
        though the provided properties; i.e. an instruction will be retained iff
        the instruction type is contained in ``instruction_types`` and
        occurs on a qubit contained in ``qubits`` (No qubits means *all* qubits).
        If unique is False, multi-qubit instrucsions occurs multiple times in the result.
        """

    def replace(self,
                reference: LengthedInstructions?Positions?,
                *targets: LengthedInstructions?Positions?):
        """Mutably replace the reference instruction just before the reference instruction.
        Raise CircuitError if total duration of the targets does not equals to the duration of reference.
        """

    def insert(self,
               reference: LengthedInstruction?Positions?,
               target: LengthedInstruction?Positions?,
               shift: bool = False):
        """Mutably insert the target instruction just before the reference instruction.
        If shift is True, all the following instructions will be shifted by the duration of the target.
        If shift is False, raise CircuitError if no enough idle time before the reference instruction.
        """

    # """Ordered instructions on a specified qubit."""
    # def instructions_on(self, qubit_index: int) -> List[LengthedInstruction]:


def build_schedule(timed_circuit: ScheduledCircuit,
                   inst_map: InstructionScheduleMap,
                   ...) -> pulse.Schedule:
    """Build schedule by replacing instructions in timed_circuit with schedules defined in inst_map.
    """
```

## Alternative Approaches
### float time vs int time:
- float (seconds) = gate time in BackendProperties
- int (number of cycle time) = pulse duration is defined by samples with dt or dtm
  - -> float time for now (convert float time into int time in pulse.build_schedule())
  - -> maybe int time in the future after pulse supports "cycle time" (unified dt)

### Relatively-timed vs Absolutely-timed data structure
Time-complexity comparison between Relatively-timed and Absolutely-timed data structure for mutable ScheduledCircuit
assuming flat instructions (no nest of circuit) is provided below.

It is found that Relatively-timed can be seen “lazy evaluation of start times”.
In all of append/replace/insert cases, Relatively-timed(REL) has better time-complexity than Absolutely-timed(ABS) by O(N), which is equivalent to the time to update start times. Note that, within schedulers, start times are usually tracked. So as for path using scheduler, REL just forget start times before its construction. In that sense, the effect of lazy evaluation may be somewhat mitigated.
On the other hand, ABS is better than REL for insert_at_time() (or find_at_time()). This is because start_time is implicitly required to find instructions at a specific time.
In summary, we can say which (REL or ABS) is better depends on how often insert/find_at_time() is required in use cases.

See also [[RFC] Qiskit Pulse: Relative timing for Schedules](https://github.com/Qiskit/rfcs/pull/16).

Notations:
```
D = self.depth (max number of instructions by qubit in self)
N = other.size (number of instructions in other (or targets))
W = other.width (number of qubis in other (or targets))
```

Relatively-timed data structure:
```
_inst_by_qubit: Dict[Qubit, List[LengthedInstruction]]
_end_time_by_qubit: Dict[Qubit, TimeType] # for speed up append()
```

Absolutely-timed data structure:
```
_timed_inst_by_qubit: Dict[Qubit, List[TimedInstruction]] # lists are sorted by (start time, duration)
```
1) append(inst: Union[LengthedInstruction, ScheduledCircuit])
- Relatively-timed:
  - O(W): fill last idle time-slot with delays
  - O(N): extend lists by qubit
  - O(W): update end time dictionary
- Absolutely-timed:
  - O(W): fill last idle time-slot with delays
  - O(N): extend lists by qubit
  - O(N): update start times

2) replace(refs: Positions, targets: LengthedInstructions)*
    - fail if refs and targets has different duration
- Relatively-timed:
  - O(N): check validity of arguments
  - O(ListReplace(D)*W): replace items on lists by qubit
- Absolutely-timed:
  - O(N): check validity of arguments
  - O(N): update start times
  - O(ListReplace(D)\*W): replace items on lists by qubit

3) insert(ref: Position, targets: LengthedInstructions)*
    - fail if no enough idle time before the reference instruction
- Relatively-timed:
  - O(N): check validity of arguments
  - O(ListInsert(D)*W): insert items on lists by qubit
- Absolutely-timed:
  - O(N): check validity of arguments
  - O(N): update start times
  - O(ListInsert(D)*W): insert items on lists by qubit

4) insert_at_time(time: Time, targets: LengthedInstructions)
    - fail if no enough idle time between the time and start time of next instruction
- Relatively-timed:
  - O(D*W): overlap check
  - O(ListReplace(D)*W): replace items on lists by qubit
- Absolutely-timed:
  - O(log(D)*W): overlap check
  - O(N): update start times
  - O(ListReplace(D)*W): replace items on lists by qubit

In addition to these, we need kinds of “find()” family for users to specify references in insert/replace etc.

\*) I assume the interface that references are given as Postions. If references should be given as Instructions, we need to accept overhead to find positions. 
Such overhead would be O(D*W) for Relatively-timed and O(log(D)*W) for Absolutely-timed unless we add another data structure for the purpose.


## Questions
- any other use case? noise simulation with custom gate (with custom duration and noise)?

- need to support "insert_at_time()"?
- reference can be "position" or should be "instruction"?
- naming:
  - delay/wait/idle ("delay" is used in pulse module)
  - duration/gate_length/length ("duration" is used in pulse module)
- mutable / immutable

## Future Extensions
How to enhance ScheduledCircuit to enable (rough but easy) cross-talk simulation on Aer.
