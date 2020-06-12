# Improved Provider Interface and Supporting New Service APIs

| **Status**        | **Proposed**                                 |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0002                                         |
| **Authors**       | Matthew Treinish (mtreinish@kortar.org)      |
| **Deprecates**    | None                                         |
| **Submitted**     | 2020-02-04                                   |
| **Updated**       | 2020-02-04                                   |


## Summary

Implement a new provider interface for terra that is based around python
objects instead of a pseudo-serialization format unique to ibmq/iqx. This new
interface will live in parallel to the existing provider interface for backwards
compatibility, but the new interface will be the model moving forward. The basic
goal is to decouple the wire protocol IBMQ uses from what developers have to
deal with. This frees up providers to interact with terra at a native terra
object level which provides much more flexibillity for non-ibmq and local
providers. Additionally, it will add the groundwork for providers to add
new types of backends that model additional services that can be provided.

## Motivation

This change is needed because right now the entire provider interace is built
around the json protocol for iqx. Basically all I/O between terra and
providers takes native terra objects and either converts them to dictionaries
which can then be just passed directly to `json.dumps()` (ie qobj) or takes the
dictionary output from `json.loads()` directly (ie result). The problem with
this is that these formats are that they're not designed to be human interfaces
or useful for development. Additionally, this makes the assumption that the
backend is remote and will always use JSON as the interface, and that the
execution model follows IQXs.

It also doesn't help that the current version of this interface is built around
marshmallow which obfuscates the actual structure of things and introduces a
big performance bottleneck. For example, the way a circuit gets passed to the
providers today is of the form:

```python
class QasmQobj:
    header = Qobjheader()
    config = QasmQobjConfig()
    experiments = [QasmQobjExperiment()]
    ...

class QasmQobjExperiment:
    header = QobjExperimentHeader()
    config = QasmQobjExperimentConfig()
    instructions = [QasmQobjInstruction]

...
```

basically every field in the ibmq json protocol becomes it's own class. This
has needless class heirarchy that makes tracking how to build one from
scratch quite difficult. It also doesn't need to work this way since we're
basically just passing:

```python
class qobj_experiment_equiv:
    header = {}
    config = {}
    experiments = [{'config': {}, 'header': {}, 'instructions': QuantumCircuit.data}]
```

just in a different format. But, what makes it worse is that because of
how marshmallow is used the structure of the classes from the current state
is not actually defined in it's class definition/constructor but instead in
a seperate schema class that gets applied via a decorator. For example the
current strucure in terra actually is:

```python
from qiskit.validation.base import BaseModel, BaseSchema, bind_schema
from qiskit.validation.fields import Nested, String
from .utils import QobjType

class QobjSchema(BaseSchema):
    """Schema for Qobj."""
    # Required properties.
    qobj_id = String(required=True)
    schema_version = String(required=True)

    # Required properties depend on Qobj type.
    config = Nested(QobjConfigSchema, required=True)
    experiments = Nested(QobjExperimentSchema, required=True, many=True)
    header = Nested(QobjHeaderSchema, required=True)
    type = String(required=True, validate=OneOf(choices=(QobjType.QASM, QobjType.PULSE)))

    @pre_load
    def add_schema_version(self, data, **_):
        """Add the schema version on loading."""
        data['schema_version'] = QOBJ_VERSION
        return data

class QasmQobjSchema(QobjSchema):
    """Schema for QasmQobj."""

    # Required properties.
    config = Nested(QasmQobjConfigSchema, required=True)
    experiments = Nested(QasmQobjExperimentSchema, required=True, many=True)

    type = String(required=True, validate=Equal(QobjType.QASM))

    @pre_load
    def add_type(self, data, **_):
        """Add the Qobj type (QASM) on loading."""
        data['type'] = QobjType.QASM.value
        return data

@bind_schema(QasmQobjSchema)
class QasmQobj(Qobj):
    """Model for QasmQobj inherit from Qobj.
    Please note that this class only describes the required fields. For the
    full description of the model, please check ``QasmQobjSchema``.
    Attributes:
        qobj_id (str): Qobj identifier.
        config (QASMQobjConfig): config settings for the Qobj.
        experiments (list[QASMQobjExperiment]): list of experiments.
        header (QobjHeader): headers.
    """
    def __init__(self, qobj_id, config, experiments, header, **kwargs):
        super().__init__(qobj_id=qobj_id,
                         config=config,
                         experiments=experiments,
                         header=header,
                         **kwargs)
```

