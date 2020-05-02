# User-friendly pulse drawer

| **Status**        | **Proposed** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Naoki Kanazawa (knzwnao@jp.ibm.com)    |
| **Deprecates**    |                  |
| **Submitted**     | 2020-05-01                                   |
| **Updated**       | YYYY-MM-DD                                   |

## Summary

The pulse drawer is one of the best ways to debug pulse programs. However, it is not very user-friendly.
We propose a new drawer implementation that should improve usability and lead to improved pulse programming efficiency.
In the new implementation, the style sheet feature is leveraged to reduce the required amount of code to generate good visualization and will provide Qiskit users with several predefined style sheets.

## Motivation
In the conventional pulse drawer, the minimum amount of code to generate a readable pulse visualization is:
```python
pulse_drawer(
    my_sched,
    label=True,
    plot_range=(0, 2000),
    channels=[pulse.DriveChannel(0), pulse.DriveChannel(1), pulse.MeasureChannel(0), pulse.MeasureChannel(1)]))
```
Users must write this boilerplate code for every pulse program they wish to draw.
In addition, the drawer has several defects in the output format.
The drawer aranges pulse channels in the order of `[DriveChannel[i], ControlChannel[i], MeasureChannel[i], AcquireChannel[i]] for i in range(max_channel_index)` (or `'channel-index-wise'`) but in IBM Quantum backends the index `i` of `ControlChannel` doesn't indicate the index of corresponding qubit.
For example, when we visualize a pulse schedule that includes a cross resonance pulse on qubit 15, the corresponding `ControlChannel` index may be 36 and the cross resonance pulse is drawn at the place far below the `DriveChannel`s of this qubit.
This makes the debugging of the program very hard.
Another defect can be seen when we try to visualize two pulses whose durations are very much different.
A good example is a single qubit pi pulse followed by a measurement stimulus pulse.
Usually a pi pulse duration is around 160 cycles whereas that of the measurement pulse is over 10000.
Currently we can truncate measurement pulse with `plot_range` option.
However when we enable conditional gates or a conditional reset in the middle in the schedule, the truncation by `plot_range` doen't work becase users may also want to know the pulse sequence after the measurement stimulus.
There is another frustrating situation based on our experience. When we wrote the Qiskit Pulse paper we needed to modify the drawer itself to adress reviewers requests and we decided not to share the code for pulse drawing in the public database.
To enrich the style sheet option will help us in such situation because we just need to create a dedicated style sheet for publication and share the style sheet instead of the whole modified Qiskit-terra.

From the point of view of contoributors, current drawer is implemented as one huge callback function and it is very hard to maintain.
Sometime a change in Qiskit Pulse module may cause an unexpected error in the drawer or even crash it, but this will not be detected because visualization level test is skipped now.
Moreover many contributors don't update reference figures in the unittest folder.
Thus, it is better to decompose this huge callback funtion into some unit functions and enable the unittest for those functions which doesn't deal with `matplotlib` objects.

This update will improve the efficiency of pulse programming and maintainability of code.

## User Benefit
Qiskit Pulse user will enjoy the benefit. From novice to professional level, users will be able to get a readable visualization with more simple function or method call for debugging purpose and then easily customize the output format for publication.
Cost of code maintainance will be also decreased.

## Design Proposal
The heart of new design concept is the simplification of the function call while keeping sufficient customizability using style sheets.
In the new implementation concept, user will call the drawer in the following manner:

```python
import qiskit.visualization.pulse_style_lib

pulse_drawer(
    my_sched,
    style=pulse_style_lib.iqx_debugging())
```

The `pulse_style_lib` contains several predefined style sheets that have different purpose of visualization.
In above example `iqx_debugging` may show many information as possible, i.e. pulse name, operand values of `PhaseShift` and `FrequencyShift`, pulse peak height, pulse duration and so on.
If those information are too much for publication user can switch the output format by using another style sheet:

```python
import qiskit.visualization.pulse_style_lib

pulse_drawer(
    my_sched,
    style=pulse_style_lib.iqx_publication())
```

The `iqx_publication` style sheet may show only pulse names and frequency values because other information will appear in the pulse envelope.
Pulse names may be reformatted into LaTex syntax, for example, systematic name `X90p_d0_123456789` is replaced by `$X_{90}$` by using regular expression operation (This is one of what we needed to do in the Qiskit Pulse paper).
As those examples illustrate, we can easily change the data to be overlayed on the plot by using style sheets.
The layout of channels and truncation of pulses are also controlled by the style sheets.
In `iqx` style sheet series, preferred channel layout must be `[DriveChannel[i] for i in range(max_channel_index)], [ControlChannel[i] for i in range(max_channel_index)], [MeasureChannel[i] for i in range(max_channel_index)], [AcquireChannel[i] for i in range(max_channel_index)]` (or `'layout': 'channel-type-wise'`), and `'truncation': True` (automatically truncate long pulse and delay).

