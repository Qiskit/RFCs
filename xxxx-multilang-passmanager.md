# Qiskit pass manager in rust

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Julien Gacon (jul@zurich.ibm.com)            |
| **Submitted**     | YYYY-MM-DD                                   |
| **Updated**       | YYYY-MM-DD                                   |

## Summary

We will implement a Qiskit pass manager in Rust, which is exposed to both the Python and C APIs.
The new pass manager supersedes the existing Python pass manager in a backward compatible fashion.
It will share the same key concepts, such as a pass, task, property set and pass manager
which facilitates backwards compatibility and adoptation. Newly, the pass manager is solely 
responsible for workflow execution, multi-IR pipelines can be verified upon construction (if
type information is available), and callbacks can be registered at different entry points.

## Motivation

Compilation of a quantum computing program is a complex process. Qiskit's existing pass manager
system served the purpose for near-term compiler pipelines, which were written in Python and mainly
using a single IR, the `DAGCircuit`, with a single builder-interface, the `QuantumCircuit`.
Even though the pass manager infrastructure is written in Python, the compiler is highly performant 
as the passes themselves, where the work is, are written in Rust.

Nevertheless, there are two main reasons to write a new pass manager in Rust:

1. Qiskit's C API requires a pass manager. For compatibility with Python extensions and for reasons
   of consistency and code maintainability, both the Python and C pass manager should be backed by
   the same Rust datastructure.

   This will also enable writing compiler passes in other languages, such as third party Rust 
   (frequently requested), by means of a C FFI.

2. Several compiler workflows are arising that use multiple IRs, most notably in the fault-tolerant
   compiler frameworks. Python does not provide adequate tooling to ensure the pipelines we are
   building have correct types and lowerings. Rust, on the other hand, does.

   This becomes especially important as we are developing a fault-tolerant compiler pipeline
   in Qiskit itself. The Rust pass manager enables us to build the entire pipeline directly in 
   Rust, with compatibility checks.

## User benefit

- The C API will get a pass manager. 

- By the C FFI, passes and pass managers can also be written 
  in other languages, such as third party Rust code. Users will also be able to write passes for their
  Python compiler frameworks using compiled Python extensions.

- In the long run, users will benefit from the inherent type checks in our predefined, mutable 
  pipelines, since compatibility errors are more easily detected and helpful errors can be raised.
  The type checks for custom passes is _very_ limited (in Python) or non-existing (in C).


## Design proposal

The core requirements for the pass manager are:

- Enable transformations, analyses and lowerings on multiple IRs
- Support of flow control, such as conditionals, or loops
- Convenient setup using preset pipelines that are mutable and pluggable
- Status information via callbacks
- An analysis manager for context-specific information for executing passes
- Implementing passes in Python and C (and other languages by extension)
- Backward compatibility with Qiskit's existing system 

We propose these four main concepts and responsibilities:

1. `Pass`: An atomic unit of work. A pass is some form of atomic transformation on the input IR, such as 
   a specific optimization or a lowering. This what users should implement when implementing a single 
   compiler pass.
2. `Task`: A single unit of execution flow. This describes how work is executed, including non-linear
   execution such as loops or switches, or grouped execution of task, or named stages.
3. `PassManager`: Orchestrates the execution of a set of tasks. This includes tracking required 
    analysis tasks, providing a runtime context to the tasks, and passing a callback.
4. `PassContext`: Contains and provides context information. Can be thought of the equivalent
   of an analysis manager is the new LLVM pass manager.

Important: This framework needs to be extensible by design.  There are additional compiler 
features on the horizon (e.g. regions, an IR trait, more data in `PassContext`, ...) and the
new system must be able to accomodate this in the future.

### Passes

We define a pass as implementable trait, where the input and output IRs are set via
associated types.  We use associated types over generics since a pass is typically tied 
to an IR and does not need to be generic over it. 
```rust
/// An atomic transformation. Runs on an input IR and produces an output IR.
pub trait Pass {
    // The IRs must be static to store them in a type-erased pass,
    // see the following sections for more detail on this requirement.
    type InputIR: 'static; 
    type OutputIR: 'static;

    /// Run the pass on the IR, given the context.
    fn run(&self, ir: Self::InputIR, context: &mut PassContext) -> anyhow::Result<Self::OutputIR>;

    /// Which analysis does this preserve?
    fn preserves(&self) -> PreservedAnalysis;
}
```
We could add a `type Error` associated type and have `run` returns a
`Result<Self::OutputIR, Self::Error>` instead of using `anyhow::Result`, though for simplicity
we're starting out with the latter.

