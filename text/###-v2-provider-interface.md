# Improved Provider Interface

| **Status**        | **Proposed**             |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0002                                         |
| **Authors**       | Matthew Treinish (mtreinish@kortar.org)      |
| **Deprecates**    | None                                         |
| **Submitted**     | 2020-02-04                                   |
| **Updated**       | 2020-02-04                                   |


# Improved Provider Interface

## Summary

Implement a new provider interface for terra that is based around python
objects instead of a pseudo-serialization format unique to ibmq/iqx. This new
interface will live in parallel to the existing provider interface for backwards
compatibility, but the new interface will be the model moving forward. The basic
goal is to decouple the wire protocol IBMQ uses from what developers have to
deal with. This frees up providers to interact with terra at a native terra
object level which provides much more flexibillity for non-ibmq and local
providers.

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
better documented interface for working with backends and results.

Additionally, it makes the domains of what's part of IBMQ/IQX and what's part
of terra much clearer. Right now to make changes to terra's interfaces this
often gets caught up in how iqx works today and limits what is possible.

## Design Proposal

### Providers Interface v1.5
The transition will happen in 2 stages. The first stage, v1.5 which is
a maintanence change for terra is to remove marshmallow from the existing
providers interface. This will make things more transparent in how they are
built, not just simplifying working with the interface, but also
significantly speed up the performance of the current interface. The first
piece of this has been pushed for the qobj class here:

https://github.com/Qiskit/qiskit-terra/pull/3383

This basically just mirrors the marshmallow class heirarchy and replaces each
class with a flat class with just attributes for each element in the schema.
The removal of marshmallow will need to be replicated for backends, jobs, and
results too before we can move on to the new interface.

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
as `json.loads()` from what the ibmq provider sends on the wire.

### Providers interface 2.0

After v1.5 is complete we can work on adding a new providers v2 interface
that basically treats the providers interface as a python interface instead of
as a pseudo-json api. The basic idea is that we use terra's objects as the
what is passed back and forth to providers instead of the intermediate formats
that exist today.

For example, the new interface for running circuits (or pulse schedules) will
look like:

```python
qc = QuantumCircuit()
provider = my_provider()
backend = my_provider.get_backend()
backend.set_config(shots=2048)
backend.run([qc]*100)
```
The abstract classes defining this new interface will live in:
`qiskit.providers.v2` in the terra repo and for terra's functionality
that interacts with anything as part of the provider interface (eg transpiler
passes that use backend configuration) will need to check if it's using a
provider based on v1 or v2 prior to interacting with it.

A base backend abstract class will be defined as:

```python
from abc import ABC

class BaseBackend(ABC):
    def __init__(self):
        configuration = self._default_config()

    @classmethod
    @abstractmethod
    def _default_config(cls):
        pass

    def set_configuration(self, **fields):
        configuration.update_config(**fields)

    @property
    def configuration(self):
        return self.configuration

    @abstractmethod
    def run(self, circuits):
        pass
```

With a configuration class that looks like:

```python

from abc import ABC


class Configuration(ABC):

    @property
    @abstractmethod
    def num_qubits(self):
        pass

    @property
    @abstractmethod
    def name(self):
        pass
```

The required fields for the base configuration are kept minimal to maximize
compatibility with all providers. Each provider is expected to define it's
own configuration subclasses which will have any extra required fields that
are specific to that backend.

On the other side of the interface, interacting with a running circuit and
it's results a new job class will be defined. This will merge in the standard
results functionality (it's still present).

```python
from abc import ABC

class BaseJob(ABC):
    """A base class for representing an async job on a backend."""

    job_id = None
    metadata = None

    def __init__(self, job_id, backend):

    def backend(self):
        return

    @abstractmethod
    def result():
        pass

    @abstractmethod
    def status():

    def cancel():
        raise NotImplementedError

    def wait_for_final_state():
        raise NotImplementedError

    def get_memory(self):
        return self.result().get_memory()

    def get_counts(self):
        return self.result().get_counts()

    def get_statevector(self):
        return self.result().get_statevector()

    def get_unitary(self):
        return self.result().get_statevector()
```

The job class is mostly the same except it is using the new fields. One thing
that has been expanded is that the result classes methods have duals here. There
are few times where you'll want or need to interact with a result directly.

```python
from abc import ABC

class Result(ABC):
    """An object representing an experiment's result"""

    @abstractmethod
    def data(self):
        pass

    def get_memory(self):
        raise NotImplementedError

    @abstractmethod
    def get_counts(self):
        """Return a Counts object for the result if applicable."""
        pass

    @abstractmethod
    def get_statevector(self):
        """Return a statevector operator object of the result if applicable."""
        pass

    @abstractmethod
    def get_unitary(self):
        """Return a unitary operator object of the result if applicable."""
        pass
```

For the job/results interface one thing to note is that there is now a Counts
class which will represent a counts result from the job (returned by
`get_counts()`). This will be a subclass of dict and the data format will be
the same as returned by `Result.get_counts()` now, but will have several helper
functions for interacting. Similarly a statevector operator object and a
unitary operator object will be returned from `get_statevector()` and
`get_unitary()` respectively.

In the v2 providers interface the conversion to ibmq's api format from the
terra objects then live in the ibmq provider, where it should live, when we
migrate it from v1 to v2. Other providers will then have more flexibility to
convert the circuit objects to whichever format is needed for interfacing the
circuit to their backends/services.

We'll need to keep `assemble()` (and the dissasembler) in terra for backwards
compat and v1.5 providers, but the ibmq provider will have it's own equivalent
and it will likely diverge from the terra version because in 1.5 the qobj class
becomes the provider interface not the wire format.

## Alternative Approaches

Well the first obvious alternative is to keep the current system in place where
our interface between providers and terra remains the same and hard coupled to
the wire format for IQX. However, I think that the motivation secion here covers
why this is not ideal fairly well and this really isn't an option moving forward.

The only obvious alternative option to this plan is to eliminate the v1.5 step
and go directly to the v2 implementation. The problem with this is it either
forces us to break all the existing provider code by making a hard transition
or forces us to keep the existing structure moving forward. Neither of those is
a situation we want to be in. We either break our users in another way which
will just make our mission of having qiskit be the open source platform for
working with quantum computers. When we constantly introduce breaking changes
it just turns people off of the platform since they can't rely on it. Or we
force other backend providers to use the existing mess of a current system
while we come up with something better.

## Questions


## Future Extensions
Once we've decoupled the providers interface and the IQX API and implemented
the v2 interface it opens up the door for adding additional resource types
to providers. This will open up the door for additional types of hosted/external
pieces of terra that integrate seemlessly.
