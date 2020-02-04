# Integer variable encoding with binary variables

| **Status**     | **Proposed**                           |
|:-------------- |:-------------------------------------- |
| **RFC #**      | ####                                   |
| **Authors**    | Takashi Imamichi (imamichi@jp.ibm.com) |
| **Deprecates** | RFC that this RFC deprecates           |
| **Submitted**  | 2020-02-06                             |
| **Updated**    | 2020-02-06                             |

## Summary

This RFC proposes the encoding of integer variable with binary variable of `OptimizationProblem`.

Note that it is based on [the converter framework](opt-prob-conv.md).


## Motivation

- `OptimizationProblem` is supposed to support binary, integer, and continuous variable.
- This RFC proposes a way of encoding integer variable with binary variables.
- We can convert `OptimzationProblem` with binary and integer variables into QUBO or a qubit Hamiltonian.
- Users of `OptimizationProblem` benefit because this RFC allows them to use integer variables as well as binary variables.

## User Benefit

- Target: users of `OptimizationProblem`.
- This RFC enables integer variables of `OptimizationProblem`.

## Design Proposal

We propose to implement a class `IntegerToBinaryConverter` that converts
`OptimizationProblem` with following conditions.

- integer variables with both lower and upper bounds
- equality constraints


### Option 1: explicit conversion

```python
op = OptimizationProblem()
op.variables.add(names=["x0", "x1"], types='I'*2, lb=[0]*2, ub=[10]*2)
... # define an optimization problem
op.variables.get_type()
# ['I', 'I']

conv = IntegerToBinaryConverter()
op2 = conv.int2bin(op)
op2.variables.get_type()
# ['B', 'B', ..., 'B']

op2.solve()
op2.solution.get_values()
# [0, 1, ..., 0]
op3 = conv.bin2int(op2)
op.solution.get_values()
# [2, 5]
```

### Option 2: implicit conversion

We have another option to hide the conversion as follows.

```python
op = OptimizationProblem()
op.variables.add(names=["x0", "x1"], types='I'*2, lb=[0]*2, ub=[10]*2)
... # define an optimization problem
op.variables.get_type()
# ['I', 'I']
op.solve()
# apply `IntegerToBinaryConverter` inside `op.solve()`
op.solution.get_values()
# [2, 5]
```

## Detailed Design

`IntegetToBinaryConverter` has two functions: one encodes integer variables with binary variables and the other converts binary numbers into integer numbers corresponding to original integer variables. We propose to adopt _bounded-coefficient encoding_ [arxiv:1706.01945 Eq. (5)](https://arxiv.org/abs/1706.01945)

When we encode an integer variable with binary variables, it would be helpful if we can add some information to each auxiliary binary variable for conversion of solution. It is necessary to define naming rule of auxiliary binary variables. For example, suppose that we convert an integer variable `x` with range [0, 3]. We represent it using 2 auxiliary binary variables `x_1` and `x_2` such that `x = x_1 + 2 * x_2`, where we use character '_' as delimiter.

Note that we need following special method that are not available as part of CPLEX API,

- `OptimizationProblem.solution._set_solution` to set decoded solution values.

- `OptimiztionProblem.copy` to deep-copy internal data structure. 

```python
class IntegerToBinaryConverter(OptimizationProblemConverter):
    _delimiter = '@' # users are supposed not to use this character in variable names

    def __init__(self):
        self._src = None
        self._dst = None
        self._conv: Dict[str, List[Tuple[str, int]]] = {}
        # e.g., self._conv = {'x': [('x@1', 1), ('x@2', 2)]}

    def encode(self, op):
        self.int2bin(op)

    def decode(self, op):
        self.bin2int(op)

    def int2bin(self, op: OptimizationProblem) -> OptimizationProblem:
        self._src = op.copy() # deep copy
        self._dst = OptimizationProblem()

        # declare variables
        names = self._src.variables.get_names()
        types = self._src.variables.get_types()
        lb = self._src.variables.get_lower_bounds()
        ub = self._src.variables.get_upper_bounds()
        for i, name in enumerate(names):
            typ = types[i]
            if typ == 'B':
                self._dst.variables.add(names=[name], types='B')
            elif typ == 'C':
                self._dst.variables.add(names=[name], types='C')
            elif typ == 'I':
                new_vars: List[Tuple[str, int]] = self._encode_var(name=name, lb=lb[i], ub=ub[i])
                self._conv[name] = new_vars
                self._dst.variables.add(names=[name for name, _ in new_vars], types='B')
            else:
                raise QiskitError('Variable type not supported: ' + typ)

        # replace integer variables with binary variables in the objective function
        self.objective.subs(self._conv)

        # replace integer variables with binary variables in the constrains
        self.linear_constraints.subs(self._conv)
        self.quadratic_constraints.subs(self._conv)
        # note: `subs` substibutes variables with sets of auxiliary variables

        return self._dst

    def _encode_var(self, name:str, lb:int, ub:int) -> List[Tuple[str, int]]:
        # bounded-coefficient encoding proposed in arxiv:1706.01945 (Eq. (5))
        n = ...
        lst = []
        for i in range(n):
            coef = ...
            new_name = name+self._delimiter+str(i)
            lst.append((new_name, coef))
        return lst

    def bin2int(self, op: OptimizationProblem) -> OptimizationProblem:
        names = op.variables.get_names()
        vals = op.solution.get_values()
        new_vals = self._decode_var(names, vals)
        ret = self._src.copy()
        ret.solution._set_values(new_vals)  # a special method that does not exist in CPLEX API
        # perhaps we need to copy some solution information of `op.solution`
        return ret
                                
    def _decode_var(self, names, vals) -> List[int]:
        # decode integer values
        sol = {name: vals[i] for i, name in enumerate(names)}
        new_vals = []
        for i, name in enumerate(self._src.variables.get_names()):
            if name in self._conv:
                new_vals.append(sum(sol[aux] * coef for aux, coef in self._conv[name]))
            else:
                new_vals.append(sol[name])
        return new_vals
```

## Questions

Open questions for discussion and an opening for feedback.

## Future Extensions

- Conversion of inequality constraints into equality constraints by introducing integer or continuous slack variables.
  
  - We need to compute ranges of integer slack variables to encode them with binary variables.