Under the hood we'll store a type-erased version of the pass, since the associated type is must be
fully specified to use the trait in a type specification.  Since the types are user-specified,
we cannot store a `dyn Pass` object in a task or pass manager, but instead need to use 
a type-erased version that does not have the associated types.  The types can instead be queried
by methods returning the `TypeId`. 
```rust
/// A type-erased version which we can call using dyn Any as input.
trait TypeErasedPass {
    fn input_ir(&self) -> TypeId;
    fn output_ir(&self) -> TypeId;
    fn run(&self, ir: Box<dyn Any>, context: &mut PassContext) -> Box<dyn Any>;
}
```
Note that `TypeId`s are only defined for `'static` types!  This implies that any IR in the 
pass manager is static.
For now, being static is the only requirement for an IR.  We are free to inject an `QiskitIR` 
trait that every IR has to implement, if the need arises.

We provide a blanket implementation for objects with the `Pass` trait
```rust
/// ... plus a blanket implementation for objects with the [Pass] trait
impl<P: Pass> AnyPass for P {
    fn input_type_id(&self)  -> TypeId { TypeId::of::<P::InputIR>() }
    fn output_type_id(&self) -> TypeId { TypeId::of::<P::OutputIR>() }
    fn preserves(&self) -> PreservedAnalysis { Pass::preserves(self) }

    fn run(&self, ir: Box<dyn Any>, context: &mut PassContext) -> anyhow::Result<Box<dyn Any>> {
        let ir = ir
            .downcast::<P::InputIR>()
            .expect("The pipeline construction guarantees the input IR has the correct type.");
        let out_ir = P::run(&self, *ir, context)?;
        Ok(Box::new(out_ir))
    }
}
```

**Questions**

- Should we return the preserved analysis information dynamically, as part of the `run` result?
  That would allow passes to decide during runtime which information is preserved, e.g. 
  declare that they did not act at all.
- Do we need to provide a `requires()` method that allows listing passes that must run before this,
  or can we leave this in Python space? Probably we need this here.

### Tasks