This has several negative qualities, the obvious downside is that we take
7 classes from the simplified example above and at least double that. This makes
it difficult to follow, but also increases the memory footprint and slows down
the creation of these objects. It also makes things like code introspection
(which is used by IDEs for auto completion) completely useless because the
properties of the actual class being used are not defined except at run time.

This then has to be multiplied for every interface between terra and a provider
that ends up in wire traffic on ibmq (basically backends, jobs, and results
too).

The other aspect is that this whole construct is built around how IBMQ's API
works. While the api schema was defined with the intention of being a general
purpose api that could be used by anyone hosting a backend. In actuality
this is just IBMQs api and no one else will use it outside of IBMQ. This means
except for qiskit-ibmq-provider all providers basically have to parse what we
pass them and convert them to their internal representations for running the
actual experiment. The AQT provider is really good example of this that
implementation takes the qobj object it gets loops over the instructions and
builds it's own json string from that. See:
https://github.com/Qiskit/qiskit-aqt-provider/blob/master/qiskit/providers/aqt/qobj_to_aqt.py
or in some cases (like the honeywell provider) they take a qobj run
`disassemble` to get circuit objects and then convert then run `.qasm()` on the
circuits to send that over the wire:
https://github.com/Qiskit/qiskit-honeywell-provider/blob/a08a4d12bb0493a9c40462cba1b4e23c4ea2972d/qiskit/providers/honeywell/honeywelljob.py#L160

There's no reason to be passing a qobj here it just makes things slower by adding
an uncessary conversion step and providing a much harder object to interact with.
A QuantumCircuit object contains all the same information needed. This gets even
worse for local simulator providers we end up converting things to wire protocol
which has to be parsed and converted locally for input. 

Then when you look at local simulators this model makes even less sense.
The simulators are running on the same device as the provider interface there
is no reason to convert an object to a json like structure when the simulator
can just interact with it directly. Passing native objects around directly is
much faster and simpler.

## User Benefit
The main target for this work is provider authors and qiskit developers,
since the pain points of working with the current interface are isolated
to the development of qiskit or qiskit plugins. But all users of qiskit
will benefit from performance imprpovements gained by the new interface
because we won't be performing needless conversions everytime we run
experiments. Another general benefit is this will result in a cleaner and
better documented interface for working with backends and results. This will
also hopefully give us a chance to refine the interface that users use for
interacting with running experiments and results.

Additionally, it makes the domains of what's part of IBMQ/IQX and what's part
of terra much clearer. Right now to make changes to terra's interfaces this
often gets caught up in how iqx works today and limits what is possible.

## Design Proposal

### Providers Interface v1.5

NOTE: Since https://github.com/Qiskit/qiskit-terra/pull/4030 merged this first
phase is now complete.

The transition will happen in 2 stages. The first stage, v1.5 which is
a maintanence change for terra is to remove marshmallow from the existing
providers interface. This will make things more transparent in how they are
built, not just simplifying working with the interface, but also
significantly speed up the performance of the current interface. This is
already in progress here:

https://github.com/Qiskit/qiskit-terra/pull/3383
https://github.com/Qiskit/qiskit-terra/pull/4096
https://github.com/Qiskit/qiskit-terra/pull/4016
https://github.com/Qiskit/qiskit-terra/pull/4030

These basically just mirrors the marshmallow class heirarchy and replaces each
class with a flat class with just attributes for each element in the schema.

This is needed for 3 reasons. The first is that the providers interface has
existed in it's current form for some time and is how any backend interfaces
with qiskit. We need to keep this around not just for a smooth transition to
the new interface but also for providers that are not maintained by our team.
We want to maximize compatibility with any providers that exist out there and
removing the current interface would potentially break a lot of people. The
second is to simplify the maintenance around this existing interface. Right
now the interface for interacting with providers is too obfuscated. Creating a
flat object makes it much simpler to work with when we need to make additions
or changes to keep the interface functioning. It also makes it easier for local
simulators to use C aligned data structures natively because everything is
just a normal python attribute and doesn't go through marshmallow's
transformation layer. The third is that doing this will greatly improves
performance. In Qiskit/qiskit-terra#3383 for assembling a qobj for 100
experiments of circuits with 53 qubits and a depth of 2048 took < 2min with the
PR removing marshmallow and 3min and 40 seconds with master (object generation
is also significantly faster, and more importantly uses far less memory).

