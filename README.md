meteor-job
======================================

**WARNING** This Package remains under development and the methods described here may change. As of now, there are no unit tests. You have been warned!

## Intro

Meteor Job is a pure Javascript implementation of the `Job` and `JobQueue` classes that form the foundation of the `jobCollection` Atmosphere package for Meteor. This package is used internally by `jobCollection` but you should also use it for any job workers you need to create and run outside of the Meteor environment as pure node.js programs.

Here's a very basic example that ignores authentication and connection error handling:

```js
var DDP = require('ddp');
var Job = require('meteor-job')

// In this case a local Meteor instance, could be anywhere...
var ddp = new DDP({
  host: "127.0.0.1",
  port: 3000,
  use_ejson: true
});

// Job uses DDP Method calls to communicate with the Meteor jobCollection.
// Within Meteor, it can make those calls directly, but outside of Meteor
// you need to hook it up with a working DDP connection it can use.
Job.setDDP(ddp);

// Once we have a valid connection, we're in business
ddp.connect(function (err) {
  if (err) throw err;

  // Worker function for jobs of type 'somejob'
  somejobWorker = function (job, cb) {
    job.log("Some message");
    // Work on job...
    job.progress(50, 100);  // Half done!
    // Work some more...
    if (jobError) {
      job.fail("Some error happened...");
    } else {
      job.done();
    }
    cb(null); // Don't forget!
  };

  // Get jobs of type 'somejob' available in the 'jobPile' jobCollection for somejobWorker
  workers = Job.processJobs('jobPile', 'somejob', somejobWorker);
});
```

## Installation

`npm install meteor-job`

Someday soon there will be tests...

## Usage

### Getting connected

