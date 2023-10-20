# Primitive options type change

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0016                                         |
| **Authors**       | Jessie Yu (jessieyu@us.ibm.com)    |
| **Deprecates**    | RFC that this RFC deprecates                 |
| **Submitted**     | 2023-10-20                                   |
| **Updated**       | YYYY-MM-DD                                   |

## Summary

This RFC proposes using Python and `pydantic` dataclasses as the object type of primitive options. This allows users to discover supported options without having to read the documentation. It also supports auto-complete, nested options in the `x.y` format, and field validation.

## Motivation

`BasePrimitive` currently accept an `options` parameter of type `dict` to its constructor. The underlying expectation is that all options would be stored in this dictionary. However, due to the large number of options, both Qiskit Runtime and Qiskit Aer primitive implementations use a more structured approach. Transpiler options, for example, is specified via `transpile_options` in Aer and `options.transpilation.x` in Qiskit Runtime.

Further more, while `BasePrimitive` takes a dictionary `options` as an input, its `options` property actually returns a `qiskit.providers.Options` instance, which is another inconsistency. There is little reason to use `qiskit.providers.Options`, which has a lot of complicated code designed to maintain backward compatibility with Qiskit Experiments. `qiskit.providers.Options` does allow setting of validators, but that feature is not used by any of known primitive implementations.

In addition, there are additional features users requested around accessing options:
- auto-complete, this can include nested auto-complete with `x.y`
- an easy way to discover the different options for each primitive
- validation on values
- consistent way to specify the same options across different implementations
- the ability to specify experimental or undocumented options
- the ability to make a copy of the options

## User Benefit

All users of the primitives stand to benefit from this proposal. It will allow users to more easily discover available options, auto-complete field names, and get prompted for invalid values when specifying them.

It will also decouple `qiskit.providers.Options` from primitives, if Qiskit wants to retire that class.

## Design Proposal

1. Use Python dataclass as the input/output type for primitive options. Dataclass allows auto-complete and is self documenting.

2. Make the `options` property callable, so a user can do `estimator.options.x`, to get/set the specific option, and `estimator.options()` to get all options. `options()` will return a copy of the options.

2. `set_options` can continue to be used for bulk update.

4. Providers that have more complex options, such as Qiskit Runtime, can use more advanced dataclass such as the one in `pydantic`. `pydantic` dataclasses are very similar to Python dataclasses but allow easier validation. It would also return `True` when using `dataclasses.is_dataclass` to check its type.

    While it is still up to each provider to implement the options, this allows a standardized `x.y` approach instead of separate `nnn_options` dictionaries. We can document the common grouping of options in base primitives, such as `transpilation` and `execution`, to encourage provider to use the same terminologies.

    Similarly, it is up to each provider on how they want to support undocumented options. Qiskit Runtime, for example, plans to support them under an `experimental` field that takes arbitrary kwargs.


Backward compatibility depends on how we do migration for the primitive interface changes (see #51). If we decide to use a different version, then no backward compatibility is required. Otherwise, we'd need to use a different attribute, such as `settings`, and `options` would continue to return `qiskit.providers.Options`.

## Detailed Design

- `BasePrimitive` would accept either a dictionary or a dataclass as the input.
- The `options` property becomes callable, which would allow both auto-complete and getting a copy of all options.
    - Alternatively, we can use `get_options` to return a copy of the options.
- `set_options` can continue to be used for bulk update:

```python
from dataclasses import dataclass, replace

@dataclass
class BasePrimitiveOptions:

    test_attr: int = 1

    # Some recommended fields:
    # transpilation: TranspilationOptions  # For transpiler options
    # execution: ExecutionOptions  # For run options
    # simualtor: SimulatorOptions  # For simulator options

    def __call__(self):
        return replace(self)


class BasePrimitive:
    """Primitive abstract base class."""

    _options_class = BasePrimitiveOptions

    def __init__(self, options: dict | BasePrimitiveOptions | None = None):
        if isinstance(options, dict):
          self._options = self._options_class(**options)
        elif options is None:
          self._options = self._options_class()
        else:
          self._options = options

    @property
    def options(self) -> BasePrimitiveOptions:
        """Return options values for the estimator.

        Returns:
            options
        """
        return self._options

    def set_options(self, **fields):
        options_dict = asdict(self._options)
        options_dict.update(fields)
        self._options = self._options_class(**options_dict)
```

```
>>> base = BasePrimitive()
>>> base.options.test_attr
1
>>> base.options()
BasePrimitiveOptions(test_attr=1)
```

`qiskit-ibm-runtime` and `qiskit-aer` can use `pydantic` to construct more complex options:

```python
from pydantic.dataclasses import dataclass as pydantic_dataclass
from pydantic import Field, ConfigDict, model_validator

@pydantic_dataclass
class TranspilationOptions:
    skip_transpilation: bool = False


@pydantic_dataclass(config=ConfigDict(validate_assignment=True))
class EstimatorOptions(BasePrimitiveOptions):

    resilience_level: int = Field(1, ge=0, le=3)
    transpilation: TranspilationOptions = Field(default_factory=TranspilationOptions)

    @model_validator(mode='after')
    def _validate_model(self):
        if self.transpilation.skip_transpilation and self.resilience_level > 0:
            raise ValueError("You cannot skip transpilation when doing error mitigation.")
        return self


class Estimator(BasePrimitive):
  _options_class = EstimatorOptions

```

```
>>> estimator = Estimator()
>>> print(estimator.options())
EstimatorOptions(test_attr=1, resilience_level=1, transpilation=TranspilationOptions(skip_transpilation=False))

>>> estimator.options.transpilation.skip_transpilation = True
>>> print(estimator.options.transpilation.skip_transpilation)
True

>>> estimator.options.resilience_level = 5
ValidationError: 1 validation error for EstimatorOptions ...

>>> estimator.set_options(transpilation={"skip_transpilation": False})
>>> print(estimator.options)
EstimatorOptions(test_attr=1, resilience_level=1, transpilation=TranspilationOptions(skip_transpilation=False))
```

## Alternative Approaches

An alternative considered was using `pydantic.BaseModel`. This class offers all the features Python dataclasses and pydantic dataclasses have, plus many useful methods, such as determining whether a field was set or defaulted to.

Unfortunately, these convenient methods would also show up when you attempt to do auto-complete and could cause confusion on which one are the options.

## Questions


## Future Extensions
