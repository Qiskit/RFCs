# Qiskit Pulse: Relative timing for Schedules

| **Status**        | **Proposed** |
|:------------------|:---------------------------------------------|
| **RFC #**         | TBD                                          |
| **Authors**       | Lauren Capelluto                             |
| **Submitted**     | 3/11/20                                      |
| **Updated**       | ---                                          |


## Summary
Update the internal structure of `Schedule`s to track instructions sequentially. `Instruction`s will not be played at an explicit time; rather, they will begin immediately after the last is done. To schedule idle time between instructions, `Delay` instructions must be used.

The user API will be updated to take advantage of this relative timing, with several new methods: `barrier`, `left_align`, `center_align`, and `right_align` (see: [new methods](#new-schedule-methods)).

## Motivation
(This section was copied from [this issue](https://github.com/Qiskit/qiskit-terra/issues/3749), written by @taalexander.)
Currently, all `pulse.Instructions` are stored in a `Schedule` with a two-tuple containing the `(start_time, Instruction)`. While this makes it clear the explicit starting time of every pulse it leads to convoluted internal data structures such as `TimeslotCollection` for tracking the time of all instructions.

It also encourages inefficient operations such as taking the union of two schedules. At first glance this operation should be efficient since all one needs to do is specifying the starting time of the instructions, but this requires verification that no instructions overlap in time in the two schedules resulting in expensive checks. The requirement for such a check to merge two schedules is obvious within the relatively schedules model, as one must look to make sure a delay exists in the first schedule for all instructions of the second schedule to merge.

Furthermore, explicit timing of instructions will become confusing with the introduction of control-flow to quantum programs as it would become difficult to program experiments containing loops that are not unrollable.

We should move to a model where instructions are explicitly scheduled with variable-duration `Delay`s (think NOPs) of a given duration. This will:

 - Make appending two schedules a much simpler operation as the dependency of instructions will be specified by their order on operands
 - This will allow a simple transformation to a DAG representation of a basic pulse schedule block
 - Have positive performance impacts.
 - Have implications for the current pulse API.

## User Benefit
 - Pulse users (primarily, experimental physicists interested in using IBM backends) should get a performance improvement.
 - When we begin implementing control flow around Schedules, we will find this new implementation much easier to work with. This benefits frontend and backend developers, which in turn will give our end users new features sooner.

## Design Proposal

### Implementation changes

Here we will use the following notation, where `self` is the `Schedule` initiating the operation in question:

 - |C| : the total number of channels in `self`
 - N: the number of instructions in `self`.
 - N<sub>C</sub> : the number of instructions on a channel within `self`
 - N<sub>new</sub>: the number of instructions in the other schedule

Notice that N<sub>C</sub> * |C| = N

| Method        | New Runtime        | Old Runtime | Change          |
|---------------|--------------------|-------------|-----------------|
| `append(self, schedule)` | O(\|C\|) | O(\|C\| + N<sub>new</sub>) (best case is O(\|C\|)) | Previously, `append` would add `schedule` to `self` with relative times preserved, at time set by `self.duration({schedule.channels}.intersection({self.channels}))`. The new method will simply extend the instructions in `self` with those in `schedule` on a per channel basis. |
|`insert(self, time, schedule)`| O(N<sub>new</sub>N<sub>C</sub>) | O(\|C\| + N<sub>new</sub> + \|C\|*N<sub>new</sub>) (best case is O(N<sub>new</sub>log(N<sub>C</sub>)) | This will be slow generally, but it can be implemented with the same runtime as `append` when the `time` is greater than `self.duration(schedule.channels)`. |
|`shift`(self, time) | O(\|C\|) | O(N) | Rather than updating every time interval, we can just prepend a delay. |


## Detailed Design

The `Schedule` will have an internal attribute, `_insts_by_chan`:

```
_insts_by_chan: Dict[Channel, List[ScheduleNode]]
```

**Example**
```
# sched._insts_by_chan
{
    DriveChannel(0):
        ('instructions'=[Delay(duration=10, ...),
                         Play(duration=100, ...),
                         Play(duration=20, ...)],
         'duration'=130),
     DriveChannel(1):
        ('instructions'=[Delay(duration=110, ...),
                         Play(duration=20, ...)],
         'duration'=130),
    ...
}
```

A rough sketch for the implementation of some of append described above could look like this:

```
# These can be done with a few more lines to avoid mutating the input, but for simplicity, I'll leave it like this to demonstrate the main idea

    def append(self, schedule):
        for chan, data in schedule._insts_by_chan:
            self._insts_by_chan[chan].instructions.extend(data.instructions)
            self._insts_by_chan[chan].duration += data.duration
```

 
## Alternative Approaches

- We could have added methods to `Schedule`, but instead we will leave that to the builder.

## Questions

 1. Should we create `insert(self, index, channel)`?
 1. We can support the current `insert` method, but the runtime with be O(|N<sup>2</sup>|). Are we committed to discouraging its use?
 2. Something like `sched |= sched << time` is *much* less efficient than `sched.insert(time, sched)` because the shift happens separately from the union. I don't know how to fix this.

## Future Extensions

A related and important companion to this RFC is the implementation of a schedule building interface. See a sketch of how this might look from our in-house hackathon project lead by @taalexander: [branch here](https://github.com/taalexander/qiskit-terra/tree/pulse-builder-interface) and [demo here](https://github.com/taalexander/qiskit-terra/blob/pulse-builder-interface/notebooks/Pulse%20DSL%20Demo.ipynb).

Altogether, this relative timing feature, the above builder, and the [instruction/command unification work](https://github.com/Qiskit/rfcs/pull/12), we will have migrated Pulse towards a very user friendly and physicist-charming interface, paving the way for wonderful Pulse-based work to be implemented in Ignis.

Additionally, as stated in the motivation, this work will be crucial for supporting general control flow in Qiskit Pulse.
