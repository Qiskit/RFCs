# User-friendly pulse drawer

| **Status**        | **Proposed** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Naoki Kanazawa (knzwnao@jp.ibm.com), Thomas Alexander    |
| **Deprecates**    |                  |
| **Submitted**     | 2020-05-01                                   |
| **Updated**       | 2020-05-02                                   |

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
Currently, we can truncate the measurement pulse with the `plot_range` option.
However, when we enable conditional gates or a conditional reset in the middle in the schedule, the truncation by `plot_range` doen't work becase users may also want to know the pulse sequence after the measurement stimulus.
There is another frustrating situation based on our experience. When we wrote the [Qiskit Pulse paper](https://arxiv.org/abs/2004.06755) we needed to modify the drawer itself to address reviewer's requests and we decided not to share the code for pulse drawing in the public database.
Having configuration be based on a style sheet will help us in such a situation because we would just need to create a dedicated style sheet for publication and share the style sheet instead of modifying source code.

From the point of view of contributors, the current drawer is implemented as one large function call and it is very hard to maintain.
Sometimes a change in the Qiskit Pulse module may cause an unexpected error in the drawer or even crash it, but this will not be detected because visualization level tests are currently skipped now.
Moreover many contributors don't update reference figures in the unittest folder.
Thus, it is better to decompose this huge callback function into some unit functions and enable unit tests for those functions which don't deal with `matplotlib` objects.

This update will improve the efficiency of pulse programming and maintainability of code.

## User Benefit
Every Qiskit Pulse user will enjoy the benefit. From novice to the professional level, users will be able to get a readable visualization with a more simple function or method call for debugging purposes and then easily customize the output format for publication.
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

The `pulse_style_lib` will contain several predefined style sheets that have different visualization purposes.
In the above example, `iqx_debugging()` is a function call which generates a style sheet object.
This style sheet is expected to show as much information as possible, i.e. pulse name, operand values of `PhaseShift` and `FrequencyShift`, pulse peak height, pulse duration and so on.
If that information is too much for publication, the user can switch the output format by using another style sheet:

```python
import qiskit.visualization.pulse_style_lib

pulse_drawer(
    my_sched,
    style=pulse_style_lib.iqx_publication())
```

The `iqx_publication` style sheet may show only pulse names and frequency values because other information will appear in the pulse envelope.
Pulse names may be reformatted into LaTex syntax, for example, systematic name `X90p_d0_123456789` is replaced by `$X_{90}$` by using regular expression operation (This is one of what we needed to do in the [Qiskit Pulse paper](https://arxiv.org/abs/2004.06755)).
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

  def to_dict(self):
    # output style sheet configuration in dictionary
    pass

  @classmethod
  def from_dict(cls, dict_):
    # load style sheet from dictionary
    pass
```

This class has `to_dict` and `from_dict` methods as, for example, the Qiskit `Result` class so that user can save the style sheet in serialized data file such as JSon.
It is important because serialized data can reduce security risks when the style sheet is provided by external providers.
A good usecase of this feature is publication.
If a publisher provides us with a style sheet comforming to their standard, we don't need to spare time for fine tuning of drawings.

Predefined style sheets are accessible with a function call:

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
      +-- core_drawer.py (CoreImageGenerator, VisualizationElement)
      +-- core_matplotlib_static.py (Core2DPulseDrawer, CoreScheduleDrawer, CoreWaveDrawer)
      +-- event_manager.py (EventManager)
      +-- pulse_style_lib.py (QiskitPulseStyle, iqx_publication, iqx_debugging, ...)
      +-- drawing_engine.py (collection of small programs to help object mapping)
```
The interface function `pulse_drawer` handles drawing of static 2D image of waveforms.
The function internally instantiates an image generator class which depends on the format of output object, i.e. 2D or 3D, static or dynamic, passive or interactive, etc...
Then the `pulse_drawer` calls `draw` method of the instance to visualize an input pulse program.
This RFC covers generation of 2D static image.

In `core_drawer.py`, the abstract class of those image generator is placed. This abstract class should be drawing backend agnostic so that any software can be used for visualization.

```python
class CoreImageGenerator(ABC):
  @abstractmethod
  def __init__(self, style_sheet, pulse_canvas=None, table_canvas=None):
    # create a canvas with style sheet
    self._style_sheet = style_sheet
    self._pulse_canvas = pulse_canvas
    self._table_canvas = table_canvas
    self._initialize()

    pass

  @abstractmethod
  def _initialize(self):
    # initialize canvas with style sheet (backgroud color, margins, etc...)
    pass

  @abstractmethod
  def refresh(self):
    # refresh canvas and show image
    pass

  @abstractmethod
  def draw(self, program):
    # generate image of given program
    pass

  @abstractmethod
  def save(self, file_name):
    # save current canvas image to file
    pass

  @abstractmethod
  def _draw_line(line_object):
    # draw line from configuration object
    pass

  @abstractmethod
  def _draw_text(text_object):
    # draw text from configuration object
    pass
```

Based on a year of an experience with the conventional drawer, we learned drawing lines and texts are sufficient functionalities required for the pulse program visualization.
Thus in our new design we abstract those functionalities so that the `CoreImageGenerator` to be drawing backend agnostic.
In addition, we prepare special class to represent data and options for those objects to draw.

```python
class VisualizationElement:
  def __init__(self, data_type, data, coordinate, **style_args):
    self.data_type = data_type
    self.data = data
    self.coordinate = coordinate
    self.style_args = style_args
```

The keyword arguments `**style_args` depends on the drawing backend and it is directly passed to the backend.
In the case of `matplotlib`, this will be something like `{'linestlye': ':', 'color': 'red', 'alpha': 0.5}`.
This can be regarded as an intermediate representation of visualization data.
The `data_type` specifies the method of visualization.
For example, there are two types of lines in the conventional drawer.
The base line of pulse is drawn by simple bold line while pulse envelope is drawn by blotting out the area under curve.
Such details of visualization appears in this property.
Those selections will be predefined with `enum`.

In Qiskit Pulse there are two drawers we need to support.
One is for `Schedule` and another one is for `SamplePulse`.
Because the functionalities and implementation of those generators are almost identical (i.e. `SamplePulse` doesn't require event table), it is quite reasonable to prepare superclass under the name of `Core2DPulseDrawer`.
This superclass and child classes are placed in `core_matplotlib.py` and this is the only file that has `matplotlib` dependency (we supporse to use `matplotlib` for those drawing backend).
In other word, all other functions are isolated from drawing environment and thus we can easily add new drawing software based upon user's requests while reusing those assets.

The image generator of `Schedule` is:

```python
class CoreScheduleDrawer(Core2DPulseDrawer):
  # image generator class for pulse schedule
  pass
```

The image generator of `SamplePulse` is:

```python
class CoreWaveDrawer(Core2DPulseDrawer):
  # image generator class for sample pulse
  pass
```

Since a pulse `Schedule` has more complicated data structure, the `core_schedule_drawer` consists of several elements to extract a set of objects to draw.

The class `EventManager` is an extension of the existing `EventsOutputChannels` class, but some method names are updated to reflect the recent change of pulse syntax. This class is defined for each `Channel` to draw and the class makes a collection of instructions tagged by the time. The class structure looks like:

```python
class EventManager:
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

First, the `core_schedule_drawer` initializes `EventManager` for all channels specified by the user input and the style sheet preference and keep them as a python `dict`. Then, those channels are rearranged as a `list` according to the `layout` option in the style sheet. If `'layout': 'channel-type-wise'` is specified, the program collects all `DriveChannel`s in the dictionary and then collects `ControlChannel`s, ... , so that channels of the same type are shown in one group.
This functionality is offloaded to a dedicated function placed in `drawing_engine.py`.

The file `drawing_engine.py` consists of many small programs that process `EventManager` instances to generate a set of `VisualizationElement` object thrown to an abstracted drawing backend.
This is a big change from the current implementation (but not impactful to users) such that existing large function is decomposed into small pieces.
It should be emphasized that `drawing_engine` doesn't deal with `matplotlib` object so that all programs can be unittested in the current testing framework.

For example,

```python
def get_labels(manager: EventManager) -> List[VisualizationElement]]:
  # return coordinate and name of each pulse instruction
  pass
```

this is the one of helper functions to extract a set of pulse names (sometime formatting names by request) and the coordinate to draw of each pulse in the `EventManager`.
Both of the arguments and the returned values don't contain the `matplotlib` object, this function can be tested by current testing framework (currently this is written as a part of drawer and there is no way to test without importing `matplotlib`).
We can prepare a set of such function in `drawing_engine.py`. Truncation and scaling can be implemented as a part of `drawing_engine.py`.

The `core_schedule_drawer` execute required callback functions based on the style sheet and generate a set of objects drawn by backends (this is usually `matplotlib`).
Finally `core_schedule_drawer` returns `matplotlib.Figure` object if `matplotlib.Axes` objects are not given.

The `core_pulse_drawer` has much simple structure because it just need to call `.samples` method of given `SamplePulse` instance and draw the returned `ndarray`.

The entire work can be separated into several small PRs.

### Phase1: Add `event_manager` and `core_drawer`
This is the fundamental element of the schedule drawer and thus this should be implemented first.
Since the input of `EventManager` is the existing pulse progam and `core_drawer` just contains the abstract class and the IR of visualization data, there is no dependency on another drawer components.
Implementation of those object will give us outline of new drawer framework.
We can prepare the unittest for those object.

### Phase2: Add `pulse_style_lib`
The design of style sheet will influence the implementation of helper functions in `drawing_engine.py` because behavior of those functions are controlled by style sheet preference.
Therefore roughly designed style sheets should be implemented first and will be refined with the actual helper function implementation.
These are candidates of style sheets we need to implement:

+ `iqx_debugging`: show every details of the pulse, may return very busy plot
+ `iqx_standard`: show instruction level information to understand the overview of program, small details are ignored
+ `iqx_publication`: show minimum information to make clean plot for publication

### Phase3: Add `drawing_engine`
Drawing engine has dependency on `EventManager` and be influenced by the desing of style sheets.
So the implementation of the drawing engine will come after style sheets.
These are candidates of programs we need to implement:

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
+ `arange_channel_layout`: arange channel layout based on style sheet preference

We need to prepare unittest for those functions.

### Phase4: Add `core_matplotlib_static` and update `pulse_visualization`
With all of above PRs we can start implementing main part of drawer.
The unittest for `matplotlib.Figure` output is skipped in the current testing framework.
We need to check the output of figure by our eyes.

Because new drawer drastically changes the interface and the module structure, it is not efficient to implement this framework while keeping the backward compatibility with the existing drawer.
We plan to introduce the new drawer slowly under different name such as `pulse_drawer_v2` (TBD) to test over time.


## Alternative Approaches
N/A

## Questions
N/A

## Future Extensions
New drawer provides higher degree of freedom for image generation.
At the Qiskit user level they can easily fine tune visualization output with style sheets.
We are able to call for cool style sheets! We can collaborate with design team to provide users with cool pulse visualization. This is also very easy because it just need to add new style sheet.

On the other hand, at the contributor level, we can provide users with new drawing functionality by adding new generator cores.
We may want to create drawer widget or dynamic drawer, something like:

```python
sched = Schedule()

my_drawer = dynamic_drawer(schedule, style_sheet)
my_drawer.start() # open clean canvas

sched += Play(rabi_pulse, d0) # Rabi pulse appears in d0
sched += Play(meas_pulse, m0) # Measurement pulse appears in m0

my_drawer.filter(channel=d0) # m0 is removed
my_drawer.filter(time_range=[0, 100]) # envelope within the time from 0 to 100 is shown
my_drawer.reset() # reset filtering
...
```

This will be a powerful tool for future pulse coding. When we consider the implementation of such new module, we can resue most of the existing code because actual drawing environment is isolated from the data processing part and thus the environment can be easily replaced depending on the visualization purpose.
We can crete new core generator based upon the `CoreImageGenerator`.