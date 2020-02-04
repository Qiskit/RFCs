# Unify Pulse Commands and Instructions

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Thomas Alexander (talexander@ibm.com)       |
| **Deprecates**    | None                 |
| **Submitted**     | 2020-02-04                                   |
| **Updated**       | 2020-02-04                                   |

## Summary
`Command`s were initially created so as to allow pulses to be only defined once as a `SamplePulse` and then have their usage tracked based on the class instance through equality checking. To enable this, every `Instruction` instance was defined as containing a `Command` and its `Channel` operands that the command would be applied to. This resulted in a confusing API as one must first define a command and then call it with a channel to emit an instruction. We we propose a way of gracefully unifying `Command`s and `Instruction`s to make pulse programming more straightforward and in-line with traditional instruction sets.

## Motivation
The duplication of `Command`s and `Instruction`s has resulted in a confusing API with many statements that look like:
```python
Command(...)(channel1, channel2, ...)
```
which yields an instruction. This can be created more explicitly with:
```python
Instruction(Command(...), channel1, channel2, ...)
```

If we were to unify commands with instructions a generic instruction would accept a list of operands that could be either a numeric parameter like a `float` or a `Channel` to operate on,
eg.:
```python
Instruction(operand1, operand2, ...)
```

For example, a frame change would go from,
```python
FrameChange(0.0)(DriveChannel(0))
```

or more explicitly,
```python
FrameChangeInstruction(FrameChange(0.0), DriveChannel(0))
```

to
```python
ShiftPhase(0.0, DriveChannel(0))
```

where `ShiftPhase` is of type `Instruction`.

The desired behavior of reusing a single pulse multiple times could still be retained by defining a `Pulse` type and defining a `Play` instruction that would accept a `Pulse` and a `Channel`:

```python
random_pulse = SamplePulse(np.random.random(10))
Play(random_pulse, DriveChannel(0))
```

This will yield a much cleaner pulse programming API and be more in lines with traditional compiler representations. The API will no longer have odd looking statements like `FrameChange(0.0)(DriveChannel(0))` but may still retain syntactic sugar for statements like:
```python
x90 = gaussian(...)
x90(DriveChannel(0))
```

## User Benefit
This will be both an enhancement for:
- Pulse programmers as there will be more standard syntax for scheduling pulses.
- Codebase contributors and maintainers as there will be fewer classes to maintain and understand.

## Design Proposal
The current root pulse `Instruction` has both a `command` attribute which holds a `Command` and a tuple of `channels` to which it applies.
We will remove the current pulse `Instruction` and replace it with an implementation that is a class with an `operands` attribute that contains all operands that the instruction acts on, including both pulses and constants such as floats.

The `Command` will be deprecated. A `Pulse` type will be defined and all pulses will inherit from this.

`Command`s and `Instruction`s will be combined into a new `Instruction` as shown below:
| `Command`      | old `Instruction`         | new `Instruction` | new Operands                                                                                                                 |
|----------------|---------------------------|-------------------|------------------------------------------------------------------------------------------------------------------------------|
| `PulseCommand` | `PulseInstruction`        | `Play`            | `pulse:Pulse, channel: PulseChannel`                                                                                         |
| `Delay`        | `DelayInstruction`        | `Delay`           | `duration: int, channel: Channel`                                                                                            |
| `FrameChange`  | `FrameChangeInstruction`  | `ShiftPhase`      | `phase: float, channel: PulseChannel`                                                                                        |
| N/A            | N/A                       | `SetPhase`        | `phase: float, channel: PulseChannel`                                                                                        |
| N/A            | N/A                       | `ShiftFrequency`  | `frequency: float, channel: PulseChannel`                                                                                    |
| `SetFrequency` | `SetFrequencyInstruction` | `SetFrequency`    | `frequency: float, channel: PulseChannel`                                                                                    |
| `Acquire`      | `AcquireInstruction`      | `Acquire`         | `duration: int, register: Union[MemorySlot, RegisterSlot], kernel: Optional[Kernel], discriminator: Optional[Discriminator]` |
| `Snapshot`     | `Snapshot`                | `Snapshot`        | `label: str, snapshot_type: str`                                                                                             |