The core style sheet is defined as a class `PusleStyleSheet` which is initialized with style sheet options.

```python
class QiskitPulseStyle:
  def __init__(self, param1=None, param2=None, param3=None, ...):
    self._param1 = param1 or default_value1
    self._param2 = param2 or default_value2
    self._param3 = param3 or default_value3
    ...

  @property
  def param1(self):
    return self.param1
  ...
```

Predefined style sheets are defined by callback function:

```python
def iqx_debugging(**kwargs):
  param1 = kwargs.get('param1', default_value1)
  param3 = kwargs.get('param3', default_value3)
  ...
  # param2 will be system default
  return QiskitPulseStyle(param1=param1, param3=param3 ...)
```

This means users can easily define own style sheet based upon favorite predefined one when they call it.

The drawer takes those style sheets for controlling information to show, channel layout and truncation.
However users are still able to control some options on the spot to focus on the waveforms they want to check.
Thus, new drawer interface becomes:

```python
def pulse_drawer(
  program: Union[SamplePulse, ScheduleComponent],
  dt: int = 1, # time resolution
  style: Union[PulseStyle, SchedStyle] = None, # style sheet instance
  filename: str = None, # if set save the plot in file
  scale: Dict[Channel, float] = None, # scaling value for each channel type
  plot_range: Tuple[Union[int, float], Union[int, float]] = None, # plot range
  channels: List[Channel] = None # list of channels to show
  pulse_ax: matplotlib.Axes = None # object to draw pulses
  table_ax: matplotlib.Axes = None # object to draw event table
):
```

Note that the new drawer interface takes `matplotlib.Axes` objects.
This is important feature from the publication viewpoint, because users may want to compare two different pulse schedules aranged in column-wise or row-wise.

Contributors must pay attention not to increase the number of arguments of this inteface otherwise the simplifity of function call will be lost and such PR will eventually decrease the efficiency of pulse programming.
Instead of adding new argument to the interface, contributors can add new options to the `QiskitPulseStyle` and callback functions of predefined style sheets.

Implementation details of this function are described in next section.


## Detailed Design
```
qiskit
  +--[visualization]
    +--pulse_visualization.py (pulse_drawer)
    +--[pulse]
      +-- drawer_core.py (schedule_drawer_core, pulse_drawer_core)
      +-- event_manager.py (EventsOutputChannels)
      +-- pulse_style_lib.py (QiskitPulseStyle, iqx_publication, iqx_debugging, ...)
      +-- utils.py (other additional callbacks)
```
The interface function `pulse_drawer` internally calls the sub function `schedule_drawer_core` or `pulse_drawer_code` which generate `matplotlib.Figure` from given arguments.

```python
def schedule_drawer_core(program: ScheduleComponent, ...):
  # draw schedule figure
  return matplotlib.Figure
```

```python
def pulse_drawer_core(program: SamplePulse, ...):
  # draw schedule figure
  return matplotlib.Figure
```

The `schedule_drawer_core` is used to draw pulse `Schedule` while the `pulse_drawer_core` is used to draw `SamplePulse`.
Since a pulse `Schedule` has more complecated data structure, the `schedule_drawer_core` consists of several elements to extract a set of objects to draw.

The class `EventsOutputChannels` is an extention of existing `EventsOutputChannels` class, but some method names are updated to reflect the recent change of pulse syntax. This class is defined for each `Channel` to draw and the class makes a collection of instructions tagged by the time. The class structure looks like:

```python
class EventsOutputChannels:
  def __init__(self, schedule: ScheduleComponent, channel: Channel):
    self._pulse = []
    self._phase = []
    self._frequency = []
    self._snapshot = []
    self._conditional = []

    self._parse_schedule(schedule, channel)

  def _parse_schedule(self, schedule, channel):
    # filter schedule by channel and make time table of instructions
    pass

  def waveform(self, t0, tf) -> List[Tuple[ndarray, str]]:
    # create waveform from time t0 to tf and return it as list of ndarray
    pass

  @property
  def max_amp(self):
    # return max amplitude in the channel
    pass

  @property
  def min_amp(self):
    # return min amplitude in the channel
    pass
```

