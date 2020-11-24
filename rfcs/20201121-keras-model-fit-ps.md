
# Distributed Keras training APIs with parameter servers

| Status        | Accepted                                             |
:---------------|:-----------------------------------------------------|
| **Author(s)** | Rick Chao, Tom O'Malley, Zhenyu Tan, Yuefeng Zhou (Google)                                  |
| **Sponsor**   | Francois Chollet, Priya Gupta (Google)                                |
| **Updated**   | 2020-11-21                                        |


## Background

With the recent release of TF2 parameter server training support ([API](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/python/distribute/parameter_server_strategy_v2.py)) ([tutorial](https://www.tensorflow.org/tutorials/distribute/parameter_server_training)), custom training loop (CTL) users have started using the `tf.distribute.experimental.ParameterServerStrategy` and `tf.distribute.experimental.coordinator.ClusterCoordinator` APIs for parameter server style distributed training. `ParameterServerStrategy` provides implementation of variable placement, and APIs for defining computation, and `ClusterCoordinator` provides APIs for dataset creation, asynchronous function scheduling and remote execution. The asynchronicity brought by `ClusterCoordinator` provides scalability and training fault tolerance, and at the same time implications such as the need for remote resource creation.

TF2 parameter server training is based on one coordinator task, multiple workers, and multiple (usually fewer than workers) parameter servers (referred to as "ps"). Workers and parameter servers run TensorFlow servers, while the coordinator creates resources on workers and parameter servers, dispatches functions, coordinates the training amd writes checkpoints etc.

While CTL user flow has been supported since the release of TF 2.4, Keras `model.fit` training API is not yet. It has been a common ask (as shown in a survey conducted earlier this year) for availability, given its simplicity and support for a variety of machine learning models, metrics, optimizers, etc. 

In this design, we will discuss the changes in `model.fit` API that we expect to make to accommodate asynchronous, coordinator-based parameter server training flow, and challenges the integration may have given the historical focus of synchronous distributed training with `model.fit`.


## Goals



*   Parameter server training support for Keras compile/fit style training API
*   Minimal code changes across usage with other strategies
*   Minimal performance implications


## Glossary

* Parameter server training: a distributed training method utilizing multiple machines to scale up model training, utilizing a set of training workers, and a set of parameter servers that store the training variables.

* Coordinator: A task (referring to a program run on a dedicated machine) where the python program creates variables on parameter servers, defines functions to be executed on remote workers, and controls the training via `ParameterServerStrategy` and `ClusterCoordinator` APIs.

* `model.fit`: A Keras API for running epoch and step based training loops, with user-provided optimizers, metrics, loss, and callbacks etc.


## Proposed options and solutions

Let’s first take a look at the proposed user flow (on the coordinator). It is expected to be largely the same with other strategies (except for the strategy swap). Unless mentioned otherwise, the discussion here applies to the python program intended to be run on the coordinator.


### User Journey

With a dataset factory:


```
cluster_resolver = ...
strategy = tf.distribute.experimental.ParameterServerStrategy(cluster_resolver)
with strategy.scope():
  preproc_stage = ... # Some Keras preproc layers
  model = ... # Building a Keras model
model.compile(optimizer=..., loss=...)  # `ClusterCoordinator` is created
def dataset_fn(): 
  return tf.data.Dataset.X... # Make use of `preproc_stage` for transformation
history = model.fit(dataset_fn, epochs=..., steps_per_epoch=...,  callbacks=[...])
logging.info("result: %r", history)
```


with a dataset instance:


```
cluster_resolver = ...
strategy = tf.distribute.experimental.ParameterServerStrategy(cluster_resolver)
with strategy.scope():
  preproc_stage = ... # Some Keras preproc layers
  model = ... # Building a Keras model
model.compile(optimizer=..., loss=...)  # `ClusterCoordinator` is created
dataset = tf.data.Dataset.X... # Make use of `preproc_stage` for transformation
# model.fit serializes and deserializes dataset onto workers
history = model.fit(dataset, epochs=..., steps_per_epoch=...,  callbacks=[...]) 
logging.info("result: %r", history)
```



### Changes in `model.fit`

This section discusses the changes needed to be made in `model.fit` API and assumes the reader has basic familiarity with Keras training APIs.


#### `dataset` vs `dataset_fn` argument in `model.fit` API

Current `model.fit` API takes a dataset from which an iterator is created, and the train function is built with this iterator. However, `ClusterCoordinator` only supports taking a no-argument function that returns a `Dataset`. We propose to have both options supported, with an initial focus on `dataset_fn` path due to its lighter implications on `model.fit` implementation.

Previous discussion indicates that although an API modification is needed, the simplicity and less overhead may well justify such callable support, with a goal to support both.


##### `dataset_fn` path

In TF2 parameter server training, `ClusterCoordinator` naturally supports a dataset function to be passed in to `create_per_worker_dataset` API, which creates datasets on remote workers. By leveraging such data factory support, `model.fit` with `dataset_fn` can be implemented by subclassing the existing Keras `DataHandler` (a Keras internal private API) to provide a worker-distributed dataset for Keras to use (i.e. call `iter` on). Please see `DataHandler` section below for proposed changes.

In terms of how users pass a dataset factory into `model.fit`, there are a couple of options:


###### Option 1: any `callable`

In the simplest case, we can allow any kind of `callable` to be passed in:


```
def dataset_fn(): 
  return tf.data.Dataset.from_tensor_slices(...)
history = model.fit(dataset_fn, epochs=..., steps_per_epoch=...,  callbacks=[...])
```

Pros: 
* `callable` does not require users to use additional APIs and may be less overhead.

Cons:
* Less future proof as there could be different intepretation of callable passed as `dataset` to `model.fit` in the future.


###### Option 2: dataset factory

For future-compatibility of `model.fit` API where a `dataset_fn` may have a signature change, a `DatasetFactory` can come handy which determines how the function is supposed to be used. In the case where a `ClusterCoordinator` is used, it is supposed to be invoked with no arguments. With this, user flow will become:


```
def dataset_fn(): return tf.data.Dataset.from_tensor_slices(...)
history = model.fit(DatasetFactory(dataset_fn), epochs=..., steps_per_epoch=...,  callbacks=[...])
```


With an additionally defined class `DatasetFactory`:


```
class DataFactory(object):

  def __init__(self, x):
    if not callable(x):
      raise TypeError('Input for `DataFactory` must be a `callable`.')
    self.x = x

  def __call__(self, *args, **kwargs):
    # We gain the flexibility of modifying args/kwargs for future-compatibility.
    # For example, if we allow different argument signature of user-provided
    # `dataset_fn`, this works as an abstraction layer. 
    # If we now only allow zero-arg, but later extend it to one-arg (e.g. input
    # context), we omit the the arg when we observe that the function doesn't
    # take any arg.
    return self.x(*args, **kwargs)
```

Pros:
* If `dataset` has a different intepretation, for example it takes an argument instead of none, we get a adapting layer with a `DatasetFactory`.

Cons:
* This requires users to use an additional symbol.


The following discussion is tentatively based on option 1, where a simple callable is taken.


###### Implication on no strategy/other strategies

If `model.fit` is allowed to take a `dataset_fn`, use cases for synchronous strategies, and no strategy, can be readily applied. That is, when a dataset is needed, the `callable` is invoked eagerly to obtain a `dataset`, and the remaining workflow will apply.


###### Signature of `dataset_fn`

The signature (input argument and return value) of `dataset_fn` taken by `model.fit` should basically follow the signature `ClusterCoordinator.create_per_worker_dataset` takes. There has been discussion around whether that should take an `InputContext` for effective sharding. Current decision is that it does not, since we do not expect users to shard the dataset considering workers can get preempted. 


##### `dataset` path


###### General support of `dataset` in `ClusterCoordinator`

To support the existing `model.fit` API contract where a `dataset` is taken, it should be preferred that `ClusterCoordinator` provides native `dataset` support, which `model.fit` can readily use, rather than `model.fit` implementing replication logic to accommodate that. Similar to `experimental_distribute_dataset` API, `ClusterCoordinator` can use `tf.data`’s `replicate` API to serialize the dataset graph, and unserialize onto workers. 


###### Variable and resource placement

In parameter server training with a `dataset_fn`, recent work recommends that resources and variables used in the input pipeline are created outside the `dataset_fn`, but within `strategy.scope`, to have variable placement on PS, and resources on the coordinator. This recommendation applies to Keras preprocessing layers (KPL), where read-only resources and variables are created at layer creation. For the `dataset` path, the variables and resources are eagerly created by the user, and they end up having the same placement as `dataset_fn` user path. However, `tf.data` `replicate` API does not support the resources referenced in the dataset graph to be accessed once serialized and deserialized, as opposed to `dataset_fn` case where resources can still be accessed, in the remotely executed `dataset_fn`. This imposes limitations on `dataset` path to use cases where resources are not used.


###### Initial focus on `dataset_fn` path

`dataset` path is concurrently explored to provide compatibility with other strategies, although `dataset_fn` path would be the initial focus to make sure we get to the bottom and user experience can be delivered end-to-end. All following sections are based on the aforementioned `dataset_fn` path, and will be updated when the `dataset` path is more understood.


###### Proposed design for `dataset` path

This is to be completed.


#### The setup of `ClusterCoordinator`

To take advantage of TF2 support of parameter server training, a `ClusterCoordinator` should be created for handling asynchronous function scheduling and joining. The preferred route should be that such an object is abstracted away from the user with `model.fit` training API as an implementation detail.

In terms of who keeps track of the `ClusterCoordinator`, and when it starts allocating threads, there are a few options. Here, we assume that the distribution `Strategy` object can provide whether or not it is supposed to be used with a `ClusterCoordinator`. See below “Changes in tf.distribute” section for more information.


##### Option 1: Attach the `ClusterCoordinator`’s lifecycle to `model.fit`

With this option, an attribute is proposed for the `Model` that keeps track of the `ClusterCoordinator`, and it is instantiated at the beginning of `model.fit` time. 


```
class Model(...):
  def __init__(self):
    self._cluster_coordinator = None
    ...

  def fit(self, ...):
    if hasattr(self.distribute_strategy, 'should_use_with_coordinator'):
      self._cluster_coordinator = cluster_coordinator.ClusterCoordinator(
          self.distribute_strategy)
    ... # the rest of `fit`
    self._cluster_coordinator.shut_down() # Shut down at the end of `fit`
    self._cluster_coordinator = None

class ClusterCoodinator(object):
  def shut_down(self):
    # Join the threads and terminate resources. We don't have this implemented yet.
```



##### Option 2: Have an attribute in `ParameterServerStrategy` that holds the `ClusterCoordinator`

With this option, an attribute is proposed for the `ParameterServerStrategy` to keep track of the `ClusterCoordinator`. We start the `ClusterCoordinator` as soon as the `model.fit` is called for the first time, and do not attempt to shut it down after `fit` completes. It will then be reused for the next `fit`.


```
class Model(...):

  def fit(self, ...):
    if (hasattr(self.distribute_strategy, 'should_use_with_coordinator') and 
        not hasattr(self.distribute_strategy, '_cluster_coordinator'):
      self.distribute_strategy._cluster_coordinator = \
          cluster_coordinator.ClusterCoordinator(self.distribute_strategy)
    ... # the rest of fit

```


This option is with the assumption that there is always only one `ParameterServerStrategy` used.


#### Keras `Model` changes

The train function in `Model.make_train_function` can be swapped with a wrapper that takes an iterator (which, when invoked, would be the worker-specific iterator), and returns the resulting `RemoteValue`.


```
class Model(...):
  def make_train_function(self):
    """Creates a function that executes one step of training."""
    def step_function(model, iterator):
      """Runs a single training step."""
      def run_step(data):
        return model.train_step(data)
      # reduce part omitted
      return model.distribute_strategy.run(run_step, args=(next(iterator),)) 

    self.train_function = ...

    if self._cluster_coordinator:
      self.train_function = lambda iterator: self._cluster_coordinator.schedule(  # pylint: disable=g-long-lambda
          train_function, args=(iterator,))

    return self.train_function
```



#### DataAdapter and DataHandler changes

Most challenges of supporting `model.fit` with `ParameterServerStrategy` are coming from the asynchronicity of dataset creation, where datasets are only created on workers when they are needed. This means the concrete dataset is not existent at the time the `DataHandler` class is instantiated, and thus some information extraction is not available, such as size of a batch, number of batches, etc.

This problem can be solved by a new `DataHandler` class which has customized logic to configure the datasets and steps. In addition, it provides how synchronization with the cluster is done, and how to interpret the result from the train function.


```
class ClusterCoordinatorDataHandler(DataHandler):
  """A `DataHandler` that is compatible with `ClusterCoordinator`."""

  def _configure_dataset_and_inferred_steps(self, strategy, x, steps_per_epoch,
                                            class_weight):
    self._dataset = self._model._cluster_coordinator.create_per_worker_dataset(x)
    if steps_per_epoch is None:
      raise RuntimeError(
          "Steps per epoch must be specified with `ParameterServerStrategy`.")
    self._inferred_steps = steps_per_epoch

  def sync(self):
    self._model._cluster_coordinator.join()

  def resolve_logs(self, logs):
    return logs.fetch()
```


And, in the existing `DataHandler`,


```
class DataHandler(object):

  def _configure_dataset_and_inferred_steps(self, strategy, x, steps_per_epoch,
                                            class_weight):
    # Does what it does currently

  def sync(self):
    context.async_wait()

  @contextlib.contextmanager
  def catch_stop_iteration(self):
    """Catches errors when an iterator runs out of data."""
    try:
      yield
      self.sync()
    except (StopIteration, errors.OutOfRangeError):
      # stop the iteration

  def resolve_logs(self, logs):
    return logs
```


A new `DataAdapter` is also needed to make sure the training API knows a dataset factory is a supported path:


```
class DataFactoryAdapter(DataAdapter):
  """Adapter that handles dataset functions."""

  @staticmethod
  def can_handle(x, y=None):
    if isinstance(x, DatasetFactory):
      assert y is None
      return True
    return False
```



#### Callbacks

With `ParameterServerStrategy`, the return value of `Model.train_function` is a dict `RemoteValue`s. This dict is passed as the `logs` argument to the `CallbackList` object. The `CallbackList` object relies on the `tf_utils.to_numpy_or_python_type` utility to convert these `logs` into `NumPy` values. We will extend the logic of this utility to support `ParameterServerStrategy`. The utility will sync the workers and fetch the `NumPy` value from the `RemoteValue`s:


```
def to_numpy_or_python_type(logs):
  if isinstance(logs, RemoteValue):
    get_strategy().cluster_coordinator.join()  # Sync the workers.
    return logs.fetch()  # Return the NumPy results.
  else:
    ...  # Existing logic.
```


This utility is only used in the `CallbackList` object, which already handles converting `tf.Tensor` logs to `NumPy`. User-defined `Callback`s do not have to be aware of this logic and will not need changes to support `ParameterServerStrategy`.  


##### Epoch-level callbacks

Since the workers will sync every epoch anyway, fetching the remote values incurs minimal additional overhead and so epoch-level `Callback`s can be easily supported.


##### Batch-level callbacks

Some users may want to use batch-level `Callback`s. When users use `steps_per_execution=N`, the `Callback`s will only execute every `N` steps, and so batch-level callbacks will not be prohibitively slow for large `N`. However in other cases, batch-level callbacks may cause a significant slowdown. 

We will support batch-level `Callback`s, but we will use existing logic in `CallbackList` to detect when batch-level `Callback`s are passed, and only incur the performance penalty of syncing workers each batch when the user has passed batch-level `Callback`s (for context, none of Keras's built-in `Callbacks` other than the `ProgressBar` will incur this penalty). This logic was originally put in place to ensure that TPU async mode was only blocked when needed, and applies equally well to `ParameterServerStrategy` without significant modifications.

We will also re-use existing logic to log a warning to the user when their batch-level `Callback`s are causing a significant slowdown in training time. This logic also resides in the `CallbackList` object.


#### Metrics variables

In Keras training APIs, users can specify custom metrics or strings for metrics at `model.compile`, and there is also built-in loss. The variables that are involved, are either created at `compile` time, which is under `strategy.scope`, or the first time they are being updated (at `fit` time, which is also under `strategy.scope`. Therefore the variables should be placed correctly in parameter servers.


#### Optimizer variables

Similarly, the hyper and slot variables an `optimizer` object uses, would be created at gradient application, at which point `model.fit` has entered `strategy.scope` for correct placement. For the variables that need to be colocated with other variables, they should work because `ParameterServerStrategy` has entered colocation device scope when these variables are being created.


#### model.evaluate and model.predict

Initially, we aim to have `evaluate` and `predict` to only be carried out on the coordinator. That is, it does not involve distribution via a `ClusterCoordinator`.

In the longer term, we seek distributed support for `model.evaluate`, where the evaluate function is scheduled onto the workers to execute. Visitation guarantee cannot be supported currently with the parameter server training API, so we can implement distributed evaluation without it, or wait until that is supported, and integrate it. 


### Changes in tf.distribute

The proposal of coordinator-based distributed training provides a way for a `Strategy` to distribute in addition to typical `run`-based synchronous training, but there needs to be a way in the strategy to specify if it is intended to be used with a `ClusterCoordinator`. We should consider having this specified by the user, possibly at `__init__`, so that the Keras training library knows whether or not it is intended for a `ClusterCoordinator`-based single-client training, or traditional multi-client training.

For now, it seems feasible that `ParameterServerStrategy` has a field `should_use_with_coordinator`, which is always True until usage without a `ClusterCoordinator` is supported.


```
  class ParameterServerStrategy(Strategy):
    self.should_use_with_coordinator = True
```



### Workers and parameter servers

With `model.fit`, the cluster expects to continue having `tf.distribute.Server`s run on parameter servers and workers. This can be done by running a python program that calls the python `Server` API:


```
cluster_resolver = tf.distribute.cluster_resolver.TFConfigClusterResolver()
server = tf.distribute.Server(
    cluster_resolver.cluster_spec(),
    job_name=cluster_resolver.task_type,
    task_index=cluster_resolver.task_id,
    protocol=cluster_resolver.rpc_layer or "grpc",
    start=True)
server.join()
```



### Evaluation

If `validation_data` is provided, and certain conditions are satisfied, `model.fit` also runs evaluation via `model.evaluate` API every epoch, in an train-evaluate alternating manner. As described above, at this time, only coordinator is used for `model.evaluate` evaluation, and we plan to extend this to worker-distributed evaluation when visitation guarantee is supported.

In addition to the built-in evaluation `model.fit` provides, sidecar evaluation is also supported with the `SidecarEvaluator` [API](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/python/keras/distribute/sidecar_evaluator.py). User is expected to kick start an additional task `evaluator`, in which the python program runs a `SidecarEvaluator` as follows:


```
model = tf.keras.models.Sequential(...)
model.compile(metrics=tf.keras.metrics.SparseCategoricalAccuracy(...)
data = tf.data.Dataset.from_tensor_slices(...)

SidecarEvaluator(
    model=model,
    data=data,
    checkpoint_dir='/tmp/checkpoint_dir',  # dir for training-saved checkpoint
    steps=None,  # Eval until dataset is exhausted
    log_dir='/tmp/log_dir',
    max_evaluations=None  # The evaluation needs to be stopped manually
).start()
```


`SidecarEvaluator` periodically loads the checkpoint files saved by the training counterpart, as long as the checkpoint captures the `model` and `optimizer` objects. As part of full integration with `model.fit` workflow, we propose to extend `SidecarEvaluator` to 



*   also accept the checkpoint files saved by `ModelCheckpoint` callback for periodic evaluation.
*   accept arbitrary callbacks to be used in its internal `model.evaluate` call

Users will then have two evaluation schemes to choose from: alternating evaluation, or sidecar evaluation, or both, depending on their needs.


### Fault tolerance

There are two goals of fault tolerance in multi-worker training:



*   Task preemption or unavailability is handled by the program or cluster, without being reported as failure
*   The training progress can withstand task preemption and subsequent recovery 

The first goal is fulfilled by the failure handling mechanism provided by the `ClusterCoordinator` library, where worker unavailability is handled within the program, and PS/coordinator unavailability is handled by the cluster’s auto-restart mechanism (by exiting with an exit code that the cluster management recognizes). From the user's perspective, they can leverage proposed `handle_restarable_error` context manager:


```
with handle_restartable_error():
  with strategy.scope():
    model = ... 
  model.compile(optimizer=..., loss=...) 
  model.fit(dataset_fn, epochs=..., steps_per_epoch=...,  callbacks=[...])
```


In the long term, we propose an in-process restart mechanism. This will apply to both custom training loop and `model.fit` cases.

The second goal is fulfilled by Keras `BackupAndRestore` callback, which automatically saves checkpoints at the end of epochs (by default), and restores at the beginning of `model.fit`, including the epoch number it should start at initially:


```
model.fit(...,
    callbacks=[tf.keras.callbacks.experimental.BackupAndRestore(backup_dir=...)])
```


See [class doc](https://github.com/tensorflow/tensorflow/blob/2d263ad1caf91fcc2eb83efa3c63939608e24901/tensorflow/python/keras/callbacks.py#L1546) for more information.


### Testing

If the assumption that model.fit user code remains the same across strategies (for the most basic flow where batch-level callback is not used) stays true, we can utilize strategy combination tests, where we add `ParameterServerStrategy` to the list of strategy combinations, and use that in the existing tests. With lack of resources currently, we may reprioritize this in Q1, and in the immediate term we aim to cover basic flow such as mnist, resnet, etc.


## Timeline



*   Current phase: exploring and protytyping
*   Design doc (ETA: mid/late-Nov)
*   Schedule design review (ETA: Early Dec)
*   User model testing (ETA: Dec)
*   Aligned design with approvals on this doc (ETA: End of Dec)
*   Demonstrable working prototype (ETA: End of Dec)
