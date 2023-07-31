# Pulse Compiler & IR

| **Status**        | **Proposed** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Tsafrir Armon (tsafrir.armon@ibm.com),  Naoki Kanazawa (knzwnao@jp.ibm.com)    |
| **Deprecates**    | -                 |
| **Submitted**     | 2023-07-31                                   |
| **Updated**       | YYYY-MM-DD                                   |

## Summary
This RFC summarizes the proposal for new Pulse Compiler & IR. The introduction of the new compiler paves the way to the transition to frame aware model, which is also discussed. 
The proposal is based on a series of discussions
within Qiskit Pulse's development team, and is brought here for the community to weigh in. The main changes proposed include:

- Introduce new pass based Pulse Compiler and the supporting Pulse IR (intermediate representation).
- Introduce new model to supplement the existing `Channel` model, that will allow writing backend-agnostic template pulse programs.

Comments are welcome.

## Motivation
Qiskit Pulse currently has no unified compilation pathway. Different tasks are handled by the different code segments. This not only creates code clutter which is hard to maintain,
but also limits the ability of vendors to adapt Qiskit Pulse, or adjust Qiskit Pulse to new HW developments. The circuit module of Qiskit already uses a pass based compilation process,
which gives the flexibility to add\change\adapt passes according to the changing needs.

A prime example for the need for a better compilation process, is given by the second part of this proposal - rework of the `Channel` model.

The legacy `Channel`s correspond to what the backend calls a mixed frame - a combination of specific HW port, and the frame (frequency and phase) needed to play pulses.
However, to specify a Channel one must be aware of the backend mapping. For example, the following pulse schedule is an ECR pulse for qubits 3 and 4:

```
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
One can schematically split the user interface of Qiskit Pulse into three layers -

- Pulse level (`SymbolicPulse`,`WaveForm`)
- Instruction level (`Play`,`Delay`...)
- Program level (`Schedule`,`ScheduleBlock`)
This proposal focuses on the instruction level, and the necessary compilation needs that will arise.
The main goal is to allow for clear and simple backend agnostic (but not architecture agnostic) pulse templates.
To do this, one must move away from the `Channel`s setup, which can't be used without specific backend mapping.

The new Pulse IR and compiler will support the transition, and provide a unified compilation tool for every need and desired output format.

No API breaking changes are needed. New and legacy options can coexist.

## Detailed Design
### Channel rework
To replace the legacy Channels we propose to specify instructions in terms of `LogicalElement`s and `Frame`s:

- `LogicalElement` - every type of element in the HW which the user can control ("play pulses on"). For example, a qubit is a `LogicalElement`, and not the specific port used to drive it. The most notable example of a logical element is of course the `Qubit`, but in the future one can imagine dedicated couplers, flux controls and so on.
- `Frame` - a combination of frequency and phase. Subclasses used to identify the frame with a backend default. Notable examples are `QubitFrame` and `MeasurementFrame` associated with the default driving and measurement frequencies (respectively) of a qubit.
Using these objects, the above code takes the form:

```
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

The new code not only allows to work with logical backend-agnostic qubits, but is also much clearer in conveying the actual actions in play (acting on qubit 3 with the frame of qubit 4 vs with the frame of qubit 3).

On top of the `LogicalElement` and the `Frame`, another layer of `MixedFrame` will be introduced.
The `MixedFrame` is simply a combination of a LogicalElement and a Frame, and in many ways is similar to the legacy `Channel`.
The difference being that a `MixedFrame` does not depend on the backend mapping and has clear relations with other `MixedFrames` as opposed to a `Channel` which hasn't.

It should be noted that custom `Frame` and `MixedFrame` will make it easier to control qudit experiments.

### Pulse IR
The Pulse IR will provide the framework for the compiler to work on and perform the necessary compilation steps. The main components of the IR will be:

- IR Block - A block of instructions (or nested blocks) with an alignment context. This structure is similar to the structure of ScheduleBlock and needed to support one of the main tasks of the compiler - scheduling.
- IR Instruction - A pulse program instruction. Typically, it will be initialized without concrete timing, and will be scheduled during the compiler operation.

The typical attributes of the instructions will be: t0 (starting time), duration, payload (instruction details) and the target associated with the instruction (`Frame`, `MixedFrame` or the legacy `Channel`). For example:

- `MixedFrameInstruction` - target is `MixedFrame` (or `Channel`). Additional attribute operand (Play\Delay). The payload attribute will be the pulse to be played (for Play) or delay duration (for Delay).
- `FrameInstruction` - target is `Frame` (or `MixedFrame`/`Channel`). Additional attribute operand (Set\Shift Frequency\Phase). The payload will be the phase\frequency value.
- `AcquireInstruction` - target is a tuple of `Qubit` and classical registry.

### Compiler
The compiler will be built with a shared design of Qiskit's pass manager (based on PR #10474), and will thus be easily extendable to other tasks. Here we will outline the minimal requirements from the compiler in light of the new proposal.

#### Scheduling
- IR initialization - For a given ScheduleBlock the IR will initialized as nested blocks (with no concrete timing).
- Canonicalization (transform pass) - If legacy `Channel`s were used, they will be mapped to `MixedFrame`s using the backend mapping. This is the only scenario where backend mapping will be needed for this stage.
- Identify frames and mixed frames (characterization pass) - `FrameInstruction`s will need to be broadcasted to every `MixedFrame` associated with that `Frame`. Therefore, we need identify every `MixedFrame`, `Frame` and the relations between them.
- Schedule each block -
  - starting from the lowest blocks, each block will be scheduled on its own.
  - `FrameInstruction` defined on a `Frame` will be broadcasted to every associated `MixedFrame`, and these `MixedFrame`s will be considered "blocked" by this IR block.
  - `FrameInstruction` defined on a `MixedFrame` will not be broadcasted to other `MixedFrame`. This will leave the user the ability to control the frame of `MixedFrame`s individually if wanted.
  - According to the alignment, the instructions will be scheduled within the block.
  - Nested blocks will be shifted according to the alignment, but their internal timing will not be altered by parent block alignment.
- Apply backend scheduling constraints.
- Validate timing and constraints.

#### Mapping to backend aware "channels"
The `Frame`s and `MixedFrame`s will have to be converted to backend aware "channels".
It remains to be seen if this conversion will be done on the frontend or the backend, but a scheme for frontend mapping could look like this:

- First, map `MixedFrame`s which natively map to the backend. For example, a `MixedFrame` associated with `Qubit(1)` and `QubitFrame(1)` is natively mapped to the backend's Q1 "channel".
  Similarly MixedFrame associate with `Qubit(1)` and `QubitFrame(2)` is natively mapped to the backend's CR12 "channel".
- Next, map remaining `MixedFrames` into unused "channels".
- Lastly, provide the backend with `MixedFrame` which couldn't have been mapped by the frontend, in hope that the backend could support them. (This will obviously require dedicated backend support).

#### Conversion to output format
There is still ongoing discussion about desired output formats, but every choice of output format could be supported at the cost of creating a conversion mechanism from IR to that format.

## Alternative Approaches
This proposal combines two seemingly separate issues - the channel model rework and the introduction of Pulse IR and Compiler. It would be possible to do one without the other, but it seems like this is
a good opportunity to do both.

## Questions
- Naming - Names were not finalized. While not in the same namespace, some names here clash with other Qiskit modules (most notably `Qubit`), and it might be better to modify the names.
- IR instructions - The existing `Instruction` class provides similar functionality. Should we use it instead of introducing new IR Instruction classes?

## Future Extensions
- Frontend & backend coordination to support custom frames and mixed frames. This has the potential to better utilize HW control options (as discussed above).