The main purpose of this class to generate a list of complex `ndarray` accompanied with the pulse name from the given set of instruction in the channel.
Note that pulse waveform should consider the phase instructions.
This is done by calling `waveform` method.
Each `ndarray` in the list represents a pulse envelope of each `Play` instruction.
The interval between `Play` instructions should be padded by `Delay` and this appears as long zero array with the length of the interval in terms of system cycle time (not actual pulse time).
This is because we need to truncate long pulses or long delays based on `truncate` preference in the style sheet.
If we output the entire waveform as a single data array it is very hard to recover a length of each pulse.
The method `max_amp` and `min_amp` are used to determine the autoscaling value of the channel.

First the `schedule_drawer_core` initialize `EventsOutputChannels` for all channels specified by the user input and the style sheet preference and keep them as a python `dict`. Then, those channels are rearanged as a `list` according to the `layout` option in the style sheet. If `'layout': 'channel-type-wise'` is specified, the program collects all `DriveChannel`s in the dictionary and then collects `ControlChannel`s, ... , so that channels of the same type are shown in one group.

In `utils.py` several helper functions are defined to extract information and coordinate to show from the `EventsOutputChannels`.
This is a big change from current implementation (but not impact to users) such that existing huge callback function is decomposed into small pieces which can be unittested.
For example,

```python
def get_labels(manager: EventsOutputChannels) -> List[Tuple[Tuple[float, float], str]]:
  # return coordinate and name of each pulse instruction
  pass
```

this is the callback function to extract a set of pulse names (sometime formatting names by request) and the coordinate to draw of each pulse in the `EventsOutputChannels`.
Both of the arguments and the returned values don't contain the `matplotlib` object, this function can be tested by current testing framework (currently this is written as a part of drawer and there is no way to test without importing `matplotlib`).
We can prepare a set of such callback function in `utils.py`. Truncation and scaling can be implemented as a part of `utils.py`.

The `schedule_drawer_core` execute required callback functions based on the style sheet and generate a set of objects drawn by backends (this is usually `matplotlib`).
Finally `schedule_drawer_core` returns `matplotlib.Figure` object if `matplotlib.Axes` objects are not given.

The `pulse_drawer_core` has much simple structure because it just need to call `.samples` method of given `SamplePulse` instance and draw the returned `ndarray`.

The entire work can be separated into several small PRs.

### Phase1: Update `EventsOutputChannels`
This is the fundamental element of the schedule drawer and thus this should be implemented first.
Since the input is existing pulse `Schedule` there is no dependency on another drawer components.
We can prepare the unittest for this object.

### Phase2: Add utility functions
Utility functions have dependency on `EventsOutputChannels`.
These are candidates of callback function we need to implement:

+ `get_labels`: return label of each pulse
+ `get_max_values`: return maximum value of each pulse
+ `get_min_values`: return minimum value of each pulse
+ `get_lengths`: return length of each pulse
+ `get_phases`: return phase change/set instruction
+ `get_frequencies`: return frequency change/set instruction
+ `get_snapshots`: return snapshot instruction
+ `scale_waveform`: apply scaling to pulses
+ `truncate_waveform`: truncate long waveform and delay
+ `interpolate_waveform`: interpolate waveform

We can prepare unittest for those functions as well.

### Phase3: Add style sheets
The style sheet doesn't have any dependency so we can start implementation of them at anytime we want.
These are candidates of style sheets we need to implement:

+ `iqx_debugging`: show every details of the pulse, may return very busy plot
+ `iqx_standard`: show instruction level information to understand the overview of program, small details are ignored
+ `iqx_publication`: show minimum information to make clean plot for publication

### Phase4: Add core function and interface
With all of above PRs we can start implementing main part of drawer.
The unittest for `matplotlib.Figure` output is skipped in the current testing framework.
We need to check the output of figure by our eyes.

## Alternative Approaches
N/A

## Questions
+ How old drawer can be deprecated? Becase interface will change drastically, I feel introducing new drawer independently to old one is easier.

## Future Extensions
We may want to create some drawer widget based upon this drawer.
When we draw a very long pulse sequence such as dynamical decoupling, this widget is very convenient, for example, to scroll over the entire pulse sequence or to filter pulse channels directly from the widget GUI.
In the proposed implementation the drawer generates a set of object to draw without the context of `matplotlib`, it is easy to introduce other drawing backends.

We are also able to call for cool style sheets! We can collaborate with design team to provide users with cool pulse visualization. This is also very easy because it just need to add new style sheet.