### Example of pulse programming with changes
```python
x90 = gaussian(duration, amp, sigma)
meas_stim = gaussian_square(duration, amp, sigma, width)

sched = Schedule()
# we will keep callable shorthand for pulses
sched += x90(DriveChannel(0))
sched += Delay(100, DriveChannel(0))
# or we can generate a Play instruction explicitly
sched += Play(x90, DriveChannel(0))
# measurements may be scheduled with delay
sched += Delay(200, AcquireChannel(0))
sched += Play(meas_stim, MeasureChannel(0))
# or shifting in time
sched += Acquire(100, AcquireChannel(0), MemorySlot(0)) << 200
# We also have instructions to modify frequency and phase
sched += SetFrequency(qubit_freq, DriveChannel(0))
sched += ShiftFrequency(0.1, DriveChannel(0))
sched += ShiftPhase(np.pi, DriveChannel(0))
sched += SetPhase(0.0, DriveChannel(0))
```

### Deprecation Path
The `Command`s will be gracefully deprecated with the procedures defined in the table below:
| Command        | Deprecation Path                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
|----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Instruction`  | This is not a user-facing class from an API interface perspective. This will be rewritten as per the outline in the detailed design section. The `name` parameter will be deprecated.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `PulseCommand` | Will be converted to a `Pulse` and all pulse classes will inherit from this. Calling a `Pulse` with a `Channel` as input will now yield an instruction of the form `Play(pulse, channel)` this will enable the same syntax for instructions to be kept.                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `Delay`        | Will be converted from a `Command` to a new `Instruction` type. For the deprecation period the `Channel` argument will be made optional but if not supplied will emit a warning and a `None` type will be set for the second operands attribute. Calling the instruction with a `Channel` will yield a new instruction with the parameters of the original instruction and the additional input `channel` operands being the input channel. If the instruction with a `channel` attribute of `None` makes it to assembly, an error shall be raised.                                                                                                                                                                               |
| `FrameChange`  | The new instruction (`ShiftPhase`) will have a different name. Initializing a `FrameChange` will be deprecated and emit a warning, calling the command to convert it to an instruction will be modified to yield the new version of the `Instruction`, `ShiftPhase`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `SetFrequency` | Will be converted from a `Command` to a new `Instruction` type. For the deprecation period the `Channel` argument will be made optional but if not supplied will emit a warning and a `None` type will be set for the second operands attribute. Calling the instruction with a `Channel` will yield a new instruction with the parameters of the original instruction and the additional input `channel` operands being the input channel. If the instruction with a `channel` attribute of `None` makes it to assembly, an error shall be raised.                                                                                                                                                                               |
| `Acquire`      | Will be converted from a `Command` to a new `Instruction` type. For the deprecation period the `Channel` argument will be made optional but if not supplied will emit a warning and a `None` type will be set for the second operands attribute. Calling the instruction with a `Channel` will yield a new instruction with the parameters of the original instruction and the additional input `channel` operands being the input channel. If the instruction with a `channel` attribute of `None` makes it to assembly, an error shall be raised. `reg_slot` and `mem_slot` will be merged, into `register` the kwargs `reg_slot` and `mem_slot` will be gracefully deprecated. If both are supplied an error will be returned. |
| `Snapshot`     | Will remain the same.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |

Communication of changes to users will be performed with a reno entry for the changes and deprecation warnings for deprecated components.

## Detailed Design

### Classes
#### Pulse

```python
class Pulse(ABC):

  @property
  @abstractmethod
  def duration(self) -> int:
    ...

  @property
  @abstractmethod
  def name(self) -> str:
    ...

  def __call__(self, channel: PulseChannel) -> Play:
    return Play(self, channel)