This also starts the process of decoupling IQX's API from the provider interface
because removing marshmallow means we don't get the 2 transformations it performs
`numpy array -> list` and `complex numbers -> [re, im]`. Basically when v1.5
starts merging we are saying that the `qobj.to_dict()`'s output is not the same
as `json.loads()` from what the ibmq provider sends on the wire, and all other
aspects of the existing providers interface is no longer mapped 1:1 to the IQX
API. This is a necessary step before we can define a more abstract python
interface for all providers, not just IQX.

### Providers interface 2.0

At the same time we're implementing the v1.5 freeze and refactor we will work
on dding a new providers v2 interface that basically treats the providers
interface as a python interface instead of as a pseudo-json api. The basic idea
is that we use terra's objects as the what is passed back and forth to providers
instead of the intermediate formats that exist today. In addition to making it
easier to work with the plan for this is to have the necessary infrastructure in
place to integrate additional types of backends (for additional types of
services from providers).

For example, the new interface for running circuits (or pulse schedules) will
look like:

```python
qc = QuantumCircuit()
provider = my_provider()
backend = provider.get_backend()
backend.set_config(shots=2048)
backend.run([qc]*100)
```

The abstract classes defining this new interface will live in:
`qiskit.providers.v2` in the terra repo and for terra's functionality
that interacts with anything as part of the provider interface (eg transpiler
passes that use backend configuration) will need to check if it's using a
provider based on v1 or v2 prior to interacting with it. This is as simple
as just doing `if isinstance(backend, qiskit.providers.v2.backend.Backend)`.
Also, these abstract class definitions are just the base data models that define
the common pieces that must be shared between all backends. Each provider will
provide it's own subclass definition of each abstract class that the backend
objects returned will contain (or be) instances of. This allows providers to
provide as much information as they would like or have available. But, because
every platform and backend are different the v2 interface is mainly formalizing
the common definition of the mininmum specification for trying into terra.

#### Backend Class

Building a new interface lets us rework our abstraction model for how a backend
is represented in Qiskit to be more agnostic and specific to the information
both qiskit-terra and end users need to get from a backend. The v1 interface (as
has been said many times prior) was built around the paticulars of what IQX JSON
payload. The current model has 4 objects (which are returned by methods of the
same name) for backends that mirror the returned information from IBMQ:
``configuration`` which is used to describe the immutable properties of the
device (like number of qubits or coupling map), properties which describe the
current measured properties of a device (for example error rates), status which
is used to describe the current backend status, and pulse defaults which is
used to describe the definition of pulses for commands. However, because these
are not actually defined by qiskit, and come from what IQX provides there
is a lot of extraneous information in all of them, it also doesn't really
make it clear what a new provider needs to implement to actually to work
with certain pieces of terra. These also all change independently of
qiskit-terra since they're defined by IQX (until v1.5 merges).

In the v2 interface we'll rework this model to define the minimum pieces of
information that are required and/or optional for a backend to work with
the transpiler. This does not preclude having additional information and for
providers that have more information than is currently consumed by terra there
is a mechanism to provide this, and for terra to access these custom attributes.
Also a path for adding additional fields to the base abstract definition moving
forward. But this results in a well defined interface where both terra/pass
developers and provider authors know what information is always available,
optionally available, and custom to a specific provider. This clarity makes
it easier to work with the model (and is also based on what other compilers
use, for exmplae [1]).

[1] https://github.com/rust-lang/rust/blob/1.44.0/src/librustc_target/spec/mod.rs#L515-L540

A base backend abstract class will be defined as:

