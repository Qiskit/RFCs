# Dataframe for Qiskit Experiments

| **Status**        | **Proposed** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Naoki Kanazawa (nkanazawa1989@gmail.com)     |
| **Deprecates**    | N/A                 |
| **Submitted**     | 2022-08-25                                   |
| **Updated**       | 2022-08-25                                   |


## Summary
This RFC proposes a new internal data structure to be used by Qiskit Experiments (QE).
New data structure based on the [pandas data frame](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.html) aims at improving the usability of current framework so that experimentalists can easily access raw experiment data to apply fancy analysis techniques for research projects.


## Motivation
With the current QE Implementation, it is very tough for novice users to tune or customize the data analysis chain because of the complexity of data structure. Data are often protected or private members, and every analysis step has own data model. Therefore it is not easy to scrutinize the actual data, and it is almost impossible to resume analysis from a particular step without hack. In addition, experimental outcome itself takes different format depending on the run options. New data structure allows users to easily access these internal data with better handling and visibility.

## User Benefit
Experimentalists will mainly enjoy the benefit of new framework. They can try fancy analysis techniques directly on the experiment data without treating it as a big deal, e.g. modifying the analysis class implementation or implementing a new class. Experiment authors (developers) will also benefit by this, especially when testing new analysis code.

## Design Proposal

#### Primitive data types

In the conventional Qiskit execution model, we will get measure outcomes with different data types depending on run options.

| Type                         | Format                 | Run option                           | Typical usage      |
|------------------------------|------------------------|--------------------------------------|--------------------|
| Classified, Counts (CC)      | `List[Dict[str, int]]` | `meas_level=2`                       | Standard outcome   |
| Classified, Single shot (CS) | `List[str]`            | `meas_level=2, memory=True`          | Restless mode      |
| Kerneled, Averaged (KA)      | `ndarray`              | `meas_level=1, meas_return="avg"`    | Device bring-up    |
| Kerneled, Single shot (KS)   | `ndarray`              | `meas_level=1, meas_return="single"` | Discriminator cal. |
| Raw, Averaged (RA)           | `ndarray`              | `meas_level=0, meas_return="avg"`    | Not available      |
| Raw, Single shot (RS)        | `ndarray`              | `meas_level=0, meas_return="single"` | Not available      |

The difference of these data format is one of the difficulties for developer to write custom data processor.
Furthermore, some formats like dict and list of string are not efficient for mathematical operations.
In QE, we should provide clean and compute-efficient data representation for these primitive data types.

It is noteworthy that the bit counts can be efficiently represented by numpy array. For example, assume we received following outcome from the CC measurement.
```python
cc_dict = {
  "00": 25,
  "01": 10,
  "10": 34,
  "11": 13,
}
```
This dictionary can be represented as follows.
```python
cc_array = array([25, 10, 34, 13])
```
Note that, for example, the index `2` is `0b10` in the binary representation, and thus we can translate bit string "10" to index 2 of the array, given we know the size of memory slot. In the same way, for the CS data,
```python
cs_list = ["00", "10", "11"]
```
would become
```python
cs_array = array([0, 2, 3])
```
The conversion from CS to CC format is straightforward. We just need to count the frequency of each number in the CS array, calculate the array index from the number, and set the frequency to the value at the corresponding index of the output CC array.

