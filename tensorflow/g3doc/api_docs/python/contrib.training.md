<!-- This file is machine generated: DO NOT EDIT! -->

# Training (contrib)
[TOC]

Training and input utilities.

## Splitting sequence inputs into minibatches with state saving

Use [`SequenceQueueingStateSaver`](#SequenceQueueingStateSaver) or
its wrapper [`batch_sequences_with_states`](#batch_sequences_with_states) if
you have input data with a dynamic primary time / frame count axis which
you'd like to convert into fixed size segments during minibatching, and would
like to store state in the forward direction across segments of an example.

- - -

### `tf.contrib.training.batch_sequences_with_states(input_key, input_sequences, input_context, input_length, initial_states, num_unroll, batch_size, num_threads=3, capacity=1000, allow_small_batch=True, pad=True, name=None)` {#batch_sequences_with_states}

Creates batches of segments of sequential input.

This method creates a `SequenceQueueingStateSaver` (SQSS) and adds it to
the queuerunners. It returns a `NextQueuedSequenceBatch`.

It accepts one example at a time identified by a unique `input_key`.
`input_sequence` is a dict with values that are tensors with time as first
dimension. This time dimension must be the same across those tensors of an
example. It can vary across examples. Although it always has to be a multiple
of `num_unroll`. Hence, padding may be necessary and it is turned on by
default by `pad=True`.

`input_length` is a Tensor scalar or an int recording the time dimension prior
to padding. It should be between 0 and the time dimension. One reason we want
to keep track of it is so that we can take it into consideration when
computing the loss. If `pad=True` then `input_length` can be `None` and will
be inferred.

This methods segments `input_sequence` into segments of length `num_unroll`.
It batches input sequences from `batch_size` many examples. These mini-batches
are available through the `sequence` property of the output. Moreover, for
each entry in the batch we can access its original `input_key` in `key` and
its input length in `total_length`. `length` records within this segment how
many non-padded time steps there are.

Static features of an example that do not vary across time can be part of the
`input_context`, a dict with Tensor values. This method copies the context for
each segment and makes it availabe in the `context` of the output.

This method can maintain and update a state for each example. It accepts some
initial_states as a dict with Tensor values. The first mini-batch an example
is contained has initial_states as entry of the `state`. If save_state is
called then the next segment will have the updated entry of the `state`.
See `NextQueuedSequenceBatch` for a complete list of properties and methods.

Example usage:

```python
batch_size = 32
num_unroll = 20
num_enqueue_threads = 3
lstm_size = 8
cell = tf.nn.rnn_cell.BasicLSTMCell(num_units=lstm_size)

key, sequences, context = my_parser(raw_data)
initial_state_values = tf.zeros((state_size,), dtype=tf.float32)
initial_states = {"lstm_state": initial_state_values}
batch = tf.batch_sequences_with_states(
    input_key=key,
    input_sequences=sequences,
    input_context=context,
    initial_states=initial_states,
    num_unroll=num_unroll,
    batch_size=batch_size,
    num_threads=num_enqueue_threads,
    capacity=batch_size * num_enqueue_threads * 2)

inputs = batch.sequences["input"]
context_label = batch.context["label"]

inputs_by_time = tf.split(1, num_unroll, inputs)
assert len(inputs_by_time) == num_unroll

lstm_output, _ = tf.nn.state_saving_rnn(
  cell,
  inputs_by_time,
  state_saver=batch,
  state_name="lstm_state")

# Start a prefetcher in the background
sess = tf.Session()

tf.train.start_queue_runners(sess=session)

while True:
  # Step through batches, perform training or inference...
  session.run([lstm_output])
```

##### Args:


*  <b>`input_key`</b>: A string scalar `Tensor`, the **unique** key for the given
    input example.  This is used to keep track of the split minibatch elements
    of this input.  Batched keys of the current iteration are made
    accessible via the `key` property.  The shape of `input_key` (scalar) must
    be fully specified.
*  <b>`input_sequences`</b>: A dict mapping string names to `Tensor` values.  The values
    must all have matching first dimension, called `value_length`. They may
    vary from input to input. The remainder of the shape (other than the first
    dimension) must be fully specified.
    The `SequenceQueueingStateSaver` will split these tensors along
    this first dimension into minibatch elements of dimension `num_unrolled`.
    Batched and segmented sequences of the current iteration are made
    accessible via the `sequences` property.

    **Note**: if `pad=False`, then `value_length` must always be a multiple
      of `num_unroll`.

*  <b>`input_context`</b>: A dict mapping string names to `Tensor` values.  The values
    are treated as "global" across all time splits of the given input example,
    and will be copied across for all minibatch elements accordingly.
    Batched and copied context of the current iteration are made
    accessible via the `context` property.

    **Note**: All input_context values must have fully defined shapes.

*  <b>`input_length`</b>: None or an int32 scalar `Tensor`, the length of the sequence
    prior to padding. If `input_length=None` and `pad=True` then the length
    will be inferred and will be equal to `value_length`. If `pad=False` then
    `input_length` cannot be `None`: `input_length` must be specified. Its
    shape of `input_length` (scalar) must be fully specified. Its value may be
    at most `value_length` for any given input (see above for the definition
    of `value_length`). Batched and total lengths of the current iteration are
    made accessible via the `length` and `total_length` properties.
*  <b>`initial_states`</b>: A dict mapping string state names to multi-dimensional
    values (e.g. constants or tensors).  This input defines the set of
    states that will be kept track of during computing iterations, and
    which can be accessed via the `state` and `save_state` methods.

    **Note**: All initial_state values must have fully defined shapes.

*  <b>`num_unroll`</b>: Python integer, how many time steps to unroll at a time.
    The input sequences of length k are then split into k / num_unroll many
    segments.
*  <b>`batch_size`</b>: int or int32 scalar `Tensor`, how large minibatches should
    be when accessing the `state()` method and `context`, `sequences`, etc,
    properties.
*  <b>`num_threads`</b>: The int number of threads enquing input examples into a queue.
*  <b>`capacity`</b>: The max capacity of the queue in number of examples. Needs to be
    at least `batch_size`. Defaults to 1000. When iterating over the same
    input example multiple times reusing their keys the `capacity` must be
    smaller than the number of examples.
*  <b>`allow_small_batch`</b>: If true, the queue will return smaller batches when
    there aren't enough input examples to fill a whole batch and the end of
    the input has been reached.
*  <b>`pad`</b>: If `True`, `input_sequences` will be padded to multiple of
    `num_unroll`. In that case `input_length` may be `None` and is assumed to
    be the length of first dimension of values in `input_sequences`
    (i.e. `value_length`).
*  <b>`name`</b>: An op name string (optional).

##### Returns:

  A NextQueuedSequenceBatch with segmented and batched inputs and their
  states.

##### Raises:


*  <b>`TypeError`</b>: if any of the inputs is not an expected type.
*  <b>`ValueError`</b>: if any of the input values is inconsistent, e.g. if
    not enough shape information is available from inputs to build
    the state saver.


- - -

### `class tf.contrib.training.NextQueuedSequenceBatch` {#NextQueuedSequenceBatch}

NextQueuedSequenceBatch stores deferred SequenceQueueingStateSaver data.

This class is instantiated by `SequenceQueueingStateSaver` and is accessible
via its `next_batch` property.
- - -

#### `tf.contrib.training.NextQueuedSequenceBatch.__init__(state_saver)` {#NextQueuedSequenceBatch.__init__}




- - -

#### `tf.contrib.training.NextQueuedSequenceBatch.batch_size` {#NextQueuedSequenceBatch.batch_size}

The batch_size of the given batch.

Usually, this is the batch_size requested when initializing the SQSS, but
if allow_small_batch=True this will become smaller when inputs are
exhausted.

##### Returns:

  A scalar integer tensor, the batch_size


- - -

#### `tf.contrib.training.NextQueuedSequenceBatch.context` {#NextQueuedSequenceBatch.context}

A dict mapping keys of `input_context` to batched context.

##### Returns:

  A dict mapping keys of `input_context` to tensors.
  If we had at input:

  ```python
  context["name"].get_shape() == [d1, d2, ...]
  ```

  then for this property:

  ```python
  context["name"].get_shape() == [batch_size, d1, d2, ...]
  ```


- - -

#### `tf.contrib.training.NextQueuedSequenceBatch.insertion_index` {#NextQueuedSequenceBatch.insertion_index}

The insertion indices of the examples (when they were first added).

These indices start with the value -2**63 and increase with every
call to the prefetch op.  Each whole example gets its own insertion
index, and this is used to prioritize the example so that its truncated
segments appear in adjacent iterations, even if new examples are inserted
by the prefetch op between iterations.

##### Returns:

  An int64 vector of length `batch_size`, the insertion indices.


- - -

#### `tf.contrib.training.NextQueuedSequenceBatch.key` {#NextQueuedSequenceBatch.key}

The key names of the given truncated unrolled examples.

The format of the key is:

```python
"%05d_of_%05d:%s" % (sequence, sequence_count, original_key)
```

where `original_key` is the unique key read in by the prefetcher.

##### Returns:

  A string vector of length `batch_size`, the keys.


- - -

#### `tf.contrib.training.NextQueuedSequenceBatch.length` {#NextQueuedSequenceBatch.length}

The lengths of the given truncated unrolled examples.

For initial iterations, for which `sequence * num_unroll < length`,
this number is `num_unroll`.  For the remainder,
this number is between `0` and `num_unroll`.

##### Returns:

  An integer vector of length `batch_size`, the lengths.


- - -

#### `tf.contrib.training.NextQueuedSequenceBatch.next_key` {#NextQueuedSequenceBatch.next_key}

The key names of the next (in iteration) truncated unrolled examples.

The format of the key is:

```python
"%05d_of_%05d:%s" % (sequence + 1, sequence_count, original_key)
```

if `sequence + 1 < sequence_count`, otherwise:

```python
"STOP:%s" % original_key
```

where `original_key` is the unique key read in by the prefetcher.

##### Returns:

  A string vector of length `batch_size`, the keys.


- - -

#### `tf.contrib.training.NextQueuedSequenceBatch.save_state(state_name, value, name=None)` {#NextQueuedSequenceBatch.save_state}

Returns an op to save the current batch of state `state_name`.

##### Args:


*  <b>`state_name`</b>: string, matches a key provided in `initial_states`.
*  <b>`value`</b>: A `Tensor`.
    Its type must match that of `initial_states[state_name].dtype`.
    If we had at input:

    ```python
    initial_states[state_name].get_shape() == [d1, d2, ...]
    ```

    then the shape of `value` must match:

    ```python
    tf.shape(value) == [batch_size, d1, d2, ...]
    ```


*  <b>`name`</b>: string (optional).  The name scope for newly created ops.

##### Returns:

  A control flow op that stores the new state of each entry into
  the state saver.  This op must be run for every iteration that
  accesses data from the state saver (otherwise the state saver
  will never progress through its states and run out of capacity).

##### Raises:


*  <b>`KeyError`</b>: if `state_name` does not match any of the initial states
    declared in `initial_states`.


- - -

#### `tf.contrib.training.NextQueuedSequenceBatch.sequence` {#NextQueuedSequenceBatch.sequence}

An int32 vector, length `batch_size`: the sequence index of each entry.

When an input is split up, the sequence values
```
0, 1, ..., sequence_count - 1
```
are assigned to each split.

##### Returns:

  An int32 vector `Tensor`.


- - -

#### `tf.contrib.training.NextQueuedSequenceBatch.sequence_count` {#NextQueuedSequenceBatch.sequence_count}

An int32 vector, length `batch_size`: the sequence count of each entry.

When an input is split up, the number of splits is equal to:
`padded_length / num_unroll`.  This is the sequence_count.

##### Returns:

  An int32 vector `Tensor`.


- - -

#### `tf.contrib.training.NextQueuedSequenceBatch.sequences` {#NextQueuedSequenceBatch.sequences}

A dict mapping keys of `input_sequences` to split and rebatched data.

##### Returns:

  A dict mapping keys of `input_sequences` to tensors.
  If we had at input:

  ```python
  sequences["name"].get_shape() == [None, d1, d2, ...]
  ```

  where `None` meant the sequence time was dynamic, then for this property:

  ```python
  sequences["name"].get_shape() == [batch_size, num_unroll, d1, d2, ...].
  ```


- - -

#### `tf.contrib.training.NextQueuedSequenceBatch.state(state_name)` {#NextQueuedSequenceBatch.state}

Returns batched state tensors.

##### Args:


*  <b>`state_name`</b>: string, matches a key provided in `initial_states`.

##### Returns:

  A `Tensor`: a batched set of states, either initial states (if this is
  the first run of the given example), or a value as stored during
  a previous iteration via `save_state` control flow.
  Its type is the same as `initial_states["state_name"].dtype`.
  If we had at input:

  ```python
  initial_states[state_name].get_shape() == [d1, d2, ...],
  ```

  then

  ```python
  state(state_name).get_shape() == [batch_size, d1, d2, ...]
  ```

##### Raises:


*  <b>`KeyError`</b>: if `state_name` does not match any of the initial states
    declared in `initial_states`.


- - -

#### `tf.contrib.training.NextQueuedSequenceBatch.total_length` {#NextQueuedSequenceBatch.total_length}

The lengths of the original (non-truncated) unrolled examples.

##### Returns:

  An integer vector of length `batch_size`, the total lengths.



- - -

### `class tf.contrib.training.SequenceQueueingStateSaver` {#SequenceQueueingStateSaver}

SequenceQueueingStateSaver provides access to stateful values from input.

This class is meant to be used instead of, e.g., a `Queue`, for splitting
variable-length sequence inputs into segments of sequences with fixed length
and batching them into mini-batches.  It maintains contexts and state for a
sequence across the segments.  It can be used in conjunction with a
`QueueRunner` (see the example below).

The `SequenceQueueingStateSaver` (SQSS) accepts one example at a time via the
inputs `input_length`, `input_key`, `input_sequences` (a dict),
`input_context` (a dict), and `initial_states` (a dict).
The sequences, values in `input_sequences`, may have variable first dimension
(the `padded_length`), though this dimension must always be a multiple of
`num_unroll`.  All other dimensions must be fixed and accessible via
`get_shape` calls.  The length prior to padding can be recorded in
`input_length`.  The context values in `input_context` must all have fixed and
well defined dimensions.  The initial state values must all have fixed and
well defined dimensions.

The SQSS splits the sequences of an input example into segments of length
`num_unroll`.  Across examples minibatches of size `batch_size` are formed.
These minibatches contain a segment of the sequences, copy the context values,
and maintain state, length, and key information of the original input
examples.  In the first segment of an example the state is still the initial
state.  It can then be updated; and updated state values are accessible in
subsequent segments of the same example. After each segment
`batch.save_state()` must be called which is done by the state_saving_rnn.
Without this call, the dequeue op associated with the SQSS will not run.
Internally, SQSS has a queue for the input examples. Its `capacity` is
configurable.  If set smaller than `batch_size` then the dequeue op will block
indefinitely.  A small multiple of `batch_size` is a good rule of thumb to
prevent that queue from becoming a bottleneck and slowing down training.
If set too large (and note that it defaults to unbounded) memory consumption
goes up.  Moreover, when iterating over the same input examples multiple times
reusing the same `key` the `capacity` must be smaller than the number of
examples.

The prefetcher, which reads one unrolled, variable-length input sequence at
a time, is accessible via `prefetch_op`.  The underlying `Barrier` object
is accessible via `barrier`.  Processed minibatches, as well as
state read and write capabilities are accessible via `next_batch`.
Specifically, `next_batch` provides access to all of the minibatched
data, including the following, see `NextQueuedSequenceBatch` for details:

*  `total_length`, `length`, `insertion_index`, `key`, `next_key`,
*  `sequence` (the index each minibatch entry's time segment index),
*  `sequence_count` (the total time segment count for each minibatch entry),
*  `context` (a dict of the copied minibatched context values),
*  `sequences` (a dict of the split minibatched variable-length sequences),
*  `state` (to access the states of the current segments of these entries)
*  `save_state` (to save the states for the next segments of these entries)

Example usage:

```python
batch_size = 32
num_unroll = 20
lstm_size = 8
cell = tf.nn.rnn_cell.BasicLSTMCell(num_units=lstm_size)
initial_state_values = tf.zeros(cell.state_size, dtype=tf.float32)

raw_data = get_single_input_from_input_reader()
length, key, sequences, context = my_parser(raw_data)
assert "input" in sequences.keys()
assert "label" in context.keys()
initial_states = {"lstm_state": initial_state_value}

stateful_reader = tf.SequenceQueueingStateSaver(
    batch_size, num_unroll,
    length=length, input_key=key, input_sequences=sequences,
    input_context=context, initial_states=initial_states,
    capacity=batch_size*100)

batch = stateful_reader.next_batch
inputs = batch.sequences["input"]
context_label = batch.context["label"]

inputs_by_time = tf.split(1, num_unroll, inputs)
assert len(inputs_by_time) == num_unroll

lstm_output, _ = tf.nn.state_saving_rnn(
  cell,
  inputs_by_time,
  state_saver=batch,
  state_name="lstm_state")

# Start a prefetcher in the background
sess = tf.Session()
num_threads = 3
queue_runner = tf.train.QueueRunner(
    stateful_reader, [stateful_reader.prefetch_op] * num_threads)
tf.train.add_queue_runner(queue_runner)
tf.train.start_queue_runners(sess=session)

while True:
  # Step through batches, perform training or inference...
  session.run([lstm_output])
```

**Note**: Usually the barrier is given to a QueueRunner as in the
    examples above.  The QueueRunner will close the barrier if the prefetch_op
    receives an OutOfRange Error from upstream input queues (i.e., reaches
    the end of the input).  If the barrier is closed no further new examples
    are added to the SQSS.  The underlying barrier might, however, still
    contain further unroll-steps of examples that have not undergone all
    iterations.  To gracefully finish all examples, the flag
    `allow_small_batch` must be set to true, which causes the SQSS to issue
    progressively smaller mini-batches with the remaining examples.
- - -

#### `tf.contrib.training.SequenceQueueingStateSaver.__init__(batch_size, num_unroll, input_length, input_key, input_sequences, input_context, initial_states, capacity=None, allow_small_batch=False, name=None)` {#SequenceQueueingStateSaver.__init__}

Creates the SequenceQueueingStateSaver.

##### Args:


*  <b>`batch_size`</b>: int or int32 scalar `Tensor`, how large minibatches should
    be when accessing the `state()` method and `context`, `sequences`, etc,
    properties.
*  <b>`num_unroll`</b>: Python integer, how many time steps to unroll at a time.
    The input sequences of length `k` are then split into `k / num_unroll`
    many segments.
*  <b>`input_length`</b>: An int32 scalar `Tensor`, the length of the sequence prior
    to padding.  This value may be at most `padded_length` for any given
    input (see below for the definition of `padded_length`).
    Batched and total lengths of the current iteration are made accessible
    via the `length` and `total_length` properties.  The shape of
    input_length (scalar) must be fully specified.
*  <b>`input_key`</b>: A string scalar `Tensor`, the **unique** key for the given
    input.  This is used to keep track of the split minibatch elements
    of this input.  Batched keys of the current iteration are made
    accessible via the `key` property.  The shape of `input_key` (scalar)
    must be fully specified.
*  <b>`input_sequences`</b>: A dict mapping string names to `Tensor` values.  The
    values must all have matching first dimension, called `padded_length`.
    The `SequenceQueueingStateSaver` will split these tensors along
    this first dimension into minibatch elements of dimension
    `num_unroll`. Batched and segmented sequences of the current iteration
    are made accessible via the `sequences` property.

    **Note**: `padded_length` may be dynamic, and may vary from input
    to input, but must always be a multiple of `num_unroll`.  The remainder
    of the shape (other than the first dimension) must be fully specified.

*  <b>`input_context`</b>: A dict mapping string names to `Tensor` values.  The values
    are treated as "global" across all time splits of the given input,
    and will be copied across for all minibatch elements accordingly.
    Batched and copied context of the current iteration are made
    accessible via the `context` property.

    **Note**: All input_context values must have fully defined shapes.

*  <b>`initial_states`</b>: A dict mapping string state names to multi-dimensional
    values (e.g. constants or tensors).  This input defines the set of
    states that will be kept track of during computing iterations, and
    which can be accessed via the `state` and `save_state` methods.

    **Note**: All initial_state values must have fully defined shapes.

*  <b>`capacity`</b>: The max capacity of the SQSS in number of examples. Needs to be
    at least `batch_size`. Defaults to unbounded.
*  <b>`allow_small_batch`</b>: If true, the SQSS will return smaller batches when
    there aren't enough input examples to fill a whole batch and the end of
    the input has been reached (i.e., the underlying barrier has been
    closed).
*  <b>`name`</b>: An op name string (optional).

##### Raises:


*  <b>`TypeError`</b>: if any of the inputs is not an expected type.
*  <b>`ValueError`</b>: if any of the input values is inconsistent, e.g. if
  not enough shape information is available from inputs to build
  the state saver.


- - -

#### `tf.contrib.training.SequenceQueueingStateSaver.barrier` {#SequenceQueueingStateSaver.barrier}




- - -

#### `tf.contrib.training.SequenceQueueingStateSaver.batch_size` {#SequenceQueueingStateSaver.batch_size}




- - -

#### `tf.contrib.training.SequenceQueueingStateSaver.close(cancel_pending_enqueues=False, name=None)` {#SequenceQueueingStateSaver.close}

Closes the barrier and the FIFOQueue.

This operation signals that no more segments of new sequences will be
enqueued. New segments of already inserted sequences may still be enqueued
and dequeued if there is a sufficient number filling a batch or
allow_small_batch is true. Otherwise dequeue operations will fail
immediately.

##### Args:


*  <b>`cancel_pending_enqueues`</b>: (Optional.) A boolean, defaulting to
    `False`. If `True`, all pending enqueues to the underlying queues will
    be cancelled, and completing already started sequences is not possible.
*  <b>`name`</b>: Optional name for the op.

##### Returns:

  The operation that closes the barrier and the FIFOQueue.


- - -

#### `tf.contrib.training.SequenceQueueingStateSaver.name` {#SequenceQueueingStateSaver.name}




- - -

#### `tf.contrib.training.SequenceQueueingStateSaver.next_batch` {#SequenceQueueingStateSaver.next_batch}

The `NextQueuedSequenceBatch` providing access to batched output data.

Also provides access to the `state` and `save_state` methods.
The first time this gets called, it additionally prepares barrier reads
and creates `NextQueuedSequenceBatch` / next_batch objects. Subsequent
calls simply return the previously created `next_batch`.

In order to access data in `next_batch` without blocking, the `prefetch_op`
must have been run at least `batch_size` times (ideally in a separate
thread, or launched via a `QueueRunner`). After processing a segment in
`next_batch()`, `batch.save_state()` must be called which is done by the
state_saving_rnn. Without this call, the dequeue op associated with the SQSS
will not run.

##### Returns:

  A cached `NextQueuedSequenceBatch` instance.


- - -

#### `tf.contrib.training.SequenceQueueingStateSaver.num_unroll` {#SequenceQueueingStateSaver.num_unroll}




- - -

#### `tf.contrib.training.SequenceQueueingStateSaver.prefetch_op` {#SequenceQueueingStateSaver.prefetch_op}

The op used to prefetch new data into the state saver.

Running it once enqueues one new input example into the state saver.
The first time this gets called, it additionally creates the prefetch_op.
Subsequent calls simply return the previously created `prefetch_op`.

It should be run in a separate thread via e.g. a `QueueRunner`.

##### Returns:

  An `Operation` that performs prefetching.