```python
class Backend(ABC):
    """Abstract class for Backends

    This abstract class is to be used for all Backend objects created by a
    provider. There are several classes of information contained in a Backend.
    The first are the properties of class itself. These should be used to
    defined the immutable characteristics of the backend that fall outside
    any of the defined abstract properties. The ``configuration`` attribute
    of the backend is used to contain the dynamic properties of the backend.
    The intent is that these will all be user configurable. It should be used
    more for runtime properties that **configure** how the backend is used.
    For example, something like a ``shots`` field for a backend that runs
    experiments which would contain an int for how many shots to execute. The
    ``properties`` attribute is optionally defined and is used to return
    measured properties, or properties of a backend that may change over time.
    The simplest example of this would be a version string, which will change
    as a backend is updated, but also could be something like noise parameters
    for backends that run experiments.

    For Backends that will run circuits you'll want to have a target defined
    as well. This will provide necessary information to the transpiler so that
    circuits will be transpiled so that they actually run on the backend.
    """

    version = 1

    def __init__(self, name, **fields):
        """Initialize a backend class

        Args:
            name (str): The name of the backend
            fields: kwargs for the values to use to override the default configuration.
        Raises:
            AttributeError: if input field not a valid configuration
        """
        self.name = name
        self._configuration = self._default_config()
        if fields:
            for field in fields:
                if field not in self._configuration.data:
                    raise AttributeError(
                        "Configuration field %s is not valid for this backend" % field)
            self._configuration.update_config(**fields)

    @classmethod
    @abstractmethod
    def _default_config(cls):
        """Return the default configuration

        This method will return a :class:`qiskit.providers.v2.BaseConfiguration`
        subclass object that will be used for the default configuration. These
        should be the default parameters to use for the configuration of the
        backend.

        Returns:
            qiskit.providers.v2.Configuration: A configuration object with
                default values set
        """
        pass

    def set_configuration(self, **fields):
        """Set the configuration fields for the backend

        This method is used to update the configuration of a backend. If
        you need to change any of the configuration prior to running just
        pass in the kwarg with the new value for the configuration.

        Args:
            fields: The fields to update the configuration

        Raises:
            AttributeError: If the field passed in is not part of the
                configuration
        """
        for field in fields:
            if field not in self._configuration.data:
                raise AttributeError(
                    "Configuration field %s is not valid for this "
                    "backend" % field)
        self._configuration.update_config(**fields)

    @property
    def properties(self):
        """Return the backend's measure properties.

        The properties of a backend represent measure properties of a backend.
        These are fields that are immutable for the property but may change
        over time based on the state of the Backend.

        This is an optional property for the Backend and will return None if a
        backend does not have properties.
        """
        if hasattr(self, '_properties'):
            return self._properties
        else:
            return None

    @property
    def configuration(self):
        """Return the configuration for the backend

        The configuration of a backend are the dynamic parameters defining
        how the backend is used. These are used to control the :meth:`run`
        method.
        """
        return self._configuration

    @property
    def target(self):
        """Return the backend's target information

        The target of a backend represents the information for the compiler
        to set this backend as the target device. These are fields that are
        immutable for the Backend.

        This is an optional property for the Backend if it targets being a
        compiler target. For classes of backends like circuit optimizers this
        doesn't apply and doesn't have to be defined.and will return None if a
        backend does not have a target.
        """
        if hasattr(self, '_target'):
            return self._target
        else:
            return None

    @property
    def pulse_defaults(self):
        """Return the backend's pulse defaults

        The pulse_defaults of a backend represent the default pulse definitions
        for the pulse commands a backend supports.
        """
        if hasattr(self, '_pulse_defaults'):
            return self._pulse_defaults
        else:
            return None
        """

    @abstractmethod
    def run(self, run_input):
        """Run on the backend.

        This method that will return a :class:`~qiskit.providers.v2.Job` object
        that run circuits. Depending on the backend this may be either an async
        or sync call. It is the discretion of the provider to decide whether
        running should  block until the execution is finished or not. The Job
        class can handle either situation.
        """
        pass
```

The new things in this model are configuration is no longer the description
of the immutable properties for the backend, it is now a container class
for the dynamic properties that are user configurable to control execution
on the backend. This is for things like setting the number of `shots` for
example. The expectation is that users will use the `set_config()` method
or the class constructor for a Backend object to set these parameters. Checking
the state of the parameters can be done from the Backend.configuration property
which will return the Configuration object with it's current state.

```python3
class Configuration:
    """Base configuration object

    This class is the abstract class that all backend configuration is based
    on. The properties of the class are intended to be all ddynamically
    adjustable so that a user can reconfigure the backend on demand. If a
    property is immutable to the user (eg something like number of qubits)
    that should be a property of the backend class itself instead of the
    configuration.
    """

    def __init__(self, **kwargs):
        self.data = kwargs if kwargs else {}

    def update_config(self, **fields):
        self.data.update(fields)

    def get(self, field, default=None):
        return self.data.get(field, default)
```