A `Task` is implemented as enum, the variants describing different execution units.
```rust
/// An execution unit.
#[non_exhaustive]
pub(crate) enum Task {
    /// A single pass.
    Transformation(Box<dyn AnyPass>), // here we need the type erasure -- <dyn Pass> is not enough!

    /// A task group. Used to conceptually group and to execute a sequence in conditionals.
    Group(Vec<Task>),

    /// A sequence of named tasks.
    Stages(Vec<(String, Task)>),

    /// A conditional execution of a task.
    /// Takes a switch function, that takes the type-erased IR and the pass context,
    /// and returns an index to which case to run.
    Switch {
        switch: fn(&dyn Any, &PassContext) -> usize,
        cases: Vec<Task>,
    },

    /// A looped execution.
    /// Runs the body until the condition function returns false.
    Loop {
        condition: fn(&dyn Any, &PassContext) -> bool,
        body: Box<Task>, // must be boxed to define Task's size
    },
}

impl Task {
    /// Return the input IR and output IR type of the task.
    /// Needed to check type compatibility.
    /// 
    /// # Returns
    /// 
    /// * Ok((TypeId, TypeId)) - The type IDs of a non-empty task.
    /// * Err(PassManagerError) - An error if the task is empty.
    fn io_types(&self) -> Result<(TypeId, TypeId), PassManagerError> { ... }
}
```
The `Task` enum is defined as [`#[non_exhaustive]`](https://doc.rust-lang.org/reference/attributes/type_system.html)
since we might add additional task variants in the future and this allows us to remain backward compatible.

There's a design question here on allowing empty `Task`s, such as
```rust
let empty = Task::Group(vec![]);
```
This can come up naturally e.g. if a user removes tasks from a group, but empty tasks 
make checking the types for compatibility cumbersome.  If in the future we would like to enable
workflows where tasks are temporarily empty, but populated later on, then we can enable empty
tasks at a more complex type check cost.

**Questions**

- Python's flow controllers only take the property set as input for controllers, such as 
  the while-loop or conditionals, not the IR itself. Do we want to keep this, or add the IR?

### Pass manager

The pass manager is in charge of executing the tasks.
```rust
pub struct PassManager {
    tasks: Vec<Task>,
}

/// A callback registry.
///
/// Callbacks have different hook points: After pass, after task and after stages.
/// Since we want the callback to handle any IR that we're going through in the compiler
/// pipeline, we're using type-erased `dyn Any` which needs to be downcast by the implementer.
pub struct CallbackRegistry {
    post_pass: Vec<Box<dyn Fn(&dyn Any, &PassContext) -> ()>>,
    post_task: Vec<Box<dyn Fn(&dyn Any, &PassContext) -> ()>>,
    post_stage: Vec<Box<dyn Fn(&dyn Any, &PassContext) -> ()>>,
}

impl PassManager {
    pub fn new() -> Self { Self { tasks: vec![] } }

    pub fn run<IRIn, IROut>(
        &self, ir: IRIn, 
        callback: Option<&CallbackRegistry>
    ) -> Result<IROut, PassManagerError>
    where
        IRIn: 'static,
        IROut: 'static,
    {
        if self.tasks.len() == 0 {
            // If there are no tasks, return the input, but cast to IROut
            return match Box::new(ir).downcast::<OutType>() {
                Ok(out) => return Ok(*out),
                Err(_) => Err(PassManagerError::FailedOutputConversion),
            }
        }

        let (first, _) = self.tasks[0].io_types()?;
        if first != TypeId::of::<IROut>() {
            return Err(PassManagerError::IncompatibleTypes);
        }

        // Erase the type to pass it through the task execution
        let mut ir: Box<dyn Any> = Box::new(ir);

        // Main iteration loop over tasks
        let mut context = PassContext::new();
        for task in self.tasks.iter() {
            ir = execute_task(task, ir, &mut context, callback)?;
        }

        match obj.downcast::<OutType>() {
            Ok(out) => Ok(*out),
            Err(_) => Err(PassManagerError::FailedOutputConversion),
        }
    }
}

/// The task runner. This should not be called standalone, passes should be run
/// via the pass manager.
fn execute_task(
    task: &Task,
    mut ir: Box<dyn Any>,
    context: &mut PassContext,
    callbacks: Option<&CallbackRegistry>,
) -> Result<Box<dyn Any>, PassManagerError> {
    let out = match task {
        Task::Transformation(pass) => {
            let out = pass
                .run(ir, context)
                .map_err(|e| PassManagerError::PassError(e))?;
            callbacks.map(|cb| cb.call_post_pass(&out, context));
            Ok(out)
        }
        Task::Group(tasks) => {
            for task in tasks.iter() {
                ir = execute_task(task, ir, context, callbacks)?;
            }
            Ok(ir)
        }
        Task::Switch { switch, cases } => {
            let index = switch(&ir, context);
            execute_task(&cases[index], ir, context, callbacks)
        }
        Task::Loop { condition, body } => {
            while condition(&ir, &context) {
                ir = execute_task(&body, ir, context, callbacks)?;
            }
            Ok(ir)
        }
        Task::Stages(stages) => {
            for (_name, task) in stages.iter() {
                ir = execute_task(task, ir, context, callbacks)?;
                callbacks.map(|cb| cb.call_post_stage(&ir, context));
            }
            Ok(ir)
        }
    };
    callbacks.map(|cb| cb.call_post_task(&out, context));
    out
}
```

The pass manager provides functionality to modify the tasks; appending, removing, inserting, ...
Every modification must leave the pass manager in a valid state.
```rust
impl PassManager {
    /// Try push a [Task] to the pass manager. Returns an error if the types are not
    /// compatible.
    pub fn try_push_task(&mut self, task: Task) -> Result<(), PassManagerError> {
        // Check that the task types are compatible, if there's an existing task and if
        // neither of the tasks are empty.
        if let Some(last_task) = self.tasks.last() {
            let (_, out_type) = last_task.io_types()?;
            let (in_type, _) = task.io_types()?;
            if in_type != out_type {
                return Err(PassManagerError::IncompatibleTypes);
            }
        }
        self.tasks.push(task);
        Ok(())
    }

    /// ... plus some other mutation methods, like removing or injecting a task.
}
```

### Pass context

The `PassContext` contains context information provided to the passes, akin to 
the existing `PropertySet`.
```rust
/// The key under which analyses results or properties are stored.
pub struct Property(String);

pub struct PassContext {
    data: HashMap<Property, Box<dyn Any>>
}

/// Specify which analyses are preserved by a pass.
pub enum PreservedAnalysis {
    All,
    None,
    Preserves(&[Property]),  // specify which are preserved
    Invalidates(&[Property]),  // specify which are invalidated
}
```

### Extension: Analysis passes

We could add dedicated analysis passes, which only take an IR by reference and does not
produce an output.  This allows to skip type checks and is the formally correct object to
implement analyses.

This is something we likely want since it provides safety guardrails and ensure that analysis
passes cannot modify the IR, but it's not a technical requirement for the first version.

```rust
pub trait AnalysisPass {
    type InputIR;

    /// Run the pass and update the context in-place.
    pub fn run(&self, ir: &Self::InputIR, context: &mut PassContext);

    /// Report which properties this pass computes.
    pub fn properties(&self) -> Vec<Property>;
}
```

In the same fashion as for the transformation `Pass` we will have 
type-erased version and blanket implmentation
```rust
trait AnyAnalysisPass { 
    fn input_type_id(&self) -> TypeId;
    fn run(&self, ir: &dyn Any, context: &mut PassContext) -> anyhow::Result<()>;
    fn properties(&self) -> Vec<Property>;
}

impl<A: AnalysisPass> TypeErasedAnalysisPass for A {
    fn input_type_id(&self)  -> TypeId { TypeId::of::<A::InputIR>() }
    fn properties(&self) -> Vec<Property> { A::properties(self) }

    fn run(&self, ir: &dyn Any, context: &mut PassContext) -> anyhow::Result<()> {
        let ir = ir
            .downcast::<P::InputIR>()
            .expect("The pipeline construction guarantees the input IR has the correct type.");
        A::run(&self, &ir, context)
    }
}
```

The `Task` enum would gain an additional variant
```rust
pub enum Task {
    /// [ ... ]
 
    /// A single pass.
    Analysis(Box<dyn AnyAnalysisPass>),

    /// [ ... ]
}
```

### Extension: Analysis registry

In the current design, analysis passes populating the property set need to be 
scheduled manually. This is inconvenient since the requirements are tied to _passes_ rather
than _properties_ (since there could be different ways of computing the required property).

Another solution is to request a property from the pass context, akin to the 
`AnalysisManager` in LLVM's new pass manager. The pass context would contain a registry of 
analysis passes it can run to compute a required property, and would be in charge of maintaining
the cached data.

```rust
pub struct PassContext {
    data: HashMap<Property, Box<Any>> 

    /// The analysis passes the context knows about.
    registry: HashMap<Property, Box<dyn AnyAnalysisPass>>
}

impl PassContext {
    /// Register an analysis we can run.
    pub fn register(&mut self, analysis: Box<dyn AnyAnalysisPass>) {
        for property in analysis.properties() {
            self.registry.insert(property, analysis);
        }
    }

    /// Update the cache given the preserved analyses.
    pub fn update(&mut self, preserved: &PreservedAnalysis) { ... }

    /// Request an analysis. 
    /// 
    /// Try fetch from the cache, else try compute using the registry, else we cannot do it.
    pub fn request(&mut self, property: &Property) -> Result<Box<dyn Any>, PassContextError> {
        ...
    }

    /// Manually add something to the cache.
    pub fn insert(&mut self, key: &AnalysisKey, value: Box<dyn Any>) {
        self.data.insert(key, value);
    }
}
```

This is a feature Qiskit's pass manager system currently does not have and as such is not
a technical requirement. If there's concrete evidence that this improves performance or quality,
we can add it.

### C API

In C, we will rely on function pointers to implement custom passes and `void`-pointers for 
IRs. We have no way of verifying that the IRs are compatible and it is entirely up to the user to
ensure the pipeline is set up correctly. Inconsistent pipelines will probably panic.

The pass manager, the pass context and the callback registry are structs and will be exposed
to C as opaque pointers. We will provide functions to interact with these objects, e.g.
```c
QkPassManager *qk_passmanager_new(void);  // create a new empty PM
QkCallbackRegistry *qk_passmanager_new_callback(void);  // create a new empty CB registry
void *qk_passmanager_run(void* ir, QkCallbackRegistry* callbacks);  // run a passmanager


QkPassManager pm *qk_preset_passmanager(int optimization_level);
qk_passmanager_append(pm, pass);
```

A pass will be exposed to C as opaque struct with functions to set the required methods.
We are opting for an opaque struct rather than exposing a `#[repr(C)]` struct to keep 
the option of modifying the trait without breaking ABI compatibility.
```rust
// note: all of this below is coded in the markdown, it's Rust pseudo-code

/// The pass' run method with signature
/// void* run(void *self, void *ir, QkPassContext *context);
type RunFunctionPtr = extern "C" fn(*mut c_void, *mut c_void, *mut PassContext) -> *mut c_void;

struct PassFromC {
    ptr_self: *mut c_void,
    ptr_run: RunFunctionPtr,
    ptr_preserved_analyses: Box<dyn Fn(*mut c_void) -> *const PreservedAnalysis>,
}

#[unsafe(no_mangle)]
pub extern "C" fn qk_pass_new() -> *mut PassFromC { ... };

#[unsafe(no_mangle)]
pub extern "C" fn qk_pass_set_self(pass: *mut PassFromC, ptr_self: *mut c_void) { ... };

#[unsafe(no_mangle)]
pub extern "C" fn qk_pass_set_run(pass: *mut PassFromC, ptr_run: RunFunctionPtr) { ... };
```
It's the user's responsibility that the functions are safe to call during the lifetime
of the pass.

With the required methods set, we can implement the `Pass` trait for `PassFromC`.
```rust
impl Pass for PassFromC {
    type InputIR = ::std::ffi::c_void;
    type OutputIR = ::std::ffi::c_void;

    fn run(&self, ir: Self::InputIR, context: &mut PassContext) -> anyhow::Result<Self::OutputIR> {
        // call to self.ptr_run ...
    }

    fn preserves(&self) -> PreservedAnalysis {
        // call to self.ptr_preserved_analyses ...
    }
 }

#[unsafe(no_mangle)]
pub extern "C" fn qk_passmanager_append_pass(pm: *mut PassManager, pass: *const PassFromC) -> QkExitCode {
    let pm = unsafe { mut_ptr_as_ref(pm) };
    let pass = unsafe { const_ptr_as_ref(pass) };
    pm.try_append_pass(pass).expect("Failed appending pass.");
}
```

The task will be exposed as opaque pointer with various functions to construct the variants.
Again, the lifetime will be tied to the C objects it is storing references to. 

### A new Python API 

There are fundamental differences of the existing Python API and the new pass manager system
we're exposing here. Differences include:

- the callback system with different hook points
- the pass context as smart object rather than a simple dictionary
- analysis passes (that have no return value)
- the `PreservedAnalysis` object to annotate preserved analyses

The existing API does not allow to enable all of these features. Instead, we propose 
to add a new Python API that can be used going forward, plus a compatibility shim for the 
existing API which can be removed in a next major release.

The new interface exposes a `Pass` base class Python-side, which uses Python's ABC module
to define the abstract interface.  Rust-side we keep a `PassFromPy` struct which inside keeps
the Python pass as `Py<PyAny>`.  This is done since it allows proper abstract interfaces in 
Python and avoids using PyO3's subclassing system, which is not super ergonomic. 
In Python we have
```python
from abc import ABC, abstractmethod
from typing import Any
from qiskit.passmanager import PassContext  # from Rust

class Pass(ABC):
    """A pass interface."""

    @abstractmethod
    def run(self, ir: Any, context: PassContext) -> Any: ...
```
and in Rust we hold
```rust
#[pyclass]
struct PassFromPy { 
    // Store the Python `Pass` instance, and we'll call it's methods where needed
    py_pass: Py<PyAny>; 
}

impl PassFromPy {
    fn new(py_pass: Py<PyAny>) -> PyResult<Self> {
        // When constructing, run instance check to verify it is a `Pass` instance
        Ok(Self { py_pass })
    }
}

impl Pass for PassFromPy {
    type InputIR = Py<PyAny>;
    type OutputIR = Py<PyAny>;

    fn run(&self, ir: Self::InputIR, context: &mut PassContext) -> anyhow::Result<Self::OutputIR> {
        Python::attach(|py| {
            // This is pseudo-code: the important thing is that the pass context
            // needs to be passed to Python's run method, too.
            self.py_pass.bind(py).call_method1(intern!("run", (ir, context))?)
        })
    }
}
```

The other objects that are key to the interface can be exposed directly by equipping them
with the `#[pyclass]` attribute and exposing Python-specific methods:

- `PassManager` + methods to modify the pass manager
- `CallbackRegistry` + methods to (de-)register callbacks
- `PassContext` + methods to query and store manual data

Python IR types are generally **not validated** upon construction of the pass. 
We could add validation for Qiskit-specific types that exist in Python and Rust, such as `DAGCircuit`,
by attempting to downcast the `Py<PyAny>` into `DAGCircuit`, but this is only a partial cover.

#### Naming conventions

The name `PassManager` is already taken from the Python model for `DAGCircuit`-specific compilation,
in the module `qiskit.transpiler`. There's several options on what to call the new pass manager.

**Option 1** Namespacing: `qiskit.passmanager.PassManager`.

`PassManager` is arguably the correct choice for a pass manager and reflects the 
importance of the object.  The obvious concern here is the confusion with the DAG-specific
`qiskit.transpiler.PassManager` which works slightly differently.  This duality is not 
necessarily only bad since it reflects the relation (but certainly will lead to some mixups.)

**Option 2** Use `MultiStagePassManager` as new pass manager.

Not introducing yet another pass manager keeps a simpler API and steers users to the new
features easily.  The `MultiStagePassManager` is already a new object and could be used as 
entry point to the new pass manager -- the only caveat is that it implies stages and has a 
`stages` attribute, which is not a requirement for the new pass manager. We might want
to have a clearer, "stronger" name for the central new pass manager.

**Option 3** Create new `NewPassManager` object (or some other new name).

This clearly shows the introduction of a new execution model.  But building the new main
execution mode on some "second-choice naming" feels a bit off.


### Backwards compatibility with the Python model

As outlined above, a new pass interface allows exposing new features of the new pass manager.
Since the existing Python API is stable and must be backward compatible,
we could simply expose the entire infrastructure as new, concurrent model -- LLVM does the
same with their new pass manager.

We propose two possible plans for backward compatibility. The common factor in both is that
`GenericPass` instances are valid passes for the new pass manager, since all Qiskit passes
are currently using that framework.

**Option 1** Almost standalone.

Build an adaptor and allow `GenericPass` instances to be used in the new `PassManager`, but 
that's the only point of contact. Leave all other Python code as is, in particular, the 
`BasePassManager` and `MultiStagePassManager` do not use the new pass manager under the hood.

This is the simplest solution.

**Option 2** Inject the new PM under the hood.

Build an adaptor for `Task` instances (below the `GenericPass` level) to be used in the new
pass manager.  Also use the new pass manager under the hood in `BasePassManager` and 
`MultiStagePassManager`.

This will no longer rely on the `FlowController` to execute and canonicalizes the execution,
but comes at more implementation cost, since we e.g. still want to support the multi-processing
of `BasePassManager` which implies implementing pickle support for the new pass manager.

**Suggestion** Option 1 seems better, unless there's things we gain from Option 2 that we missed?

<!--

Below we discuss the current public Python API in more detail, but already state the high-level plan 
here:

1. Make the new pass manager compatible with `qiskit.passmanager.Task` instances. This 
   allows using existing passes and tasks in the 
2. Keep `BasePassManager` and `MultiStagePassManager` implemented Python side as is. 

To make a decision, we're discussing the differences and how/if they can be reconciled.
The current public API of the Python pass manager includes:

- `BasePassManager` and the built-upon main objects, `PassManager` and `StagedPassManager`:
  A key problem here is the multi-processing these classes enable, which we cannot faithfully
  reproduce using Rust-only. Keeping the multi-processing intact requires us to keep a Python-side
  pass manager, that could internally hold a Rust pass manager and calls it in a multi-processing
  environment.  To enable this we would need pickling support in the Rust pass manager.

  These classes also have publicly documented `_passmanager_frontend` and `_passmanager_backend`
  conversion methods, which are not part of the Rust pass manager.

  We likely need to keep the `BasePassManager` for this interface, including the multi-processing
  execution logic.

- `MultiStagePassManager`: This corresponds to a Rust pass manager with a single `Task::Staged` 
  task.  It does not have any multi-processing and we could implement it by e.g. subclassing
  the new Python-exposed pass manager and fixing the tasks to a single `Task::Staged` or 
  work by composition.  Either way we would retain the existing class.

- `GenericPass`: All passes in Python derive from this base class.  If we make the new pass
  manager accept this pass type, we get compatibility of the existing passes.

  The public interface of `GenericPass` requires users to implement `.run(ir: Any) -> Any`, and 
  assumes that a `self.property_set` is available to be written into.
   
- `Task`: This interface for any executable does not have a Python-exposed equivalent in the new 
  passmanager.

- Flow controllers, such as `FlowControllerLinear` and `DoWhileLoop` via `BaseController`:
  In the new model, we only have preset controllers available, such as a while-loop and a 
  conditional.  We can support appending existing equivalent objects, but there is no concept
  of a `FlowControllerLinear`, since the execution logic is owned by the pass manager,
  and no generic `BaseController` supported.

- `PassManagerState` with `PropertySet` and `WorkflowStatus`: 
  The `PropertySet` is equivalent to `PassContext.data`, however we have no concept of 
  a `WorkflowStatus`. 

-->
 
The callback in the existing Python pass manager is fundamentally different from the new
callback registry.  In the new proposal, the _pass manager_ manages callbacks, but previously
the _pass_ itself called the callback.  To make the callback available during pass execution,
we have to store it in the `PassContext`.

Similarly, we can handle the difference in `PassContext` and the existing 
`PassManagerState` which holds a `PropertySet`, equal to `PassContext.data`, and 
a `WorkflowStatus`, for which we do not have an equivalent but which can be stored as plain
Python object. We have:
```rust
pub struct PassContext {
    data: HashMap<Property, Box<dyn Any>>

    #[cfg(feature = "python_binding")]  // only include it if Python is present
    py_legacy_callback: Option<Py<PyAny>>,

    #[cfg(feature = "python_binding")] 
    py_workflow_status: Option<Py<PyAny>>,
}

impl PassContext {
    fn as_py_state(&self, py: Python) -> PyResult<Py<PyAny>> {
        // build a PassManagerState from self.data and self.py_workflow_status
    }

    fn update_from_py_state(&self, py: Python, state: Py<PyAny>) -> PyResult {
        // store the updated WorkflowStatus and data (aka. PropertySet)
    }
}
```
Then we can write an adaptor for legacy passes based on `GenericPass`. It differs from the 
above `PassFromPy` in that the `LegacyPassFromPy::run` method needs to go through the legacy
signature of `GenericPass.run(ir: Any, state: PassManagerState, callback: Callback[Any]) -> Any`.
```rust
#[pyclass]
struct PassFromPyTask { 
    // Store the `GenericPass` instance, and we'll call it's methods where needed
    py_pass: Py<PyAny>; 
}

impl PassFromPyTask {
    fn new(py_pass: Py<PyAny>) -> PyResult<Self> {
        // When constructing, run instance check to verify it is a `GenericPass`
        Ok(Self { py_pass })
    }
}

impl Pass for PassFromPyTask {
    type InputIR = Py<PyAny>;
    type OutputIR = Py<PyAny>;

    fn run(&self, ir: Self::InputIR, context: &mut PassContext) -> anyhow::Result<Self::OutputIR> {
        Python::attach(|py| {
            let callback_handle = context.py_legacy_callback;
            let state = context.as_py_state(py)?;
            // This is pseudo-code: the important thing is that the pass context
            // needs to be passed to Python's run method, too.
            let (ir_out, state_out) = self
                .py_pass
                .bind(py)
                .call_method1(intern!("execute", (ir, state, callback_handle))?)?
                .extract::<(Py<PyAny>, Py<PyAny>)>()?;
            context.update_from_py_state(py, state_out)?;
            Ok(new_ir)
        })
    }
}
```
This adaptor allows appending a legacy pass from Python.

## Implementation plan

P0: 2.6 critical features
- [ ] Base Rust pass manager, without Analysis passes or the registry
- [ ] C pass manager
- [ ] Expose the new pass to Python 

P1: New features 
- [ ] Analysis passes and registry
