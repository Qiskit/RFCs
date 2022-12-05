# Dataframe for Qiskit Experiments

| **Status**        | **Proposed** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Naoki Kanazawa (nkanazawa1989@gmail.com), Gadi Aleksandrowicz (gadial@gmail.com)    |
| **Deprecates**    | N/A                 |
| **Submitted**     | 2022-08-25                                   |
| **Updated**       | 2022-12-06                                   |


## Summary
This RFC proposes a new representation of the experimental data and experiment service payload format in Qiskit Experiments (QE).
New data structure based on the [pandas data frame](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.html) aims at improving the usability so that an experimentalist can easily manage batched experiment data from IBM Falcon-size device and easily access to raw experiment data for custom analysis and visualization.


## User stories

#### User1
User is operating an 'agent' like monitoring that generates a ton of data over time. The monitoring agent might be run on a regular basis, and a human experimentalist would request the past results in certain time window to investigate the trend of the device parameter.

_Pain points_:

Currently the experiment outcomes are `List[AnalysisResult]` and the query through the REST API is created per single result. This user suffers from the overhead of saving and loading the results from the remote database via the experiment service.

_Solutions_:

This user wants to use data frames to consolidate and simplify data exchange.

#### User2
- (story1) User wants to do analysis of many experiments data taken over time. Such analysis may include, for example, [TLS landscape visualization](https://www.nature.com/articles/s41534-022-00643-y/figures/4), in which multiple relaxation spectroscopy (freq. v.s. P1) curves are arranged in line with time.
- (story2) User wants to get multiple curve data subsets from batched `CurveAnalysis` data and visualize them with non-QE plotter, such as [seaborn](https://seaborn.pydata.org/). For example, one may run `T1` experiment with multiple qubits in parallel, and compare the decay curves of different qubits in a single figure.

_Pain points_:

Currently the curve data visualization is tied to the protected `CurveAnalysis._run_analysis` method and we cannot easily access to the curve data. We are lucking the portable data format for curve data, i.e. (x, y, y_err).

_Solutions_:

This user wants to use data frames to make the curve data portable, and want to store it in `ExperimentData.artifacts`.


## Design Proposal

In this proposal, we are going to replace the data format of analysis results, based off of our users survey.

### A. Experiment data

We propose to replace `ExperimentData._analysis_results` data format from `ThreadSafeOrderedDict` to the thread safe wrapper of the pandas data frame. This data frame may consists of following columns:

#### Primary columns

These columns exist for all analysis results.

- name (str): Same as `AnalysisResultData.result_type`. This name will appear in the result database.
- value (Any): Same as `AnalysisResultData.result_data["_value"]`. Any value that QE analysis will generate. This value must be serializable through the `ExperimentEncoder`.
- quality (str): Same as `AnalysisResultData.quality`. Systematically evaluated quality of the fitting to yield this data.
- components (List[DeviceComponent]): Same as `AnalysisResultData.device_components`. Usually `.physical_qubits` of the experiment.
- experiment_id (str): Same as `AnalysisResultData.experiment_id`. The ID of the associated experiment.
- tags (List[str]): Same as `AnalysisResultData.tags`. An arbitrary strings to help filtering this entry. If this entry is verified by human experimentalist, "verified" tag will be automatically added.
- result_id (str): Same as `AnalysisResultData.result_id`. A unique ID of this entry.
- backend (str): Same as `AnalysisResultData.backend_name`. Name of the associated backend.
- run_time (datetime): A date time at which the experiment data received the job results. Experiment base class must implement new feature to track this time.
- experiment (str): A string representation for the associated experiment for filtering.

#### Extra columns

Analysis class can generate the analysis result with arbitrary number of extra information. If two analysis result entries have different extra information, the data frame column will be created for union of two data set and lacking data is filled with NaN value. Data frame can collapse or expand these extra information. These are the typical information belong to extra.

- chisq (float): Chi-squared value from the curve analysis. This represents a goodness of fitting.
- unit (float): Physical (SI) unit of the value.

Each table row corresponding to a single analysis result entry, i.e. `AnalysisResultData` equivalent.
The columns must contain all information that `AnalysisResultData` provides for the backward compatibility.
Index value is a truncated result ID which is guaranteed to be unique in the table, such as `17636b47`. We can use this ID to get a particular analysis result entry from the table.
Some data field might be hidden from the user view.

For backward compatibility, we add the `dataframe=True` option to `ExperimentData.analysis_results()` method. This options is `False` by default in the first release, and it will be `True` in future versions and eventually deprecated and removed. When `dataframe=False`, the experiment data internally generates the legacy `AnalysisResult` object from each table row.

#### Example 1: Parallel experiment.

```python
pexp = ParallelExperiment(
  [
    T1(qubit=0, delays=delays),
    T1(qubit=1, delays=delays),    
  ],
  backend=ibm_washington,
)
experment_data = pexp.run()

experiment_data.analysis_results(dataframe=True)
```

Analysis results:

|          | name   | value  | quality | components |  backend        | run_time                   | experiment | ... |
|----------|--------|--------|---------|------------|-----------------|----------------------------|------------|-----|
| 03fe91s1 | T1     | 100e-6 | good    | [Q0]       |  ibm_washington | 2022-11-22 03:55:30.313872 | T1         | ... |
| 1c4ad2a1 | T1     | 120e-6 | good    | [Q1]       |  ibm_washington | 2022-11-22 03:55:30.313872 | T1         | ... |


#### Example 2: Batch experiment.

```python
bexp = BatchExperiment(
  [
    T1(qubit=0, delays=delays),
    T2Ramsey(qubit=0, delays=delays),    
  ],
  backend=ibm_washington,
)
experment_data = bexp.run()

experiment_data.analysis_results(dataframe=True)
```

Analysis results:

|          | name   | value  | quality | components |  backend        | run_time                   | experiment | ... |
|----------|--------|--------|---------|------------|-----------------|----------------------------|------------|-----|
| 17636b47 | T1     | 100e-6 | good    | [Q0]       |  ibm_washington | 2022-11-22 03:58:36.523230 | T1         | ... |
| 2f50a741 | T2star |  80e-6 | good    | [Q0]       |  ibm_washington | 2022-11-22 03:58:36.523230 | T2Ramsey   | ... |


#### Example 3: Repeating the same experiment.

```python
exp = T1(qubit=0, delays=delays)

trend_data = []
for _ in range(2):
  experment_data = exp.run()
  trend_data.append(experment_data)
  time.sleep(3600)

# combine trend data
...
```

Analysis results (combined):

|          | name   | value  | quality | components |  backend        | run_time                   | experiment | ... |
|----------|--------|--------|---------|------------|-----------------|----------------------------|------------|-----|
| 03fe91s1 | T1     | 100e-6 | good    | [Q0]       |  ibm_washington | 2022-11-22 04:01:30.313872 | T1         | ... |
| 1c4ad2a1 | T1     | 103e-6 | good    | [Q0]       |  ibm_washington | 2022-11-22 05:01:14.123201 | T1         | ... |


#### Complement

As these examples show, experimentalist can distinguish the outcomes of batch (with `experiment`) and parallel experiments (with `components`). Even though there are multiple entries for the identical parameter (example 3), they have different result and experiment IDs and run times. In other words, one can sort the entries by `run_time` and get the time-ordered sequence of `value` for the trend analysis:

```python
sorted_frame = result_dataframe.sort_values(by=["run_time"])
```

alternatively one can directly visualize the data frame with seaborn:

```python
import seaborn as sns

sns.relplot(
    data=result_dataframe,
    kind="line",
    x="run_time",
    y="value",
)
```

See pandas documentation for more details.

### B. Introduction of artifacts

Result database allows experimentalists to store `artifact` data per experiment ID. Thus, in QE, we can add `ExperimentData.artifact` to save supplementary data such as raw curve data points and figures. In principle this can be a bare python data container, but it is better to use [thread-safe container](https://github.com/Qiskit/qiskit-experiments/blob/b150427f86c74a02d638e06507864c7bb060c27c/qiskit_experiments/database_service/utils.py#L168-L245) subclass.
The `BaseAnalysis.run` method can call `ExpeirmentData.add_artifacts` to save extra data in the database, which is running on a Python concurrent thread, and therefore we need to protect the artifact container from the data collision from different analysis threads.

#### Data structure

Experiment data artifact is a thread same list of `ArtifactData`. For example, T1 experiment analysis may generate three objects to store:
- Figure of curve fitting
- Curve data points (data frame, see below)
- Fit status (dictionary, see below)

Once each data is added to the `ExperimentData`, the data is converted into a `ArtifactData` class that wraps the raw data with metadata:

- .data (Any): Raw data
- .artifact_id (str): Random UUID (or unique ID) for this entry.
- .created_time (datetime): When this data is created.
- .experiment_id (str): ID of experiment that this data is associated with.
- .name (str): Arbitrary name for this entry, such as "t1_plot".

An experimentalist can filter artifact by metadata, i.e. `exp_data.artifacts(name="t1_curves_data")`. Thus, the `ExperimentData` must implement the filtering mechanism.

#### Common entries

- Curve data points

  This is a data frame to store the formatted data points used in the `CurveAnalysis`. For example, an experimentalist can visualize fitting data with preferred format at a later time for publication by taking this entry. Note that this is currently stored as analysis result with the name `@Data_*`.

  - x_val (float): X value.
  - y_val (float): Y value.
  - y_err (float): Y standard error.
  - samples (int): Number of samples used to evaluate this y value and error.
  - model (str): Name of fit model associated with this data.
  - group (str): Name of fit group if composite curve analysis, e.g. Hamiltonian tomography analysis.
  - data_kind (str): Data type. Typically "raw" or "formatted". Raw data is used for visualization.
  - components (List[DeviceComponent]): Device component associated with this data point.


- Fit status

  A dictionary to represent status of curve fitting, which is generated by, not limited to, the `CurveAnalysis`. In the curve analysis, this data is generated based off of the output of the LMFIT solver, and includes fit models, solver status, initial and final fit parameters and so forth. Note that this is currently stored as analysis result with the name `@Parameters_*`, but this sort of metadata to tell how the analysis parameter is estimated. Thus, it's more natural to move to artifact.


### C. Experiment service

The experiment service acts as an intermediate layer between QE
and the resultDB. The current structure of an analysis result contains the following fields:

- uuid (str): Same as the proposed `result_id` field.
- experiment_uuid (str): Same as the proposed `experiment_id` field.
- type (str): Same as the proposed `name` field.
- chisq (float): Same as the proposed `chisq` in the `_extra` dictionary.
- device_components (List[DeviceComponent]): Same as the proposed `components` field.
- quality (str): One of three possible values: `Bad`, `No Information`, `Good`. This is slightly different than the strings being used in QE.
- tags (List[str]): Same as the proposed `tags` field.
- verified (bool): Same as the proposed `verified` field.
- device_name (str): Same as the proposed `backend` field.
- created_at (string): An ISO 8601 timestamp indicating the creation time of the analysis
result inside the resultDB. This is similar to the `date` field but not the same.
Usually experimentalists are interested in the time at which the experiment was done, rather than a timestamp for the resultDB entry.
This suggests these time information must be handled separately.
- fit (dict): A free-form dictionary allowing the storage of arbitrary data.
This can be used to store the `value`, `extra` and `source` fields.

Two of the proposed fields do not make sense in a resultDB context and won't be saved/loaded but only handled by QE:
- `created_in_db`: This has been conventionally managed internally in the `ExperimentData`.
- `experiment`: This may be stored in the `fit` field. Not only experiment UUID, but also human readable representation of experiment is important to help experimentalists with filtering and sorting analysis parameters.

To reduce the amount of work done by QE, we suggest that the experiment service `qiskit-ibm-experiment` will implement
the capability to work with pandas data frames in the suggested structure, and perform the following changes:
- Breakdown the data frame to individual results.
- Translate the fields
   - `uuid` <--> `result_id`
   - `experiment_uuid` <--> `experiment_id`
   - `type` <--> `name`
   - `device_name` <--> `backend`
   - `quality` (db format) <--> `quality` (QE format)
   - `fit` <--> `value`, `extra`, `source`
   - TODO: still need to determine the use of `date`
- Efficiently upload/download multiple results from the server. This depends on the
API supplied by the server; currently mass loading/updating is possible in one API call,
but mass creation is not supported, meaning the service might have to resort to multithreading.

To preserve backward compatibility, we suggest all data frame handling in `qiskit-ibm-experiment` to be
indicted using an optional parameter, `dataframe` (bool) which defaults to `False`, inside the methods

- `IBMExperimentService.create_analysis_results()`
- `IBMExperimentService.update_analysis_results()`
- `IBMExperimentService.analysis_result()`
- `IBMExperimentService.analysis_results()`

## Examples

### A. Writing an analysis class

When an experiment author writes a custom analysis class, one must override the `_run_analysis()` method of a `BaseAnalysis` subclass.
This style doesn't change by our proposal to minimize the impact to user code.
Even though the author can directly call the `ExperimentData.add_analysis_results()` and `ExperimentData.add_artifacts()` from the `_run_analysis()`, this pattern must be prohibited because this ties analysis to executor (i.e. we want to make `_run_analysis()` agnostic to the `ExperimentData` in future update).

At present, the `_run_analysis()` method takes the `ExperimentData` as input and returns the `Tuple[List[AnalysisResult], List[Figure]]`. With this proposal, the return type will be generalized with artifact.

```python
from qiskit_experiments.framework import ExperimentData, AnalysisResult, ArtifactData

class MyAnalysis(BaseAnalysis):

  def _run_analysis(
    experiment_data: ExperimentData
  ) -> Tuple[List[AnalysisResult], List[ArtifactData]]:

    # In future, _run_analysis will directly receive this.
    data = experiment_data.data()
    ...
    # Do some analysis. Generate result and figure (and other artifacts).
    ...
    result1 = AnalysisResult(
      name="param1",
      value=0.1,
      quality="good",
      unit="Hz",
    )
    artifact1 = ArtifactData(
      data=mpl_figure,
      name="curve_plot",
    )
    artifact2 = ArtifactData(
      data=curve_data_table,
      name="curve_data",
    )
    return [result1], [artifact1, artifact2]

```

As you can see, some expected data fields are missing in `result1` in this example (such as component and run time). Note that the minimum required field that the author must provide are name and value. The missing data are implicitly filled by the following base class routine. After this canonicalize, `BaseAnalysis.run` calls `add_analysis_results()` and `add_artifacts()`. The formatted data frame is created (updated) in `ExperimentData` by `add_analysis_results()` call.

### B. Adding new analysis results or artifacts.

An end user (experimentalist) can directly call `add_analysis_results()` and `add_artifacts()` to create custom entry. For example,

```python
exp_data = MyExperiment(...).run().block_for_result()

exp_data.add_analysis_results(
  name="my_param1",
  value="0.1",
)
exp_data.add_artifacts(
  name="my_second_plot",
  data=my_plot2,
)
```

In this case, the user can bypass creation of `AnalysisResult` or `ArtifactData` since these are assumed to be a container used by an experiment author to write `_run_analysis`.

### C. Updating existing results or artifacts.

An end user (experimentalist) can modify analysis result or artifact after experiment.

```python
exp_data = MyExperiment(...).run().block_for_result()

exp_data.update_analysis_results(
  index="1fe131b2",
  tags=["my_project1"],
)
exp_data.update_artifacts(
  index="0ef121s1",
  data = modified_plot,
)
```

Because modified entry must be uniquely specified, `ExperimentData` newly provides methods `update_analysis_results()` and `update_artifacts()` that takes entry ID as a require argument.

### D. Saving results.

(currently save is done by experiment data, but this could be moved to service to decouple service API from data, i.e. `service.save(experiment_data)`)

### E. Requesting particular analysis results to the service.

(what is the syntax of the query?, what is the return format? -- returning the data frame?)


## Migration plan

Even though this is the change of internal data structure, this may impact community developers who write own experiment on top of our base classes. Thus, the change should be made step by step to guarantee the backward compatibility.

##### qiskit_experiments v.0.6
Implement the thread safe data class and subclasses for the analysis results. Replace internal analysis results data with the data frame.
In this version, `ExpeirmentData.analysis_results()` method still returns a list of `AnalysisResult` by default, and calling with the option `dataframe=True` returns the data frame. When data frame is disabled, it internally converts data frame into a list of `AnalysisResult` to return.
`ExperimentData.add_analysis_results()` still receives conventional `Union[AnalysisResult, List[AnalysisResult]]`, and it internally converts them into rows of the data frame to add.

`ExperimentData.artifacts()` and `.add_artifacts()` methods are newly implemented with new protected member `ExpeirmentData._artifacts` which is a thread safe list of artifacts.
This method may provide a filtering function based on the artifact metadata.

_Deprecations_
- `CurveData` will be replaced with the data frame and thus immediately deprecated. This is in internal use for curve analysis, thus the impact is minimum.
- Curve analysis options `return_data_points` (curve data) and `return_fit_parameters` (fit status). Now these are moved to artifacts rather than in the analysis results.
- Add deprecation warning when `ExpeirmentData.analysis_results()` is called with `dataframe=False` (default).
- Add deprecation warning when a user analysis class returns raw `Figure` object.

_Remove_
- None

##### qiskit_experiments v0.7
In this version we switch the default option of `ExpeirmentData.analysis_results()` to `dataframe=None`. It returns data frame by defaults.

_Deprecations_
- Calling with `ExpeirmentData.analysis_results()` with dataframe argument.

_Remove_
- `CurveData`
- Remove fit status and curve data entry generation in analysis results. Curve analysis will complete migration to the artifact at this point.
- No longer accept raw `Figure` object return from `_run_analysis`.

##### qiskit_experiments v0.8
In this version we complete migration to data frame and legacy QE classes and methods are completely removed.

_Deprecations_
- None

_Remove_
- dataframe argument from the `ExpeirmentData.analysis_results()`.


## Alternative Approaches
N/A

## Questions
N/A

## Future Extensions

In future, we can also replace `ExperimentData.data` with data frames, which is currently represented as a `List[Dict]`. We can also provide convenient QE helper methods by using the [pandas accessors](https://pandas.pydata.org/docs/development/extending.html#registering-custom-accessors). Probably we can implement custom handling of extra field with this accessors.