The other change is the introduction of the target class. The target class
is used in a similar manner to how configuration worked in the v1 model. It is
used to describe the immutable properties of backend. But unlike configuration
the target class is much simpler and only defines the fields that transpiler
is using, or optionally can use depending on configuration. This makes it clear
what is actually necessary for a provider to provide with backends to fully
integrate with terra and for pass authors to know what is always expected to
be present for all backends, what is optionally available for some backends, and
what is specific to a particular provider's backends. Note, this does not preclude
backends from defining and offering custom fields with additional information.
These are expected to either be provided as python properties on the backend
classes themselves or in the target subclass for a provider. But the provider
will need to define and document this. If terra transpiler starts needing
additional information for any of it's passes we can either add fields and define
a new target version.

```python3
class Target(ABC):
    """Abstract class for Backend Target

    This abstract class is used to provide target information to the compiler
    about a backend. These are immutable properties about the backend. The
    properties defined here are what information may get consumed by terra,
    however if it doesn't apply to a backend they can return ``None`` to
    indicate this. The version field is specifically around the target format.
    If in the future the fields in a target grow we should create a subclass
    and that subclass should increase the version number. This will let the
    transpiler with extra fields know beforehand which version the target
    is (and ignore fields if they need a newer version). Additionally, in
    the future this will be needed if/when a serialization format is added for
    this.
    """

    version = 1

    @property
    @abstractmethod
    def num_qubits(self):
        """Return the number of qubits for the backend."""
        pass

    @property
    @abstractmethod
    def basis_gates(self):
        """Return the list of basis gates for the backend."""
        pass

    @property
    @abstractmethod
    def supported_instructions(self):
        """Return the list of supported non-gate instructions for the backend."""
        pass

    @property
    @abstractmethod
    def coupling_map(self):
        """Return the qiskit.transpiler.CouplingMap object"""
        pass

    @property
    @abstractmethod
    def conditional(self):
        """Return bool whether the target can execute gates with classical conditions."""
        pass

    @property
    @abstractmethod
    def gates(self):
        """Return a list of dictionaries describing the properties of each gate."""
        pass
```

In the v2 model there are a few things which remain mostly the same, mainly
properties and pulse defaults. These fields are optional and are only provided
by backends that have the necessary information. We can also base the minimum
set of required and optional fields based on what's in the v1 interface since
IBMQ is the only backend that provides the information to-date. However,
two things that will have to be kept in mind is that we want to have a flatter
class heirarachy to make it easier to construct and work with the objects.
So for example the `BackendProperties` <-> `Gate` <-> `Nduv` heirarchy
probably should be flattened. So that a provider doesn't need to nest loops
of object creations to use the python interface for construction a
`Properties` object. These definitions should be modeled after the target
class which will define the required and optional fields for a provider to
define a properties and defaults class for it's backends.

It's worth pointing out that everything in this model has an explicit version
parameter. This is because as we continue to develop qiskit-terra and backends
improve we will almost certainly find that we need to modify the interface
between qiskit-terra and providers. If we need to add anything to these base
abstract classes we can't modify them in place because a change in requirements
for a implementation of one of theese classes would break any existing released
provider. By versioning the definitions we can transition forward and make
changes, it also makes it simple to describe what a backend is going to provide
by which version it is using. This will let us make forward progress as things
change over time without necessarily breaking people (which has been a big
issue for the v1 interface).

##### New run model

On the other side of the interface, interacting with running on a backend.
The first change that will need to be made is removing the implicit assumption
that run() and jobs are always async. In backends. In v1/v1.5 there is a `run`
method that expects an async job execution that will return a job object which
is handle for that async execution. However, This model doesn't make sense for
all classes of work executed by a backend. For, example on local simulators the
execution is a syncronous task and doesn't need the overhead of wrapping it
with an async handle. To facilitate this the ``run`` method for v2 backends
could be sync or async at the discrestion of the provider. The job class in
the v2 provider will enable either returning a handle to the async execution
or the completed result.

