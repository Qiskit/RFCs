# Primitive options type change

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0016                                         |
| **Authors**       | Jessie Yu (jessieyu@us.ibm.com)    |
| **Deprecates**    | RFC that this RFC deprecates                 |
| **Submitted**     | 2023-10-20                                   |
| **Updated**       | 2023-11-06                                   |

## Summary

This RFC proposes using Python and `pydantic` dataclasses as the object type of primitive options. This allows users to discover supported options without having to read the documentation. It also supports auto-complete, nested options in the `x.y` format, and field validation.

## Motivation

`BasePrimitive` currently accept an `options` parameter of type `dict` to its constructor. The underlying expectation is that all options would be stored in this dictionary. However, due to the large number of options, both Qiskit Runtime and Qiskit Aer primitive implementations use a more structured approach. Transpiler options, for example, is specified via `transpile_options` in Aer and `options.transpilation.x` in Qiskit Runtime.

While `BasePrimitive` takes a dictionary `options` as an input, its `options` property actually returns a `qiskit.providers.Options` instance, which is another inconsistency. There is little reason to use `qiskit.providers.Options`, which has a lot of complicated code designed to maintain backward compatibility with Qiskit Experiments. `qiskit.providers.Options` does allow setting of validators, but that feature is not used by any of known primitive implementations.

In addition, there are features users requested around accessing options that this RFC wishes to address:
- auto-complete, this can include nested auto-complete with `x.y`
- an easy way to discover the different options for each primitive
- validation on values
- consistent way to specify the same options across different providers
- the ability to specify experimental or undocumented options
- the ability to make a copy of the options

## User Benefit

All users of the primitives stand to benefit from this proposal. It will allow users to more easily discover available options, auto-complete field names, and get prompted for invalid values when specifying them.

It will also decouple `qiskit.providers.Options` from primitives, if Qiskit wants to retire that class.

## Design Proposal

1. Use Python dataclass as the input/output type for primitive options. Dataclass allows auto-complete, and users can easily discover what fields are available.

2. The `options` property would return the dataclass, and a user can do `estimator.options.x`, to get/set the specific option. The users can also use the builtin `asdict` method to convert the dataclass to a dictionary if they wish.

2. Instead of a `set_options()` method attached to the primitive, the dataclass will have an `update()` method for bulk updates.

4. Providers that have more complex options, such as Qiskit Runtime, can use more advanced dataclass type such as the one in `pydantic`.

5. It is still up to each provider on how they want to support undocumented/experimental options.


Per #51, we will start versioning the primitives. Therefore, no backward compatibility is needed, and the changes will only apply to v2 primitives.

## Detailed Design

- `BasePrimitive` would accept either a dictionary or a dataclass for `options`.
- A new `BasePrimitiveOptions` dataclass will serve as the base class, since there is no type hint for dataclasses.
- `_options_class` class attribute is used to identify the dataclass the subclass wishes to use.
- The `options` property wll return the dataclass.
- The options dataclass will have a `update()` method for bulk update

```python
from dataclasses import dataclass, replace

@dataclass
class BasePrimitiveOptions:

    test_attr: int = 1

    def update(self, **kwargs):
        for key, val in kwargs.items():
            setattr(self, key, val)


class BasePrimitiveV2:
    """Primitive abstract base class."""

    _options_class = BasePrimitiveOptions

    def __init__(self, options: dict | BasePrimitiveOptions | None = None):
        if isinstance(options, dict):
          self.options = self._options_class(**options)
        elif options is None:
          self.options = self._options_class()
        else:
          self.options = options
```

```
>>> base = BasePrimitive()
>>> base.options.test_attr
1
>>> base.options
BasePrimitiveOptions(test_attr=1)
>>> base.options.update(test_attr=2)
>>> base.options.test_attr
2
```

## Alternative Approaches

An alternative considered was using `pydantic.BaseModel`. This class offers all the features Python dataclasses and pydantic dataclasses have, plus many useful methods, such as determining whether a field was explicitly set or defaulted to.

Unfortunately, these useful methods would also show up when you attempt to do auto-complete and could cause confusion on which ones are the actual options.

## Questions


## Future Extensions
