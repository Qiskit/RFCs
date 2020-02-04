# Converters of Optimization Problems

| **Status**     | **Proposed**                           |
|:-------------- |:-------------------------------------- |
| **RFC #**      | ####                                   |
| **Authors**    | Takashi Imamichi (imamichi@jp.ibm.com) |
| **Deprecates** | RFC that this RFC deprecates           |
| **Submitted**  | 2020-02-06                             |
| **Updated**    | 2020-02-06                             |

## Summary

This RFC proposes a framework to convert `OptimizationProblem`.

## Motivation

- We need to transform`OptimizationProblem` to apply quantum algorithms
  - Convert inequality constraints into equality constraints
  - Replace continuous variables with numbers
  - Make an unconstrained problem by penalizing equality constraints
  - Encode integer variables with binary variables
- We propose a common framework to implement such converters.

## User Benefit

- Target: users of `OptimizationProblem`.
- It allows users to implement their own converters of `OptimizationProblem`
- It allows users to apply various converters with the same API.

## Design Proposal

We propose an abstract class `OptimizationProblemConverter` and implement actual converters based on it.

```python
class OptimizationProblemConverter:
    def encode(self, op: OptimizationProblem) -> OptimizationProblem:
        # convert an original problem into a specific type of problem
        pass

    def decode(self, op: OptimizationProblem) -> OptimizationProblem:
        # convert the solution of `op` into that of the original problem
        pass
```

### Use case

```python
op = OptmizationProblem()
op.variables.add(names=["x", "y"], types="C"*2, lb=[0]*2)
op.objective.set_linear([("x", 1.0), ("y", 2.0)])
op.linear_constraints.add(
    lin_expr=[
        SparsePair(ind=["x", "y"], val=[1.0, -1.0]),
    senses = ["L"],
    rhs = [1.0],
    names=["c"]
)
# (op)
# min  x + 2 * y
# s.t. x - y <= 1.0
#      x, y >= 0.0

i2e = IneqalityToEqualityConverter(continuous_slack=True)
op2 = i2e.encode(op)
# (op2)
# min x + 2 * y
# s.t. x - y + z = 1.0
# x, y, z >= 0.0

v2r = ReplaceVariables({'y': 1})
op3 = v2r.encode(op)
# (op3)
# min x + 2
# s.t. x - 1 + z = 1.0
# x, z >= 0.0

op3.solve()
# (op3 solution)
# x = 0, z = 2, obj = 2

op4 = v2r.decode(op3)
# (op2 & op4 solution)
# x = 0, y = 1, z = 2, obj = 2

op5 = i2e.decode(op4)
# (op1 & op5 solution)
# x = 0, y = 1, obj = 2
# note that the solution is not optimal because we fix y = 1 in `op3`.
```

## Detailed Design

It is useful to apply various converters at once as follows.

### Use case

```python
op = OptimizationProblem()
op.variables.add(...)
...

conv = SetOfConverters([
    InequalityToEqualityConverter(contiuous_slack=True),
    IntegerToBinaryConverter(),
    ReplaceVariables({'x': 0, 'y': 1})
])

op2 = conv.encode(op)
op2.solve()
op3 = conv.decode(op2)
```

```python
class SetOfConverters:
    def __init__(lst: List[OptimizationProblemConverter]):
        self._lst = lst
    
    def encode(op):
        new_op = op.copy()
        for conv in self._lst:
            new_op = conv.encode(new_op)
        return new_op

    def decode(op):
        new_op = op.copy()
        for conv in self._lst[::-1]:
            new_op = conv.decode(new_op)
        return new_op
```



## Questions

It is not efficient to generate an object every time we replace a variable with a different number. Should such a function be embedded in `OptimizationProblem`?

## Future Extensions