```
class Job(ABC):
    """A base class for representing an async or sync job on a backend.

    This provides the base structure required for building an async or
    sync job handler. The underlying concept around this is that this
    object gets returned by :meth:`qiskit.providers.v2.Backend.run` and
    depending on the nature of the backend will either be the record of
    the execution with results, or in the case of async backends will
    be a handle to the record of the async execution that will populate
    the result data after the execution is complete. The ``result_data``
    instance attribute can either be the data from a single execution
    (depending on the backend this would be a
    :class:`~qiskit.circuit.QuantumCircuit`, :class:`~qiskit.result.Counts`,
    :class:`~qiskit.quantum_info.Statevector`, etc. object), or in the case
    there were multiple inputs an ``OrderedDict`` class where the key is the
    name of the input circuit or schedule.
    """

    version = 1

    job_id = None
    metadata = None

    def __init__(self, job_id, backend, time_taken=None):
        """Initialize a new job object

        Args:
            job_id (str): A unique identifier for the job
            backend (qiskit.providers.v2.BaseBackend): The backend the job is
                being run on
            time_taken (float): Te duration of the job in seconds, should only
                be set during init for sync jobs.
        """

        self.result_data = None
        self.job_id = job_id
        self.backend = backend
        self.time_taken = None

    def _result_conversion(self, result, name=None):
        if isinstance(result, Counts):
            if 'seed_simulator' in result.metadata:
                seed_simulator = result.metadata.pop(
                    'seed_simulator')
            else:
                seed_simulator = None
            header = result.metadata
            header['name'] = result.name
            result_dict = {
                'shots': result.shots,
                'data': result.hex_raw,
                'success': True,
                'time_taken': result.time_taken,
                'header': header,
            }
            if seed_simulator:
                result_dict['seed_simulator'] = seed_simulator
                result_dict['seed'] = seed_simulator
        elif isinstance(result, Statevector):
            result_dict = {
                'data': result.data,
                'status': 'DONE',
                'success': True,
            }
        elif isinstance(result, np.ndarray):
            if name:
                header = {'name': name}
            else:
                header = {}
            result_dict = {
                'data': {'unitary': result},
                'status': 'DONE',
                'success': True,
                'shots': 1,
                'header': header
                }
        else:
            raise Exception
        return result_dict

    def result(self):
        result_list = {}
        self.wait_for_final_state()
        warnings.warn("The result method is deprecated instead access the "
                      "result_data attribute to access the result from the "
                      "job", DeprecationWarning, stacklevel=2)
        result_dict = {
            'backend_name': self.backend.name,
            'qobj_id': '',
            'backend_version': '',
            'success': True,
            'job_id': self.job_id,
        }
        if isinstance(self.result_data, OrderedDict):
            result_list = []
            for result in self.result_data:
                result_list.append(
                    self._result_conversion(self.result_data[result], result))
        elif isinstance(self.result_data, Counts, Statevector, np.ndarray):
            result_list = [self._result_conversion(self.result_data)]
        else:
            raise TypeError(
                "Result for job %s is not a circuit result and a backwards "
                "compat Result object can't be constructed for "
                "it" % self.job_id)
        result_dict['results'] = result_list
        return Result.from_dict(result_dict)

    @abstractmethod
    def status(self):
        pass

    def cancel(self):
        raise NotImplementedError

    @abstractmethod
    def wait_for_final_state():
        pass

    def _get_experiment(self, key=None):
        # Automatically return the first result if no key was provided.
        if key is None:
            if isinstance(self.result_data,
                          OrderedDict) and len(self.result_data) != 1:
                raise QiskitError(
                    'You have to select a circuit or schedule when there is more than '
                    'one available')
            key = 0
        else:
            if not isinstance(self.result_data, OrderedDict):
                raise QiskitError("You can't specify a key if there is only "
                                  "one result")

        # Key is a QuantumCircuit/Schedule or str: retrieve result by name.
        if isinstance(key, (QuantumCircuit, Schedule)):
            key = key.name
        # Key is an integer: return result by index.
        if isinstance(key, int):
            return list(self.result_data.values())[key]
        elif isinstance(key, str):
            return self.result_data[key]
        else:
            raise TypeError('Invalid key type %s' % type(key))

    def get_memory(self):
        raise NotImplementedError

    def get_counts(self, experiment=None):
        self.wait_for_final_state()
        if isinstance(self.result_data, Counts):
            return self.result_data
        elif isinstance(self.result_data, OrderedDict):
            exp_result = self._get_experiment(experiment)
            if isinstance(exp_result, Counts):
                return exp_result
            else:
                raise TypeError(
                    "Result for job %s is not a Counts object" % self.job_id)
        else:
            raise TypeError(
                "Result for job %s is not a Counts object" % self.job_id)
```

