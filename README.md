![Panopticon](https://raw.github.com/qubyte/panopticon/develop/icon.png)

## Introduction

Panopticon is a Node.js utility to aggregate samples generated by a Node.js application. It works in regular, single instance applications but shines in applications that make use of the `cluster` core module. Samples generated by all processes are sent to the master process and emitted so that the aggregated data may be used.

Panopticon exposes a constructor. Instances live across the cluster, and expose a very simple API for a user to sample data. At regular intervals, the data acquired across the cluster is emitted by the master part of the instance.

In order to minimise parsing load due to interprocess communication, data is sub-aggregated on each worker/master for an interval, and then forwarded to the master for final aggregation, reducing the parsing load to a single object per machine per instance, rather than parsing a small object for every sample taken.

Panopticon has no production dependencies!

## Usage

Require Panopticon like a regular node.js module:

```javascript
var Panopticon = require('panopticon');
```

Panopticon itself is a constructor, so when you're ready to start it, make a new object

```javascript
var panopticon = new Panopticon(startTime, name, interval, scaleFactor, persist, transformer);
```

where `startTime` (ms since the unix epoch) is an optional time to start from, `interval` is the time delay (in ms) between batches of data and `scaleFactor` scales the reporting from some of the reporter types. If no `startTime` is provided, then it defaults to `0`. Similarly, if no sane `interval` is provided, it defaults to 10 seconds. By default the scale of reporting is in kilohertz. `persist` is a boolean, and tells the panopticon if it should be keeping data paths around after each interval.

The `startTime`, if used, must be the same across your cluster. This is simple to manage using the optional environment  argument to `cluster.fork`. i.e. the master can use `startTime = Date.now()`, and pass this value to the forked workers with `cluster.fork({ START_TIME: startTime })`. If not used (undefined or otherwise falsy) then it defaults to 0, so the first interval will be short, but all workers will have the same starting point without communicating a value. A modulo function is used internally to calculate when the current interval ends, so there is no additional cost associated with starting from 0.

If no value is passed in for `scaleFactor`, it defaults to `1` (reports in kHz). Panopticon internally calculates the rate of increments, so it needs to be told if this scale is wrong. For example, to change the reporting of incrementers and timed samples to Hz, set this value to 1000. This only affects incrementers and timed samples, since these are concerned with timing. Sets and samples are your responsibility, so if these should be reporting in something other than kHz for those, then you must give the panopticon the data in the scale desired.

By default the PID of each worker and the master are logged, as well as the number of workers (not including the master). Everything else needs to be sent to the panopticon object using one of its acquisition methods. In each case the `id` is the identifier that should be associated with this piece of data, and path is an array of strings representing subkeys in descending order. The methods are

 - `panopticon.set(path, id, n)`, where `n`, a finite number, may replace a previous `n` for this `id`.
 - `panopticon.inc(path, id, n)`, where `n` is added to the previous value if `n` is a finite number. If `n` is not a finite number, then it defaults to `1`.
 - `panopticon.sample(path, id, n)`, which keeps track of the max, min, average and standard deviation of `n` over an interval.
 - `panopticon.timedSample(path, id, dt)`, which is like sample, but takes the output of a high resolution timer `dt` (or rather the difference between two timers). It also provides a count and total.

When your application is shutting down, it should call `panopticon.stop()` to clear timers.

On the master, halfway between collections from the workers and itself the panopticon object emits aggregated data. This *only happens on the master*.

```javascript
panopticon.on('delivery', function (aggregatedData) {
	// Do something with aggregatedData
});
```

The delivered data is an object containing all of the data collected over the cluster.

### `persist`

Without `persist` turned on, a completely fresh batch of data is started by each panopticon every interval. This means that loggers that get fired rarely are only represented in intervals in which they have occurred. Since this is not always desirable, `persist` tells a panopticon not to start from fresh, but simply to set the loggers to a null state. For `inc`s this is as simple as resetting to `0`, and for `set`s the value from the previous interval is kept. In the case of `sample` and `timedSample`, the subfields are set to `null` since no data recorded should be interpreted as a need for interpolation.

### `transformer`

A transformer function can be used to rearrange the aggregated data. For example:

```javascript
function transformer(data, id) {
	function checkValue(obj) {
		if (typeof obj !== 'object') {
			return;
		}

		if (obj.hasOwnProperty('value')) {
			obj.values = {};
			obj.values[id] = obj.value;
			delete obj.value;
			return;
		}

		for (var key in obj) {
			if (obj.hasOwnProperty(key)) {
				checkValue(obj[key]);
			}
		}
	}

	checkValue(data);

	return data;
}
```

The function takes raw data, and looks for occurrences of the `'value'` key, associated with panopticon data types. When it finds one, it puts the content into a small object called `'values'` against a key which is the worker ID. Panopticon merges objects together on aggregation, so values objects are merged, keeping related data together.

## Panoptica

Multiple panoptica may be instantiated. The motivation for this is sampling over different intervals concurrently. Internally Panopticon keeps track of instances with IDs counting up from zero. To ensure consistency panoptica must be instantiated in the same order, meaning that you should avoid instantiating panoptica in separate asynchronous functions with indefinite execution order. Try to keep them in a synchronous group.

If a worker goes down, you may safely restart it. New panoptica instances catch up to the current interval and report to the master as normal.

To differentiate between different panoptica, each aggregated data has an `id` key, which is the same as the `id` of the panopticon responsible for it.

## Points to note

The node.js implementation of setTimeout is buggy. The resulting timeout can ([and does](https://github.com/joyent/node/issues/5103)) fire early sometimes, contrary to expectations. This lead to some acrobatics to ensure that when it does fire early, it is reinitialised. This can be seen in `Panopticon.prototype.timeUp`.

The standard deviation method used by `panopticon.sample` is single pass. This leaves it more prone than a two pass algorithm to round off errors. A single pass method is used to avoid growing arrays whilst accumulating a batch. The specific algorithm used is the one found in *The Art of Computer Programming, Volume 2: Seminumerical Algorithms*, section 4.2.2.

## Testing

Tests for Panopticon are written in nodeunit. To run them, execute the following command in the Panopticon directory:

```bash
npm test
```

If you want to inspect the test coverage, use the following:

```bash
npm run-script cover
open coverage/lcov-report/index.html
```

(the last line assumes that you're on a mac)

## Contributing

Contributions are welcome! Please observe the coding style of Panopticon. If you add functionality, then this *must* be accompanied by tests. If you break tests, you must have a good reason for doing so and provide updates to existing tests to fix the breakages. Please run JavaScript files changed in your branch through jshint to catch problems. A jshint config file is provided, and jshint is installed as a development dependency.

Panopticon was something of an experiment in using node.js module architecture best practices. Throughout it uses the *module-constructor* pattern, a form of the substack pattern. Every JavaScript file exposes a constructor function on `module.exports`. Whatever your thoughts on this, please abide by the choices made for this module.

# API

## Class: `Panopticon`

Instances are event emitters.

### `Panopticon.count`

Returns the number of panopticon instances that have been started. Useful for testing.

### `Panopticon._reset`

Resets the count of instances. Strictly for testing use only. Do not use this.

### Event: 'delivery'

```javascript
function (data) { }
```

Master only. A panopticon instance emits this event when it has a dataset object to deliver.

### Event: 'sample'

```javascript
function(data, id) { }
```

Master and workers. A panopticon instance emits this event when it has data to be
aggregated. This is a private event and should not be acted upon. Use the 'delivery' event.

### Event: 'newInterval'

```javascript
function () { }
```

Master and workers. A panopticon instance emits this event when a new interval begins. This is
useful for sets, which may be useful to do once per interval.

### Event: 'reset'

```javascript
function () { }
```

Master and workers. If the panopticon instance is persistent, then this event is emitted at the end
of an interval, immediately before 'newInterval'.

### Event: 'stopping'

```javascript
Master and workers. Emitted when `panopticon.stop()` is called.
```

### `panopticon.sample(path, id, n)`

The sample method keeps track of the maximum, minimum and standard deviation of `n` over multiple
calls in an interval. The sample is registered to the aggregated object on the given `path` with a
key given by `id`.

### `panopticon.timedSample(path, id, dt)`

Similar to the sample method, but instead of a number `n` it takes `dt`, the result of a [high
resolution timer](http://nodejs.org/api/process.html#process_process_hrtime) call (a length 2
array of numbers). This method also keeps track of the count and total time over all calls in an
interval.

### `panopticon.inc(path, id, n)`

Increments by `n`. If this the first call of inc with this `path` and `id`, then the starting
value is assumed to be 0.

### `panopticon.set(path, id, value)`

Set a value on a path with id.