First you need to establish a [DDP connection](https://github.com/oortcloud/node-ddp-client) with the Meteor server hosting the jobCollection you wish to work on.

```js
var DDP = require('ddp');
var Job = require('meteor-job')

// See DDP package docs for options here...
var ddp = new DDP({
  host: "127.0.0.1",
  port: 3000,
  use_ejson: true
});

Job.setDDP(ddp);

ddp.connect(function (err) {
  if (err) throw err;

  // You will probably need to authenticate here unless the Meteor
  // server is wide open for unauthenticated DDP Method calls, which
  // it really shouldn't be.
  // See DDP package for information about how to use:

    // ddp.loginWithToken(...)
    // ddp.loginWithEmail(...)
    // ddp.loginWithUsername(...)

  // The result of successfully authenticating will be a valid Meteor authToken.
  ddp.loginWithEmail('user@server.com', 'notverysecretpassword', function (err, response) {
    if (err) throw err;
    authToken = response.token

    // From here we can get to work, as long as the DDP connection is good.
    // See the DDP package for details on DDP auto_reconnect, and handling socket events.

    // Do stuff!!!

  });
}
```

### Job workers

Okay, so you've got an authenticated DDP connection, and you'd like to get to work, now what?

```js
// 'jobQueue' is the name of the jobCollection on the server
// 'jobType' is the name of the kind of job you'd like to work on
Job.getWork('jobQueue', 'jobType', function (err, job) {
  if (job) {
     // You got a job!!!  Better work on it!
     // At this point the jobCollection has changed the job status to 'running'
     // so you are now responsible to eventually call either job.done() or job.fail()
  }
});
```

However, `Job.getWork()` is kind of low-level. It only makes one request for a job. What you probably really want is to get some work whenever it becomes available and you aren't too busy:

```js
workers = Job.processJobs('jobQueue', 'jobType', { concurrency: 4 }, function (job, cb) {
  // This will only be called if a job is obtained from Job.getWork()
  // Up to four of these worker functions can be oustanding at
  // a time based on the concurrency option...

  cb(); // Be sure to invoke the callback when this job has been completed or failed.

});
```

Once you have a job, you can work on it, log messages, indicate progress and either succeed or fail.

```js
// This code assumed to be running in a Job.processJobs() callback
var count = 0;
var total = job.data.emailsToSend.length;
var retryLater = [];

// Most job methods have optional callbacks if you really want to be sure...
job.log("Attempting to send " + total + " emails", function(err, result) {
  // err would be a DDP or server error
  // If no error, the result will indicate what happened in jobCollection
});

job.progress(count, total);

if (networkDown()) {
  // You can add a string message to a failing job
  job.fail("Network is down!!!");
  cb();
} else {
  job.data.emailsToSend.forEach(function (email) {
    sendEmail(email.address, email.subject, email.message, function(err) {
      count++;
      job.progress(count, total);
      if (err) {
        job.log("Sending email to " + email.address + "failed"., {level: 'warning'});
        retryLater.push(email);
      }
      if (count === total) {
        // You can attach a result object to a successful job
        job.done({ retry: retryLater });
        cb();
      }
    });
  });
}
```

However, the retry mechanism in the above code seems pretty clunky... How do those failed messages get retried?
This approach probably will probably be easier to manage:

```js
workers = Job.processJobs('jobQueue', 'jobType', { payload: 20 }, function (jobs, cb) {
  // jobs is an array of jobs, between 1 and 20 long, triggered by the option payload > 1
  var count = 0;

  jobs.forEach(function (job) {
    email = job.data.email // Only one email per job
    sendEmail(email.address, email.subject, email.message, function(err) {
      count++;
      if (err) {
        job.log("Sending failed with error" + err, {level: 'warning'});
        job.fail("" + err);
      } else {
        job.done();
      }
      if (count === jobs.length) {
        cb();  // Tells the processJobs we're done
      }
    });
  });
});
```

With the above logic, each email can succeed or fail individually, and retrying later can be directly handled by the jobCollection itself.

The jobQueue object returned by `Job.processJobs()` has methods that can be used to determine its status and control it's behavior. See the jobQueue API reference for more detail.

### Job creators

If you'd like to create an entirely new job and submit it to a jobCollection, here's how:

```js
job = new Job('jobQueue', 'jobType', { work: "to", be: "done" });

// Set some options on the new job before submitting it. These option setting
// methods do not take callbacks because they only affect the local job object.
// See also: job.repeat(), job.after(), job.depends()

job.priority('normal')                    // These methods return job and so are chainable.
   .retry({retries: 5, wait: 15*60*1000}) // Retry up to five times, waiting 15 minutes per attempt
   .delay(15000);                         // Don't run until 15 seconds have passed

job.save(function (err, result) { //Save the job to be added to the Meteor jobCollection via DDP
  if (!err && result) {
    console.log("New job saved with Id: " + result);
  }
});
```

**Note:** It's likely that you'll want to think carefully about whether node.js programs should be allowed to create and manage jobs. Meteor jobCollection provides an extremely flexible mechanism to allow or deny specific actions that are attempted outside of trusted server code. As such, the code above (specifically the `job.save()`) may be rejected by the Meteor server depending on how it is configured. The same caveat applies to all of the job management methods described below.

### Job managers

Management of the jobCollection itself is accomplished using a mixture of Job class methods and methods on individual job objects:

```js
// Get a job object by Id
Job.getJob('jobQueue', id, function (err, job) {
  // Note, this is NOT the same a Job.getWork()
  // This call returns a job object, but does not change the status to 'running'.
  // So you can't work on this job.
});

// If your job object's infomation gets stale, you can refresh it
job.refresh(function (err, result) {
  // job is refreshed
});

// Make a job object from a job document (which you can obtain by subscribing to a jobCollection)
job = Job.makeJob('jobQueue', jobDoc);  // No callback!
// Note that jobCollections are reactive, just like any other Meteor collection. So if you are
// subscribed, the job documents in the collection will autoupdate. Then you can use Job.makeJob
// to turn a job doc into a job object whenever necessary without another DDP roundtrip

// Once you have a job object you can change many of its settings (but only while it's paused)
job.pause(function (err, result) {   // Prohibit the job from running on the queue
  job.priority('low');   // Change its priority
  job.save();            // Update its priority in the jobCollection
                         // This also automatically triggers a job.resume()
                         // which is how you'd otherwise get it running again.
});

// You can also cancel jobs that are running or are waiting to run.
job.cancel();

// You can restart a cancelled or failed job
job.restart();

// Or re-run a job that has already completed successfully
job.rerun();

// And you can remove a job, so long as it's cancelled, completed or failed
// If its running or in any other state, you'll need to cancel it before you can remove it
job.remove();

// For bulk operations on acting on more than one job at a time, there are also Class methods
// that take arrays of job Ids.  For example, cancelling a whole batch of jobs at once:
Job.cancelJobs('jobQueue', Ids, function(err, result) {
  // Operation complete. result is true if any jobs were cancelled (assuming no error)
});
```

## API

### class Job

`Job` has a bunch of Class methods and properties to help with creating and managing Jobs and getting work for them.

#### `Job.setDDP(ddp)`

This class method binds `Job` to a specific instance of `DDPClient`. See [node-ddp-client](https://github.com/oortcloud/node-ddp-client) for more details. Currently it's only possible to use a single DDP connection at a time.

```js
var ddp = new DDP({
  host: "127.0.0.1",
  port: 3000,
  use_ejson: true
});

Job.setDDP(ddp);
```

#### `Job.getWork(root, type, [options], [callback])`

Get one or more jobs from the job Collection, setting status to `'running'`.

`options`:
* `maxJobs` -- Maximum number of jobs to get. Default `1`  If `maxJobs > 1` the result will be an array of job objects, otherwise it is a single job object, or `undefined` if no jobs were available

`callback(error, result)` -- Optional only on Meteor Server with Fibers. Result will be an array or single value depending on `options.maxJobs`.

```js
if (Meteor.isServer) {
  job = Job.getWork(  // Job will be undefined or contain a Job object
    'jobQueue',  // name of job Collection
    'jobType',   // type of job to request
    {
      maxJobs: 1 // Default, only get one job, returned as a single object
    }
  );
} else {
  Job.getWork(
    'jobQueue',                 // root name of job Collection
    [ 'jobType1', 'jobType2' ]  // can request multiple types in array
    {
      maxJobs: 5 // If maxJobs > 1, result is an array of jobs
    },
    function (err, jobs) {
      // jobs contains between 0 and maxJobs jobs, depending on availability
      // job type is available as
      if (job[0].type === 'jobType1') {
        // Work on jobType1...
      } else if (job[0].type === 'jobType2') {
        // Work on jobType2...
      } else {
        // Sadness
      }
    }
  );
}
```

#### `Job.processJobs(root, type, [options], worker)`

Create a `JobQueue` to automatically get work from the job Collection, and asyncronously call the worker function.

See the `JobQueue` section for documentation about the methods and attributes on a `JobQueue` instance.

`options:`
* `concurrency` -- Maximum number of async calls to `worker` that can be outstanding at a time. Default: `1`
* `cargo` -- Maximum number of job objects to provide to each worker, Default: `1` If `cargo > 1` the first paramter to `worker` will be an array of job objects rather than a single job object.
* `pollInterval` -- How often to ask the remote job Collection for more work, in ms. Default: `5000` (5 seconds)
* `prefetch` -- How many extra jobs to request beyond the capacity of all workers (`concurrency * cargo`) to compensate for latency getting more work.

`worker(result, callback)`
* `result` -- either a single job object or an array of job objects depending on `options.cargo`.
* `callback` -- must be eventually called exactly once when `job.done()` or `job.fail()` has been called on all jobs in result.

```js
queue = Job.processJobs(
  'jobQueue',   // name of job Collection
  'jobType',    // type of job to request, can also be an array of job types
  {
    concurrency: 4,
    cargo: 1,
    pollInterval: 5000,
    prefetch: 1
  },
  function (job, callback) {
    // Only called when there is a valid job
    job.done();
    callback();
  }
);

// The job queue has methods... See JobQueue documentation for details.
queue.pause();
queue.resume();
queue.shutdown();
```

#### `Job.makeJob(root, jobDoc)`

Make a Job object from a job Collection document.

```js
job = Job.makeJob('jobQueue', doc);  // doc is obtained from a job Collection subscription
```

#### `Job.getJob(root, id, [options], [callback])`

Creates a job object by id from the server job Collection, returns `null` if no such job exists.

`options`:
* `getLog` -- If `true`, get the current log of the job. Default is `false` to save bandwidth since logs can be large.

`callback(error, result)` -- Optional only on Meteor Server with Fibers. `result` is a job object or `null`

```js
if (Meteor.isServer) {
  job = Job.getJob(  // Job will be undefined or contain a Job object
    'jobQueue',  // name of job Collection
    id,          // job id of type EJSON.ObjectID()
    {
      getLog: false  // Default, don't include the log information
    }
  );
  // Job may be null
} else {
  Job.getJob(
    'jobQueue',    // root name of job Collection
    id,            // job id of type EJSON.ObjectID()
    {
      getLog: true  // include the log information
    },
    function (err, job) {
      if (job) {
        // Here's your job
      }
    }
  );
}
```

#### `Job.getJobs(root, ids, [options], [callback])`

Like `Job.getJob` except it takes an array of ids and is much more efficicent than calling `Job.getJob()` in a loop because it gets Jobs from the server in batches.

#### `Job.pauseJobs(root, ids, [options], [callback])`

Like `job.pause()` except it pauses a list of jobs by id.

#### `Job.resumeJobs(root, ids, [options], [callback])`

Like `job.resume()` except it resumes a list of jobs by id.

#### `Job.cancelJobs(root, ids, [options], [callback])`

Like `job.cancel()` except it cancels a list of jobs by id.

#### `Job.restartJobs(root, ids, [options], [callback])`

Like `job.restart()` except it restarts a list of jobs by id.

#### `Job.removeJobs(root, ids, [options], [callback])`

Like `job.remove()` except it removes a list of jobs by id.

#### `Job.startJobs(root, [options], [callback])`

This feature is still immature. Starts the server job Collection.

`options`: No options currently

`callback(error, result)` -- Result is true if successful.

```js
Job.startJobs('jobQueue');  // Callback is optional
```

#### `Job.stopJobs(root, [options], [callback])`

This feature is still immature. Stops the server job Collection.

`options`:
* `timeout`: In ms, how long until the server forcibly fails all still running jobs. Default: `60*1000` (1 minute)

`callback(error, result)` -- Result is true if successful.

```js
Job.stopJobs(
  'jobQueue',
  {
    timeout: 60000
  }
);  // Callback is optional
```

#### `Job.forever`

Constant value used to indicate that something should repeat forever.

```js
job = new Job('jobQueue', 'jobType', { work: "to", be: "done" })
   .retry({ retries: Job.forever })    // Default for .retry()
   .repeat({ repeats: Job.forever });  // Default for .repeat()
```

#### `Job.jobPriorities`

Valid non-numeric job priorities.

```js
Job.jobPriorities = {
  low: 10
  normal: 0
  medium: -5
  high: -10
  critical: -15
};
```

#### `Job.jobStatuses`

Possible states for the status of a job in the job collection.

```js
Job.jobStatuses = [
    'waiting'
    'paused'
    'ready'
    'running'
    'failed'
    'cancelled'
    'completed'
];
```

#### `Job.jobLogLevels`

Valid log levels. If these look familiar, it's because they correspond to some the Bootstrap [context](http://getbootstrap.com/css/#helper-classes) and [alert](http://getbootstrap.com/components/#alerts) classes.

```js
Job.jobLogLevels: [
    'info'
    'success'
    'warning'
    'danger'
];
```

#### `Job.jobStatusCancellable`

Job status states that can be cancelled.

```js
Job.jobStatusCancellable = [ 'running', 'ready', 'waiting', 'paused' ];
```

#### `Job.jobStatusPausable`

Job status states that can be paused.

```js
Job.jobStatusPausable = [ 'ready', 'waiting' ];
```

#### `Job.jobStatusRemovable`

Job status states that can be removed.

```js
Job.jobStatusRemovable = [ 'cancelled', 'completed', 'failed' ];
```

#### `Job.jobStatusRestartable`

Job status states that can be restarted.

```js
jobStatusRestartable = [ 'cancelled', 'failed' ];
```

### Instances of Job

#### `j = Job(root, type, data)`

Create a new `Job` object.  Data should be reasonably small, if worker requires a lot of data (e.g. video, image or sound files), they should be included by reference (e.g. with a URL pointing to the data, and another to where the result should be saved).

```js
job = new Job(  // new is optional
  'jobQueue',   // job collection name
  'jobType',    // type of the job
  { /* ... */ } // Data for the worker, any valid EJSON object
);
```

#### `j.depends([dependencies])`

Adds jobs that this job depends upon (antecedents). This job will not run until these jobs have successfully completed. Defaults to an empty array (no dependencies). Returns `job`, so it is chainable.

```js
job.depends([job1, job2]);  // job1 and job2 are Job objects, and must successfully complete before job will run
```

#### `j.priority([priority])`

Sets the priority of this job. Can be integer numeric or one of `Job.jobPriorities`. Defaults to `'normal'` priority, which is priority `0`. Returns `job`, so it is chainable.

```js
job.priority('high');  // Maps to -10
job.priority(-10);     // Same as above
```

#### `j.retry([options])`

Set how failing jobs are rescheduled and retried by the job Collection. Returns `job`, so it is chainable.

`options:`
* `retries` -- Number of times to retry a failing job. Default: `Job.forever`
* `wait`  -- How long to wait between attempts, in ms. Default: `300000` (5 minutes)

Note that the above stated defaults are those when `.retry()` is explicitly called. When a new job is created, the default number of `retries` is `0`.

```js
job.retry({
  retries: 5,   // Retry 5 times,
  wait: 20000   // waiting 20 seconds between attempts
});
```

#### `j.repeat([options])`

Set how many times this job will be automatically re-run by the job Collection. Each time it is re-run, a new job is created in the job collection. This is equivalent to running `job.rerun()`. Only `'completed'` jobs are repeated. Failing jobs that exhaust their retries will not repeat. By default, if an infinitely repeating job is added to the job Collection, any existing repeating jobs of the same type that are cancellable, will be cancelled.  See `option.cancelRepeats` for `job.save()` for more info. Returns `job`, so it is chainable.

`options:`
* `repeats` -- Number of times to rerun the job. Default: `Job.forever`
* `wait`  -- How long to wait between re-runs, in ms. Default: `300000` (5 minutes)

Note that the above stated defaults are those when `.repeat()` is explicitly called. When a new job is created, the default number of `repeats` is `0`.

```js
job.repeat({
  repeats: 5,   // Rerun this job 5 times,
  wait: 20000   // wait 20 seconds between each re-run.
});
```

#### `j.delay([milliseconds])`

How long to wait until this job can be run, counting from when it is initially saved to the job Collection. Returns `job`, so it is chainable.

```js
job.delay(0);   // Do not wait. This is the default.
```

#### `j.after([time])`

`time` is a date object. This sets the time after which a job may be run. It is not guaranteed to run "at" this time because there may be no workers available when it is reached. Returns `job`, so it is chainable.

```js
job.after(new Date());   // Run the job anytime after right now. This is the default.
```

#### `j.log(message, [options], [callback])`

Add an entry to this job's log. May be called before a new job is saved. `message` must be a string.

`options:`
* `level`: One of `Jobs.jobLogLevels`: `'info'`, `'success'`, `'warning'`, or `'danger'`.  Default is `'info'`.
* `echo`: Echo this log entry to the console. `'danger'` and `'warning'` level messages are echoed using `console.error()` and `console.warn()` respectively. Others are echoed using `console.log()`.

`callback(error, result)` -- Result is true if logging was successful. When running as `Meteor.isServer` with fibers, for a saved object the callback may be omitted and the return value is the result. If called on an unsaved object, the result is `job` and can be chained.

```js
job.log(
  "This is a message",
  {
    level: 'warning'
    echo: true   // Default is false
  },
  function (err, result) {
    if (result) {
      // The log method worked!
    }
  }
);
```

#### `j.progress(completed, total, [options], [cb])`

Update the progress of a running job. May be called before a new job is saved. `completed` must be a numnber `>= 0` and `total` must be a number `> 0` with `total >= completed`.

`options:`
* `echo`: Echo this progress update to the console using `console.log()`.

`callback(error, result)` -- Result is true if progress update was successful. When running as `Meteor.isServer` with fibers, for a saved object the callback may be omitted and the return value is the result. If called on an unsaved object, the result is `job` and can be chained.

```js
job.progress(
  50,
  100,    // Half done!
  {
    echo: true   // Default is false
  },
  function (err, result) {
    if (result) {
      // The progress method worked!
    }
  }
);
```

#### `j.save([options], [callback])`

Submits this job to the job Collection. Only valid if this is a new job, or if the job is currently paused in the job Collection. If the job is already saved and paused, then omost properties of the job may change (but not all, e.g. the jobType may not be changed.)

`options:`
* `cancelRepeats`: If true and this job is an infinitely repeating job, will cancel any existing jobs of the same job type. Default is `true`. This is useful for background maintainance jobs that may get added on each server restart (potentially with new parameters).

`callback(error, result)` -- Result is true if save was successful. When running as `Meteor.isServer` with fibers, the callback may be omitted and the return value is the result.

```js
job.save(
  {
    cancelRepeats: false  // Do not cancel any jobs of the same type, even if this job repeats forever.  Default: true.
  }
);
```
#### `j.refresh([options], [callback])`

Refreshes the current job object state with the state on the remote job Collection. Note that if you subscribe to the job Collection, the job documents will stay in sync with the server automatically via Meteor reactivity.

`options:`
* `getLog` -- If true, also refresh the jobs log data (which may be large).  Default: `false`

`callback(error, result)` -- Result is true if refresh was successful. When running as `Meteor.isServer` with fibers, the callback may be omitted and the return value is the result.

```js
job.refresh(function (err, result) {
  if (result) {
    // Refreshed
  }
});
```

#### `j.done(result, [options], [callback])`

Change the state of a running job to `'completed'`. `result` is any EJSON object.  If this job is configured to repeat, a new job will automatically be cloned to rerun in the future.

`options:` -- None currently.

`callback(error, result)` -- Result is true if completion was successful. When running as `Meteor.isServer` with fibers, the callback may be omitted and the return value is the result.

```js
job.done(function (err, result) {
  if (result) {
    // Status updated
  }
});
```

#### `j.fail(message, [options], [callback])`

Cause this job to fail. It's next state depends on how the job's `job.retry()` settings are configured. It will either become `'failed'` or go to `'waiting'` for the next retry. `message` is a string.

`options:`
* `fatal` -- If true, no additional retries will be attempted and this job will go to a `'failed'` state. Default: `false`

`callback(error, result)` -- Result is true if completion was successful. When running as `Meteor.isServer` with fibers, the callback may be omitted and the return value is the result.

```js
job.fail(
  'This job has failed again!',
  {
    fatal: false  // Default case
  },
  function (err, result) {
    if (result) {
      // Status updated
    }
  }
});
```

#### `j.pause([options], [callback])`

#### `j.resume([options], [callback])`

#### `j.cancel([options], [callback])`

#### `j.restart([options], [callback])`

#### `j.rerun([options], [callback])`

#### `j.remove([options], [callback])`

#### `j.type`

Contains the type of a job. Useful for when `getWork` or `processJobs` are configured to accept multiple job types. This may not be changed after a job is created.

#### `j.data`

Always an object, contains the job data needed by the worker to complete a job of a given type. This may not be changed after a job is created.

### class JobQueue

JobQueue is similar in spirit to the [async.js](https://github.com/caolan/async) [queue](https://github.com/caolan/async#queue) and [cargo]([queue](https://github.com/caolan/async#cargo)) except that it gets its work from the Meteor jobCollection via calls to `Job.getWork()`

#### `q = Job.processJobs()`

#### `q.resume()`

#### `q.pause()`

#### `q.shutdown()`

#### `q.length()`

#### `q.full()`

#### `q.running()`

#### `q.idle()`