It is officially supported by numpy to define [custom array subclass](https://numpy.org/doc/stable/user/basics.subclassing.html#subclassing-ndarray). Thus, we can easily write a dedicated wrapper class for each primitive data type on their framework. Type cast (view casting) between primitive data can be also implemented naturally. For example,
```python
cc_array = cs_array.view(ClassifiedCounts)
```
Like this, we can offload complicated type management to numpy to some extent.
The rest of primitive data types are basically numpy.
Now everything becomes numpy array, and we can perform data processing very compute-effectively because we can directly apply numpy functions to every experiment outcomes.


#### Experiment data set

The workflow of experiment analysis is roughly classified into following steps: (1) Extract data from job `Result` object. (2) Pass the data to the data processing routine. This might be a dedicated [data processor](https://qiskit.org/documentation/experiments/stubs/qiskit_experiments.data_processing.DataProcessor.html) class. This might perform discrimination, single value decomposition, restless formatting, computation of probability or expectation value, depending on the configuration of the experiment. (3) Pass the processed data to analysis routine. Some extra formatting might be performed here, such as computing new quantity by combining two series, applying smoothing filters to data, or averaging the outcomes over the same experiment settings.

In the current implementation, these steps have own data model. Specifically in an example of the typical [curve analysis](https://qiskit.org/documentation/experiments/apidocs/curve_analysis.html) workflow, the step-1 generates `List[Dict[str, Any]]`, followed by the step-2 that converts the data into `ndarray` or [uncertainties UFloat](https://pythonhosted.org/uncertainties/) object, and lastly the step-3 converts them into a dedicated class `CurveData` managing x and y values with series index. Some experiment may consist of multiple sub-experiments (series) that are simultaneously analyzed, e.g. [Ramsey XY](https://qiskit.org/documentation/experiments/stubs/qiskit_experiments.library.characterization.RamseyXY.html) experiment. Usually these experiments are managed by circuit metadata, thus an outcome value should be tied to the associated metadata from the experimental circuit.

The pandas data frame will nicely fit in with our framework in consideration of this situation. The outcome of a single circuit execution generates a single row in the data frame table, and each row consists of the outcome value along with some metadata. We can define several data frames in the `ExperimentData` class.

#### `ExperimentData.primitive_data()`

A data frame that provides raw outcomes returned by backend or low-level runtimes (not supported yet). Usually, it contains one of `CC`, `CS`, `KA`, `KS`, `RA`, `RS` data types. Such data frame may look like:

index     | outcome | slots | shots | xval | parameter | ...
----------|:-------:|:-----:|:-----:|:----:|:---------:|----
circuit-1 |   CC    |   1   | 1024  |  3   |     X     |
circuit-2 |   CC    |   1   | 1024  |  3   |     Y     |
circuit-3 |   CC    |   1   | 1024  |  4   |     X     |
...

#### `ExperimentData.processed_data()`

A data frame of data processed by analysis class. For example, this might be a probability or expectation value, projected IQ data (SVD, real, imag), new quantity computed from multiple outcomes, and so on. In the curve analysis chain, this data might take the form (analysis specific):

index     | yval | xval |   state   | parameter | ...
----------|:----:|:----:|:---------:|:---------:|----
element-1 | 0.3  |  3   | formatted |     X     |
element-2 | 0.2  |  3   | formatted |     Y     |
element-3 | 0.4  |  4   | formatted |     X     |
...

Here we can find several benefits of using data frame from multiple aspects.

#### 1. functionality

Data frame is widely accepted by the community of data science, and offers many useful functionality for grouping, sorting, and filtering the data. Experimentalists can easily find tutorials and user guides provided by external communities and we can save our bandwidths for documentation.

Particularly, the [groupby](https://pandas.pydata.org/docs/reference/api/pandas.core.groupby.GroupBy.get_group.html) offers a convenient syntax to extract data at certain analysis step. For example, an experimentalist may prepare data for curve fitting, and then test a fancy algorithm they write. For example,

```python
# Get data frame from the experiment data object.
# This contains full data set.
data = exp_data.processed_data()

# Get subset of data prepared for curve fitting.
x_series_data = data.groupby(["state", "parameter"]).get_group(("formatted", "X"))

# Run custom fitter with typecasted numpy arrays.
xvals = x_series_data["xval"].to_numpy()
yvals = x_series_data["yval"].to_numpy()
popt, pcov = my_curve_fitter(xvals, yvals)
```

#### 2. centralization

Because we can dynamically add arbitrary columns to the data frame, in principle we can manage multiple data sets in a single data frame.

For example, the curve analysis generates two data sets of "processed" and "formatted". In the standard RB analysis, they correspond to multiple probabilities for different circuit seed, and statistical probability averaged over the seeds, respectively. The `groupby` with "status" column allows us to keeps these data in a single object rather than having two `CurveData` instance.
Similarly, `CompositeAnalysis` can store multiple `primitive_data` for each child experiment in the same data frame by using `groupby` with another column presenting a child experiment identifier.

This indicates we can drastically simplify our codebase by offloading data grouping to the data frame.

#### 3. portability

Data frame natively supports [variety of file formats](https://pandas.pydata.org/docs/reference/io.html) from popular CSV and JSON to practical one such as HDF5 and SQL. This drastically enhances the portability of analysis data. Especially, the JSON format will fit well in the IBM experiment service API where we could upload these data as an extra artifact entry.


## Detailed Design

Even though this is the change of internal data structure, this may impact community developers who write own experiment on top of our base classes. Thus, the change should be made step by step to guarantee the backward compatibility.

#### In Qiskit Experiments 0.5

Implement custom numpy array containers for primitive data types. When the `ExperimentData._add_result_data()` method is called, this converts Qiskit `Result` outcomes into corresponding primitive data type (formatting). This data constructs the primitive data frame `ExperimentData._primitive_data` in the experiment data.
Note that the data frame is [not thread safe](https://stackoverflow.com/questions/13592618/python-pandas-dataframe-thread-safe), and we need to implement a subclass of `ThreadSafeContainer` for the data frame.
Accordingly, we can remove `ExperimentData._result_data` that is the conventional thread safe list to store the result dictionaries.

In this version, `ExperimentData.primitive_data()` lives together with the conventional `ExperimentData.data()` method that returns a conventional list of result dictionary. This is important for backward compatibility.
When the `.data()` method is called, we rebuild a list of dictionaries on the fly from the data frame in `_primitive_data`.
Primitive data types might be type casted back into the conventional format (e.g. dict or list of str) not to break existing codes.

The data processor class and processing nodes might be updated to leverage the data frame. The data processor can push formatted data to data frame, so that analysis class can utilize this intermediate data for detailed analysis or visualization. For example, when the processor converts an RS data into CC data through kerneling and classifying, it can push the intermediate KS data to the primitive data frame so that the analysis class can visualize IQ plane. Detailed discussion is skipped here since this is bit outside the scope of this RFC.

#### In Qiskit Experiments 0.6

Deprecate the `ExperimentData.data()` method. Accordingly, all analysis classes relying on `.data()` call must be updated. Specifically, these are the analysis classes that don't use the data processor framework, such as tomography and quantum volume analysis. This change is pretty straightforward.

```python
# Conventional code
for datum in experiment_data.data():
  metadata = data["metadata"]
  if metadata["A"] == "X":
     counts_x.append(data["counts"])
  else:
     counts_y.append(data["counts"])
  ...

# New style
counts_x = experiment_data.frame().groupby("A").get_group("X")["outcome"]
counts_y = experiment_data.frame().groupby("A").get_group("Y")["outcome"]
```

In addition, `CurveAnalysis` class is updated to use the data frame, i.e. `ExperimentData.processed_data()`. This will allow experimentalists to easily deal with custom curve fitting algorithms by using the formatted data set.

#### In Qiskit Experiments 0.7

Completely remove the `ExperimentData.data()` method. Support artifact of the data frame. The support from the experiment service team would be essential to achieve this.


## Alternative Approaches
N/A

## Questions
N/A

## Future Extensions
Upgrade data model of composite experiment. The `groupby` can drastically simplify the current implementation.
Integration of actual runtime primitives must be considered at some point.