Aside from provisions and documentation on how to make a ``Job`` work with a
sync execution The key change is the flattening of the ``Job`` and result
classes together. This was an arbitrary distiction because there are very
few instances where having a job and not wanting the result from it were needed
(and those were mostly for failure cases). Because we're flattening and
combining the `Result` class into job the v1/v1.5  `Result` class's methods
have duals in the new Job class, since the job already should contain all the
result data we need. Additionally, as part of this it has a couple new fields
like `metadata` that were previously used in the `Result`.

For the job/results interface one thing to note is that there is now a Counts
class which will represent a counts result from the job (returned by
`get_counts()`). See https://github.com/Qiskit/qiskit-terra/pull/4501 for
this implementation. This will be asubclass of dict and the data format will be
the same as returned by `Result.get_counts()` in v1, but will have several
helper functions for interacting with counts data and support multiple input
formats to make it easier to construct. This is also needed to contain per
experiment metadata about the execution in the flattened model. Similarly a
statevector operator object (or a wrapper around it) and unitary object returned
from `get_statevector()` and `get_unitary()` respectively. The counts model can
also be mirrored for simulators that offere returning additional types of data
(via snapshots etc), but that would be a custom definition in that simulator's
provider. The model in terra abstract interface defines the required pieces
(or common optional) return types, anything custom to a single provider are
defined in that provider. If we decide to add something to the base definitions
we have to increase the version and create a new abstract object.

For backwards compatibility though we need to be able handle add a deprecated
`result()` method to the job class with a lengthy deprecation period (longer
than the minimum 3 months since this is a core pattern in Terra) to give users
ample time to migrate to this new model. This is necessary because while the
provider interface is a major version difference this is not actually user
facing. From the user perspective the day they migrate from qiskit-terra 0.14.0
to 0.15.0 and qiskit-aer 0.5.0 to 0.6.0 that included the migration to the new
provider interface on both sides they would just get an error when they run
`execute().result()`. Additionaly, for backwards compatibility we can't remove
the `Result` class yet, so the `Job` because we need it for `result()` method
which will generate a v1/v1.5 object from the job.

In the v2 providers interface the conversion to ibmq's api format from the
terra objects then live in the ibmq provider, where it should live. When we
migrate it from v1 to v2. Other providers will then have more flexibility to
convert the circuit objects to whichever format is needed for interfacing the
circuit to their backends/services.

We'll need to keep `assemble()` (and the dissasembler) in terra for backwards
compat and v1.5 providers, but the ibmq provider will have it's own equivalent
and it will likely diverge from the terra version because in v1.5 the qobj class
becomes the provider interface not the wire format.

## Alternative Approaches

The first obvious alternative is to keep the current system in place where
our interface between providers and Terra remains the same and hard coupled to
the wire format for IQX. However, I think that the motivation secion here covers
why this is not ideal fairly well and this really isn't an option moving forward.

The only alternative option to this plan is to eliminate the v1.5 step and go
directly to the v2 implementation. The problem with this is it either
forces us to break all the existing provider code by making a hard transition
or forces us to keep the existing structure moving forward. Neither of those is
a situation we want to be in. We either break our users in another way which
will just make our mission of having qiskit be the open source platform for
working with quantum computers. When we constantly introduce breaking changes
it just turns people off of the platform since they can't rely on it. Or we
force other backend providers to use the existing mess of a current system
while we come up with something better. (the v1.5 step is also completed at
this point)

## Questions

## Future Extensions
Once we've decoupled the providers interface and the IQX API and implemented
the v2 interface it opens up the door for adding additional resource types
to providers. This will open up the door for additional types of hosted/external
pieces of terra that integrate seemlessly. We will be able to build these
additional base classes off the model introduced here.

Additionally, one thing not included here is serialization formats for any
of the backend pieces. While the explicit goal here is to decouple the terra
interface from what IQX uses having a serialization format for things like
Backends, target, properties, etc has proven quite useful. It provides the
ability for saving backends locally which can be useful for testing, or reusing
backends in distributed execution. By moving to an abstract model a follow up
would be to define a common serialization format and add methods that
explicitly return the **serialized** format. While providers like IBMQ that
rely on webservices calls can still easily do this by just storing the payload
from the service, having a common method which can be used for all backends
will be useful and would be a simple extension on the this new model. What
we don't want to repeat though is the pseudo-serialized `to_dict()` that is a
holdover from marshmallow which both complicates the class definition and
user expectations.