```

#### Instruction
```python
class Instruction(ABC):
  def __init__(self, *operands) -> Instruction:
    ...

  @property
  @abstractmethod
  def duration(self) -> int:
    ...

  @property
  def operands(self) -> Tuple[Any, ...]:
    return operands

  @property
  @abstractmethod
  def channels -> Tuple[Channel, ...]:
    ...
```

#### SamplePulse
```python
class SamplePulse(Pulse):
  # otherwise unchanged
  ...
```

#### `ParametricPulse`
```python
class ParametricPulse(Pulse):
  # otherwise unchanged
  ...
```

#### Play
```python
class Play(Instruction):
  def __init__(self, pulse: Pulse, channel: PulseChannel) -> Play:
    super().__init__(pulse, channel)

  @property
  def duration(self) -> int:
    return pulse.duration

  @property
  def channels:
    return (channel,)

  @property
  def pulse(self) -> Pulse:
    return pulse

  @property
  def pulse_name(self) -> str:
    return pulse.name
```

#### ShiftPhase
```python
class ShiftPhase(Instruction):
  def __init__(self, phase: float, channel: PulseChannel) -> ShiftPhase:
    super().__init__(phase, channel)

  @property
  def duration(self) -> int:
    return 0

  @property
  def channels:
    return (channel,)

  @property
  def phase(self) -> float:
    return phase
```

#### SetPhase
```python
class SetPhase(Instruction):
  def __init__(self, phase: float, channel: PulseChannel) -> SetPhase:
    super().__init__(phase, channel)

  @property
  def duration(self) -> int:
    return 0

  @property
  def channels:
    return (channel,)

  @property
  def phase(self) -> float:
    return phase
```

#### SetFrequency
```python
class SetFrequency(Instruction):
  def __init__(self, frequency: float, channel: PulseChannel) -> SetFrequency:
    super().__init__(frequency, channel)

  @property
  def duration(self) -> int:
    return 0

  @property
  def channels:
    return (channel,)

  @property
  def frequency(self) -> float:
    return frequency
```

#### ShiftFrequency
```python
class ShiftFrequency(Instruction):
  def __init__(self, frequency: float, channel: PulseChannel) -> ShiftFrequency:
    super().__init__(frequency, channel)

  @property
  def duration(self) -> int:
    return 0

  @property
  def channels:
    return (channel,)

  @property
  def frequency(self) -> float:
    return frequency
```

#### Acquire
```python
class Acquire(Instruction):
  def __init__(self, duration: float, channel: AcquireChannel,
               register: Union[MemorySlot, RegisterSlot],
               kernel: Optional[Kernel], discriminator: Optional[Discriminator]) -> Acquire:
    super().__init__(duration, channel, register,
                     kernel, discriminator)

  @property
  def duration(self) -> int:
    return duration

  @property
  def channels(self):
    return (channel, register)

  @property
  def kernel(self) -> Kernel:
    return kernel

  @property
  def discriminator(self) -> Discriminator:
    return discriminator
```

#### Snapshot
```python
class Snapshot(Instruction):
  def __init__(self, label: str, snapshot_type: str) -> Snapshot:
    super().__init__(label, snapshot_type)

  @property
  def label(self):
    return label

  @property
  def snapshot_type(self):
    return snapshot_type

  @property
  def duration(self):
    return 0

  @property
  def channels:
    return (SnapshotChannel(0),)
```
## Alternative Approaches
At this time the alternative approach is to leave the API as is and continue to have both `Command`s and `Instruction`s. This would allow some useful features such as being able to define reusable fixed `FrameChange`s but this comes at the cost of a cumbersome user interface.

## Questions
- Have all possible breaking changes for the current interface been covered? If so do graceful deprecation paths exist?

## Future Extensions
- Defining an accompanying textual representation (language) for the instruction set defined within this RFC should be explored. This will enable a convenient interface for textual interface pulse programming and also provide an on-disk storage format.
- Explore the possibility of creating a builder interface with Python within a Python context that generates the underlying schedule data structure. This could make pulse programming much easier and would like a Python DSL for pulse programming rather than building up a data structure as pulse programming is currently done.
