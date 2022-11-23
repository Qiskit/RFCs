# Dataframe for Qiskit Experiments

| **Status**        | **Proposed** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Naoki Kanazawa (nkanazawa1989@gmail.com), gadial     |
| **Deprecates**    | N/A                 |
| **Submitted**     | 2022-08-25                                   |
| **Updated**       | 2022-11-21                                   |


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

#### User3

User wants to easily perform custom analysis on outcomes of experiments. For example, the [probabilistic error cancellation](https://arxiv.org/abs/2201.09866) first estimates fidelities of various Pauli channels with RB family experiment, then an experimentalist will develop a noise model tailored to a quantum circuit with these fidelities.

_Pain points_:

Currently analysis results are retuned as a list of `AnalysisResult` and they are not well organized. The experimentalist needs to iterate over all list items and check the entry name and extra metadata to filter the result of interest.

_Solutions_:

This user wants to use data frame to organize the analysis results so it is easy to work with.


## Design Proposal

In this proposal, we are going to replace the data format of analysis results, based off of our users survey.

### A. Experiment data

We propose to replace `ExperimentData._analysis_results` data format from `ThreadSafeOrderedDict` to the thread safe wrapper of the pandas data frame. This data frame may consists of following columns:

- name (str): Same as `AnalysisResultData.result_type`. This name will appear in the result database.
- value (Any): Same as `AnalysisResultData.result_data["_value"]`. Any value that QE analysis will generate. This value must be serializable through the `ExperimentEncoder`.
- chisq (float): Same as `AnalysisResultData.chisq`. Chi-squared value of the entry. Probably this can be moved to extra dict.
- quality (str): Same as `AnalysisResultData.quality`. Systematically evaluated quality of the fitting to yield this data.
- device_components (List[DeviceComponent]): Same as `AnalysisResultData.device_components`. Usually `.physical_qubits` of the experiment.
- extra (Dict): Same as `AnalysisResultData.result_data["_extra"]`. Free-form dictionary for analysis metadata.
- experiment_id (str): Same as `AnalysisResultData.experiment_id`. The ID of the associated experiment.
- verified (bool): Same as `AnalysisResultData.verified`. A boolean value to indicate if this entry is checked by human experimentalist. This can be removed since this is rarely used.
- tags (List[str]): Same as `AnalysisResultData.tags`. An arbitrary strings to help filtering this entry.
- source (Dict): Same as `AnalysisResultData.result_data["_source"]`. Version information of QE that was used to generate this entry.
- created_in_db (bool): Same as `AnalysisResultData._created_in_db`. A boolean value to indicate wether this is loaded from the database or just created locally.
- result_id (str): Same as `AnalysisResultData.result_id`. A unique ID of this entry.
- backend (str): Same as `AnalysisResultData.backend_name`. Name of the associated backend.
- creation_datetime (datetime): Same as `AnalysisResultData.creation_datetime`. A date time at which the experiment data received the job results.
- experiment (str): A string representation for the associated experiment for filtering.

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

|          | name   | value  | quality | device_components |  backend        | creation_datetime          | experiment | ... |
|----------|--------|--------|---------|-------------------|-----------------|----------------------------|------------|-----|
| 03fe91s1 | T1     | 100e-6 | good    | [Q0]              |  ibm_washington | 2022-11-22 03:55:30.313872 | T1         | ... |
| 1c4ad2a1 | T1     | 120e-6 | good    | [Q1]              |  ibm_washington | 2022-11-22 03:55:30.313872 | T1         | ... |


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

|          | name   | value  | quality | device_components |  backend        | creation_datetime          | experiment | ... |
|----------|--------|--------|---------|-------------------|-----------------|----------------------------|------------|-----|
| 17636b47 | T1     | 100e-6 | good    | [Q0]              |  ibm_washington | 2022-11-22 03:58:36.523230 | T1         | ... |
| 2f50a741 | T2star |  80e-6 | good    | [Q0]              |  ibm_washington | 2022-11-22 03:58:36.523230 | T2Ramsey   | ... |


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

|          | name   | value  | quality | device_components |  backend        | creation_datetime          | experiment | ... |
|----------|--------|--------|---------|-------------------|-----------------|----------------------------|------------|-----|
| 03fe91s1 | T1     | 100e-6 | good    | [Q0]              |  ibm_washington | 2022-11-22 04:01:30.313872 | T1         | ... |
| 1c4ad2a1 | T1     | 103e-6 | good    | [Q0]              |  ibm_washington | 2022-11-22 05:01:14.123201 | T1         | ... |


#### Complement

As these examples show, experimentalist can distinguish the outcomes of batch (with `experiment`) and parallel experiments (with `device_components`). Even though there are multiple entries for the identical parameter (example 3), they have different result IDs and creation date times. In other words, one can sort the entries by `creation_datetime` and get the time-ordered sequence of `value` for the trend analysis:

```python
sorted_frame = result_dataframe.sort_values(by=["creation_datetime"])
```

alternatively one can directly visualize the data frame with seaborn:

```python
import seaborn as sns

sns.relplot(
    data=result_dataframe,
    kind="line",
    x="creation_datetime",
    y="value",
)
```

See pandas documentation for more details.

### B. Introduction of artifacts

Result database allows experimentalists to store `artifact` data per experiment ID. Thus, in QE, we can add `ExperimentData.artifact` to save supplementary data such as raw curve data points. In principle this can be a free-from dictionary, but it would be better to provide the artifact base class and subclasses to support custom JSON serialization to generate the REST API payload and the input data type validation. In the following we describe `CurveAnalysisArtifact` for `CurveAnalysis`. Note that artifact must be the [thread-safe container](https://github.com/Qiskit/qiskit-experiments/blob/b150427f86c74a02d638e06507864c7bb060c27c/qiskit_experiments/database_service/utils.py#L168-L245) subclass. Each analysis class must know the associated artifact subclass to populate the supplementary data.

The artifact is a part of `ExpeirmentData`, however, each `BaseAnalysis._run_analysis` method must instantiate and return a list of artifact instances to the experiment data. For example, a composite analysis instance may return multiple artifacts for each qubit or experiment.


#### BaseArtifact

The base class of the artifact. Probably Python `dataclass`. This class implements `__json_encode__` and `__json_decode__` methods and subclass can override if necessary.
This base class may have the field for list of figures. In addition to this, artifact object must provide metadata including
- experiment (str): Name of experiment that generated this data.
- device_components (List[DeviceComponent]): List of device components that this data is associated with.
- creation_datetime (datetime): When this data is created. For example, an experimentalists can sort the artifacts by date to visualize the trend of curves in a heat map in line with time.


#### CurveAnalysisArtifact

This artifact have two extra fields.

- fit_status: A dictionary (that may provide nice html view) of fitting status. This is currently provided as analysis result entry with the name `@Parameters_*`. This is not really the analysis results and thus moved to artifact. This contains information about fitting, such as fit models, solver status, initial and final fit parameters and so forth. For example,

- curve_data: Curve data points in the data frame format. This table may have the column:
  - x_val (float): X value.
  - y_val (float): Y value.
  - y_err (float): Y standard error.
  - samples (int): Number of samples used to evaluate this y value and error.
  - model (str): Name of fit model associated with this data.
  - group (str): Name of fit group if composite curve analysis, e.g. Hamiltonian tomography analysis.
  - data_kind (str): Data type. Typically "raw" or "formatted". Raw data is used for visualization.


### C. Experiment service

TODO: write this


## Migration plan

Even though this is the change of internal data structure, this may impact community developers who write own experiment on top of our base classes. Thus, the change should be made step by step to guarantee the backward compatibility.

##### qiskit_experiments v.0.6
Implement the thread safe data class and subclasses for the analysis results. Replace internal analysis results data with the data frame.
In this version, `ExpeirmentData.analysis_results()` method still returns a list of `AnalysisResult` by default, and calling with the option `dataframe=True` returns the data frame.
When data frame is disabled, it internally converts data frame into a list of `AnalysisResult` to return.

`ExperimentData.add_analysis_results()` dispatches the add logic.
The method signature is updated to `results: Optional[Union[AnalysisResult, List[AnalysisResult]]] = None, **kwargs: Any`.
If results argument is not None, this must be the legacy analysis result data.
This implies existing analysis classes can still return this legacy object.
Legacy result objects are internally converted into a single row of the data frame and added.
If there are only keyword arguments, we can directly populate the analysis result data frame and bypasses generation of redundant `AnalysisResult` which no longer exists in the `ExperimentData`.
Existing analysis classes in the Qiskit Experiments namespace will switch to this kwargs style, namely, it must directly populate analysis data from `_run_analysis` method.

`ExperimentData.artifacts()` and `.add_artifacts()` methods are newly implemented with new protected member `ExpeirmentData._artifacts` which is a thread safe list of artifacts.
This method may provide a filtering function based on the artifact metadata.

_Deprecations_
- `CurveData` will be replaced with the data frame and thus immediately deprecated. This is in internal use for curve analysis, thus the impact is minimum.
- Curve analysis options `return_data_points` (curve data) and `return_fit_parameters` (fit status). Now these are moved to artifacts rather than in the analysis results.
- Add deprecation warning when `ExpeirmentData.analysis_results()` is called with `dataframe=False` (default).

_Remove_
- None

##### qiskit_experiments v0.7
In this version we switch the default option of `ExpeirmentData.analysis_results()` to `dataframe=None`. It returns data frame by defaults.
We must write analysis classes so that it adds analysis results with keyword arguments.

_Deprecations_
- `AnalysisResult` and `AnalysisResultData`.
- Deprecate calling `ExperimentData.add_analysis_results()` with results argument.
- Calling with `ExpeirmentData.analysis_results()` with dataframe argument.

_Remove_
- `CurveData`
- Remove fit status and curve data entry generation in analysis results. Curve analysis will complete migration to the artifact at this point.

##### qiskit_experiments v0.8
In this version we complete migration to data frame and legacy QE classes and methods are completely removed.

_Deprecations_
- None

_Remove_
- `AnalysisResult` and `AnalysisResultData`.
- results argument from the `ExpeirmentData.add_analysis_results()`.
- dataframe argument from the `ExpeirmentData.analysis_results()`.

##### qiskit_ibm_experiment  v.0.x
TODO write migration plan on service side.


## Alternative Approaches
N/A

## Questions
- What data frame columns must be shown to end users.
- How we expose `extra` column? A single column of a free-form dict is enough? We cannot apply convenient data frame filtering methods to such column. This must be flattened? This may cause problem when we have multiple experiments, i.e. BatchExperiment. Every experiment may have different extra dict keys.

## Future Extensions

In future, we can also replace `ExperimentData.data` with data frames, which is currently represented as a `List[Dict]`. We can also provide convenient QE helper methods by using the [pandas accessors](https://pandas.pydata.org/docs/development/extending.html#registering-custom-accessors).
