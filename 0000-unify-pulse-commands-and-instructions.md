# RFC Title

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Thomas Alexander (talexander@ibm.com)       |
| **Deprecates**    | None                 |
| **Submitted**     | 2020-02-04                                   |
| **Updated**       | 2020-02-04                                   |

## Summary
Pulse `Command`s were initially created to allow the pulse to be only defined once as a SamplePulse and then have its usage tracked based on the pulse instance. Every `Instruction` instance was then defined as containing a `Command` and its `Channel` operands that the command would be applied to.

## Motivation
The duplication of `Command`s and `Instruction`s has resulted in a confusing API with many statements that look like:
```python
PulseCommand(...)(channel1, channel2, ...)
```
This yields an instruction. This can be created more explicitly with:
```python
PulseInstruction(PulseCommand(...), channel1, channel2, ...)
```

If we were to unify commands with instructions a generic instruction would accept a list of operands that could be either a numeric parameter like a complex or a channel to operate on,
eg.:
```python
PulseInstruction(operand1, operand2, ...)
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

The desired behavior of reusing a single pulse multiple times could then be retained by defining a `Pulse` type and defining a `Play` instruction that would accept a `Pulse` and a `Channel`:

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
This is the focus of the document. Explain the proposal from the perspective of
educating another user on the proposed features.

This generally means:
- Introducing new concepts and nomenclature
- Using examples to introduce new features
- Implementation and Migration path with associated concerns
- Communication of features and changes to users

Focus on giving an overview of impact of the proposed changes to the target
audience.

Factors to consider:
- Performance
- Dependencies
- Maintenance
- Compatibility

## Detailed Design
Technical reference level design. Elaborate on details such as:
- Implementation procedure
  - If spans multiple projects cover these parts individually
- Interaction with other features
- Dissecting corner cases
- Reference definition, eg., formal definitions.

## Alternative Approaches
Discuss other approaches to solving this problem and why these were not
selected.

## Questions
- Can we tabulate all possible breaking changes for the current interface. If so do graceful deprecation paths exist?

## Future Extensions
- Defining an accompanying textual representation (language) for the instruction set defined within this RFC should be explored. This will enable a convenient interface for textual interface pulse programming and also provide an on-disk storage format.
- Explore the possibility of creating a builder interface with Python within a Python context that generates the underlying schedule data structure. This could make pulse programming much easier and would like a Python DSL for pulse programming rather than building up a data structure as pulse programming is currently done.
